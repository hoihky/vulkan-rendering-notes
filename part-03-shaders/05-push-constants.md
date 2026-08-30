# Push Constants

> **Build-Along Milestone:** Every `SceneDrawItem` reaches the GPU as 192 bytes of push constants — same layout in C++, `scene.vert`, and `scene.frag`.

**You will learn:**

- When push constants beat UBOs and dynamic offsets.
- Mirror `ModelPushConstants` across C++ and GLSL with size and offset proofs.
- Record `vkCmdPushConstants` in the draw loop without validation errors.

## 1. The Problem We Are Solving

Drawing a scene with hundreds of meshes, each with its own transform and material, you need **per-draw** GPU data. Options:

| Approach | Pros | Cons |
|----------|------|------|
| One UBO per object | Simple mental model | Descriptor explosion, updates costly |
| Dynamic UBO offsets | One buffer | Alignment padding, still large binds |
| **Push constants** | No descriptor slot, fast for small data | Size limit (~128–256 B typical) |
| SSBO indexing | Huge flexibility | Overkill for model matrix + material |

Spark pushes **192 bytes** per draw: model matrix, albedo tint, texture layer, PBR factors, shading flags, shadow flags. That fits comfortably within Vulkan minimum push constant range (128 bytes minimum guaranteed, Spark targets 256).

This chapter is about the **contract**: identical struct layout in C++ and both shader stages, pipeline layout range declaration, and command buffer updates.

### Push constants for beginners

**Push constants** are a small block of data (up to ~128–256 bytes) you upload **immediately before each draw** without binding a separate buffer:

```cpp
vkCmdPushConstants(cmd, layout, stages, offset, size, &push);
vkCmdDrawIndexed(cmd, ...);
```

Spark's `ModelPushConstants` holds per-object data that changes every draw:

| Field group | Examples |
|-------------|----------|
| Transform | `model` mat4 |
| Material | `albedo`, `metallic`, `roughness`, `textureLayer` |
| Maps | `normalMapLayer`, `metallicRoughnessMapLayer` |
| Flags | `shadingModel`, `shadowFlags`, glTF extensions |

**Why not put model matrix in the UBO?** You would need either thousands of UBOs or dynamic offsets with strict alignment. Push constants are designed for "small, frequent, per-draw" updates.

### CPU struct must match GLSL exactly

C++ (`VulkanSceneOpaquePass.hpp`):

```cpp
struct ModelPushConstants {
    float model[16];
    float albedo[4];
    int textureLayer;
    // ...
};
```

GLSL (`scene.vert` / `scene.frag`):

```glsl
layout(push_constant) uniform Push {
    mat4 model;
    vec4 albedo;
    int textureLayer;
    // ...
} push;
```

Same order, same sizes, same padding rules. Spark uses `static_assert` on sizeof. If you add a field on one side only, colors and transforms scramble silently.

## 1b. Concept Deep Dive — scene_push.glsl and ModelPushConstants

### The shared push block in both shader stages

`scene_push.glsl` defines the per-draw uniform both stages include:

```glsl
// shaders/scene_push.glsl
layout(push_constant) uniform Push {
    mat4 model;
    vec4 albedoTint;
    int textureLayer;
    int skyMode;
    float metallic;
    float roughness;
    vec4 emissive;
    int useSkinning;
    int jointCount;
    int shadingModel;
    int toonDiffuseBands;
    float toonRimIntensity;
    float toonRimPower;
    int normalMapLayer;
    int metallicRoughnessMapLayer;
    int emissiveMapLayer;
    // glTF map UV transforms, clearcoat, transmission, iridescence ...
} push;
```

Vertex code reads `push.model`, `push.useSkinning`, `push.jointCount`, and material varyings. Fragment code reads map layers, `push.shadingModel`, `push.shadowFlags`, and `push.alphaCutoff`. Identical layout in both stages is required — Vulkan links push ranges across the pipeline, not per stage.

### C++ struct with offset proofs

```cpp
// include/spark/render/scene/VulkanSceneOpaquePass.hpp (excerpt)
struct ModelPushConstants {
    float model[16]{};
    float albedo[4]{1, 1, 1, 1};
    std::int32_t textureLayer = -1;
    std::int32_t skyMode = 0;
    float metallic = 0.0F;
    float roughness = 0.45F;
    float emissive[4]{};
    std::int32_t useSkinning = 0;
    std::int32_t jointCount = 0;
    std::int32_t shadingModel = 0;
    // normalMapLayer, mapUvScale[5], clearcoat, transmission ...
};
```

Spark proves layout in `VulkanSceneOpaquePass.cpp`:

```cpp
static_assert(sizeof(ModelPushConstants) == 348);
static_assert(offsetof(ModelPushConstants, mapUvScale) == 176);
static_assert(offsetof(ModelPushConstants, clearcoatFactor) == 296);
```

