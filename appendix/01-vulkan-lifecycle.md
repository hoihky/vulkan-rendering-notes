# Vulkan Object Lifecycle Cheat Sheet

> **Build-Along Milestone:** Clean app exit with zero validation object-tracking errors after resize session.

**You will learn:**

- Destroy Vulkan objects in reverse dependency order after device idle.
- Map handles to Spark owner classes.
- Distinguish swapchain recreate from full application shutdown.

## 1. The Problem We Are Solving

Vulkan handle types have strict parent/child rules. When validation reports obscure errors at exit or resize, engineers need a **lifecycle cheat sheet** — not another tutorial chapter.

Use this appendix during Part I bootstrap audits and Part XVI teardown reviews. Spark centralizes device/swapchain in `VulkanDeviceContext` and partial teardown in `RecreateSwapchain`.


Appendix chapter 01 supports active development — bookmark it. When stuck mid-chapter on Parts XV–XVI, grep appendix first before re-reading entire prior part. Keep local notes column for your renamed types alongside Spark names.

Schedule 30-minute appendix refresh after any PR touching `SceneSubmit`, descriptor layouts, or `RecreateSwapchain`.


Print the lifecycle reference table and tape it next to your monitor during Part XVI hardening week — resize bugs and exit leaks are easier when destroy order is visible without switching PDF pages.

## 2. Core Concepts

### Creation order (outside-in)

Vulkan objects form a **dependency tree**. Create parents before children:

```text
VkInstance
  → VkSurfaceKHR
    → VkPhysicalDevice (selected)
      → VkDevice + queues
        → VkSwapchainKHR + image views
          → VkRenderPass / VkFramebuffer
            → VkPipelineLayout → VkPipeline
              → VkDescriptorPool → VkDescriptorSet
                → VkCommandPool → VkCommandBuffer
                  → VkSemaphore / VkFence
```

Spark centralizes early steps in `VulkanDeviceContext`; `VulkanRenderer` owns passes and pipelines. Demos never call `vkCreateDevice` directly.

### Destruction order (inside-out)

After `vkDeviceWaitIdle`, destroy in **reverse** creation order:

```text
WaitIdle
  → sync objects (fences/semaphores in flight)
  → command buffers + pool
  → descriptor sets + pool
  → pipelines + pipeline layout
  → framebuffers
  → render pass
  → image views → images → memory
  → swapchain
  → device
  → surface
  → debug messenger
  → instance
```

Destroying a parent while children live → validation `VUID-*-object-in-use` or silent corruption.

### Recreate vs full shutdown

**Swapchain resize** is partial teardown:

| Kept | Destroyed/recreated |
|------|---------------------|
| `VkDevice`, `VkInstance` | Swapchain image views |
| Queue handles | Depth/HDR sized to extent |
| Most descriptor pool layout | Framebuffers, some pipelines |

**App exit** destroys device entirely — no swapchain without device. Appendix helps when validation fires at resize **or** quit — different fix lists.

### Single owner rule

| Handle | Spark owner |
|--------|-------------|
| Instance, device, swapchain | `VulkanDeviceContext` |
| Render passes, pipelines per feature | Pass classes (`HdrTonemapPass`, `ShadowPass`) |
| Scene descriptors | `VulkanSceneDescriptors` |
| Command buffers | `VulkanRenderer` |

Gameplay never holds `VkDevice`. Matches Part I three-layer architecture — GPU handles stay in backend module.

### Common validation themes

| Message theme | Usual fix |
|---------------|-----------|
| Object in use | Wait idle or fence |
| Child destroyed after parent | Reverse destroy order |
| Image layout mismatch | Barrier before use |
| Swapchain out of date | RecreateSwapchain |
| Memory leak at exit | Missing Destroy on pass shutdown |

Keep a personal checklist pasted next to `Shutdown()` — tick each `vkDestroy*`.

### RAII vs manual order

RAII wrappers (`UniqueVkBuffer`) help but **member declaration order** still defines destroy order in C++. Pass objects as members destroyed before `VulkanDeviceContext` in renderer destructor.

Spark mixes explicit `Destroy()` methods on passes with context destructor — read destructor body when adding new pass member.

### Debug messenger lifetime

`VkDebugUtilsMessengerEXT` must be destroyed **before** `vkDestroyInstance` — Spark creates in VulkanDeviceContext bootstrap. Leaked messenger spam CI logs with validation on every test process exit.

Validation layers themselves are not VkObjects — but messenger callback must not call Vulkan after instance teardown begins.

### Surface before swapchain destroy

