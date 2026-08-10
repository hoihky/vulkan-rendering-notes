# Uniform Buffer Objects and std140

> **Build-Along Milestone:** `SceneUniformGpu` on the CPU and `SceneUBO` in GLSL describe the same 656-byte block — verified by `static_assert` and a RenderDoc buffer view.

**You will learn:**

- Explain std140 alignment rules and why they matter for Vulkan UBOs.
- Map every field in Spark's scene UBO from C++ writer to shader `ubo` access.
- Update UBOs safely with frame-in-flight buffering.

## 1. The Problem We Are Solving

Uniform buffer objects (UBOs) carry **per-frame** data shared by many draws: camera matrices, light direction, shadow matrices, cluster grid metadata. Unlike push constants (small, per-draw), UBOs can hold hundreds of bytes but must obey strict **memory layout** rules.

If C++ struct packing disagrees with GLSL `std140`, the GPU reads garbage — lights drift, shadows shear, cluster indices point at wrong lights. Symptoms are subtle: "shadows almost work" or "ambient color tints wrong channel."

Spark's scene UBO is **656 bytes** with nested `mat4` arrays for cascaded shadows. This chapter teaches std140 mechanics on that real layout, not a toy `vec4 color` example.

## 2. Core Concepts

### UBO vs push constants vs SSBO

| Mechanism | Typical size | Update rate | Layout |
|-----------|--------------|-------------|--------|
| Push constants | ≤ 128–256 B (device limit) | Per draw | Packed, no std140 |
| Uniform buffer | ≤ 64 KB (min spec) | Per frame / per material | **std140** |
| Storage buffer | Large | Per frame / per object | **std430** |

Spark puts frame-global data in UBO (binding 0) and punctual light **arrays** in SSBOs (bindings 3+) because std140 would waste padding on large arrays.

### std140 alignment rules (essential subset)

| Type | Base alignment | Size in array |
|------|----------------|---------------|
| `float`, `int` | 4 | 4 |
| `vec2` | 8 | 8 |
| `vec3`, `vec4` | 16 | 16 |
| `mat4` | 16 (columns aligned) | 64 |
| `struct` | round up to 16 | each element starts at 16-byte boundary |

Rules that bite beginners:

1. **`vec3` is padded to 16 bytes** — next scalar may waste 4 bytes.
2. **`mat4` starts at 16-byte offset** — good for `viewProj` at offset 0.
3. **Array of `mat4`** — each matrix is 64 bytes, stride 64 (not 16).
4. **No tight packing** — you cannot memcpy a C struct with `#pragma pack(1)` and expect it to match GLSL.

### Spark SceneUBO GLSL declaration

```glsl
layout(std140, set = 0, binding = 0) uniform SceneUBO {
    mat4 viewProj;           // offset 0
    vec4 lightDir;           // 64
    vec4 cameraPos;          // 80
    vec4 lightColor;         // 96
    vec4 ambientColor;       // 112
    vec4 ambientSky;         // 128
    vec4 ambientProbe;       // 144
    mat4 invViewProj;        // 160
    vec4 viewportSize;       // 224
    vec4 timeGlobal;         // 240
    mat4 worldToShadowClip[4]; // 256
    vec4 cascadeSplits;      // 512
    vec4 cascadeAtlas[4];    // 528
    vec4 shadowParams;       // 592
    vec4 clusterGrid;        // 608
    vec4 clusterDepth;       // 624
    vec4 iblParams;          // 640
} ubo;
```

Total: **656 bytes**. The C++ mirror is `SceneUniformGpu` with `float` arrays instead of GLSL types — same offsets if fields are ordered identically.

### C++ mirror and static_assert

```cpp
struct SceneUniformGpu {
    float viewProj[16]{};
    float lightDir[4]{};
    // ... same field order as GLSL ...
    float iblParams[4]{};
};
static_assert(sizeof(SceneUniformGpu) == 656);
```

`VulkanSceneUniformWriter.cpp` adds offset checks:

```cpp
static_assert(offsetof(Ubo, ambientSky) == 128);
static_assert(offsetof(Ubo, worldToShadowClip) == 256);
static_assert(offsetof(Ubo, iblParams) == 640);
```

When you add a field, update **three places**: GLSL header, C++ struct, static_assert block — or validation layers stay silent while pixels lie.

### Field semantics (what shaders actually read)

| Field | Shader usage |
|-------|--------------|
| `viewProj` / `invViewProj` | Vertex clip transform; sky ray reconstruction |
| `lightDir`, `lightColor` | Directional sun/moon |
| `cameraPos` | View vector, IBL, sky |
| `ambientColor`, `ambientSky`, `ambientProbe` | Hemispherical ambient blend |
| `viewportSize` | NDC reconstruction, shadow fade, flip flags |
| `timeGlobal` | Animation time, shadow blend, time-of-day |
| `worldToShadowClip[4]` | Cascaded shadow map sampling |
| `cascadeSplits`, `cascadeAtlas` | Which cascade, atlas UV transform |
| `shadowParams` | Bias, normal bias, texel size |
| `clusterGrid`, `clusterDepth` | 3D light cluster indexing |
| `iblParams` | Environment map layer and intensity |

### Descriptor binding and update flow

```text
CPU: VulkanSceneUniformWriter::Write → mapped host-visible buffer
GPU: vkCmdBindDescriptorSets (set 0) → scene.frag reads ubo.*
```

Spark uses **frame-in-flight** UBO slices: each swapchain frame index maps to a different region of a larger `VkBuffer` so the GPU reads last frame's data while CPU writes this frame — no hazard without explicit sync beyond fence wait.

Minimum UBO alignment: `VkPhysicalDeviceLimits::minUniformBufferOffsetAlignment` (often 256). Spark allocates `kSceneUniformGpuBytes` per frame slot, rounded up to alignment.

