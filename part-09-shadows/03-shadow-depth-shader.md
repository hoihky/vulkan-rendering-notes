# Shadow Depth Shader

> **Build-Along Milestone:** A single cube casts a visible depth silhouette in the directional shadow pass.

**You will learn:**

- Author a depth-only vertex shader that outputs light-space clip position.
- Match vertex input layout to the lit pass for shared mesh buffers.
- Configure pipeline state with depth write and no color attachments.

## 1. The Problem We Are Solving

The shadow pass does not need albedo, normals, or lighting — only **correct clip-space depth** from the light's view. Most bugs are pipeline mismatch (color writes left on) or matrix mismatch (different light projection than the sampling shader expects).

Spark's `shadow_depth.vert` is the reference implementation you will mirror in your engine.

### Depth-only pass for beginners

The shadow pass is simpler than the main lit pass:

```text
Main pass shader:     position + normals + UVs → color + depth
Shadow pass shader:   position only           → depth ONLY (no color buffer)
```

There is **no fragment shader** (or an empty one) — the GPU writes depth automatically from the vertex shader's `gl_Position.z`.

Pipeline differences:

| Setting | Lit pipeline | Shadow pipeline |
|---------|--------------|-----------------|
| Color attachment | RGBA16F HDR | **None** |
| Depth attachment | D32 | D32 shadow map |
| Depth write | ON | ON |
| Color write | ON | **OFF** |

The vertex shader transforms position by **light view-projection** instead of camera `viewProj`:

```glsl
// shadow_depth.vert (concept)
gl_Position = ubo.worldToShadowClip[cascade] * push.model * vec4(position, 1.0);
```

Same `push.model` and vertex buffer as the main pass — only the matrix changes. That is why winding and mesh layout must match.

## 1b. Concept Deep Dive — shadow_depth.vert Contract

### Shared vertex input with lit pass

```glsl
// shaders/shadow_depth.vert (lines 3–8)
layout(location = 0) in vec3 inPosition;
layout(location = 1) in vec3 inNormal;
layout(location = 2) in vec2 inTexCoord;
layout(location = 3) in vec4 inTangent;
layout(location = 4) in vec4 inJointsPacked;
layout(location = 5) in vec4 inWeights;
```

Same locations as `scene.vert` — one interleaved vertex buffer serves both pipelines. Normals and UVs are unused in the shadow pass but must be present for attribute layout compatibility.

### Push constants — model + lightViewProj only

```glsl
// shadow_depth.vert (lines 14–21, 52)
layout(push_constant) uniform ShadowPush {
    mat4 model;
    mat4 lightViewProj;
    int useSkinning;
    int jointCount;
} sp;
// ...
gl_Position = sp.lightViewProj * vec4(worldPos, 1.0);
```

No material, no camera `viewProj`. `RecordShadowCastMeshes` memcpy's `d.model` and the cascade-specific light matrix per draw.

### RecordShadowCastMeshes — indexed draws from scene snapshot

```cpp
// src/spark/render/shadow/VulkanShadowCastDraw.cpp (lines 43–98)
for (std::size_t di = 0; di < ctx.scene->draws.GetSize(); ++di) {
    const SceneDrawItem& d = ctx.scene->draws[di];
    if (d.skyMode != SceneSkyMode::None) continue;
    if ((d.shadowFlags & kSceneShadowCast) == 0) continue;
    // bind static or custom VB/IB, push constants, vkCmdDrawIndexed
}
```

Sky draws are skipped. Only casters enter the depth pass. Static scene meshes (cube, plane) and custom dynamic meshes share the same shader — binding switches when geometry source changes.

## 2. Core Concepts

### Vertex shader contract

The shadow pass only needs clip-space depth from the light's view:

```glsl
gl_Position = lightViewProj * model * skinnedOrRigidPosition;
```

No varyings are required when depth comes from rasterization and the fragment stage is empty or minimal.


### Push constant layout

Shadow draws carry `model`, `lightViewProj`, and optional skinning flags in push constants — small, per-draw data that fits in the 128-byte default limit without extra descriptor bindings for matrices.


