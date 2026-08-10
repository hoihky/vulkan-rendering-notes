# Bias and Artifacts

> **Build-Along Milestone:** Flat floor shows no acne and character feet show no Peter panning at default camera angles.

**You will learn:**

- Diagnose shadow acne, Peter panning, and cascade seams.
- Tune constant, slope-scaled, and per-cascade bias parameters.
- Apply practical content-side fixes when math limits are reached.

## 1. The Problem We Are Solving

Shadow maps discretize continuous geometry into texels. **Acne** and **Peter panning** come from depth precision and texel quantization. **Seams** appear at cascade boundaries when splits or biases disagree.

Tuning is half math, half art — you need a systematic debug workflow with ImGui sliders and false-color modes.

## 2. Core Concepts

### Shadow acne

Surface points sample a texel whose stored depth is slightly closer due to surface tilt → false self-shadowing. Fix with **receiver bias** before compare.


### Peter panning

Bias too large → compare always passes → shadow **detaches** from contact. Reduce bias or use normal-offset bias.


### Cascade seams

Adjacent cascades use different texel grids → intensity jump at splits. Overlap splits slightly or blend cascades (advanced).


### Pipeline vs shader bias

Vulkan pipeline depth bias (`depthBiasConstantFactor`, `depthBiasSlopeFactor`) complements shader-side bias — do not double both aggressively.


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


### Acne vs Peter panning tradeoff

**Acne**: receiver `refZ` slightly greater than true surface depth vs map texel → false shadow. Fix: increase bias. **Peter panning**: bias so large compare always passes → shadow detaches from contact. Tune on flat floor (acne) and character feet (panning) simultaneously.

Spark directional bias:
```glsl
depthBias = max(shadowParams.x, 1e-5) * (1.0 + 6.25 * gracing)
depthBias += shadowParams.y * 0.22 * gracing
```
where `gracing = 1 - dot(N, L)`.


### Pipeline vs shader bias

Vulkan `depthBiasConstantFactor` / `depthBiasSlopeFactor` on shadow **pass** pipeline shift caster depth during rasterization. Receiver bias in scene shader shifts compare reference. Using **large values on both** over-separates shadows — pick one primary knob per project and document it.


### Cascade seam tuning

Adjacent cascades use different texel grids — intensity jumps at splits. Mitigations: split overlap, `blendSunShadowCascades`, per-cascade bias scale, or increase near cascade resolution before extreme bias values. TerrainDemo exposes seams quickly — tune outdoors, not only on test cubes.


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


### ImGui tuning workflow

Expose `shadowParams.x` (constant), `.y` (slope), `.z` (texel size), `.w` (enable) to ImGui during development. Tune on **TerrainDemo** flat floor and **ThreeDDemo** character feet simultaneously — save values per profile (outdoor/interior) in config JSON for reproducible QA.


### 16-bit vs 32-bit shadow depth

D16_UNORM shadow maps band depth steps — bias tuning shows stairstepping on flat surfaces. Spark uses D32 for directional and punctual. If porting to memory-constrained targets, expect longer bias tuning sessions and more PCF noise.

Compare D16 vs D32 captures on the same floor normal — document quality tradeoff for your platform matrix.


### Content-side fixes

When math limits hit: pull contact geometry up 1–2 mm, increase near cascade resolution, add slight split overlap via blend fraction, use contact hard shadows only on hero assets. Bias alone cannot fix 0.5 m texel footprint on distant cascade — resolution or cascade count must change.


### Directional vs punctual bias tables

Maintain separate tuned constants:

| Path | gracing multiplier | notes |
|------|-------------------|-------|
| Directional 5×5 | 6.25 × + 0.22 slope | scene.frag |
| Spot 3×3 | 4.0 × + 0.18 slope | punctual_shadows.glsl |
| Point 3×3 | 3.5 × + 0.15 slope | cube face perspective |

Copying directional values to spot lights causes Peter panning on flashlight cones — tune each path on representative geometry.


### Cascade-specific bias scale