Destroy swapchain **before** surface on shutdown, but surface outlives swapchain on resize (same surface, new swapchain). Confusing these two paths causes "surface lost" on Android if you destroy surface during recreate.

### Semaphore vs fence quick ref

**Fence** CPU waits for GPU (frame completion). **Semaphore** GPU-GPU sync between acquire/submit/present queue operations. Destroy fences after wait, semaphores after device idle on shutdown.

Mixing up which to wait causes validation or deadlock — Spark `VulkanFrameSync` encapsulates; copy pattern not individual vkWait calls scattered in gameplay.

### Object naming

`VK_EXT_debug_utils` object names appear in validation messages — name swapchain images `SwapchainImage[i]` when learning; speeds correlation with RenderDoc captures.

### Quick reference usage pattern

Use appendix chapters during **debugging**, not first read — open lifecycle when validation reports destroy order; open descriptor map when shadows black; open SceneRenderParams when draw missing despite visible entity.

Keep local copy of binding table CSV updated when forking shaders — appendix lags code if you diverge from Spark.

### Grep-first workflow

```bash
rg 'SceneDrawItem' src/scene/SceneSubmit.cpp
rg 'binding = 0' shaders/
rg 'RecreateSwapchain' src/spark/render/
```

Three greps resolve most student blockers — teach juniors grep before asking for help.

### Cross-link to book parts

Each appendix section maps to prior parts — when extending engine, update appendix notes in same PR as code so next reader inherits accurate binding indices and component gather list.

Stale appendix worse than none — wrong binding number wastes hours.

### Printable cheat sheet

Export markdown sections to single PDF "Spark GPU Quick Ref" for desk — lifecycle diagram one page, binding table one page, demo catalog one page.

Physical printouts survive IDE crashes during all-nighters.

### Fork naming

When renaming Spark types in your engine, keep appendix role table (camera block, draw arrays) even if struct renamed — conceptual map survives rebrand from Spark to your studio engine name.

### Production debugging discipline

When shipping milestone fails, bisect: validation on/off, ImGui on/off, single demo mode, default quality preset. Half of "retail build broken" reports trace to missing assets copy step not renderer logic.

Keep a `known-good` git tag with attached RenderDoc capture and profiler JSON — compare against feature branch when investigating regressions introduced during Part XVI hardening work.

### Build-along diff habit

After each chapter section, `diff` your engine against Spark for the cited file — not wholesale copy, but line-by-line understanding. Book teaches patterns; your class names may differ. If behavior diverges, fix data contract before reading next chapter — compound errors across Parts IX–XVI are painful to unwind.

Maintain a personal changelog mapping your fork file → Spark equivalent — appendix references assume Spark paths.

### Appendix chapter 1 maintenance

Appendix chapter 01 goes stale when you fork shaders or rename components — schedule appendix review in same PR as binding layout or SceneSubmit gather changes. Future you (and teammates) grep appendix PDF at 2am during milestone crunch — accuracy saves hours.

Print binding table and lifecycle diagram for desk reference; update sticky notes when adding binding 10+ in custom forward+ passes.

### Full lifecycle reference table

| Phase | VkObject | Spark owner | Create trigger | Destroy trigger |
|-------|----------|-------------|----------------|-----------------|
| Bootstrap | VkInstance | VulkanDeviceContext | Engine init | Engine shutdown |
| Surface | VkSurfaceKHR | VulkanDeviceContext | Window create | After swapchain |
| Device | VkDevice | VulkanDeviceContext | After queue select | Before instance |
| Swapchain | VkSwapchainKHR | VulkanPresentationSwapchain | Init / recreate | Recreate / shutdown |
| Depth | VkImage | VulkanRenderer | Init / recreate | RecreateSwapchain |
| HDR color | VkImage | HdrFlightTargets | Init / recreate | RecreateSwapchain |
| RenderPass | VkRenderPass | Pass classes | Init / recreate | Pass Destroy |
| Pipeline | VkPipeline | Pass classes | Init / recreate | Pass Destroy |
| DescriptorPool | VkDescriptorPool | VulkanSceneDescriptors | Init | Before device |
| CommandPool | VkCommandPool | VulkanRenderer | Init | Recreate / shutdown |
| Fence/Semaphore | sync objects | VulkanFrameSync | Init | Shutdown |

**Recreate subset (resize):** swapchain, image views, depth, HDR, framebuffers, pipelines tied to render pass compatibility, command buffer free + reallocate.

**Never recreated on resize:** VkDevice, VkInstance, descriptor pool layout (usually), mesh VB/IB on GPU.

