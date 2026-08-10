# PCF and Cascades

> **Build-Along Milestone:** Directional sun term visibly softens at shadow boundaries with 5×5 PCF enabled.

**You will learn:**

- Implement percentage-closer filtering with a weighted kernel in the scene fragment shader.
- Select the correct cascade from **camera distance** before sampling.
- Combine PCF weights with directional light attenuation in forward shading.

## 1. The Problem We Are Solving

Hard shadow compares produce **stairstepped** edges. With CSM, PCF must sample within the **correct atlas sub-rect** and use cascade-appropriate bias, or you get seams and acne at split boundaries.

You will implement PCF in the lit fragment shader using the same depth convention as the shadow pass.

## 2. Core Concepts

### Hard compare vs PCF

A single depth test produces stairstepped shadow edges. **Percentage-closer filtering (PCF)** averages multiple depth tests with offset UVs, approximating soft penumbra cheaply.


### 3×3 PCF kernel

```glsl
for (y = -1..1)
  for (x = -1..1)
    sum += (refDepth - bias <= sampleDepth) ? 1 : 0
return sum / 9
```

Scale texel size by cascade atlas scale when sampling sub-rectangles.


### Cascade selection

Spark uses **world-space distance** from the camera to `vWorldPos` compared against `ubo.cascadeSplits` — not view-space Z. Distance thresholds stay intuitive when the camera tilts without moving forward.

```glsl
int selectShadowCascade(float distFromCamera) {
    if (distFromCamera <= ubo.cascadeSplits.x) return 0;
    if (distFromCamera <= ubo.cascadeSplits.y) return 1;
    if (distFromCamera <= ubo.cascadeSplits.z) return 2;
    return 3;
}
```


### Receiver bias

Combine constant bias with slope-scaled term proportional to surface tilt relative to the light:

```glsl
bias = constant + slopeScale * (1.0 - dot(N, L))
```

Too much bias causes Peter panning; too little causes acne.


### Build-along verification checklist

After each milestone, verify four contracts before adding features:

| Check | Tool | Pass criteria |
|-------|------|---------------|
| Data layout | RenderDoc | Push/UBO bytes match shader struct |
| Pass order | RenderDoc event list | Dependencies before consumers |
| Format | Texture viewer | D32 / RGBA16F as documented |
| Visual | Demo run | No swimming, acne, or black pass |

When output disagrees with Spark, diff **data paths** (struct fields, matrix multiply order, binding indices) — not just Vulkan object names. Validation VUID strings link to spec language for the exact contract violated.


### Matrix and space conventions

Spark uses **column-major** `mat4` in GLSL matching C++ `Matrix4` storage. World-to-shadow transforms apply as:

```text
clip = worldToShadowClip * vec4(P_world, 1)
```

Normal vectors for bias use world-space **N** dotted with light direction **L** — not tangent space. Cascade selection in Spark uses **camera distance** from `length(vWorldPos - cameraPos)`, not clip-space Z, so split thresholds stay intuitive when the camera tilts.

Document your engine's handedness (Y-up, Z-forward) beside Spark's when porting cascade math — a transposed multiply shows up as shadows sliding opposite camera motion.


### Directional 5×5 PCF kernel

Spark's sun shadow uses a **Gaussian-weighted 5×5 PCF** with per-pixel rotation — not a naive 3×3 axis-aligned grid:

```glsl
float kShadowPcfW5[5] = float[](0.0625, 0.25, 0.375, 0.25, 0.0625);
```

`penumbraScale` widens the kernel when `N·L` is low (grazing surfaces) for softer terminators. `shadowPcfRotation01` hashes fragment coordinates to rotate offset vectors — reduces square grid banding from the shadow map raster grid.


### Manual compare for portability

`layout(set=0, binding=3) uniform sampler2D shadowMap` — **not** `sampler2DShadow`. Spark compares depths manually (`refZ <= mapZ`) for MoltenVK and `VK_KHR_portability_subset` compatibility. Hardware compare samplers behave differently across drivers; manual compare matches across Spark targets.


### Atlas-aware UV mapping

