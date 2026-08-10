# Swapchain Recreation

> **Build-Along Milestone:** ThreeDDemo resize during play produces no validation errors and fills the new extent.

**You will learn:**

- Handle `OUT_OF_DATE` from acquire and present without leaks.
- Rebuild extent-dependent resources in Spark's deterministic order.
- Debounce resize callbacks through the `framebufferResized` flag.

## 1. The Problem We Are Solving

Production apps live with **window resize** for years — not just once at launch. Treating swapchain images as immortal guarantees validation spam and black frames when the user drags a corner.

**Swapchain recreation** is a protocol: idle the GPU, tear down dependents sized to the old extent, recreate the swapchain, rebuild pipelines and framebuffers. Spark implements the full chain in `VulkanRenderer::RecreateSwapchain`. You will mirror that checklist and wire the same acquire/present error handling.

Milestone: drag-resize ThreeDDemo for 30 seconds with validation layers on — zero errors, image fills client area after release.

## 2. Core Concepts

### When swapchains go stale

The **swapchain** is a contract between your `VkSurfaceKHR` and the display compositor: image count, format, extent, present mode. It breaks when:

| Event | Result |
|-------|--------|
| Window resize / maximize | Extent mismatch |
| Dragging splitters (editor) | Repeated extent changes |
| Display scale / monitor change | Suboptimal or out of date |
| Minimize (0×0) | Must defer recreation |

`vkAcquireNextImageKHR` and `vkQueuePresentKHR` return `VK_ERROR_OUT_OF_DATE_KHR` when images no longer match the surface. Ignoring this produces validation errors, black frames, or `VK_ERROR_DEVICE_LOST` on some drivers.

### RecreateSwapchain protocol

Spark centralizes rebuild in `VulkanRenderer::RecreateSwapchain`:

```text
WaitDeviceIdle
  → free command buffers tied to old swapchain
  → CleanupSwapchain (framebuffers, depth, HDR flights, passes...)
  → deviceContext.RecreateSwapchain()
  → rebuild depth, render passes, HDR targets, pipelines, framebuffers
  → reallocate command buffers + swapchain sync + ImGui backend
```

**Never** record draws at the previous extent after recreate. Skip or retry the frame — Spark returns early from `DrawFrame` when acquire reports `OUT_OF_DATE`.

### Zero-extent guard

Minimized or hidden windows often report **0×0** framebuffer size. Creating swapchain images at zero violates validation and wastes work.

`VulkanDeviceContext::TryGetDrawableSize` returns `false` when extent is zero — defer recreate and keep `framebufferResized` set until a non-zero size arrives. During 0×0, skip present; do not spin acquire in a tight loop.

### Debouncing resize storms

Dragging a window edge can fire **dozens** of resize callbacks per second. Recreating full HDR + SSAO + sprite pipelines on every callback stalls the UI thread.

Spark sets `framebufferResized` from `NotifySwapchainResize` (GLFW callback) and recreates **once per DrawFrame** when the flag is set — not inside the OS callback directly. The callback thread must not call `vkDeviceWaitIdle`.

The same path serves Part XIII editor dock splits — viewport resize is not a special case.

### SUBOPTIMAL vs OUT_OF_DATE

`VK_SUBOPTIMAL_KHR` means images are still presentable but not ideal (often compositor scaling changed). Spark treats present `SUBOPTIMAL` like resize pending — schedule recreate next frame.

Distinguish from hard `OUT_OF_DATE` on acquire — Spark calls `RecreateSwapchain()` immediately and **returns** without submit to avoid recording at stale indices.

### Dependent resource checklist

Anything sized to `swapchain.extent` or swapchain image views must rebuild:

| Resource | Why |
|----------|-----|
| Depth buffer | Match pixel size |
| HDR color targets | Post stack resolution |
| Framebuffers | Attachment sizes |
| Screen-space effects | SSAO noise buffer extent |
| Tonemap / UI pipelines | Viewport scissor |
| ImGui display size | Input hit testing |

Missing one item → cropped image, wrong aspect, or validation "image extent mismatch".

### Resize repro matrix

Run before every milestone:

| Scenario | Pass criteria |
|----------|---------------|
| Slow drag (30 s) | No validation errors; stable FPS after release |
| Fast maximize | Single recreate; image fills client |
| Minimize 10 s | No spin loop; no DEVICE_LOST |
| Monitor move | SUBOPTIMAL then stable; aspect correct |
| Editor dock split | Same path as window resize flag |

Log `RecreateSwapchain` duration each run — regression above 2× baseline signals accidental full pipeline rebuild.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### RecreateSwapchain single entry point

