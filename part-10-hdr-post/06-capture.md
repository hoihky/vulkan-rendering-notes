# Capture Tools

> **Build-Along Milestone:** F12 saves a PNG of the current frame without validation errors.

**You will learn:**

- Save HDR and LDR screenshots from specific render stages.
- Read back images with correct layout transitions and fences.
- Wire dev hotkeys without stalling every frame.
- Extend capture service beyond swapchain to HDR diagnostics.

## 1. The Problem We Are Solving

Debugging HDR, shadows, and SSAO requires **seeing intermediate buffers** — not just the final tonemapped image. **Capture tools** copy GPU images to CPU files after safe layout transitions, fence waits, and correct row pitch alignment.

Spark includes `VulkanFrameCapture` on the renderer with swapchain PNG and video paths — you will follow the record hook, then extend the same staging pattern to HDR attachments for linear diagnosis.

## 1b. Concept Deep Dive — GPU Readback Without Stalling Every Frame

### Request flag defers copy to command buffer record

Capture is dev-triggered, not per-frame. `RequestScreenshotSave` sets a pending path; the next `RecordSceneCommandBuffer` inserts the copy after tonemap, screen UI, and ImGui so captures include overlays when desired:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 409–412)
    frameCapture.RecordCopyFromSwapchain(
            commandBuffer,
            presentSwapchain().images[imageIndex],
            presentSwapchain().extent,
            frameIndex);
```

File I/O runs in `OnFlightFenceSignaled` after fence wait — not inside submit from gameplay thread.

### Staging path: layout transition → copy → restore present layout

Swapchain image must transition `PRESENT_SRC_KHR` → `TRANSFER_SRC_OPTIMAL` → copy to HOST_VISIBLE staging → restore before WSI continues. Row pitch must respect `bufferImageGranularity` — wrong pitch skews PNG diagonally.

```text
DEVICE_LOCAL swapchain
  → TRANSFER_SRC layout
  → vkCmdCopyImageToBuffer → staging buffer
  → fence wait → mmap → stb PNG encode
  → restore PRESENT_SRC_KHR
```

### Extending to HDR float16 capture

Swapchain PNG is display-referred UNORM8. HDR stage capture reuses the same pattern against `R16G16B16A16_SFLOAT` with different row pitch — apply exposure + ACES on CPU when writing 8-bit preview, or export `.exr` to preserve values above 1.0. Guard during swapchain `OUT_OF_DATE` resize to avoid corrupt captures.

## 2. Core Concepts

### Readback pipeline

Copying GPU images to CPU files requires a **staging path**:

```text
DEVICE_LOCAL color image
  → layout TRANSFER_SRC
  → vkCmdCopyImageToBuffer
  → HOST_VISIBLE staging buffer
  → fence wait
  → mmap → encode PNG / EXR
  → restore image layout for present
```

Each transition must respect queue family and `bufferImageGranularity` alignment for row pitch.


### Capture stages

| Stage | Format | Use case |
|-------|--------|----------|
| Swapchain (tonemap out) | UNORM8 | Share screenshots, video |
| HDR scene color | RGBA16F | Linear diagnosis, exposure debug |
| Shadow depth | D32 | Shadow acne debugging |
| SSAO scratch | RGBA16F | AO tuning |

Parameterize capture by **stage enum** — one service, many attachment sources.


### Dev-only triggers

Capture on **key press**, not every frame — readback stalls the pipeline. Guard during swapchain `OUT_OF_DATE` resize to avoid copying invalid images.

```cpp
if (input.IsKeyPressed(Key::F12))
    frameCapture.RequestScreenshotSave(path);
```

Strip from shipping builds with `DEVELOPER_TOOLS` ifdef if capture adds measurable frame time.


### Swapchain capture hook

Spark records swapchain copy at end of scene command buffer when screenshot or video requested:

```text
RecordSceneCommandBuffer
  → ... tonemap, UI, ImGui ...
  → frameCapture.RecordCopyFromSwapchain(swapchainImage, ...)