Optional multiplier after cascade index selection: far cascades cover more world units per texel — increase bias slightly for cascade 2–3 to suppress acne on distant terrain without over-biasing near cascade where contact detail matters. Implement as `bias *= 1.0 + 0.15 * float(cascade)` starting only after near cascade validated.


### HDR tuning before tonemap

Validate bias on **HDR linear floor** before Part X tonemap — low contrast tonemap hides acne that returns when exposure changes. Capture RenderDoc HDR buffer on flat floor: sun shadow terminator should show smooth gradient without speckled noise before ACES crushes dynamic range.

Document final `shadowParams` in engine config per lighting profile JSON alongside exposure defaults.


### Screenshot documentation pack

Ship three labeled screenshots with final bias: (1) zero bias acne on floor, (2) tuned neutral, (3) over-biased Peter panning on feet. Attach `shadowParams` vec4 values. Future engineers revert bad tuning faster than re-deriving from scratch when art reports regression on new terrain content.


### Normal offset bias alternative

Some engines bias along **surface normal** in light space instead of depth slope — `posOffset = N * normalOffsetScale`. Spark uses depth bias in compare (`refZ - depthBias`) with gracing from `N·L`. If Peter panning persists on flat surfaces with high slope bias, try reducing slope term and adding small normal offset in world space before light transform — document hybrid approach if adopted.


### CSM split overlap tuning

Increase `cascadeBlendFraction` in lighting profile before maxing bias — overlap hides resolution discontinuities that bias alone cannot fix. Log split distances and blend width in meters when walking forward on TerrainDemo — aim for 0.5–2 m blend band depending on scene scale.


### Validation layer sync

After aggressive bias experiments, watch validation for depth compare out of range — refZ clamped to [0,1] hides bugs until border pixels flicker. Temporarily visualize refZ and mapZ difference as heatmap in scene debug mode.


### Worked example

### Systematic bias tuning session (30 minutes)

| Minute | Action |
|--------|--------|
| 0–5 | Zero bias screenshot — mark acne regions |
| 5–15 | Increase constant bias until floor clears |
| 15–22 | Add slope term; fix grazing stair edges |
| 22–27 | Walk TerrainDemo splits — note seam pixels |
| 27–30 | Record final vec4 in engine.ini |

Repeat separately for spot flashlight — do not reuse directional numbers. Document environment: indoor vs outdoor profile names.


### Profile-specific shadowParams

Store outdoor, interior, and cinematic profiles in JSON — swap at runtime when loading lighting profile from SceneLightingProfile. TerrainDemo and ThreeDDemo may use different defaults; do not copy bias vec4 between demos without visual verify.


### Summary checkpoint

Milestone: flat floor shows no acne and character feet show no Peter panning at default ThreeDDemo camera — document final shadowParams vec4 in engine config.


### Engine porting note

Wire ImGui sliders to ubo.shadowParams during development, then bake tuned values into lighting profile presets for shipping builds — avoid shipping debug UI dependency while keeping artist-tuned constants in data files.


### Closing note

Add before/after screenshot pair to engine wiki when closing shadow tuning ticket — future regressions compare against known-good imagery.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Slope-scaled shader bias

**Concept:** Shadow acne

**Decision:** Slope-scaled shader bias

**Why:** Scale bias by `1 - dot(N,L)` so grazing surfaces get more offset without over-biasing flat floors.

**In Spark:** `gracing` term in `sampleSpotShadowAtSlot` and directional helpers.

### ImGui-tunable shadowParams

**Concept:** Peter panning

**Decision:** ImGui-tunable shadowParams

**Why:** Expose constant and slope in UBO for live tuning during development.

**In Spark:** `ubo.shadowParams` in scene and punctual includes.

### Per-cascade bias scale

**Concept:** Cascade seams

**Decision:** Per-cascade bias scale

**Why:** Far cascades cover more world space per texel — multiply bias by cascade index blend.

**In Spark:** Optional multiplier in scene.frag after cascade selection.

