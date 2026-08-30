# SSAO

> **Build-Along Milestone:** Indoor corners darken with SSAO enabled; toggle removes the effect without rebuilding pipelines.

**You will learn:**

- Build a screen-space ambient occlusion pass using depth.
- Composite occlusion into HDR scene color before tonemap.
- Gate SSAO with SceneRenderParams.ssaoEnabled.
- Tune radius, bias, and strength via push constants.

## 1. The Problem We Are Solving

Direct lighting and shadow maps handle **sun visibility**, but indirect ambient still fills creases uniformly — indoor corners look like open parking lots. **SSAO** approximates short-range ambient shadowing by sampling depth around each pixel in view space.

It is not physically exact, yet it is cheap and convincing when tuned. Spark implements SSAO in `VulkanScreenSpaceEffectsPass` between the HDR scene pass and tonemap — you will trace depth copy, composite, and the renderer branch that selects SSAO output for tonemap input.

## 1b. Concept Deep Dive — Screen-Space Ambient Occlusion Before Tonemap

### Scene depth stays attachment-only; SSAO copies to a sample image

Main scene depth is bound as `DEPTH_STENCIL_ATTACHMENT_OPTIMAL` during the HDR pass — not safely sampleable for random kernel reads on all drivers. `RecordDepthCopy` transitions scene depth to transfer source, copies into per-flight `depthSampleImage`, then restores layouts:

```cpp
// src/spark/render/post/VulkanScreenSpaceEffectsPass.cpp (lines 525–536)
    barriers[0].oldLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
    barriers[0].newLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
    barriers[0].image = sceneDepth.image;
    // ...
    barriers[1].image = flight.depthSampleImage;
    barriers[1].newLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
```

SSAO runs in **linear HDR** before tonemap — applying occlusion after display encoding produces wrong contrast.

### Scratch RGBA16F avoids read-modify-write on scene HDR

`VulkanScreenSpaceEffectsPass` writes a separate scratch color image: `scratchColor = hdrSceneColor * lerp(1.0, occlusion, strength)`. Tonemap descriptor switches to scratch view when `pendingScene.ssaoEnabled` is true; otherwise raw HDR.

Push constants carry tuning without pipeline rebuild:

```cpp
// include/spark/render/post/VulkanScreenSpaceEffectsPass.hpp — PostPushConstants
float ssaoRadius = 0.35F;
float ssaoBias = 0.02F;
float ssaoStrength = 0.65F;
```

### SSAO modulates ambient, not direct sun shadows

Part IX CSM handles sun visibility. SSAO darkens creases based on **screen-space depth neighbors** — it fails on off-screen occluders and thin geometry, but cheaply sells contact shading on terrain ground and indoor corners. Toggle `ssaoEnabled` off for profiling; zero-cost path skips `Record` entirely.

## 2. Core Concepts

### SSAO purpose

Direct lighting ignores **occluded ambient** — creases, corners, and contact regions look flat. **Screen-space ambient occlusion (SSAO)** darkens pixels whose depth neighbors are consistently closer, approximating short-range shadowing for the ambient term.

```text
For each pixel P:
  reconstruct view-space position from depth
  sample N hemisphere offsets around P
  count samples with depth < expected surface depth
  occlusion = f(sample hits) → modulate ambient
```

SSAO is **not** physically correct global illumination — it is a cheap perceptual cue. It modulates **ambient/skylight**, not direct sun shadows (Part IX).


### Required inputs

| Input | Source | Notes |
|-------|--------|-------|
| HDR color | Scene RGBA16F | Composited output after AO multiply |
| Depth | Scene D32 | Copied to sampleable image |
| Normals | Reconstructed from depth or scene UBO | Kernel orientation |
| Scene UBO | set 0 binding 0 | Inverse projection helpers |

Main scene depth stays **attachment-only** during SSAO — Spark copies depth to a `SAMPLED` image in `RecordDepthCopy` to avoid layout hazards on the live depth buffer.

SSAO runs in **linear HDR** before tonemap — applying AO after display encoding produces wrong contrast.


### Kernel and radius

SSAO quality depends on **sample radius in world/view units**:

```text
occlusion ≈ Σ weight(i) * [depth(sample_i) closer than surface?]
```

Too small a radius — only sub-pixel noise. Too large — false occlusion on background through thin foreground geometry. Spark exposes `ssaoRadius`, `ssaoBias`, and `ssaoStrength` via push constants (defaults in `PostPushConstants`).

**Bias** pushes samples off the surface to reduce self-occlusion stripes on flat floors. **Strength** scales how much ambient is removed — typical 0.5–0.8.


### Half-resolution option

Full-res SSAO at 1080p costs ~2M kernel evaluations per frame. Common optimizations:

| Technique | Tradeoff |
|-----------|----------|
| Half-res SSAO | 4× fewer taps; needs bilateral upsample |
| Rotated kernel | Reduces banding; Spark rotates offsets per pixel |
| Temporal accumulation | Smoother; requires motion vectors (Spark: single-frame) |

Spark runs composite at **full resolution** but tunable radius keeps cost acceptable on integrated GPUs for demo scenes.


### Composite model

`VulkanScreenSpaceEffectsPass` writes a **scratch RGBA16F** image:

```text
scratchColor = hdrSceneColor * lerp(1.0, occlusion, strength)
```

Tonemap reads scratch when SSAO active; otherwise raw HDR. The pass is a dedicated render pass with fullscreen triangle — same mechanical shape as tonemap (Chapter 2).


### Toggle flag

`ssaoEnabled` on `SceneRenderParams` gates the entire pass — essential for profiling and low-end targets. Post volumes (Chapter 4) can override SSAO per region via `PostProcessVolumeComponent`.

```text
ssaoActive = sceneParamsValid && pendingScene.ssaoEnabled
```

When false, renderer skips `screenSpaceEffectsPass.Record` and tonemap binds raw HDR color view.


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


### Depth copy rationale

Scene depth image layout during HDR pass is **DEPTH_STENCIL_ATTACHMENT_OPTIMAL** — not sampleable in all drivers during simultaneous attachment use. SSAO copies depth to `depthSampleImage` with `SAMPLED` usage in `RecordDepthCopy` before kernel loops — avoids layout hazards and validation errors.


### SSAO limitations

SSAO darkens based on **screen-space** depth neighbors — fails on off-screen occluders, thin geometry, and moving objects without temporal reprojection. It modulates ambient, not direct sun shadows (Part IX). Treat as cosmetic contact enhancement, not physical AO.


### Tonemap input switch

When `ssaoActive`, `UpdateTonemapDescriptor` binds SSAO scratch view; else HDR color view. Forgetting to update descriptor after toggle shows scene without AO or black tonemap input — verify binding in RenderDoc after `ssaoEnabled` changes.


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

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanScreenSpaceEffectsPass

**Concept:** SSAO purpose

**Decision:** VulkanScreenSpaceEffectsPass

**Why:** Dedicated class owns depth copy, composite render pass, pipeline, and scratch targets — keeps scene.frag free of multi-pass AO logic.

**In Spark:** Header comment: 'SSAO composite pass (HDR → scratch, before tonemap)'.

### Depth copy to sample image

**Concept:** Required inputs

**Decision:** Depth copy to sample image

**Why:** Scene depth buffer remains bound for attachment layout; SSAO copies to `depthSampleImage` safe for random kernel reads.

**In Spark:** `RecordDepthCopy` in `VulkanScreenSpaceEffectsPass.cpp`.

### Push constants for tuning

**Concept:** Kernel and radius

**Decision:** Push constants for tuning

**Why:** `PostPushConstants` carries enabled flag, radius, bias, strength, depthFlipV — tweak in ImGui without pipeline rebuild.

**In Spark:** `VulkanScreenSpaceEffectsPass.hpp` lines 76–85.

### Scratch HDR target per flight

**Concept:** Composite model

**Decision:** Scratch HDR target per flight

**Why:** Separate RGBA16F image avoids read-modify-write on scene HDR during composite; tonemap descriptor switches input view.

**In Spark:** `UpdateTonemapDescriptor(device, frameIndex, ssaoOutputView)`.