Print this table — mark rows you added in custom fork with asterisk.

### Exit validation ritual

Run before every release candidate:

```text
1. Enable validation layers
2. Launch demo 17 — play 60 s, resize twice, toggle SSAO
3. Exit via menu (not kill -9)
4. Grep log for "Validation Error" — must be zero
5. Grep object tracker for leaked VkDeviceMemory — zero
6. Repeat with retail config on clean machine (layers off, no leak spam)
```

**Common exit leaks and fixes:**

| Leaked object | Missing call | Owner file |
|---------------|--------------|------------|
| VkImageView | DestroyImageView | Pass Destroy |
| VkFramebuffer | DestroyFramebuffer | Recreate cleanup |
| VkPipeline | DestroyPipeline | Pass Destroy |
| VkDescriptorPool | DestroyDescriptorPool | Renderer shutdown |
| VkDebugUtilsMessengerEXT | Destroy before instance | DeviceContext |

If leak appears only after resize session, bug is in **RecreateSwapchain cleanup** branch — not Shutdown. Reproduce: resize only, exit, compare leak list.

Document ritual in `CONTRIBUTING.md` — new engineers run on day one.

### RecreateSwapchain partial teardown checklist

When validation fires **during resize** (not exit), walk this ordered checklist against Spark `RecreateSwapchain`:

| Step | Action | Validation if skipped |
|------|--------|----------------------|
| 1 | WaitDeviceIdle | Object in use |
| 2 | Free command buffers | CB still recording |
| 3 | Destroy framebuffers | FB size mismatch |
| 4 | Destroy depth image + view | Attachment extent |
| 5 | Destroy HDR color + view | Tonemap sample bad |
| 6 | RecreateHdrFlightTargets | SSAO extent wrong |
| 7 | deviceContext.RecreateSwapchain | OUT_OF_DATE persists |
| 8 | Recreate depth + views | Depth test wrong |
| 9 | Recreate render passes if format changed | Rare |
| 10 | Recreate pipelines | Render pass incompatible |
| 11 | Recreate framebuffers | Draw to wrong size |
| 12 | Update ImGui DisplaySize | UI hit box drift |

Custom passes (bloom, motion blur) must insert at steps 8–10 — grep your fork for `extent.width` consumers.

**Not destroyed:** VkDevice, mesh VB, descriptor pool (usually), scene texture array images (content preserved — rebind views if swapchain-only pass).

Compare to **full shutdown** — shutdown also destroys device, pools, all meshes. Mixing the two mental models causes "why did my mesh survive resize but not quit" confusion — document both paths on wiki.

### Fence vs semaphore destroy timing

Sync object destroy timing causes exit validation noise if wrong:

| Object | Wait before destroy | Destroy when |
|--------|---------------------|--------------|
| VkFence (per frame) | vkWaitForFences(infinite) | After wait, before pool destroy |
| VkSemaphore (acquire) | vkDeviceWaitIdle | Shutdown only |
| VkSemaphore (render complete) | vkDeviceWaitIdle | Shutdown only |

Spark `VulkanFrameSync` owns fence ring — do not destroy individual fences mid-session except shutdown. RecreateSwapchain waits idle — implicitly waits all fences.

**Deadlock pattern:** CPU waits on fence never signaled because command buffer never submitted — happens when acquire OUT_OF_DATE returns early without advancing frame index; verify Spark early-return path does not skip fence signal schedule.

Name fences `FrameFence[i]` in debug utils — validation "fence in use" messages correlate to frame index.

### Instance and layer enumeration quick ref

| Step | API | Spark location |
|------|-----|----------------|
| Enable layers | VkInstanceCreateInfo ppEnabledLayerNames | VulkanDeviceContext bootstrap |
| Extensions | VK_KHR_surface, swapchain, debug utils | Same |
| Physical device | vkEnumeratePhysicalDevices + score | PickDiscreteGpu helper |
| Queue families | graphics + present same or different | QueueFamilyIndices |
| Logical device | VkDeviceCreateInfo | After queue selection |

Validation layers are **not** VkObjects — disable in retail by omitting layer names at instance create, not by failing to destroy layers.

MoltenVK on macOS may report different memory heap layout — lifecycle order identical; budget spreadsheet values differ (appendix ch02 VRAM row notes unified memory).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanDeviceContext bootstrap

**Concept:** Creation order (outside-in)

**Decision:** VulkanDeviceContext bootstrap

**Why:** One code path creates instance through swapchain — demos cannot skip queue family selection.