### Pick one primary knob

**Concept:** Pipeline vs shader bias

**Decision:** Pick one primary knob

**Why:** Spark leans on shader bias for receivers; shadow pass pipeline bias is optional for casters.

**In Spark:** Document which path your engine uses and stick to it.

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
- `include/spark/engine/SceneRenderParams.hpp`

### Step 1 — Baseline with zero bias

Set bias to 0; observe acne on floor under sun at grazing angles. Screenshot for before/after comparison.

### Step 2 — Read Spark spot bias formula

Note constant minimum, gracing multiplier, and secondary slope term.

*Slope-scaled punctual bias*

```glsl
// shaders/punctual_shadows.glsl (lines 66–72)
    float gracing = 1.0 - clamp(dot(N, L), 0.0, 1.0);
    float depthBias = max(ubo.shadowParams.x, 1e-5) * (1.0 + 4.0 * gracing);
    depthBias += ubo.shadowParams.y * 0.18 * gracing;
    float refZ = clamp(proj.z - depthBias, 0.0, 1.0);
    vec2 texel = vec2(1.0 / 512.0);
    float rot01 = shadowPcfRotation01(gl_FragCoord.xy);
    return samplePunctualShadowPcf3(spotShadowMap, uv, refZ, texel, rot01);
```

### Step 3 — Add constant bias incrementally

Increase in small steps until acne clears on test floor; watch for Peter panning on character feet.

### Step 4 — Add slope term for directional

Apply the same gracing concept to directional PCF in `scene.frag`.

### Step 5 — Cascade-specific tuning

Far cascades may need larger bias — scale after cascade index selection.

### Step 6 — Content fixes

Increase near cascade resolution before chasing extreme bias. Pull floor mesh up 1–2 mm in authoring for edge cases. Tune in **HDR linear space** before tonemap hides contrast issues.

### Step 7 — Document final values

Record per-project `shadowParams` for directional, spot, and point paths — they differ due to perspective foreshortening.

### Step 9 — Read directional bias in scene.frag

Slope-scaled bias before 5×5 PCF — primary directional tuning knob.

*Directional depth bias and PCF call*

```glsl
// shaders/scene.frag (lines 156–164)
    float g = sunGracing;
    float depthBias = max(ubo.shadowParams.x, 1e-5) * (1.0 + 6.25 * g);
    depthBias += ubo.shadowParams.y * 0.22 * g;
    float refZ = clamp(proj.z - depthBias, 0.0, 1.0);
    vec2 texel = vec2(ubo.shadowParams.z);
    float penumbraScale = clamp(1.0 + 2.75 * g, 1.0, 2.85);
    float rot01 = shadowPcfRotation01(gl_FragCoord.xy);
    float sh = sampleSunShadowPcf(shadowMap, uv, refZ, texel, penumbraScale, rot01);
    return pow(clamp(sh, 0.0, 1.0), 1.08);
```

## 5. How This Connects to the Rest of Your Engine

Bias interacts with **PCF kernel size** (Chapter 4) and **CSM splits** (Chapter 2). Part **X** tonemap can hide contrast issues — tune shadows in HDR linear space first.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Different bias for CSM vs spot** — keep separate params per light type.
- **Winding order + bias** — fix cull before maxing bias.
- **16-bit shadow formats** — banding; prefer D32 for tuning.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Screenshot acne, tuned, and over-biased (Peter panning) side by side.
2. Find cascade seam in TerrainDemo; try 5% split overlap.
3. Plot bias vs dot(N,L) for your slope formula in a spreadsheet.
4. Tune `shadowParams.x` / `.y` live in TimeOfDayDemo if available.
5. Document pipeline depth bias vs shader bias — which Spark uses for directional.

## Summary

Shadow artifacts come from depth precision and cascade discretization. Combine constant and slope bias, optional pipeline depth bias, and per-cascade tuning — validate in HDR before tonemap.

---

Continue to [Part X — HDR and Post](../part-10-hdr-post/index.md).
