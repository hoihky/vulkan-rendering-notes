# ACES Tonemapping

> **Build-Along Milestone:** Bright sun and emissive surfaces compress to visible detail on the swapchain after tonemap.

**You will learn:**

- Implement a full-screen tonemap pass from HDR texture to swapchain.
- Apply ACES filmic approximation with exposure multiplier.
- Handle gamma encoding for correct display output.
- Place tonemap after SSAO and before screen UI in the frame graph.

## 1. The Problem We Are Solving

HDR scene color holds values far above 1.0 — a physically plausible sun term can reach 5–20 in linear space. Monitors expect **display-referred** signals in 0–1. Without tonemapping, either highlights clip to white or the entire scene looks dim.

**Tonemapping** compresses dynamic range while preserving hue. **ACES** is the industry starting point before artists apply LUTs or grade in compositing. Spark's `tonemap.frag` is a minimal, readable reference: exposure multiply, fitted ACES, gamma encode.

You will follow `RecordTonemap` in the frame graph and reproduce the fullscreen pass in your engine.

### Where tonemap sits in the full rendering pipeline

Tonemap is the bridge between **scene-referred HDR** and **display-referred swapchain**. In Spark's frame graph it runs after all world content and optional SSAO:

```text
Shadow passes
  → HDR render pass (scene + transparent resume + 2D + particles)
  → TransitionColorToShaderRead
  → [optional] SSAO pass (reads HDR + depth)
  → RecordTonemap (HDR → swapchain, present render pass)
  → Screen UI + ImGui (display space, not exposure-scaled)
```

Emissive materials authored in Part VII can exceed 1.0 in the HDR buffer **before** tonemap. PBR specular highlights on metals similarly accumulate in linear space. If you skip HDR offscreen targets (Part X Chapter 1) and draw directly to swapchain, tonemap has nothing meaningful to compress — highlights clip at 1.0.

`VulkanHdrTonemapPass::RecordTonemap` binds the HDR color image as a combined image sampler, multiplies by `resolvedLighting.exposure`, runs ACES in `tonemap.frag`, and writes UNORM swapchain color. UI draws afterward so widgets are not exposure-scaled.

## 1b. Concept Deep Dive — Scene-Referred to Display-Referred in One Draw

### Tonemap is a fullscreen pass with push constants, not a UBO

Spark's tonemap shader samples linear HDR, scales by exposure, applies Stephen Hill's ACES fitted curve, then gamma-encodes for UNORM swapchain output. No vertex buffer — three corners generate UVs from clip-space positions:

```glsl
// shaders/tonemap.frag (lines 16–29)
vec3 acesTonemap(vec3 x) {
    const float a = 2.51;
    const float b = 0.03;
    const float c = 2.43;
    const float d = 0.59;
    const float e = 0.14;
    return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}

void main() {
    vec3 hdr = texture(hdrColor, vUv).rgb * max(push.exposure, 1e-4);
    vec3 ldr = acesTonemap(hdr);
    ldr = pow(ldr, vec3(max(push.invGamma, 1e-3)));
    outColor = vec4(ldr, 1.0);
}
```

Exposure and `invGamma` ride in push constants — tweak in ImGui without descriptor churn.

### SSAO may feed tonemap input

When `ssaoActive`, the renderer updates the tonemap descriptor to sample SSAO scratch RGBA16F instead of raw scene HDR. When false, tonemap binds the HDR color view directly. UI and ImGui record **after** tonemap so HUD colors stay display-referred and ignore exposure volumes.

```text
HDR scene end → [optional] SSAO composite → RecordTonemap(exposure) → screenUi → ImGui
```

### Why ACES beats naive Reinhard

Reinhard `x/(1+x)` desaturates bright yellows and blues differently — skies look chalky. ACES preserves hue through the shoulder region where sun-facing diffuse (~5–20 linear) maps to visible monitor detail. Verify in RenderDoc: tonemap pass should be exactly **one draw** per swapchain image.

## 2. Core Concepts

### Tonemap pipeline shape

Tonemap is a **fullscreen pass** with no depth attachment:

```text
Input:  sampler2D hdrColor   (linear RGB16F from scene or SSAO scratch)
Output: swapchain image     (UNORM display-referred)
Uniform: exposure, invGamma (push constants)
```

```text
  ┌─────────────┐     ┌──────────────┐     ┌────────────┐
  │ HDR texture │ ──► │ Tonemap FS   │ ──► │ Swapchain  │
  └─────────────┘     │ exposure×    │     └────────────┘
                      │ ACES curve   │
                      │ gamma pow    │
                      └──────────────┘
```

Spark draws a **two-triangle strip** via `tonemap.vert` — three vertices, UV from position. Load op on swapchain can be `DONT_CARE` when the pass covers the full framebuffer.


### Exposure multiplier

Before tonemap, scale HDR linear color:

```text
hdrScaled = hdrLinear * exposure
```