```

Post-submit work (`OnFlightFenceSignaled`) maps staging and writes PNG via stb — keeps command buffer recording separate from file I/O.


### HDR to 8-bit preview

Saving HDR as PNG requires **tonemap on CPU** or documented preview exposure — otherwise float16 values clip to white in 8-bit encoders.

```text
pngRgb = TonemapCpu(hdrRgb * previewExposure)
```

Optional `.exr` export preserves true HDR for technical review. Match preview exposure to in-game tonemap for WYSIWYG captures.


### Video recording

`VulkanFrameCapture` multiplexes screenshot and **video** capture through shared staging setup. Video encodes consecutive swapchain frames after fence wait — profile disk bandwidth separately from PNG single-shot cost.


### Linear workflow audit

Trace one texel from albedo file to monitor:

```text
sRGB PNG → sparkSrgbToLinear → BRDF → RGBA16F → × exposure → ACES → gamma → UNORM swapchain
```

Any extra `pow(..., 2.2)` or missing decode step washes out contrast. HDR headroom only helps if sun values **exceed 1.0** in the float buffer before tonemap — verify in RenderDoc texture inspect.


### Post pass isolation

Tonemap and SSAO share the **fullscreen triangle** pattern but target different render passes and input views. Keeping them in separate classes prevents scene code from accidentally binding swapchain during world draw. Each post pass owns its push constants — exposure vs ssaoRadius — avoiding UBO reallocation per frame.


### Operational depth and demo validation

### Operational depth

Spark's reference demos (**ThreeDDemo**, **TerrainDemo**, **TimeOfDayDemo**) exist to validate each rendering contract in isolation before you merge subsystems in your engine. Run the demo that matches this chapter's topic after every implementation step — if Spark looks correct and your port does not, the bug is almost always struct layout, pass order, or matrix convention rather than a missing Vulkan feature.

**Descriptor set 0** carries the scene UBO, texture arrays, shadow maps, and punctual shadow SSBO. When debugging binding errors, use RenderDoc's pipeline state view to compare Spark's bound descriptors against yours for the same draw index. A common regression when adding shadows or HDR is recreating the descriptor set layout without re-binding shadow map views after swapchain recreate.

**Frame-in-flight indexing** must stay consistent: CPU writes UBO slot *i* while GPU may read slot *i* from the previous frame's submit. Shadow matrices uploaded in `WriteUniformBuffer(frameIndex)` must match the `frameIndex` passed to `RecordShadowMapPass` and the scene pass descriptor bind — mixing indices produces one-frame-late shadows that look like temporal filtering bugs.

When extending Spark's pattern, prefer **pure CPU modules** (cascade math, render volume resolve) that you can unit-test without Vulkan instances. GPU record functions should consume precomputed arrays — not re-derive frustum corners inside tight draw loops.


### Staging buffer alignment

`vkCmdCopyImageToBuffer` row pitch must respect `bufferImageGranularity`. PNG encoders need padded rows — stb_image_write uses width × components, not necessarily `rowPitch`. Wrong pitch skews captured images diagonally.


### Swapchain layout chain

Capture path: `PRESENT_SRC_KHR` → `TRANSFER_SRC_OPTIMAL` → copy → restore present layout before WSI continues. Missing restore breaks subsequent frames or validation in present chain.


### HDR capture encoding

RGBA16F → PNG requires CPU tonemap preview. Copy raw half-float to `.exr` if preserving linear radiance for technical review. Match preview exposure to in-game `resolvedLighting.exposure` for WYSIWYG screenshots.


### HDR debugging in RenderDoc

After capturing **ThreeDDemo**, select the HDR color attachment before tonemap. Texture inspect on sun-facing pixels — linear float values should exceed 1.0. If all values ≤ 1.0, either exposure is too low, sun intensity is clamped in scene shader, or you are viewing the wrong attachment (swapchain instead of RGBA16F).

Compare the same pixel after tonemap pass — display values should sit in 0–1 without channel clipping to exactly 1.0 on all RGB unless intentionally blown out.

SSAO scratch (when enabled) should show modulated HDR — not identical to raw scene unless strength is zero.


### Post stack ordering invariants

These invariants must hold every frame:

```text
1. Shadow passes complete before HDR scene begin
2. HDR scene end → color in SHADER_READ layout
3. SSAO (if on) reads HDR + copied depth → scratch RGBA16F
4. Tonemap reads scratch OR raw HDR → swapchain
5. Screen UI + ImGui after tonemap (display space)
```

Violating order 4 with UI before tonemap makes widgets exposure-scaled. Violating order 2 causes validation errors when SSAO or tonemap sample HDR.


### Video capture path

`VulkanFrameCapture` multiplexes PNG and MP4 through shared staging — video records consecutive frames after fence wait. Profile disk IO separately from PNG — encoding can dominate frame time if synchronous on gameplay thread; keep finalize on `OnFlightFenceSignaled` path off critical user input.


### ScreenshotCapture implementation notes

`VulkanScreenshotCapture` allocates staging buffer sized to swapchain extent × 4 bytes RGBA. `RecordCopyFromSwapchain` inserts barriers, copies, records — file write deferred to `OnFlightFenceSignaled` after fence wait so gameplay thread does not block inside `vkQueueSubmit`.

For HDR stage capture, duplicate pattern with `R16G16B16A16_SFLOAT` format — row pitch may differ from swapchain RGBA8; verify mapped pointer stride when writing PNG.


### Developer tools guard

Wrap capture request APIs in `#ifdef DEVELOPER_TOOLS` for shipping builds — readback stalls and staging memory have no place in release. Hotkey registration in demo layer, not core renderer — keeps production binary lean while demos retain F12 screenshot workflow.


