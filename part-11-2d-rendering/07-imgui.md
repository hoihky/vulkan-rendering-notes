# Dear ImGui

> **Build-Along Milestone:** ImGui exposure slider live-tunes tonemap without recompile; input captured correctly.

**You will learn:**

- Integrate ImGui as a debug overlay recorded after tonemap.
- Feed ImGui draw lists through the swapchain render pass via Vulkan backend.
- Route input using WantCaptureMouse and WantCaptureKeyboard.

## 1. The Problem We Are Solving

Tuning HDR exposure, spawning debug entities, and inspecting frame flags needs **fast iteration UI**. Dear ImGui generates draw lists each frame; your engine uploads them like screen UI but with its own Vulkan backend and descriptor pool.

Spark records ImGui immediately after `screenUi` in the present pass. Milestone: exposure slider live-tunes tonemap while gameplay input pauses when the mouse hovers ImGui panels.

## 1b. Concept Deep Dive — Immediate-Mode Debug UI on the Present Pass

### ImGui rebuilds every frame; screen UI is retained arrays

Dear ImGui is **immediate mode** — each tick you call `ImGui::Button`, ImGui builds transient widget state and draw lists from scratch. Spark's production screen UI (Chapter 6) pushes retained `ScreenRectDraw` structs. ImGui excels at dev tools: exposure sliders, spawn menus, frame stats.

```cpp
// Typical integration loop (concept)
ImGui_ImplVulkan_NewFrame();
ImGui_ImplGlfw_NewFrame();
ImGui::NewFrame();
// ... debug panels ...
ImGui::Render();
RecordImGuiDrawData(commandBuffer);  // after tonemap + screenUi
```

### Same present pass as screen UI, after tonemap

`VulkanRenderer::RecordImGuiDrawData` runs at the end of the present pass so widget colors match display space. NewFrame happens in demo `OnUpdate` **before** snapshot submit — draw data must be recorded after tonemap target is bound.

### Input capture via WantCaptureMouse / WantCaptureKeyboard

Poll ImGui IO flags before gameplay camera controls — when the mouse hovers a panel, disable fly-camera input so sliders do not fight WASD. Spark wraps ImGui behind `IImGuiLayer`; shipping builds can strip the layer while retaining screen UI HUD art.

## 2. Core Concepts

### Immediate-mode vs retained UI

**Dear ImGui** is **immediate mode**: every frame you call `ImGui::Button("Spawn")` and ImGui builds transient widget state, layout, and draw lists from scratch. Spark's screen UI (Chapter 6) is **retained** — you push `ScreenRectDraw` structs meant to ship in production.

ImGui excels at **developer tools**: exposure sliders, spawn menus, collision debug overlays, frame stats. It is not a replacement for player-facing HUD art. Spark wraps ImGui behind `IImGuiLayer` and records draw lists through the same present render pass as `screenUi`, after tonemap, so widget colors match display space.

ImGui rebuilds widget state every frame — no retained tree on engine side. Screen UI arrays are retained per frame snapshot. Do not mix ImGui inside ScreenRectDraw — separate record paths.

Dear ImGui rebuilds draw lists **every frame** from UI function calls — no persistent widget tree in GPU memory. That fits engine tooling: sliders, checkboxes, debug plots. Shipping game menus may still use ImGui for rapid iteration before migrating to screen UI rects. Each ImGui frame calls `NewFrame`, panel code, then `Render` producing draw lists consumed by the Vulkan backend. State is implicit in C++ variables you bind to widgets — document which globals ImGui panels mutate (exposure, spawn flags).

Widget ID strings (`ImGui::PushID`) prevent collisions when multiple panels expose similarly named sliders — use entity ids or pass names as ID scope. Multi-column layouts in debug windows help compare sprite lighting presets side by side without overlapping controls. Persist ImGui `.ini` layout files only in developer builds; shipping builds should not write imgui.ini beside the executable.


### Frame flow each tick

Typical integration loop:

```cpp
ImGui_ImplVulkan_NewFrame();
ImGui_ImplGlfw_NewFrame();  // or your window backend
ImGui::NewFrame();
// ... widgets ...
ImGui::Render();
backend->RecordDrawData(commandBuffer);
```