**Concept:** When swapchains go stale

**Decision:** One function lists the full dependency chain.

**Why:** Scattered half-rebuilds caused demo bugs — one place to audit when adding new extent-sized passes.

**In Spark:** `VulkanRenderer::RecreateSwapchain` in `VulkanRenderer.cpp`.

### WaitDeviceIdle before destroy

**Concept:** RecreateSwapchain protocol

**Decision:** `deviceContext.WaitDeviceIdle()` as the first line.

**Why:** Pending command buffers may still reference swapchain image views — idle first.

**In Spark:** `VulkanRenderer::RecreateSwapchain` line 97.

### framebufferResized flag + NotifySwapchainResize

**Concept:** Debouncing resize storms

**Decision:** OS callback only sets a bool; GPU work stays on the render thread.

**Why:** Calling `vkDeviceWaitIdle` from a GLFW callback can deadlock on some drivers.

**In Spark:** `Engine.cpp` framebuffer callback → `renderer->NotifySwapchainResize()`.

### Rebuild HDR, post, 2D pipelines after swapchain

**Concept:** Dependent resource checklist

**Decision:** Recreate pipelines after `RecreateHdrFlightTargets`.

**Why:** Render pass compatibility depends on extent and formats — pipelines are not freely resizeable.

**In Spark:** `RecreateSwapchain` lines 118–133.

### TryGetDrawableSize before recreate

**Concept:** Zero-extent guard

**Decision:** Query drawable size; defer when zero.

**Why:** Prevents validation spam when the window is minimized.

**In Spark:** `VulkanDeviceContext::TryGetDrawableSize`.

### Early return on acquire OUT_OF_DATE

**Concept:** SUBOPTIMAL vs OUT_OF_DATE

**Decision:** Recreate and return without submit when acquire fails hard.

**Why:** Avoids recording command buffers against stale image indices.

**In Spark:** `VulkanRenderer::DrawFrame` lines 417–419.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/core/VulkanRenderer.cpp`
- `include/spark/render/core/VulkanDeviceContext.hpp`
- `src/Engine.cpp`

### Step 1 — Read full RecreateSwapchain

Walk the destroy → recreate → pipeline list. Compare against your engine checklist — add missing passes as you grow beyond Part X.

*Swapchain recreation chain*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 96–133)
void VulkanRenderer::RecreateSwapchain() {
    deviceContext.WaitDeviceIdle();

    if (!commandBuffers.IsEmpty()) {
        vkFreeCommandBuffers(
                device(),
                commandPool,
                static_cast<std::uint32_t>(commandBuffers.GetSize()),
                commandBuffers.GetData());
        commandBuffers.Clear();
    }

    CleanupSwapchain();

    deviceContext.RecreateSwapchain();

    frameCapture.RecreateSwapchainResources(
            physicalDevice(), device(), presentSwapchain().imageFormat, presentSwapchain().extent);

    sceneDepthFormat = VulkanRendererGpu::FindDepthFormat(physicalDevice());
    CreateDepthResources();
    CreateRenderPass();
    screenSpaceEffectsPass.CreateRenderPass(device());
    RecreateHdrFlightTargets();
    screenSpaceEffectsPass.RecreateFlightTargets(
            physicalDevice(), device(), presentSwapchain().extent, sceneDepthFormat, VulkanFrameSync::kMaxFramesInFlight);
    scenePipeline.CreateGraphicsPipeline(
            device(), hdrTonemapPass.HdrRenderPass(), sceneDescriptors.Layout(), shaderLoader);
    screenUi.CreatePipelines(device(), presentRenderPass.vkPass);
    particlePass.CreateGraphicsPipeline(device(), hdrTonemapPass.HdrRenderPass());
    tilemapPass.CreateGraphicsPipeline(
            device(), hdrTonemapPass.HdrRenderPass(), sceneDescriptors.Layout(), shaderLoader);
    spritePass.CreateGraphicsPipeline(
            device(), hdrTonemapPass.HdrRenderPass(), sceneDescriptors.Layout(), shaderLoader);
    CreateFramebuffers();
    hdrTonemapPass.CreateTonemapPipeline(
            physicalDevice(), device(), presentRenderPass.vkPass, VulkanFrameSync::kMaxFramesInFlight, shaderLoader);
    screenSpaceEffectsPass.CreatePipeline(physicalDevice(), device(), VulkanFrameSync::kMaxFramesInFlight, shaderLoader);
```

Note the function continues with descriptor updates, command buffer reallocation, `frameSync.RecreateSwapchainSync`, and `InitImGuiBackend`.

### Step 2 — Wire GLFW resize callback