### std140 vs std430

| Layout | Use case | Array stride |
|--------|----------|--------------|
| std140 | Uniform buffers | 16-byte minimum per element |
| std430 | Storage buffers | Tight packing |

Spark skin bones use `layout(std430) buffer SkinSSBO` — smaller stride for `mat4 bones[]`. Never put large dynamic arrays in UBOs; padding explodes.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Shared scene_ubo.glsl include

**Concept:** GLSL SceneUBO declaration

**Decision:** Single header included by `scene.vert`, `scene.frag`, shadow shaders

**Why:** One binding 0 definition; impossible for stages to disagree on layout.

**In Spark:** `shaders/scene_ubo.glsl` with include guard `SPARK_SCENE_UBO_GLSL`.

### float[16] instead of Matrix4 in GPU struct

**Concept:** C++ mirror

**Decision:** `SceneUniformGpu` uses raw floats, `memcpy` from `Matrix4::m`

**Why:** GPU struct stays POD and matches GLSL memory exactly; no hidden C++ padding from class alignment.

**In Spark:** `VulkanSceneUniformWriter::Write` first lines.

### Aggressive static_assert on offsetof

**Concept:** std140 alignment

**Decision:** Assert critical offsets, not only total size

**Why:** Total size 656 can match while a middle field shifted — offsetof catches insertions.

**In Spark:** Lines 24–33 in `VulkanSceneUniformWriter.cpp`.

### Shadow matrices in UBO not push

**Concept:** UBO vs push constants

**Decision:** Four `worldToShadowClip` matrices in frame UBO

**Why:** 256 bytes of matrices exceed practical push constant budget; same for all draws in frame.

**In Spark:** Cascade update in uniform writer + shadow pass integration (Part V).

### Cluster metadata in UBO, lights in SSBO

**Concept:** std140 vs std430

**Decision:** `clusterGrid` / `clusterDepth` vec4s in UBO; light lists in SSBO

**Why:** Grid dimensions are tiny; hundreds of lights need std430 packing.

**In Spark:** `clustered_lights.glsl` reads SSBO binding 3+.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `shaders/scene_ubo.glsl`
- `include/spark/render/scene/VulkanSceneUniformGpu.hpp`
- `src/spark/render/scene/VulkanSceneUniformWriter.cpp`
- Descriptor setup in scene renderer (binding 0, `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER`)

### Step 1 — Dump UBO from shader

Temporarily in `scene.frag`:

```glsl
outColor = vec4(ubo.lightColor.rgb, 1.0);
```

Change `lightColor` on CPU — if green channel moves when you changed red, offsets are wrong.

### Step 2 — Verify viewProj

Write identity to `viewProj` on CPU. Mesh should appear in canonical clip space. Write known translation — object should shift predictably.

### Step 3 — Add offsetof when extending

To add `vec4 debugColor` at the end:

1. Append to GLSL and C++ struct
2. `static_assert(offsetof(Ubo, debugColor) == 656)` (new size 672)
3. Update `kSceneUniformGpuBytes` and buffer allocation
4. Rebuild all shaders including the header

### Step 4 — Frame-in-flight slice

If using two frames in flight, allocate `2 * alignedSize` bytes. On frame `i`:

```text
offset = i * alignedSize
vkCmdBindDescriptorSets(..., dynamicOffset = offset)  // if using dynamic uniform buffer
```

Spark may use separate descriptor sets per frame or dynamic offsets — check your descriptor pool implementation when mirroring.

### Step 5 — RenderDoc buffer inspection

Capture a frame. In the resource inspector, open the bound uniform buffer for draw call. Compare hex dump at offset 64 (`lightDir`) against CPU writer values.

## 5. How This Connects to the Rest of Your Engine

Chapters **2–3** consume `ubo.viewProj`, `ubo.lightDir`, and shadow fields. Chapter **5** handles per-draw data that does **not** belong in the UBO. Part **V** fills `worldToShadowClip` from the shadow pass. Part **IV** uses `clusterGrid` for light loops.

The UBO is the **frame snapshot** of lighting and camera — keep it free of per-object transforms.

## 6. Pitfalls and Debugging

- **Wrong size only in fragment stage** — different include or duplicate `SceneUBO` block with different order.
- **`vec3` followed by `float` in custom structs** — unexpected 4-byte hole; prefer `vec4` packs in shared headers.
- **Matrix transpose** — `memcpy` column-major C++ to GLSL `mat4` is correct; row-major math library is not.
- **Stale UBO for frame N** — writing slot 0 while GPU reads slot 1; match frame index to fence slot.
- **minUniformBufferOffsetAlignment** — binding offset 64 when alignment requires 256 causes validation error or garbage on some devices.

## 7. Exercises

1. Print `sizeof(SceneUniformGpu)` and every `offsetof` — build a markdown table matching GLSL comments.
2. Intentionally swap `lightDir` and `cameraPos` in C++ only — describe the visual symptom.
3. Add a `debugMode` float in a new `vec4` at the end; wire to shader false-color modes.
4. Compare std140 vs std430 size for `struct Light { vec4 pos; vec3 color; }` array of 64 lights — calculate padding waste.
5. Use `spirv-cross` or RenderDoc to disassemble UBO member offsets in generated SPIR-V.

## Summary

Spark's scene UBO is a 656-byte std140 block shared across vertex and fragment stages via `scene_ubo.glsl`. C++ writes `SceneUniformGpu` with matching layout guarded by `static_assert`. Understand vec4 alignment, mat4 array stride, and frame-in-flight buffering before adding shadow or cluster fields.

---

Next: [Push Constants](05-push-constants.md).