Spark calls `RecordImGuiDrawData` at the end of the present pass (line 369). NewFrame happens in demo `OnUpdate` or editor hooks **before** snapshot submit. Draw data must be recorded **after** tonemap target is bound — otherwise widgets inherit wrong color space and render pass compatibility.

NewFrame before widget code; Render before RecordDrawData. Spark RecordImGuiDrawData at end of present pass — same command buffer as screenUi.

Order matters: **`ImGui_ImplVulkan_NewFrame`**, **`ImGui_ImplGlfw_NewFrame`**, `ImGui::NewFrame`, gameplay ImGui code, `ImGui::Render`, then engine `RecordImGuiDrawData` inside present pass. Calling NewFrame twice or skipping Render leaves stale draw lists. Input polling should follow ImGui IO capture flags so camera controls disable when hovering widgets. Multi-viewport ImGui is outside Spark baseline — single viewport only. Profile ImGui cost separately from gameplay submit; heavy debug windows can CPU-bound the frame.

Poll GLFW events before ImGui NewFrame so mouse position matches input state ImGui reads. RenderImGui after gameplay submit but before command buffer record ensures slider changes affect same-frame GPU constants if upload happens before record. Missing ImGui::End() on early-return paths corrupts stack — use RAII scope guards in debug panel functions during rapid iteration.


### Vulkan backend responsibilities