### RecordCopyFromSwapchain ordering

Spark records video copy before screenshot copy in `RecordCopyFromSwapchain` — both may request the same swapchain image in one frame when recording video while screenshot hotkey fires. Ensure staging buffer sizing accommodates max(extent) and that layout transitions happen once per target image per frame to avoid redundant barriers.


### Regression PNG workflow

Automate capture: launch demo headless or scripted camera pose, request screenshot, compare SHA256 against golden PNG in CI. Spark's capture path is deterministic enough for sun angle + exposure fixed — flaky tests usually mean missing fence wait or OUT_OF_DATE during resize. Retry capture after swapchain stable.


### EXR export note

For HDR archival, write half-float RGBA to OpenEXR instead of 8-bit PNG — preserves values above 1.0 for technical review in Nuke or Resolve without guessing preview exposure.


### Capture checklist

Before merging HDR work: (1) F12 PNG saves, (2) no validation on capture frame, (3) row pitch matches width×4, (4) layout restored to PRESENT_SRC, (5) resize during capture handled gracefully.


### Swapchain format capture

PNG from swapchain assumes B8G8R8A8 or similar 8-bit RGBA layout — verify component swizzle when importing into image tools. Screenshot alpha may be opaque 1.0 always if tonemap writes alpha=1.


### Summary checkpoint

Milestone: F12 saves PNG without validation errors after tonemap and UI draws complete.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Centralized VulkanFrameCapture

**Concept:** Readback pipeline

**Decision:** Centralized VulkanFrameCapture

**Why:** One class owns staging buffers, layout transitions, and fence synchronization — demos request captures through renderer API.

**In Spark:** `include/spark/render/core/VulkanFrameCapture.hpp`.

### Start with swapchain; extend enum

**Concept:** Capture stages

**Decision:** Start with swapchain; extend enum

**Why:** Swapchain capture validates present path; HDR/shadow stages reuse same staging pattern with different source image.

**In Spark:** Extend `VulkanScreenshotCapture` pattern in Spark.

### Request flag, record on next frame

**Concept:** Dev-only triggers

**Decision:** Request flag, record on next frame

**Why:** Defer copy to command buffer record after user input — avoids mid-frame synchronization from gameplay thread.

**In Spark:** `RequestScreenshotSave` sets pending path for next frame.

### RecordCopyFromSwapchain end of CB

**Concept:** Swapchain capture hook

**Decision:** RecordCopyFromSwapchain end of CB

**Why:** After UI/ImGui so captures include overlays when desired.

**In Spark:** `VulkanRenderer.cpp` lines 373–378.

### Apply exposure on CPU encode

**Concept:** HDR to 8-bit preview

**Decision:** Apply exposure on CPU encode

**Why:** When adding HDR stage capture, multiply by `resolvedLighting.exposure` before ACES/gamma on CPU.

**In Spark:** Document preview exposure in capture README.

### Shared staging with screenshot

**Concept:** Video recording

**Decision:** Shared staging with screenshot

**Why:** `VulkanVideoCapture` sub-object under `VulkanFrameCapture` — same recreate on swapchain path.

**In Spark:** `frameCapture.BeginVideoRecording` API on renderer.

### color_space.glsl include

**Concept:** Linear workflow audit

**Decision:** color_space.glsl include

**Why:** Single sRGB decode function shared across scene and UI paths that sample albedo.

**In Spark:** `shaders/color_space.glsl` sparkSrgbToLinear.

### Separate VulkanHdrTonemapPass and VulkanScreenSpaceEffectsPass

**Concept:** Post pass isolation

**Decision:** Separate VulkanHdrTonemapPass and VulkanScreenSpaceEffectsPass

**Why:** Scene module never imports swapchain present pass — only HDR handle.

**In Spark:** `include/spark/render/post/` module split.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/core/VulkanFrameCapture.hpp`
- `src/spark/render/core/VulkanFrameCapture.cpp`
- `src/spark/render/core/VulkanRenderer.cpp`
- `src/spark/render/capture/VulkanScreenshotCapture.cpp`
- `src/spark/render/capture/VulkanVideoCapture.cpp`

### Step 1 — Read VulkanFrameCapture header

Understand public API: recreate on swapchain, request screenshot, record copy, post-submit finalize.

*Frame capture API surface*

```cpp
// include/spark/render/core/VulkanFrameCapture.hpp (lines 14–44)
/** Swapchain-tied PNG screenshot and MP4 video readback (shared staging setup). */
class VulkanFrameCapture {
public:
    void RecreateSwapchainResources(
            VkPhysicalDevice physicalDevice,
            VkDevice device,
            VkFormat swapchainFormat,
            VkExtent2D extent);
    void Destroy(VkDevice device) noexcept;

