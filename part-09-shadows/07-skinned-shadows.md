# Skinned Mesh Shadows

> **Build-Along Milestone:** Animated character shadow silhouette tracks limb motion in RenderDoc.

**You will learn:**

- Reuse the bone palette SSBO in the shadow vertex shader.
- Keep joint count and skinning path identical between lit and shadow draws.
- Verify skinned casters match lit pass silhouettes in the depth map.

## 1. The Problem We Are Solving

Animated characters share **skinned vertices** between lit and shadow passes. If the shadow vertex shader treats them as rigid, you get **floating shadows** that do not follow limbs.

The fix is the **same skinning math** — same SSBO, same joint indices/weights — in `shadow_depth.vert`.

## 2. Core Concepts

### Skinning recap

For each vertex, blend bone matrices by joint indices and weights, then apply the object `model` matrix:

```glsl
skinMat = Σ weight[k] * bones[joint[k]]
worldPos = model * skinMat * vec4(position, 1)
gl_Position = lightViewProj * worldPos
```


### SSBO binding parity

Lit and shadow passes must bind the **same** bone palette SSBO at the **same** descriptor binding for each skinned draw.


### Push constant skinning flag

`useSkinning` and `jointCount` in push constants select the skinning path without a separate pipeline per skinned mesh.


### One upload per draw

Bone data uploads once per skinned draw per frame; both lit and shadow passes read identical GPU memory for that frame slot.


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


### Skinning parity checklist

| Resource | Lit pass | Shadow pass |
|----------|----------|-------------|
| Bone SSBO binding | set 0 binding 2 | set 0 binding 2 |
| Joint/weight locations | 4, 5 | 4, 5 |
| useSkinning push field | yes | yes |
| jointCount clamp | kMaxSkinJoints | same |

Break any row → floating or collapsed shadow silhouettes for animated meshes.


### Single bone upload

Bone palette uploads once per skinned draw per frame. Shadow and lit passes read **identical** GPU memory for that frame slot — do not upload different palettes per pass unless intentionally debugging.


### Normals not required

Shadow vertex shader needs **position only** after skinning. Do not require tangent/normal attributes in shadow pipeline layout — keeps vertex shader minimal and avoids linking failures when shadow layout differs from lit.


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


### Joint index packing

Spark packs joint indices in `vec4` floats for MoltenVK compatibility — shadow and lit shaders unpack with `floatBitsToUint`. Shadow path must use **identical** unpack logic; copying only rigid shadow shader to skinned meshes collapses limbs to origin.


### Animation timing vs shadow record

Skin matrices must be finalized in `OnUpdate` / animation solve **before** submit builds joint palette. Shadow and lit passes read the same palette for frame index — if animation updates mid-record, lit and shadow can disagree for one frame causing shimmering silhouettes.

Verify: pause game logic, step one frame, compare shadow depth silhouette overlay with lit pass wireframe in RenderDoc texture overlay tools.


### Max joints enforcement

`kMaxSkinJoints` caps palette size on CPU upload and in shader `jointCount` push field. Vertices referencing joints above cap are skipped in skinning loop — shadow and lit must share the same cap and skip logic or limbs diverge.


### scene.vert vs shadow_depth.vert parity

Diff the skinning loops side-by-side — joint unpack, weight skip, `jointCount` clamp, matrix multiply order must match byte-for-byte logic. Any `continue` condition difference changes limb positions. Shared include file (`skinning.glsl`) is the long-term fix if loops currently duplicate.

After animation state machine transitions, verify shadow catch-up — one-frame stale palette shows as shadow lagging pose during rapid transitions.


### Descriptor bind order for skinned shadow draw

Typical order: bind pipeline → bind descriptor set (scene) → bind skin SSBO dynamic offset if used → push constants with useSkinning=1 → draw indexed. Lit pass must use identical order — reordering causes rare bugs only on skinned draws.

RenderDoc pipeline state: confirm SSBO binding 2 populated for shadow draw call on skinned character.


### Regression test for skinned shadows

Automated test: spawn skinned character, advance animation 30 frames, capture shadow depth hash or golden image compare. Compare against lit pass depth from light view — hashes should match within tolerance. Manual break test: force `useSkinning=0` in shadow push — feet detach — re-enable confirms fix.