`imgui_impl_vulkan` (wrapped by Spark's `IImGuiVulkanBackend`) handles:

| Task | Why it matters |
|------|----------------|
| Font atlas upload | Glyph texture + sampler binding |
| Pipeline compatible with present pass format | UNORM swapchain vs RGBA16F HDR |
| Dynamic scissor from ImDrawCmd | Clipped panels |
| Per-draw texture IDs | Image buttons |

Use a **separate descriptor pool** for ImGui — exhausting the game pool with UI dynamic textures causes mesh draws to fail descriptor allocation. Pipeline creation must list the same render pass handle as the swapchain present pass or compatibility validation fails at startup.

Pipeline must list present renderPass and subpass 0 — HDR pass incompatible. Separate descriptor pool recommended in imgui_impl_vulkan docs.

The backend creates **font image**, sampler, descriptor set, and pipeline compatible with the swapchain render pass used after tonemap. `RecordImGuiDrawData` translates ImDrawData into indexed draws with dynamic vertex/index buffers or scratch buffers per frame. Descriptor pool sizing must account for ImGui texture uploads if you add image widgets — default font is one texture. Pipeline blend state matches premultiplied alpha expectations from ImGui draw lists. Wrong render pass handle at pipeline creation fails at draw time with opaque validation errors — match `VulkanScreenUiPass` pass handle.

Font texture upload uses staging buffer like Part IV textures — track completion before first draw referencing font descriptor. Dynamic vertex buffer ring per frame in flight avoids orphaning ImGui geometry when GPU still reads last frame. Backend shutdown order: wait idle, destroy pipeline, destroy pool, then ImGui context — reversing order causes validation errors on exit harmless in dev but noisy.


### Input capture routing

ImGui sets `io.

WantCaptureMouse` and `io.

WantCaptureKeyboard` when widgets need focus. Spark demos check these flags **before** gameplay camera or movement handlers:

```cpp
if (!ImGui::GetIO().

WantCaptureMouse) {
    camera.

OnMouseMove(...);
}
```

Without routing, clicking an exposure slider also picks entities in the scene or rotates the fly camera. Keyboard focus conflicts (WASD in text field vs player movement) use the same gate. Handle capture in `OnUpdate`, not in Vulkan record.

WantCaptureKeyboard blocks gameplay movement while typing in ImGui text fields — route in shell demo input handler before Platformer movement.`ImGui::GetIO().

WantCaptureMouse` and `WantCaptureKeyboard` gate gameplay input after ImGui frame build. Spark demos skip camera mouse drag when capture is true — replicate in your engine's input facade. Without routing, clicking sliders also picks world objects behind panels. Touch input on mobile builds needs parallel capture flags if you port GLFW patterns. Record capture state in debug overlay when diagnosing "stuck" camera controls during tuning sessions.

Keyboard focus for text input widgets requires WantCaptureKeyboard — platformer jump key must not fire while renaming entity in ImGui text field. Gamepad support in ImGui needs separate IO backend wiring not covered in Spark baseline desktop path. Log input capture flags when diagnosing camera drift during tuning sessions with overlapping fullscreen ImGui windows.


### Live tuning hooks

ImGui sliders often bind directly to CPU pointers already uploaded to GPU — e.g., tonemap exposure float in `ResolvedLighting` or SSAO radius in scene params. Because ImGui runs before submit, changes appear **same frame** without recompile. Document which pointers are safe for live edit (per-frame copied values) vs which require recreation (swapchain extent, render pass handle).

ThreeDDemo and TerrainDemo expose lighting toggles this way — mirror the pattern for your 2D demos when tuning sprite lighting params from Chapter 2.

Sliders on stack locals corrupt GPU next frame — bind to member vars copied in submit. Tonemap exposure lives in resolved lighting struct uploaded per frame.

Exposure, fog density, and sprite lighting params expose well to ImGui sliders because they mutate **scene params or UBO fields** uploaded before render. Avoid Vulkan calls inside ImGui callbacks — queue param changes for next frame upload. Label sliders with units (EV stops, Hz for pulse) so designers communicate with engineering. Serialize tuned values to JSON when leaving a tuning session so gameplay defaults improve without re-sliding. ImGui tuning is development-only in shipping builds — strip panels behind `#ifdef SPARK_EDITOR` or equivalent.

Sliders bound directly to scene param floats propagate next UBO upload — avoid mutating GPU-mapped memory from ImGui thread without synchronization if you ever split threads. Reset-to-default buttons should copy documented demo baseline structs, not zero memory — zero exposure blacks out scene. Serialize tuned params to JSON alongside level files for reproducible bug reports from designers.


### Shipping and compile flags

Spark gates ImGui with `SPARK_ENABLE_IMGUI` — `RecordImGuiDrawData` compiles to empty when disabled. Production builds strip developer overlay while retaining `screenUi` game HUD. Font atlas upload cost is paid once at init; per-frame cost is draw list translation to Vulkan — keep panel count reasonable on low-end targets when profiling Part XVI.

SPARK_ENABLE_IMGUI off compiles empty RecordImGuiDrawData — verify shipping config disables macro. Font atlas GPU memory freed on renderer shutdown.

Disable ImGui **init and record** in shipping configurations to drop descriptor pool and backend memory. Keep screen UI for player-facing HUD — it has fewer dependencies than ImGui. If retaining ImGui for PC debug builds, guard with compile flags and never link in console certification paths. Font embedding increases binary size marginally; texture atlas is runtime-generated. Verify present pass record skips ImGui cleanly when disabled — null checks on backend pointers prevent crashes if init order changes.

Strip ImGui link dependency in shipping via compile definition — guard all includes with same macro. Screen UI remains for player-facing elements with smaller third-party footprint than full ImGui. Certification builds on consoles often forbid runtime shader recompile toggles exposed in ImGui — separate QA-only executables if needed.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### IImGuiLayer abstraction

**Concept:** Immediate-mode vs retained UI

**Decision:** IImGuiLayer abstraction

**Why:** Game code includes ImGui headers only in demo/editor TU — not in core engine lib if stripping builds.

**In Spark:** `include/spark/imgui/IImGuiLayer.hpp`.

### Record after screenUi

**Concept:** Frame flow each tick

**Decision:** Record after screenUi

**Why:** Game HUD draws first; ImGui overlays debug on top without reordering paintOrder arrays.

**In Spark:** `VulkanRenderer.cpp` lines 363–369.

### Dedicated ImGui Vulkan backend impl

**Concept:** Vulkan backend responsibilities

**Decision:** Dedicated ImGui Vulkan backend impl

**Why:** Isolates descriptor pool and pipeline from game resources.

**In Spark:** `RecordImGuiDrawData` → `IImGuiVulkanBackend::RecordDrawData`.

### Check WantCapture* in demos

**Concept:** Input capture routing

**Decision:** Check WantCapture* in demos

**Why:** Prevents debug UI from breaking gameplay input paths.

**In Spark:** ThreeDDemo / shell demo input handlers.

### Float* on tonemap/exposure

**Concept:** Live tuning hooks

**Decision:** Float* on tonemap/exposure

**Why:** Same-frame feedback loop for artists tuning HDR post chain.

**In Spark:** Part X tonemap uniform path.

### SPARK_ENABLE_IMGUI guard

**Concept:** Shipping and compile flags

**Decision:** SPARK_ENABLE_IMGUI guard

**Why:** Zero record cost in shipping when macro off.

**In Spark:** `VulkanRenderer::RecordImGuiDrawData` lines 835–841.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/core/VulkanRenderer.cpp`
- `include/spark/imgui/IImGuiLayer.hpp`

### Step 1 — Find RecordImGuiDrawData

Backend RecordDrawData hook.

*RecordImGuiDrawData*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 835–841)
void VulkanRenderer::RecordImGuiDrawData(const VkCommandBuffer commandBuffer) {
#if SPARK_ENABLE_IMGUI
    if (IImGuiVulkanBackend* backend = TryGetImGuiVulkanBackend(imguiLayer.Get())) {
        backend->RecordDrawData(commandBuffer);
    }
#endif
    (void)commandBuffer;
```

### Step 2 — Confirm present-pass ordering

Tonemap, screenUi, ImGui.

*present order*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 355–371)
    hdrTonemapPass.RecordTonemap(
            commandBuffer,
            imageIndex,
            frameIndex,
            presentRenderPass.vkPass,
            presentSwapchain().extent,
            presentationFramebuffers,
            resolvedLighting.exposure);
    screenUi.Record(
            commandBuffer,
            frameIndex,
            presentSwapchain().extent,
            pendingScene,
            sceneParamsValid);
    RecordImGuiDrawData(commandBuffer);

    vkCmdEndRenderPass(commandBuffer);