### Shared vertex layout

Shadow and lit passes must agree on vertex attribute locations — position, joints, weights — so the same interleaved VBs work in both pipelines.


### Pipeline depth state

```text
depthTestEnable  = true
depthWriteEnable = true
depthCompareOp   = LESS
color attachments = none
cullMode         = back (or front for Peter-panning mitigation)
```


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


### Push constant byte budget

Vulkan guarantees at least **128 bytes** push constant space per pipeline layout. Spark's `ShadowPush` carries two `mat4` (128 bytes) plus skinning ints — at the limit. If you add material indices, use SSBO or dynamic UBO offsets instead of expanding push constants.

Verify with:
```text
sizeof(VulkanShadowPushConstants) == GLSL ShadowPush size
```
Mismatch causes **offset shadows** where model matrix is wrong but shader compiles.


### Pipeline compatibility rules

Shadow pipeline must declare the **same vertex input bindings and attribute locations** as the lit pipeline. Differences in stride, format, or location cause silent misreads — positions collapse or skinning weights read wrong memory.

Create shadow pipeline against **shadow render pass** handle, not HDR. Depth bias enable is optional on the pipeline — complement shader-side receiver bias (Chapter 8), do not max both.


### Empty shadow map diagnosis

If RenderDoc shows cleared depth but no draws: (1) no casters with `kSceneShadowCast`, (2) light frustum excludes geometry, (3) wrong cull mode for winding, (4) pipeline bound after descriptor mismatch. Disable cull temporarily to test winding before rewriting projection math.


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


### VulkanShadowCastDraw flow

`RecordShadowCastMeshes` iterates `SceneRenderParams::draws`, skips sky and non-casters, resolves mesh binding (static vs custom), binds VB/IB once per binding change, sets `ShadowPush` with `lightViewProj`, optionally binds skin SSBO offset, pushes constants, draws indexed. Centralizing here prevents duplicating caster logic in directional and punctual pass recorders.


### Shadow pipeline create parameters

When creating the shadow pipeline, set `VkPipelineRasterizationStateCreateInfo` cull mode to back-face unless art requires double-sided foliage shadows. Depth bias enable at pipeline level adds constant/slope factors during rasterization — coordinate with shader receiver bias (Chapter 8).

`VkPipelineVertexInputStateCreateInfo` must reference `VulkanSceneVertexLayout` — six attributes for rigid+skin path. Fragment shader stage can be omitted entirely (no FS module) when depth writes come from rasterizer alone — Spark uses minimal FS or empty depending on pass setup; verify in RenderDoc whether FS runs.


### Per-draw push update cadence

Every caster draw updates push constants with `model` and cascade-specific `lightViewProj`. Batch draws that share the same mesh binding but differ in model matrix — each needs its own push before draw. Skinned draws additionally bind SSBO offset for bone palette slice.

Profile CPU cost: shadow record is often draw-call bound on large scenes — instancing or GPU-driven draws are future optimizations; correctness first via Spark's explicit loop.


### GLSL ShadowPush layout

```glsl
layout(push_constant) uniform ShadowPush {
    mat4 model;           // 64 bytes
    mat4 lightViewProj;   // 64 bytes
    int useSkinning;      // 4 bytes
    int jointCount;
    int _sp0;
    int _sp1;
} sp;
```

Total 136 bytes — exceeds 128-byte minimum guarantee on some mobile parts if padding differs. Spark relies on desktop-class limits; verify `VkPhysicalDeviceLimits::maxPushConstantsSize` on min-spec target. C++ `VulkanShadowPushConstants` must match GLSL layout exactly — use static_assert on sizeof.


### Depth-only render pass gotchas

Without color attachments, `VkRenderPassBeginInfo::clearValueCount` includes only depth clear value. Some validation layers warn if color clear is specified — shadow pass uses depth clear 1.0 (far). After pass, depth image layout must be `DEPTH_STENCIL_READ_ONLY_OPTIMAL` for scene shader sampling — mismatch produces `-13` style compare failures or validation sync errors.