### pendingScene.ssaoEnabled

**Concept:** Toggle flag

**Decision:** pendingScene.ssaoEnabled

**Why:** Renderer branch at line 334 — zero cost path when disabled (no Record, raw HDR tonemap).

**In Spark:** `const bool ssaoActive = sceneParamsValid && pendingScene.ssaoEnabled;`

### Full-res default with tunable radius

**Concept:** Half-resolution option

**Decision:** Full-res default with tunable radius

**Why:** Spark prioritizes clarity in demos over half-res upsample complexity; radius default 0.35 world units.

**In Spark:** `PostPushConstants::ssaoRadius = 0.35F`.

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

- `include/spark/render/post/VulkanScreenSpaceEffectsPass.hpp`
- `src/spark/render/post/VulkanScreenSpaceEffectsPass.cpp`
- `src/spark/render/core/VulkanRenderer.cpp`
- `include/spark/engine/SceneRenderParams.hpp`
- `src/spark/scene/RenderVolumes.cpp`

### Step 1 — Read ScreenSpaceEffectsPass header

Understand flight targets, depth copy image, and push constant tuning fields before opening the `.cpp` record path.

*SSAO pass structure and PostPushConstants*

```cpp
// include/spark/render/post/VulkanScreenSpaceEffectsPass.hpp (lines 15–85)
/** SSAO composite pass (HDR → scratch, before tonemap). */
class VulkanScreenSpaceEffectsPass {
public:
    static constexpr VkFormat kColorFormat = VulkanHdrTonemapPass::kColorFormat;

    struct FlightTarget {
        VkImage colorImage = VK_NULL_HANDLE;
        VkDeviceMemory colorMemory = VK_NULL_HANDLE;
        VkImageView colorView = VK_NULL_HANDLE;
        VkFramebuffer framebuffer = VK_NULL_HANDLE;
        VkImageLayout colorLayout = VK_IMAGE_LAYOUT_UNDEFINED;

        /** Depth copied from the scene pass — safe to sample (main depth stays attachment-only). */
        VkImage depthSampleImage = VK_NULL_HANDLE;
        VkDeviceMemory depthSampleMemory = VK_NULL_HANDLE;
        VkImageView depthSampleView = VK_NULL_HANDLE;
        VkImageLayout depthSampleLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    };

    void CreateRenderPass(VkDevice device);
    void DestroyRenderPass(VkDevice device);

    void RecreateFlightTargets(
            VkPhysicalDevice physicalDevice,
            VkDevice device,
            VkExtent2D extent,
            VkFormat depthFormat,
            std::uint32_t framesInFlight);
    void DestroyFlightTargets(VkDevice device);

    void CreatePipeline(
            VkPhysicalDevice physicalDevice,
            VkDevice device,
            std::uint32_t framesInFlight,
            const VulkanSpvShaderLoader& shaders);
    void DestroyPipeline(VkDevice device);
    void UpdateDescriptor(
            VkDevice device,
            std::uint32_t flightIndex,
            VkBuffer sceneUbo,
            const VulkanHdrTonemapPass::FlightTarget& hdrFlight,
            const FlightTarget& effectsFlight);

    void Record(
            VkCommandBuffer commandBuffer,
            std::uint32_t frameIndex,
            VkExtent2D extent,
            const VulkanDepthResources& sceneDepth,
            const SceneRenderParams& scene);

    [[nodiscard]] bool HasFlight(std::uint32_t frameIndex) const noexcept {
        return frameIndex < flights.GetSize();
    }
    [[nodiscard]] VkImageView OutputView(std::uint32_t frameIndex) const noexcept {
        return (frameIndex < flights.GetSize()) ? flights[frameIndex].colorView : VK_NULL_HANDLE;
    }
    [[nodiscard]] const FlightTarget& Flight(std::uint32_t frameIndex) const noexcept {
        return flights[frameIndex];
    }

private:
    struct PostPushConstants {
        float ssaoEnabled = 0.0F;
        float ssaoRadius = 0.35F;
        float ssaoBias = 0.02F;
        float ssaoStrength = 0.65F;
        float depthFlipV = 0.0F;
        float pad0 = 0.0F;
        float pad1 = 0.0F;
        float pad2 = 0.0F;
    };
```