`cascadeAtlasUv` applies scale/bias from `ubo.cascadeAtlas[cascade]` after NDC→UV mapping. PCF margin clamp uses texel size from `ubo.shadowParams.z` — scale when cascades use different atlas tile resolutions. PCF kernels straddling atlas gutters sample neighboring cascade texels — add padding or clamp margins (Chapter 8).


### RenderDoc shadow debugging workflow

Capture a frame with **ThreeDDemo** shadows enabled. In the event browser, expand the directional shadow pass and confirm indexed draws for casters. Select a shadow draw → verify push constant bytes for `lightViewProj` match the matrix uploaded to UBO for the same cascade index.

Switch to the HDR scene pass → select an opaque draw → fragment shader inputs should show `worldToShadowClip` arrays in the UBO. Texture viewer on binding 3 should show depth atlas tiles — not cleared white.

Common false leads: debugging PCF when casters never drew (empty shadow pass); tuning bias when projection is inverted (whole shadow map slides). Always confirm **pass order** and **non-empty shadow pass** before shader math.


### Memory and bandwidth budget

Directional CSM atlas at 2048² D32 ≈ 16 MB per flight target. Four spot 512² tiles plus two point cube faces add punctual memory on top. Budget:

```text
bytes ≈ width * height * 4 (D32) * cascades_or_faces * framesInFlight
```

PCF sample cost scales with scene resolution × kernel taps × cascade blend bands. Spark's 5×5 directional PCF with cascade blend is ~25–50 extra depth fetches per pixel in worst case — profile before enabling on mobile targets.

Reduce cost: lower atlas resolution on far cascades only (requires split atlas sizes), fewer punctual shadow slots, or smaller PCF kernel on distant LOD materials.


### Sun shadow integration in main()

Directional shadow multiplies the sun BRDF term after `blendSunShadowCascades` returns visibility ∈ [0,1]. Punctual lights call `sampleSpotShadowAtSlot` / point variant inside clustered loops — separate atlas and bias paths. `ubo.shadowParams.w` master-enables directional shadows; push `shadowFlags` bit 2 gates receive per draw.


### selectShadowCascade distance metric

Spark selects cascade by **Euclidean distance** from camera to `vWorldPos` compared against `ubo.cascadeSplits` — not view-space Z. Distance thresholds correlate with artist intuition when walking forward through splits; view-Z splits can shift when looking up/down without moving forward.

False-color debug: output `float(selectShadowCascade(dist)) / 3.0` as RGB in scene pass temporarily — verify index increments at split distances logged from CPU.


### Punctual vs directional PCF

Directional sun uses 5×5 Gaussian-weighted kernel in `scene.frag`; punctual uses 3×3 rotated kernel in `punctual_shadows.glsl`. Bias formulas differ — `gracing` multipliers 6.25 vs 4.0 for spot. Do not copy directional bias constants to punctual paths without retuning on flashlight test scene.


### blendSunShadowCascades detail

Near split boundaries, Spark lerps between cascade *i* and *i±1* shadow results using `smoothstep` over a band proportional to split distance × `blendFrac`. Minimum blend length `kMinBlendM = 0.35` meters prevents zero-width blend when splits are close. Cost: up to three full PCF evaluations per pixel in blend zones — watch GPU cost on 4K with 5×5 kernels.

Disable blend temporarily when debugging raw cascade seams — re-enable after splits and biases stabilize.


### shadowFadeEnd distance fade

`ubo.viewportSize.z` stores shadow fade end distance — beyond this, sun shadow visibility lerps to 1.0 (fully lit) to hide low-resolution far-cascade artifacts on distant hills. Paired with `timeGlobal.y` fade start fraction — outdoor profiles tune horizon shadow softness without disabling CSM entirely.


### Kernel rotation hash

`shadowPcfRotation01` uses sin/dot fract hash on `gl_FragCoord.xy` — stable per pixel per frame, rotates PCF offset kernel to break axis-aligned shadow map grid alignment. Without rotation, 5×5 grids produce visible square patterns on flat surfaces under low sun angles.

Use same hash for punctual PCF (`punctual_shadows.glsl`) — consistent artifact reduction across light types.


### Hard compare baseline