### Reference shadow_depth.vert walkthrough

Lines 23–52 implement the full transform: unpack joints, skinning loop with weight/joint guard, multiply by `model`, then `lightViewProj`. No varyings exported — depth comes purely from rasterizer interpolating clip-space Z. This minimalism keeps shadow VS fast and easy to validate against lit path skinning.

When diffing against `scene.vert`, ignore normal/tangent outputs — shadow path intentionally omits them to reduce register pressure and layout coupling.


### Single-cube shadow milestone

Place one `castsShadow` cube and one directional light. Capture RenderDoc — shadow pass should show exactly one indexed draw; depth texture shows square silhouette. Only then enable CSM, skinning, and PCF. This isolates pipeline/layout bugs from cascade math bugs that otherwise compound debugging time across multiple subsystems changed simultaneously.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Dedicated shadow_depth.vert

**Concept:** Vertex shader contract

**Decision:** Dedicated shadow_depth.vert

**Why:** Do not branch the lit vertex shader for depth-only draws — a slim shader keeps pipeline creation fast and validation clear.

**In Spark:** `shaders/shadow_depth.vert` — compiled into the directional and punctual shadow pipelines.

### ShadowPush with lightViewProj per cascade

**Concept:** Push constant layout

**Decision:** ShadowPush with lightViewProj per cascade

**Why:** Per-draw `model` plus per-cascade `lightViewProj` in push constants avoids dynamic UBO offsets inside tight caster loops.

**In Spark:** GLSL block `ShadowPush` matches C++ `ShadowPushConstants`.

### Identical input to scene pass

**Concept:** Shared vertex layout

**Decision:** Identical input to scene pass

**Why:** Rigid meshes share layout; skinned paths include joint/weight attributes at the same locations as `scene.vert`.

**In Spark:** `VulkanSceneVertexLayout` is referenced when creating the shadow pipeline.

### Optional pipeline depth bias

**Concept:** Pipeline depth state

**Decision:** Optional pipeline depth bias

**Why:** Slope and constant depth bias on the shadow pass pipeline complement receiver-side shader bias (Chapter 8).

**In Spark:** `VulkanDirectionalShadowPass` pipeline creation sets depth bias enable when configured.

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
- `src/spark/render/shadow/VulkanShadowCastDraw.cpp`
- `src/spark/render/shadow/VulkanDirectionalShadowPass.cpp`

### Step 1 — Read shadow_depth.vert

The vertex shader handles rigid and skinned paths with the same bone SSBO binding as the lit pass.

*Shadow push constants and skinning*

```glsl
// shaders/shadow_depth.vert (lines 14–53)
layout(push_constant) uniform ShadowPush {
    mat4 model;
    mat4 lightViewProj;
    int useSkinning;
    int jointCount;
    int _sp0;
    int _sp1;
} sp;

void main() {
    vec3 worldPos;
    uvec4 inJoints = uvec4(
            floatBitsToUint(inJointsPacked.x),
            floatBitsToUint(inJointsPacked.y),
            floatBitsToUint(inJointsPacked.z),
            floatBitsToUint(inJointsPacked.w));

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
}
```

### Step 2 — Compare joint SSBO binding

Binding 2 is the skin palette — shadow and scene passes must use the same index or skinned casters produce detached silhouettes.

*Skin SSBO binding*

```glsl
// shaders/shadow_depth.vert (lines 10–12)
layout(std430, set = 0, binding = 2) readonly buffer SkinSSBO {
    mat4 bones[];
} skin;
```

### Step 3 — Trace caster draw helper

Spark centralizes shadow draw recording in `VulkanShadowCastDraw` — filter casters, bind pipeline, push constants, draw indexed.

*Shadow cast draw module (open file and follow Record calls)*