### Step 2 — Follow SSAO branch in renderer

When active: update descriptors with scene UBO + HDR flight + effects flight, record screen space pass, point tonemap at SSAO output.

*SSAO active path and tonemap descriptor update*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 334–353)
    const bool ssaoActive = sceneParamsValid && pendingScene.ssaoEnabled;
    if (ssaoActive && frameIndex < sceneDescriptors.DescriptorSetCount() && hdrTonemapPass.HasFlight(frameIndex) &&
        screenSpaceEffectsPass.HasFlight(frameIndex)) {
        screenSpaceEffectsPass.UpdateDescriptor(
                device(),
                frameIndex,
                sceneDescriptors.UniformBuffer(frameIndex),
                hdrTonemapPass.Flight(frameIndex),
                screenSpaceEffectsPass.Flight(frameIndex));
        screenSpaceEffectsPass.Record(
                commandBuffer,
                frameIndex,
                presentSwapchain().extent,
                sceneDepthResources[frameIndex],
                pendingScene);
        hdrTonemapPass.UpdateTonemapDescriptor(
                device(), frameIndex, screenSpaceEffectsPass.OutputView(frameIndex));
    } else {
        hdrTonemapPass.UpdateTonemapDescriptor(device(), frameIndex);
    }
```

### Step 3 — Enable SSAO in demo

Run **ThreeDDemo** or **TimeOfDayDemo** with `ssaoEnabled` true. Toggle off via post volume or scene params and compare corner shading in an enclosed space.

### Step 4 — Debug raw occlusion

Temporarily output occlusion factor as grayscale before composite multiply. Creases and contacts should highlight — not the entire screen. Mask sky pixels at far depth to avoid false darkening.

### Step 5 — Tune radius and bias

Increase radius until ground contact darkens convincingly; add bias if flat surfaces show stripes. Log push constant values that work for your test scene scale (meters per unit).

### Step 6 — Measure GPU cost

Profile scene only vs +SSAO on your target GPU. Record timestamp queries around `screenSpaceEffectsPass.Record` — budget typically 0.5–2 ms at 1080p.

### Step 7 — Verify linear composite

Confirm modulation happens in linear HDR **before** tonemap — not on swapchain. Capture HDR scratch buffer in RenderDoc; values should remain float16, not UNORM8.

### Step 8 — Depth copy layout

Trace `RecordDepthCopy` barriers: scene depth `DEPTH_STENCIL_ATTACHMENT_OPTIMAL` → copy → `depthSampleImage` `SHADER_READ_ONLY_OPTIMAL`. Missing barrier causes validation error or stale depth.

## 5. How This Connects to the Rest of Your Engine

SSAO depends on **HDR depth** from Chapter 1. Part **XII** terrain benefits from ground contact AO. Part **IX** sun shadows are independent — SSAO does not replace them. If you later build deferred G-buffer, SSAO inputs become richer (explicit normals, motion vectors).


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **SSAO on sky pixels** — mask by depth at far plane or reconstruct position validity.
- **Resolution mismatch** — half-res needs careful bilateral upsample to avoid edge halos.
- **Temporal noise without accumulation** — rotate kernel per frame; static grid banding shimmers.
- **Composite after tonemap** — wrong space; contrast looks baked and cannot be exposure-corrected.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Visualize raw SSAO factor, then blurred, then composited HDR.
2. Disable SSAO — compare corner shading indoors on ThreeDDemo.
3. Measure GPU ms: scene only vs +SSAO at 1080p and 4K.
4. Sweep ssaoRadius from 0.1 to 1.0 — document artifact thresholds.
5. Verify tonemap reads SSAO scratch view when ssaoEnabled is true.

## Summary

SSAO darkens ambient based on nearby depth in screen space. Spark copies scene depth for sampling, composites into HDR scratch via VulkanScreenSpaceEffectsPass, and routes scratch to tonemap when enabled.

---

Next: [Exposure Volumes](04-exposure-volumes.md).