Exposure is a **linear multiplier**, not EV — doubling exposure doubles radiance. Spark passes `resolvedLighting.exposure` from CPU resolve (global + post volumes in Chapter 4). Typical outdoor values range 0.4–1.5 depending on sun intensity and artistic intent.

Exposure affects **all pixels equally** in the tonemap pass — it does not replace per-light intensity. Use it to frame the scene inside the filmic curve's sweet spot without re-authoring light energies.


### ACES fitted curve

Naive Reinhard `x / (x + 1)` desaturates highlights. **ACES** (Academy Color Encoding System) filmic approximations preserve hue better:

```glsl
// Stephen Hill's fit — tonemap.frag
(x * (a*x + b)) / (x * (c*x + d) + e)
// a=2.51, b=0.03, c=2.43, d=0.59, e=0.14
```

```text
input linear HDR  ──► ACES ──► LDR [0,1] per channel
        10.0              ~0.95
         1.0              ~0.78
         0.1              ~0.12
```

The curve has **toe** (shadow detail), **linear midsection**, and **shoulder** (highlight rolloff). Compare against Reinhard side-by-side on a sun-facing wall to see hue retention.


### Gamma encode

After tonemap, apply **display gamma**:

```glsl
ldr = pow(ldr, vec3(1.0 / gamma));   // gamma ≈ 2.2
```

When the swapchain format is **UNORM without sRGB flag**, the GPU does not apply transfer function — the shader must encode. If the swapchain is **`VK_FORMAT_B8G8R8A8_SRGB`**, some engines skip shader gamma — Spark uses push constant `invGamma` for artist tuning regardless.

**Double-encoding** (sRGB albedo decode + tonemap gamma + sRGB swapchain) makes images dull. Decode albedo **once** on input; encode **once** at display output.


### Post-scene ordering

```text
HDR scene pass
  → TransitionColorToShaderRead
  → (optional) SSAO composite → scratch RGBA16F
  → RecordTonemap → swapchain
  → screen UI (post-tonemap)
  → ImGui
```

When SSAO is active, tonemap samples **SSAO-composited** scratch instead of raw scene HDR. UI draws **after** tonemap so widgets stay in display space and ignore exposure volumes (Chapter 4).


### Fullscreen pass mechanics

Tonemap avoids vertex buffers by generating clip-space positions from vertex index:

```glsl
// tonemap.vert — UV from [-1,1] quad positions
vUv = inPos * 0.5 + 0.5;
gl_Position = vec4(inPos, 0.0, 1.0);
```

Descriptor set binds `hdrColor` at set 0 binding 0. Push constants carry exposure and invGamma — no UBO churn per frame. RenderDoc should show **exactly one draw** in the tonemap pass for a given swapchain image.


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


### ACES vs Reinhard hue

Reinhard `x/(1+x)` desaturates bright yellows and blues differently — skies look chalky. ACES fitted curve (Hill/Naughty Dog) preserves hue through the shoulder region — preferred default for outdoor PBR until LUT grading ships.

Plot both curves for R=G=B inputs 0–16 in a spreadsheet to build intuition for exposure slider range.


### Exposure and physical lights

Exposure scales **tonemap input**, not sun `lightColor` in scene pass. Artists crank exposure in dark interiors without rebalancing directional intensity — decouples lighting authorship from display grading. Post volumes (Chapter 4) write `params.exposure` consumed here.


### Gamma and swapchain format

If swapchain is **sRGB format**, some engines skip shader gamma — Spark keeps `invGamma` push for UNORM targets and artist tuning. Document which path your engine uses; double gamma is a common HDR migration bug.


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

### Dedicated tonemap render pass

**Concept:** Tonemap pipeline shape

**Decision:** Dedicated tonemap render pass

**Why:** Separate pass targeting swapchain — depth test off, blend off, full-screen triangle. Isolates display encoding from scene lighting code.

**In Spark:** `hdrTonemapPass.RecordTonemap` writes present framebuffer.

### resolvedLighting.exposure from CPU

**Concept:** Exposure multiplier

**Decision:** resolvedLighting.exposure from CPU

**Why:** Global lighting resolve and post volumes feed one float per frame — no gameplay code touches Vulkan for exposure tweaks.

**In Spark:** `RecordTonemap(..., resolvedLighting.exposure)` at line 362.

### acesTonemap in tonemap.frag

**Concept:** ACES fitted curve

**Decision:** acesTonemap in tonemap.frag

**Why:** Single function applied per RGB channel after exposure multiply. Industry-standard starting point before custom LUTs.

**In Spark:** `shaders/tonemap.frag` lines 16–23.

### SSAO may feed tonemap input

**Concept:** Post-scene ordering

**Decision:** SSAO may feed tonemap input

**Why:** When SSAO active, `UpdateTonemapDescriptor` points at SSAO output view; otherwise raw HDR color view.

**In Spark:** `VulkanRenderer.cpp` lines 334–353 SSAO branch.