```

### Step 3 — Init Vulkan backend once

Descriptor pool and present pass pipeline.

### Step 4 — Upload font atlas

GetTexDataAsRGBA32 upload path.

### Step 5 — Add exposure slider

Live-tune tonemap exposure.

### Step 6 — Input routing test

WantCaptureMouse blocks camera.

### Step 7 — Show Demo Window

ImGui demo over ThreeDDemo.

### Step 8 — Milestone — live exposure tuning

Slider works same frame.

## 5. How This Connects to the Rest of Your Engine

ImGui shares the **present pass** with Chapter 6 screen UI — both after Part X tonemap. Not in HDR scene pass. Works alongside **frame capture** (Part X) and **profiler** (Part XVI). **Sprite lighting** params can expose ImGui sliders in 2D demos for mode 1–3 tuning.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

ImGui shares present pass with Chapter 6 — both display-referred. HDR tuning sliders mutate scene params read by tonemap and lit sprites indirectly through shared UBO fields where wired.

## 6. Pitfalls and Debugging

- **Wrong render pass compatibility** — pipeline creation fails or validation screams on draw.
- **Font texture not bound** — magenta squares instead of glyphs.
- **ImGui before tonemap** — widgets wrong brightness when HDR chain changes exposure.
- **Shared descriptor pool exhaustion** — separate ImGui pool is mandatory at scale.
- **Forgetting WantCapture*** — “buggy” camera while debugging; always route input.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

Pipeline render pass mismatch is the top ImGui Vulkan failure mode — recreate pipeline when swapchain pass handle changes. Forgetting `Render()` before RecordDrawData yields empty overlays with no validation error.

## 7. Exercises

1. Show ImGui demo window over ThreeDDemo.
2. Add slider hooked to SSAO or exposure uniform.
3. Toggle `SPARK_ENABLE_IMGUI` off — confirm link and zero ImGui record cost.
4. Capture RenderDoc — ImGui draws use swapchain format after tonemap.
5. Verify `WantCaptureMouse` blocks gameplay input when hovering panels.

## Summary

Dear ImGui generates immediate-mode overlay draw lists; Spark records them after tonemap via RecordImGuiDrawData and a dedicated Vulkan backend, with input capture gates in demo update loops.

---

Next: [2D Game Pipeline](08-platformer-2d.md).