Document in QA: any skinning shader change requires shadow VS parity review.


### Skeletal mesh QA pass

Animation QA: run walk cycle in place, jump, and turn-in-place while watching shadow feet contact. Shadow toes should not slide on ground during walk — sliding indicates skin matrix timing or root motion mismatch between passes, not bias issues. Fix skinning parity before entering bias tuning (Chapter 8).


### Custom mesh pool skinning

Spark custom dynamic meshes participate in shadow pass through same `VulkanShadowRecordContext` custom VB bindings as lit pass. Skinned custom draws must upload bone SSBO slice and set `useSkinning` push — procedural animated meshes follow identical rules as glTF skinned characters.


### Instancing future note

Current Spark path is one draw per skinned mesh with push constants — instanced skinning would require SSBO per-instance bone palettes or texture bone lookup. Shadow pass must match whatever instancing scheme lit pass adopts; diverging breaks parity silently for instance 1..N.


### Shadow pass mesh LOD

If lit pass uses LOD but shadow pass always draws LOD0, shadow silhouette may be wider than lit mesh at distance — rare but visible on impostor transitions. Match LOD selection in shadow record or force shadow-specific LOD bias for distant casters.


### Worked example

### Shared skin SSBO upload timing

`WriteUniformBuffer` and skin palette uploads for each draw happen during scene record preparation — shadow pass reads the same GPU buffer without re-uploading bones. If your engine uploads bones inside lit record only, shadow pass sees stale bind pose — add bone SSBO write before **both** record functions or share a `PrepareSkinnedDraws(frameIndex)` hook called once per frame.

Frame index ring: bone SSBO slice for frame *i* must match descriptor bind during shadow record for the same *i* — identical rule as Part VI animation chapter.


### MoltenVK skinning note

Joint indices packed as floats in vec4 for MoltenVK — shadow and lit shaders must use identical floatBitsToUint unpack. Apple Silicon validation is strict about SSBO dynamic offset alignment — verify skin SSBO offset is multiple of 256 when using dynamic offsets.


### Summary checkpoint

Milestone: animated character shadow silhouette tracks limbs frame-by-frame in RenderDoc depth view — overlay compare with lit pass mesh positions shows sub-pixel alignment.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Shared skinning.glsl include

**Concept:** Skinning recap

**Decision:** Shared skinning.glsl include

**Why:** Extract `Skin()` to an include consumed by both `scene.vert` and `shadow_depth.vert` — one implementation, two pipelines.

**In Spark:** `shadow_depth.vert` lines 31–46 mirror scene skinning loop.

### Binding 2 bone SSBO

**Concept:** SSBO binding parity

**Decision:** Binding 2 bone SSBO

**Why:** Descriptor set layout for shadow pass includes the skin buffer if not using push-only skinning.

**In Spark:** `layout(set=0, binding=2)` in `shadow_depth.vert`.

### ShadowPush useSkinning

**Concept:** Push constant skinning flag

**Decision:** ShadowPush useSkinning

**Why:** Same push layout as lit pass `ModelPushConstants` — verify sizeof matches across C++ and GLSL.

**In Spark:** `ShadowPush` block in `shadow_depth.vert`.

### Shared skinSsboMapped pointer

**Concept:** One upload per draw

**Decision:** Shared skinSsboMapped pointer

**Why:** Shadow record context receives the same mapped SSBO pointer as opaque scene recording.

**In Spark:** `ctx.skinSsboMapped` in `RecordShadowMapPass`.

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

- `shaders/shadow_depth.vert`
- `shaders/scene.vert`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Compare skinning branches

Read rigid vs skinned paths in shadow_depth.vert side-by-side with scene.vert.

*Skinned and rigid paths*

```glsl
// shaders/shadow_depth.vert (lines 31–52)
    if (sp.useSkinning != 0) {
        vec4 spv = vec4(0.0);
        for (int k = 0; k < 4; k++) {
            float w = inWeights[k];
            if (w <= 0.0) {
                continue;
            }
            uint ji = inJoints[k];
            if (int(ji) >= sp.jointCount) {
                continue;
            }
            mat4 BM = skin.bones[ji];
            spv += w * BM * vec4(inPosition, 1.0);
        }
        vec4 wp = sp.model * vec4(spv.xyz, 1.0);
        worldPos = wp.xyz;
    } else {
        vec4 wp = sp.model * vec4(inPosition, 1.0);
        worldPos = wp.xyz;
    }

    gl_Position = sp.lightViewProj * vec4(worldPos, 1.0);
```