Spark sets the callback in the Engine constructor — callback only sets a flag; GPU work happens in DrawFrame.

*NotifySwapchainResize and DrawFrame resize branch*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 394–402)
void VulkanRenderer::NotifySwapchainResize() {
    framebufferResized = true;
}

void VulkanRenderer::DrawFrame() {
    if (framebufferResized) {
        framebufferResized = false;
        RecreateSwapchain();
        return;
```

*GLFW framebuffer resize callback registration*

```cpp
// src/Engine.cpp (lines 110–112)
        window.SetFramebufferResizeCallback([this](int /*width*/, int /*height*/) {
            renderer->NotifySwapchainResize();
        });
```

### Step 3 — Handle acquire OUT_OF_DATE

Study DrawFrame early-out after fence wait and retire queues:

*Acquire OUT_OF_DATE → RecreateSwapchain*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 413–420)
    std::uint32_t imageIndex = 0;
    const VkResult acquireResult =
            frameSync.AcquireNextImage(device(), presentSwapchain().khr, imageIndex);

    if (acquireResult == VK_ERROR_OUT_OF_DATE_KHR) {
        RecreateSwapchain();
        return;
    }
```

### Step 4 — Handle present SUBOPTIMAL / OUT_OF_DATE

After present, Spark checks `presentResult` and `framebufferResized` together:

*Present result handling*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 462–469)
    const VkResult presentResult =
            frameSync.PresentFrame(presentQueue(), presentSwapchain().khr, imageIndex);
    if (presentResult == VK_ERROR_OUT_OF_DATE_KHR || presentResult == VK_SUBOPTIMAL_KHR || framebufferResized) {
        framebufferResized = false;
        RecreateSwapchain();
    } else if (presentResult != VK_SUCCESS) {
        throw std::runtime_error("vkQueuePresentKHR failed");
    }
```

### Step 5 — Test drag resize 30 seconds

Run ThreeDDemo, drag window edge slowly and rapidly. Validation layers should stay clean; after release, image fills client area without letterboxing from stale viewport.

### Step 6 — Editor viewport parity

Part XIII SceneEditor3DDemo embeds a viewport — confirm it calls the same `NotifySwapchainResize` when dock panel splits. Divergent paths are a common editor-only bug class.

### Step 7 — Log recreate timing

Wrap `RecreateSwapchain` with a CPU timer in debug builds:

```cpp
LOG("RecreateSwapchain %ux%u took %.1f ms", w, h, ms);
```

Target under 50 ms on a dev laptop — if higher, profile `WaitDeviceIdle` vs pipeline create.

### Step 8 — Minimize / restore test

Iconify window 5 s, restore. Expect no crash, no leak spam on restore. If recreate loops, verify the zero-extent guard defers work until drawable size returns.

## 5. How This Connects to the Rest of Your Engine

Part **I** created the initial swapchain; Parts **X** HDR and **XI** 2D composite depend on extent. Part **XIII** editor viewport must share this path. Appendix **lifecycle** contrasts partial recreate vs full shutdown.

Run Time of Day demo (17) after swapchain changes — it exercises HDR + SSAO + shadow recreate together.

## 6. Pitfalls and Debugging

- **Recreate without WaitDeviceIdle** — use-after-free of swapchain image views still in flight.
- **Static viewport in pipeline dynamic state forgotten** — set scissor each frame after recreate.
- **Recording at old extent** — camera aspect wrong one frame; verify `pendingScene` viewport fields.
- **Recreate inside GLFW callback** — deadlock risk with driver on some platforms.
- **Forgetting ImGui display size** — UI hit boxes drift after resize until `DisplaySize` updated.
- **Skipping editor dock in resize matrix** — viewport-only bug class in Part XIII forks.
- **Recreate in loop on SUBOPTIMAL every frame** — debounce; Spark schedules once per frame start.

## 7. Exercises

1. Run the full resize repro matrix; attach a pass/fail table to your PR.
2. Log `RecreateSwapchain` ms for slow drag vs maximize; explain the delta.
3. Capture RenderDoc after resize — verify scissor equals new extent.
4. Force `OUT_OF_DATE` by moving the window to a second monitor; document SUBOPTIMAL vs OUT_OF_DATE behavior.
5. List every resource in your fork that depends on `swapchain.extent` and confirm each appears in `RecreateSwapchain`.

## Summary

Swapchain recreation idles the GPU, rebuilds swapchain dependents, and recreates HDR/post/2D pipelines. Spark debounces resize via `framebufferResized` and handles `OUT_OF_DATE` on acquire and present.

---

Next: [GPU Memory Strategy](02-memory.md).