**In Spark:** `include/spark/render/core/VulkanDeviceContext.hpp`.
### RecreateSwapchain partial teardown

**Concept:** Recreate vs full shutdown

**Decision:** RecreateSwapchain partial teardown

**Why:** Resize must not recreate device — too slow; rebuild extent-sized attachments only.

**In Spark:** `VulkanRenderer::RecreateSwapchain`.
### Pass Destroy() before device

**Concept:** Destruction order (inside-out)

**Decision:** Pass Destroy() before device

**Why:** Each pass frees pipelines in `Destroy(device)`; renderer calls all passes then context.

**In Spark:** ShadowPass, HdrTonemapPass destructors.
### No VkDevice in gameplay

**Concept:** Single owner rule

**Decision:** No VkDevice in gameplay

**Why:** Enforced by include policy — only `render/core` and `render/gpu` include vulkan.h.

**In Spark:** Part I architecture rule.
### Explicit pass Destroy for teaching

**Concept:** RAII vs manual order

**Decision:** Explicit pass Destroy for teaching

**Why:** Clearer in debugger than implicit destructor chain alone for learners.

**In Spark:** VulkanRenderer shutdown sequence.
### CONTRIBUTING.md exit validation ritual

**Concept:** Full lifecycle reference table

**Decision:** CONTRIBUTING.md exit validation ritual

**Why:** Leaks after resize differ from exit leaks — ritual catches both.

**In Spark:** VulkanDeviceContext destructor order.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/core/VulkanDeviceContext.hpp`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Trace RecreateSwapchain partial teardown

Compare what survives vs full shutdown — note device persists.

*WaitIdle and swapchain cleanup start*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 96–110)
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
```
### Step 2 — Read VulkanDeviceContext destructor

List destroy calls for swapchain, device, surface, messenger, instance — number them 1–N on paper.
### Step 3 — Annotate your Shutdown()

Add comments `// step 7: destroy pipelines` matching cheat sheet — code review enforces order.
### Step 4 — Map validation error to fix

Reproduce one intentional leak (skip DestroyImageView) — save validation text in team doc with fix column.
### Step 5 — Adding new pass checklist

Create: render pass → layout → pipeline → framebuffers.
Destroy: pipeline → layout → render pass (before framebuffers if attached).
Insert into RecreateSwapchain list if extent-sized.
### Step 6 — RenderDoc object owner

Capture frame — click image → see debug name if set — correlate to C++ owner class for leak chase.
### Step 7 — Resize vs exit test

Run app, resize, exit — two validation sessions; confirm zero errors in both if lifecycle correct.
### Step 8 — Document in README

Paste creation/destruction ASCII diagram into project wiki — update when adding post-processing pass.

## 5. How This Connects to the Rest of Your Engine

Part **I** walked creation; Part **XVI** documents recreate subset. **Descriptor map** appendix covers pool destroy after pipelines. **SceneRenderParams** appendix is separate CPU contract — not Vulkan handles.


Appendix entries cross-reference each other: lifecycle ↔ descriptor destroy order; SceneRenderParams ↔ component ref; demo catalog ↔ profiling baselines. Part **XVI** shipping template should include printed appendix PDF in onboarding kit for new engineers.


Part **XVI** ch01 RecreateSwapchain partial checklist extends this chapter's full lifecycle table — keep both printed; resize bugs use partial list, quit bugs use full destroy order.

## 6. Pitfalls and Debugging

- **Destroying surface before swapchain** — invalid parent chain on some loaders.
- **Recreate without idle** — flicker or DEVICE_LOST on mobile.
- **Leaking debug messenger** — noisy CI on every test run.
- **Double-destroy after move** — null handles after first Destroy; assign VK_NULL_HANDLE.
- **Command pool destroyed while buffers recording** — finish recording first.


- **Confusing recreate cleanup with shutdown** — different leak signatures; run both rituals.
- **Destroying command pool while recording** — end recording before recreate frees buffers.

## 7. Exercises

1. Print lifecycle reference table; mark custom fork additions.
2. Run exit validation ritual after 60 s demo 17 + resize; zero leaks.
3. Intentionally leak VkImageView; document VUID and fix in team doc.

## Summary

Create outside-in; destroy inside-out after vkDeviceWaitIdle. Spark centralizes device/swapchain in VulkanDeviceContext and partial teardown in RecreateSwapchain. Lifecycle table plus recreate partial checklist and sync destroy timing cover both resize and exit validation failures.

---

Next: [SceneRenderParams Field Reference](02-scene-params-ref.md).