    void RequestScreenshotSave(const char* pathUtf8);
    [[nodiscard]] bool BeginVideoRecording(
            UniquePtr<VideoRecorder> recorder,
            const VideoRecordingSettings& settings,
            VkExtent2D extent);
    void EndVideoRecording() noexcept;
    [[nodiscard]] bool IsVideoRecording() const noexcept;
    [[nodiscard]] VideoRecorder* GetRecorder() noexcept;

    [[nodiscard]] bool NeedsPostSubmitWork() const noexcept;

    void RecordCopyFromSwapchain(
            VkCommandBuffer commandBuffer,
            VkImage swapchainImage,
            VkExtent2D extent,
            std::uint32_t flightIndex);

    /** After the per-flight fence has been waited on at frame start. */
    void OnFlightFenceSignaled(std::uint32_t waitedFlightIndex) noexcept;

    void FlushPendingCaptures(VkDevice device, const VkFence* inFlightFences) noexcept;
```

### Step 2 — Find swapchain copy record site

Copy records at end of scene command buffer when capture requested for that frame index.

*Swapchain capture record hook*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 373–378)
    if (imageIndex < presentSwapchain().images.GetSize()) {
        frameCapture.RecordCopyFromSwapchain(
                commandBuffer,
                presentSwapchain().images[imageIndex],
                presentSwapchain().extent,
                frameIndex);
```

### Step 3 — Implement request flag

Wire hotkey in demo: `RequestScreenshotSave("capture.png")` — verify PNG appears after one frame without validation errors.

### Step 4 — Layout transitions for copy

Transition swapchain image `PRESENT_SRC_KHR` → `TRANSFER_SRC_OPTIMAL` → copy → restore before present chain continues. Missing restore breaks WSI.

### Step 5 — Fence and map staging

Submit copy with fence; in `OnFlightFenceSignaled`, map staging buffer respecting `rowPitch`, encode with stb_image_write.

### Step 6 — HDR capture variant

Add stage selector: copy HDR RGBA16F after scene pass (before tonemap). Apply exposure + ACES on CPU when writing 8-bit PNG preview.

### Step 7 — Compare with RenderDoc

RenderDoc supersedes many manual captures for one-off debug — use automated PNG for regression tests and artist folder exports.

### Step 8 — Guard resize

Skip capture when swapchain returns OUT_OF_DATE — queue recreation before readback or row pitch mismatches corrupt PNG.

### Step 9 — Trace OnFlightFenceSignaled

Post-submit screenshot finalize path — keeps file I/O off record thread.

*PNG save after fence signaled*

```cpp
// src/spark/render/core/VulkanFrameCapture.cpp (lines 62–67)
void VulkanFrameCapture::OnFlightFenceSignaled(const std::uint32_t waitedFlightIndex) noexcept {
    (void)screenshotCapture.TrySavePendingPngForFlight(waitedFlightIndex);
    if (videoCapture.IsRecording()) {
        videoCapture.TryCommitFlightCapture(waitedFlightIndex);
    }
}
```

## 5. How This Connects to the Rest of Your Engine

Capture validates Part **I** sync understanding (fences, layouts). Works with Parts **IX–X**: shadow depth, HDR, AO, tonemap — parameterize source image by enum. Part **XI** UI included if capture records after ImGui.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Reading swapchain without layout transition** — validation error or corrupted PNG.
- **Row pitch alignment** — respect `bufferImageGranularity`; pad rows when mapping.
- **Capturing during resize** — guard OUT_OF_DATE; extent mismatch corrupts image.
- **HDR saved as 8-bit without tonemap** — all white pixels; apply preview exposure on CPU.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Capture swapchain PNG via F12 — verify no validation errors.
2. Add HDR stage capture; compare sun pixel value before/after CPU tonemap.
3. Write depth linearization for grayscale shadow debug PNG.
4. Measure stall ms from synchronous readback vs async fence path.
5. Document capture hotkeys in demo README for your team.

## Summary

Capture tools copy GPU images through staging buffers after layout transitions. Spark's `VulkanFrameCapture` records swapchain copies at command buffer end; extend the same pattern to HDR float16 for linear debugging.

---

Continue to [Part XI — 2D Rendering](../part-11-2d-rendering/index.md).