Before enabling PCF, implement single-tap compare — stairstepped shadow edges confirm UV and depth convention are correct. Then enable 5×5 kernel — softening appears without moving shadow position. If hard compare position is wrong, PCF only blurs an incorrect result — always validate hard compare first on fence mesh or stair step geometry with known light angle.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### 3×3 grid PCF default

**Concept:** Hard compare vs PCF

**Decision:** 3×3 grid PCF default

**Why:** Start with a regular kernel for predictability; upgrade to rotated Poisson for punctual lights later.

**In Spark:** Directional PCF in scene fragment; punctual uses `samplePunctualShadowPcf3` in `punctual_shadows.glsl`.

### View-space depth thresholds

**Concept:** Cascade selection

**Decision:** View-space depth thresholds

**Why:** Compare abs view Z against UBO split distances — stable across camera orientations when splits are stored in view space.

**In Spark:** `worldToShadowClip[cascade]` selected in scene.frag from UBO splits.

### UBO shadowParams

**Concept:** Receiver bias

**Decision:** UBO shadowParams

**Why:** Expose constant and slope terms in the scene UBO so ImGui sliders can tune without recompile.

**In Spark:** `ubo.shadowParams` used in directional and punctual shadow helpers.

### Shared GLSL include

**Concept:** 3×3 PCF kernel

**Decision:** Shared GLSL include

**Why:** Keep PCF helpers in an include file consumed by `scene.frag` — avoids duplicating bias math.

**In Spark:** `shaders/punctual_shadows.glsl` for spot/point; directional helpers in scene shading includes.

### RenderDoc-first debugging

**Concept:** Build-along verification checklist

**Decision:** RenderDoc-first debugging

**Why:** Capture one known-good frame from ThreeDDemo before changing splits, bias, or HDR order — regression compare pass list and bound descriptors each milestone.

**In Spark:** Demo README capture instructions; compare against Spark reference captures.

### Trace comment in cascade math

**Concept:** Matrix and space conventions

**Decision:** Trace comment in cascade math

**Why:** Spark documents CPU→GPU shadow path in source comments — update your port's equivalent when conventions change.

**In Spark:** `VulkanDirectionalShadowCascadeMath.cpp` lines 13–20 pipeline trace.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `shaders/punctual_shadows.glsl`
- `shaders/scene.frag`
- `src/spark/render/shadow/VulkanDirectionalShadowCascadeMath.cpp`

### Step 1 — Read punctual PCF helper

Spark rotates the PCF kernel per pixel to reduce banding. Study the margin clamp that prevents atlas edge bleed.

*Rotated 3×3 PCF*

```glsl
// shaders/punctual_shadows.glsl (lines 30–49)
float samplePunctualShadowPcf3(sampler2D map, vec2 uvCenter, float refZ, vec2 texel, float rot01) {
    float c = cos(rot01 * 6.28318530718);
    float s = sin(rot01 * 6.28318530718);
    mat2 rot = mat2(c, -s, s, c);
    float acc = 0.0;
    float wsum = 0.0;
    vec2 margin = texel * 2.0;
    vec2 uvC = clamp(uvCenter, margin, vec2(1.0) - margin);
    for (int j = -1; j <= 1; ++j) {
        for (int i = -1; i <= 1; ++i) {
            float w = 1.0;
            vec2 off = rot * (vec2(float(i), float(j)) * texel);
            vec2 suv = clamp(uvC + off, vec2(0.0), vec2(1.0));
            float mapZ = textureLod(map, suv, 0.0).r;
            acc += w * (refZ <= mapZ ? 1.0 : 0.0);
            wsum += w;
        }
    }
    return acc / max(wsum, 1e-5);
}
```

### Step 2 — Follow spot shadow UV build

Perspective divide, atlas scale/bias, viewport Y flip, and slope-scaled bias happen before the PCF loop.

*Spot shadow sample*