The 4-byte `padMapUvAlign` before `mapUvScale` mirrors GLSL `vec2[]` 8-byte alignment — a common push-constant footgun when adding `int` fields before UV arrays.

### Typical per-draw usage in scene.vert / scene.frag

| Push field | Set from C++ | Shader use |
|------------|--------------|------------|
| `model` | Entity world matrix | Vertex transform; normal transport |
| `textureLayer` | Material atlas slot | `flat vTextureLayer` → `texture()` layer |
| `skyMode` | Sky draw flag | Vertex clip Z hack; fragment early-out |
| `shadingModel` | Material enum | PBR / toon / unlit dispatch |
| `shadowFlags` | Material flags | `(push.shadowFlags & 2)` receive shadows |
| `mapUvScale`, `mapTexCoordSet` | glTF extensions | `sparkMapUv()` in fragment |

Recording order in the opaque pass: memcpy skin SSBO (if needed) → `vkCmdPushConstants` with full 348 bytes → `vkCmdDrawIndexed`. RenderDoc's push constant viewer is the fastest way to verify C++ and GLSL agree after adding a field.

## 2. Core Concepts

### Push constant memory model

Push constants are a small, **command-buffer-scoped** memory window. The driver copies your `vkCmdPushConstants` payload into GPU-accessible storage visible to shader stages you specify in `VkPushConstantRange`.

```text
vkCmdPushConstants(cmd, layout, stages, offset, size, pData)
    → visible in shader as layout(push_constant) uniform Push { ... }
```

Unlike descriptor sets, push constants do not require a bind call — they are overwritten each push. Last push before `vkCmdDraw` wins.

### Spark ModelPushConstants layout (C++)

```cpp
struct ModelPushConstants {
    float model[16]{};              // 0, 64 bytes
    float albedo[4]{1,1,1,1};       // 64
    std::int32_t textureLayer = -1; // 80
    std::int32_t skyMode = 0;
    float metallic = 0.0F;
    float roughness = 0.45F;
    float emissive[4]{};              // 96
    std::int32_t useSkinning = 0;     // 112
    std::int32_t jointCount = 0;
    std::int32_t shadingModel = 0;
    std::int32_t toonDiffuseBands = 3;
    float toonRimIntensity = 0.35F;
    float toonRimPower = 4.0F;
    std::int32_t normalMapLayer = -1;
    std::int32_t metallicRoughnessMapLayer = -1;
    std::int32_t emissiveMapLayer = -1;
    float metallicFactor = 1.0F;
    float roughnessFactor = 1.0F;
    float occlusionStrength = 1.0F;
    std::int32_t shadowFlags = 0;
    std::int32_t pbrPad0 = 0;
    float std430PadEmissive[2]{};
    float emissiveFactor[4]{1,1,1,0};
};
static_assert(sizeof(ModelPushConstants) == 192);
static_assert(offsetof(ModelPushConstants, emissiveFactor) == 176);
```

GLSL duplicates the same block in `scene.vert` and `scene.frag` — SPIR-V linking does not cross-check C++, so `static_assert` on the C++ side is mandatory.

### GLSL push block

```glsl
layout(push_constant) uniform Push {
    mat4 model;
    vec4 albedoTint;
    int textureLayer;
    int skyMode;
    float metallic;
    float roughness;
    vec4 emissive;
    int useSkinning;
    int jointCount;
    int shadingModel;
    // ... maps, factors, shadowFlags, emissiveFactor ...
} push;
```

Access: `push.model`, `push.textureLayer`, `push.skyMode` in both stages.

### Pipeline layout push range

```cpp
VkPushConstantRange pushConstantRange{};
pushConstantRange.stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT;
pushConstantRange.offset = 0;
pushConstantRange.size = sizeof(VulkanSceneOpaquePass::ModelPushConstants);
```

Both stages read the same block — vertex uses `model` and skinning flags; fragment uses material and `shadingModel`. Alternative: split ranges with different offsets if you exceed limits (not needed in Spark).

### Recording in the draw loop

```cpp
std::memcpy(push.model, d.model.m, sizeof(push.model));
push.textureLayer = d.textureLayer;
push.metallic = d.metallic;
// ... fill from SceneDrawItem ...

vkCmdPushConstants(
    commandBuffer,
    ctx.pipelineLayout,
    VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
    0,
    sizeof(ModelPushConstants),
    &push);
vkCmdDrawIndexed(commandBuffer, indexCount, 1, firstIndex, vertexOffset, 0);
```

Order matters: push **after** bind pipeline/layout, **before** draw. Changing push without rebinding is fine.

### Push constants vs specialization constants

| Feature | Push constants | Specialization constants |
|---------|----------------|--------------------------|
| Changes | Every draw | At pipeline create (sparse) |
| Size | Small | 32-bit ints, limited count |
| Use | Model, material | Workgroup size, static flags |

Spark uses push for per-draw material; pipeline variants (opaque/transparent/sky) handle blend and depth — not specialization.