### Step 2 — Verify shadow context bindings

Shadow recording passes skin SSBO mapped memory and max joint count — same as opaque pass.

*Shadow context skin bindings*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 628–644)
    const VulkanCustomMeshPool::Bindings customMesh = customMeshPool.GetBindings();
    VulkanShadowRecordContext ctx{};
    ctx.scene = &pendingScene;
    ctx.sceneParamsValid = sceneParamsValid;
    ctx.vertexBuffer = vertexBuffer;
    ctx.indexBuffer = indexBuffer;
    ctx.customVertexBuffer = customMesh.vertexBuffer;
    ctx.customIndexBuffer = customMesh.indexBuffer;
    ctx.cubeIndexCount = cubeIndexCount;
    ctx.planeIndexCount = planeIndexCount;
    ctx.planeFirstIndex = planeFirstIndex;
    ctx.cubeVertexOffset = cubeVertexOffset;
    ctx.planeVertexOffset = planeVertexOffset;
    ctx.customDrawPacked = &customDrawPacked;
    ctx.skinSsboMapped = &sceneDescriptors.SkinSsboMapped();
    ctx.descriptorSet = sceneDescriptors.DescriptorSet(frameIndex);
    ctx.maxSkinJoints = VulkanSceneDescriptors::kMaxSkinJoints;
```

### Step 3 — Record skinned shadow draws

For each skinned draw: bind bone SSBO offset, set `useSkinning = 1`, push constants, draw indexed — mirror lit pass order.

### Step 4 — Validate silhouettes

Animate a character in place. Compare RenderDoc shadow depth silhouette frame-by-frame with lit pass mesh positions — they must align.

### Step 5 — Break test

Disable skinning in shadow VS only — observe detached shadow feet. Re-enable and confirm fix.

### Step 6 — Receivers unchanged

Skinned meshes usually receive shadows too — Chapter 6 flags apply unchanged.

### Step 9 — Compare skinning in shadow VS

Skinning loop must match scene.vert joint/weight handling.

*Skinned path in shadow pass*

```glsl
// shaders/shadow_depth.vert (lines 31–52)
    if (sp.useSkinning != 0) {
        vec4 spv = vec4(0.0);
        for (int k = 0; k < 4; k++) {
            float w = inWeights[k];
            if (w <= 0.0) {
                continue;
            }
            uint ji = inJoints[k];
            if (int(ji) >= sp.jointCount) {
                continue;
            }
            mat4 BM = skin.bones[ji];
            spv += w * BM * vec4(inPosition, 1.0);
        }
        vec4 wp = sp.model * vec4(spv.xyz, 1.0);
        worldPos = wp.xyz;
    } else {
        vec4 wp = sp.model * vec4(inPosition, 1.0);
        worldPos = wp.xyz;
    }

    gl_Position = sp.lightViewProj * vec4(worldPos, 1.0);
```

## 5. How This Connects to the Rest of Your Engine

Built on Part **VI** animation — bone upload must be complete before shadow skinning. **CSM** only changes `lightViewProj` per cascade, not skinning.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Different joint indexing** — shadow and lit diverge.
- **Max bones exceeded in shadow shader** — clamp or split meshes.
- **Normals not needed in shadow VS** — do not require tangent in shadow layout.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Capture skinned character shadow pass — overlay wireframe from lit pass.
2. Disable skinning in shadow VS only — observe detached feet.
3. Count VS invocations: rigid vs skinned shadow draw for same mesh.
4. Verify `skinSsboMapped` pointer in `VulkanShadowRecordContext`.
5. Compare joint palette upload order: lit pass vs shadow pass.

## Summary

Skinned shadow casters reuse the bone SSBO and skinning function from the lit vertex shader. Spark passes identical skin bindings in `VulkanShadowRecordContext`.

---

Next: [Bias and Artifacts](08-shadow-tuning.md).