```glsl
// shaders/punctual_shadows.glsl (lines 51–73)
float sampleSpotShadowAtSlot(int slot, vec3 worldPos, vec3 N, vec3 L) {
    vec4 ls = punctualShadow.spotWorldToClip[slot] * vec4(worldPos, 1.0);
    if (ls.w <= 1e-5) {
        return 1.0;
    }
    vec3 proj = ls.xyz / ls.w;
    if (abs(proj.x) > 1.0 || abs(proj.y) > 1.0 || proj.z < 0.0 || proj.z > 1.0) {
        return 1.0;
    }
    vec2 uv = vec2(proj.x * 0.5 + 0.5, proj.y * 0.5 + 0.5);
    vec4 atlas = punctualShadow.spotAtlas[slot];
    uv = uv * atlas.zw + atlas.xy;
    if (ubo.viewportSize.w > 0.5) {
        uv.y = 1.0 - uv.y;
    }
    float gracing = 1.0 - clamp(dot(N, L), 0.0, 1.0);
    float depthBias = max(ubo.shadowParams.x, 1e-5) * (1.0 + 4.0 * gracing);
    depthBias += ubo.shadowParams.y * 0.18 * gracing;
    float refZ = clamp(proj.z - depthBias, 0.0, 1.0);
    vec2 texel = vec2(1.0 / 512.0);
    float rot01 = shadowPcfRotation01(gl_FragCoord.xy);
    return samplePunctualShadowPcf3(spotShadowMap, uv, refZ, texel, rot01);
}
```

### Step 3 — Implement directional PCF in scene.frag

After selecting cascade index `c`:

1. Transform world position with `worldToShadowClip[c]`.
2. Early-out if outside the light frustum.
3. Map NDC XY to atlas UV with `cascadeAtlas[c]`.
4. Run PCF and multiply the sun diffuse term.

### Step 4 — Bind shadow sampler

Directional shadow map binds at a fixed descriptor slot — verify layout matches `scene.frag` expectations in RenderDoc's pipeline view.

### Step 5 — Compare 1-tap vs 9-tap

Screenshot a fence mesh or stair edge with hard compare, then enable PCF. The softening should be visible without changing sun intensity.

### Step 6 — Tune bias with ImGui

Hook `shadowParams` to sliders. Find values that remove acne on a flat floor without detaching shadows from character feet (Chapter 8).

### Step 9 — Study blendSunShadowCascades

Cascade blend reduces seam visibility — read full function before disabling for debug.

*Cascade shadow blend start*

```glsl
// shaders/scene.frag (lines 167–179)
/** Soft blend between adjacent CSM cascades near split distances (hides resolution pops). */
float blendSunShadowCascades(float distCam, vec3 worldPos, vec3 N, vec3 Ld, float sunGracing) {
    int cascade = selectShadowCascade(distCam);
    float sh = sampleSunShadowAtCascade(cascade, worldPos, N, Ld, sunGracing);

    float blendFrac = clamp(ubo.timeGlobal.w, 0.001, 0.35);
    const float kMinBlendM = 0.35;

    if (cascade > 0) {
        float splitLow = ubo.cascadeSplits[cascade - 1];
        float blendLen = max(splitLow * blendFrac, kMinBlendM);
        float wPrev = 1.0 - smoothstep(splitLow - blendLen, splitLow, distCam);
        if (wPrev > 1e-4) {
```

## 5. How This Connects to the Rest of Your Engine

PCF runs in the **HDR scene pass** before Part **X** tonemap. Part **VIII** particles typically skip shadow receive. Part **XI** sprites may use simplified or no shadow receive — keep paths separate.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **PCF across atlas gutter** — bleed from neighboring cascade; add padding in atlas.
- **Forgot receiver bias** — acne on flat surfaces facing the light.
- **Too large kernel on low-res maps** — moire patterns; tie kernel to cascade resolution.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Compare 1-tap vs 5×5 PCF screenshots on a fence mesh.
2. Log cascade index at camera position vs 100 m away.
3. Add slope bias slider and document final values for your test scene.
4. Disable cascade blend — find seam in TerrainDemo.
5. Visualize `shadowPcfRotation01` as false color on flat floor.

## Summary

PCF softens shadow edges by averaging depth compares; CSM adds cascade selection and atlas-aware UVs. Spark separates directional and punctual PCF helpers in GLSL includes.

---

Next: [Punctual Shadows](05-punctual-shadows.md).