### Device limits

Query:

```cpp
props.limits.maxPushConstantsSize  // often 128 or 256
```

192-byte block requires ≥ 192. All mainstream desktop GPUs satisfy this. Mobile: verify early.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### 192-byte unified push block

**Concept:** Spark ModelPushConstants layout

**Decision:** One struct for vertex + fragment material/transform data

**Why:** Single `vkCmdPushConstants` per draw; RenderDoc shows one blob; avoids offset math across ranges.

**In Spark:** `VulkanSceneOpaquePass.hpp`, duplicated GLSL `Push` block.

### Duplicate GLSL in vert and frag

**Concept:** GLSL push block

**Decision:** Copy-paste `layout(push_constant)` into both shader files

**Why:** glslang compiles stages separately; include file for push block would work but Spark inlines for clarity next to usage.

**In Spark:** `scene.vert` and `scene.frag` identical `Push` declarations — must stay in sync manually.

### Both stage flags in range

**Concept:** Pipeline layout push range

**Decision:** `VERTEX_BIT | FRAGMENT_BIT` on one range

**Why:** Fragment needs `shadingModel`, `textureLayer`; vertex needs `model`, `useSkinning`.

**In Spark:** `VulkanScenePipeline.cpp` lines 189–192.

### skyMode in push not UBO

**Concept:** Recording in draw loop

**Decision:** Per-draw `skyMode` selects pipeline branch and shader path

**Why:** Sky and mesh draws interleave in same pass; frame UBO cannot distinguish per object.

**In Spark:** `VulkanSceneOpaquePass` switches `pipelineSky` vs `pipelineLit`.

### Padding fields for alignment

**Concept:** C++ static_assert

**Decision:** `pbrPad0`, `std430PadEmissive` pad to stable 192 bytes

**Why:** Future fields without reshuffling entire shader layout; matches GPU std140-like rules for mixed int/float.

**In Spark:** offsetof asserts at top of `VulkanSceneOpaquePass.cpp`.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/render/scene/VulkanSceneOpaquePass.hpp`
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp`
- `src/spark/render/scene/VulkanScenePipeline.cpp`
- `shaders/scene.vert`, `shaders/scene.frag` (`Push` block)

### Step 1 — Minimum push: model matrix only

Shrink GLSL and C++ to `mat4 model` only (64 bytes). Push identity, then translation. Confirm mesh moves.

### Step 2 — Expand to full 192 bytes

Add fields back one group at a time (material, skinning, maps). Re-run `static_assert` after each addition.

### Step 3 — Fragment-only field test

Set `push.shadingModel` to toon on one draw item. Others stay PBR — proves per-draw independence from UBO.

### Step 4 — Validation layer check

Wrong size triggers `VUID-vkCmdPushConstants-size-08878`. Wrong stage flags: fragment reads uninitialized push.

### Step 5 — RenderDoc

Select draw call → Pipeline State → View Push Constants. Compare hex to C++ struct dump for that `SceneDrawItem`.

### Step 6 — Transparent pass

`RecordTransparent` uses the same push layout and `pipelineLitTransparent` — only blend state differs. Confirms push contract is pass-independent.

## 5. How This Connects to the Rest of Your Engine

Chapter **2** uses `push.model` and `push.useSkinning`. Chapter **3** reads material and `shadingModel`. Chapter **4** holds frame data push deliberately avoids. Sprite and UI passes define **different** push structs (`SpriteBatch`, text PC) — same mechanism, different layouts per pipeline layout.

Push constants are the right tool for **draw frequency** data under ~200 bytes.

## 6. Pitfalls and Debugging

- **GLSL/C++ field order drift** — add field to C++ only: albedo tints metallic instead of color.
- **int vs float alignment** — after `vec4`, ints align to 4 bytes; inserting `float` before `int` without padding shifts everything.
- **Push without bind layout** — undefined which layout receives data.
- **Size 192 but range 128** — fails on strict devices; query limits at startup.
- **Forgot push on instanced path** — all instances share last push from previous draw.

## 7. Exercises

1. Add `int debugId` to push block — output as hue in fragment shader per draw.
2. Split into two push ranges (vertex 64 B, fragment 128 B) — when is this worth the complexity?
3. Measure CPU cost: 1000 draws with push vs dynamic UBO offsets (expect push cheaper).
4. Document byte offset of every `ModelPushConstants` field in a table.
5. Break `sizeof` assert by 4 bytes — read validation error and fix padding.

## Summary

Spark delivers per-draw transform and material data through a 192-byte push constant block shared by vertex and fragment shaders. Declare `VkPushConstantRange` for both stages, fill `ModelPushConstants` from each `SceneDrawItem`, and call `vkCmdPushConstants` immediately before each draw. Keep C++ and GLSL layouts identical with `static_assert` — the GPU will not forgive drift.

---

Next: [Shader Includes and Shared Code](06-shader-includes.md).