```cpp
// src/spark/render/shadow/VulkanShadowCastDraw.cpp (lines 1–40)
#include "spark/render/shadow/VulkanShadowCastDraw.hpp"

#include "spark/math/Matrix4.hpp"
#include "spark/render/lighting/SceneLightingProfile.hpp"
#include "spark/render/scene/VulkanSceneMeshDraw.hpp"

#include <cstring>

namespace Spark {

void RecordShadowCastMeshes(
        const VkCommandBuffer commandBuffer,
        const VkPipeline pipeline,
        const VkPipelineLayout pipelineLayout,
        const VulkanShadowRecordContext& ctx,
        const float lightViewProj[16],
        const std::uint32_t frameIndex) {
    if (!ctx.sceneParamsValid || ctx.scene == nullptr || ctx.vertexBuffer == VK_NULL_HANDLE ||
        ctx.indexBuffer == VK_NULL_HANDLE || pipeline == VK_NULL_HANDLE || pipelineLayout == VK_NULL_HANDLE) {
        return;
    }

    enum class ShadowGeom : std::uint8_t { None, StaticScene, CustomDynamic };

    const SceneMeshDrawBindings meshBindings{
            .staticVertexBuffer = ctx.vertexBuffer,
            .staticIndexBuffer = ctx.indexBuffer,
            .customVertexBuffer = ctx.customVertexBuffer,
            .customIndexBuffer = ctx.customIndexBuffer,
            .cubeIndexCount = ctx.cubeIndexCount,
            .planeIndexCount = ctx.planeIndexCount,
            .planeFirstIndex = ctx.planeFirstIndex,
            .cubeVertexOffset = ctx.cubeVertexOffset,
            .planeVertexOffset = ctx.planeVertexOffset,
            .customDrawPacked = ctx.customDrawPacked,
    };

    VulkanShadowPushConstants sh{};
    std::memcpy(sh.lightViewProj, lightViewProj, sizeof(sh.lightViewProj));

```

### Step 4 — Create your depth-only pipeline

When porting, create the pipeline against the **shadow render pass** handle, not the HDR pass. Vertex input state must match your scene mesh layout exactly.

Compile with your existing glslang rule from Part III.

### Step 5 — Debug empty shadow map

If RenderDoc shows a cleared but empty depth target:

1. Confirm casters have `castsShadow == true`.
2. Confirm light frustum encloses scene bounds.
3. Temporarily disable cull mode to catch inverted winding.

### Step 6 — Single-cube proof

Place one cube caster, one directional light, and capture shadow depth. You should see a silhouette before enabling CSM or PCF.

### Step 9 — Cross-check ShadowPush sizeof

Verify C++ push struct size against GLSL with static_assert and SPIR-V reflection if available — 136-byte layouts are tight on mobile push constant limits.

*ShadowPush constant block*

```glsl
// shaders/shadow_depth.vert (lines 14–21)
layout(push_constant) uniform ShadowPush {
    mat4 model;
    mat4 lightViewProj;
    int useSkinning;
    int jointCount;
    int _sp0;
    int _sp1;
} sp;
```

## 5. How This Connects to the Rest of Your Engine

This shader pairs with **PCF sampling** in `scene.frag` (Chapter 4). The same `worldToShadowClip` used for sampling must match `lightViewProj * model` used here, plus cascade atlas transform for UV.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Z-fighting in shadow pass** — enable pipeline depth bias or tune near/far planes.
- **Double-sided foliage** — may need `CULL_NONE` for shadow pass only.
- **Skinned mesh with rigid shadow shader** — collapsed mesh; wire skinning (Chapter 7).

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Render shadow pass for one cube; confirm depth image updates when it moves.
2. Compare `sizeof` push constant range against GLSL `ShadowPush` block.
3. Flip cull mode temporarily — observe missing casters on inverted winding.
4. Capture one draw — verify no color attachments bound.
5. List push constant fields in `shadow_depth.vert` vs `scene.vert`.

## Summary

Shadow depth rendering is a light-view vertex transform with depth write enabled and no color output. Spark shares mesh layouts and skin SSBO bindings with the lit pass.

---

Next: [PCF and Cascades](04-pcf-cascades.md).