### invGamma push constant

**Concept:** Gamma encode

**Decision:** invGamma push constant

**Why:** Artist-tunable gamma without recompile — typical default near 2.2 for UNORM swapchains.

**In Spark:** `TonemapPush` block in `tonemap.frag` lines 7–12.

### Procedural tonemap.vert

**Concept:** Fullscreen pass mechanics

**Decision:** Procedural tonemap.vert

**Why:** No vertex buffer allocation — three-vertex draw indexed by `gl_VertexIndex` pattern via explicit buffer.

**In Spark:** `shaders/tonemap.vert` — UV from position.

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

- `shaders/tonemap.frag`
- `shaders/tonemap.vert`
- `src/spark/render/post/VulkanHdrTonemapPass.cpp`
- `src/spark/render/core/VulkanRenderer.cpp`
- `src/spark/render/lighting/SceneLightingProfile.cpp`

### Step 1 — Read tonemap.frag

Follow the chain: sample HDR → multiply exposure → ACES → gamma pow → write swapchain.

*ACES tonemap and gamma encode*

```glsl
// shaders/tonemap.frag (lines 16–30)
vec3 acesTonemap(vec3 x) {
    const float a = 2.51;
    const float b = 0.03;
    const float c = 2.43;
    const float d = 0.59;
    const float e = 0.14;
    return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}

void main() {
    vec3 hdr = texture(hdrColor, vUv).rgb * max(push.exposure, 1e-4);
    vec3 ldr = acesTonemap(hdr);
    ldr = pow(ldr, vec3(max(push.invGamma, 1e-3)));
    outColor = vec4(ldr, 1.0);
}
```

### Step 2 — Read tonemap.vert

Fullscreen pass generates UVs from clip-space corner positions — no mesh asset required.

*Procedural fullscreen UVs*

```glsl
// shaders/tonemap.vert (lines 1–10)
#version 450

layout(location = 0) in vec2 inPos;

layout(location = 0) out vec2 vUv;

void main() {
    vUv = inPos * 0.5 + 0.5;
    gl_Position = vec4(inPos, 0.0, 1.0);
}
```

### Step 3 — Trace RecordTonemap call site

Tonemap records after optional SSAO, before screen UI and ImGui. Note the present render pass handle comes from `presentRenderPass`, not HDR.

*SSAO branch and tonemap record with exposure*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 334–362)
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

    hdrTonemapPass.RecordTonemap(
            commandBuffer,
            imageIndex,
            frameIndex,
            presentRenderPass.vkPass,
            presentSwapchain().extent,
            presentationFramebuffers,
            resolvedLighting.exposure);
```

### Step 4 — Build tonemap pipeline

Create pipeline against **present render pass** with no depth attachment. Fragment shader samples `hdrColor`; push constants sized to `TonemapPush`. Add to swapchain recreate path when present pass handle changes.

### Step 5 — Wire exposure source

`resolvedLighting.exposure` passes into `RecordTonemap` — connect to post volumes in Chapter 4. Log exposure during camera movement through volume boundaries.

### Step 6 — Tune outdoor scene

Walk **ThreeDDemo** or **TerrainDemo** — adjust exposure until sky and ground retain detail without gray haze. Screenshot before/after ACES at exposure 0.5 vs 1.5.

### Step 7 — Compare tonemap curves

Graph ACES vs Reinhard `x/(1+x)` for inputs 0–10 in a spreadsheet. RenderDoc: tonemap pass should be exactly **one draw call** per frame.

### Step 8 — Verify UI ordering

Confirm screen UI and ImGui record **after** tonemap — UI colors should not be exposure-scaled. If UI looks washed out, check whether it accidentally targets HDR pass.

## 5. How This Connects to the Rest of Your Engine

Tonemap sits after optional SSAO (Chapter 3). Part **XI** screen UI draws **after** tonemap. Part **VIII** particle HDR values finally become visible on monitor here. Part **IX** shadow contrast should be tuned in HDR **before** tonemap hides acne.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Double sRGB** — decode albedo once, encode display once.
- **Tonemapping already-linear albedo without decode** — colors look dull or overly contrasty.
- **Wrong UV Y flip** — upside-down tonemap; match Vulkan viewport convention.
- **Tonemap before SSAO** — ambient darkening applied in wrong space.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Add exposure slider; find three settings for indoor/outdoor/sunset moods.
2. RenderDoc: tonemap pass should be exactly one draw call.
3. Compare ACES vs Reinhard screenshot side by side on emissive geometry.
4. Plot ACES output for inputs 0–16; identify shoulder rolloff region.
5. Verify UI draws after tonemap and is not exposure-scaled.

## Summary

ACES tonemapping is a fullscreen pass: sample HDR, multiply exposure, apply filmic curve, encode for display. Spark records it after SSAO and before UI, passing exposure from resolved lighting each frame.

---

Next: [SSAO](03-ssao.md).
