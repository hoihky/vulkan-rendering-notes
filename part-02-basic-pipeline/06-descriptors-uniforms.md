# Descriptor Sets and Uniform Buffers

> **Build-Along Milestone:** You can explain set 0 binding 0 (`SceneUBO`), per-flight uniform buffers, and when descriptor sets are bound before draw.

**You will learn:**

- Create a descriptor set layout matching shader `set` and `binding` declarations.
- Allocate per-frame uniform buffers and map them for CPU writes.
- Update descriptor sets to point at buffers and textures.

## 1. The Problem We Are Solving

Shaders need **per-frame** data (view matrix, lights) and **per-draw** data (model matrix). Vulkan exposes two main mechanisms:

| Mechanism | Best for | Spark usage |
|-----------|----------|-------------|
| Uniform buffer (UBO) | Small, frequently changing frame data | `SceneUBO` at set 0 binding 0 |
| Push constants | Small per-draw data, low latency | `model` matrix in `scene.vert` |

**Descriptor sets** bind UBOs, texture arrays, and SSBOs to shader slots. This chapter covers the UBO path — without it, shaders read uninitialized memory and everything renders black or magenta validation errors.

## 2. Core Concepts

### Descriptor set layout

Shader declares:

```glsl
// shaders/scene_ubo.glsl
layout(std140, set = 0, binding = 0) uniform SceneUBO {
    mat4 viewProj;
    vec4 lightDir;
    // ...
} ubo;
```

CPU creates matching `VkDescriptorSetLayoutBinding`:

| binding | type | stages |
|---------|------|--------|
| 0 | `UNIFORM_BUFFER` | vertex + fragment |
| 1 | `COMBINED_IMAGE_SAMPLER` | fragment (textures — Part IV) |
| ... | lights, shadows | later parts |

Layout is specified at **pipeline layout** creation time.

### std140 alignment

GLSL `std140` packs with strict alignment rules:

- `vec3` starts on 16-byte boundary
- `mat4` is 64 bytes
- arrays of structs stride to 16-byte multiples

CPU struct in `VulkanSceneUniformWriter` must match exactly — use static asserts or generated headers. Part III expands alignment rules.

### Per-flight uniform buffers

With two frames in flight, you need **two uniform buffer regions** — CPU writes frame N while GPU reads frame N-1:

```text
uniformBuffers[0]  ← written when currentFrame == 0
uniformBuffers[1]  ← written when currentFrame == 1
```

Map once at creation (`HOST_VISIBLE`), `memcpy` each frame after fence wait.

### Descriptor pool and sets

`VkDescriptorPool` allocates `VkDescriptorSet` handles from the layout. Spark creates one set per flight index and updates bindings when textures change (Part IV).

Typical allocation flags: `FREE_DESCRIPTOR_SET` optional; pool sizes count UBO + combined image sampler slots.

### Binding before draw

```cpp
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, 1, &descriptorSet, 0, nullptr);
```

Set index `0` matches `set = 0` in GLSL. Push constants use `vkCmdPushConstants` separately — not part of descriptor sets.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanSceneDescriptors owns layout + pool + sets

**Concept:** Descriptor set layout

**Decision:** One class creates layout, uniform buffers, pool, and per-flight sets

**Why:** Binding slots 0–9 evolve together; splitting risks mismatch between layout and updates.

**In Spark:** `include/spark/render/scene/VulkanSceneDescriptors.hpp`.

### SceneUBO at binding 0 only for Part II essentials

**Concept:** std140 UBO

**Decision:** `VulkanSceneUniformWriter` fills viewProj, camera, directional light, ambient

**Why:** Enough data for lit cube in Part II; shadow and cluster bindings added without changing the pattern.

**In Spark:** `VulkanSceneUniformWriter.cpp` + `scene_ubo.glsl`.

### Mapped uniform memory

**Concept:** Per-flight buffers

**Decision:** `vkMapMemory` at init; `UniformMapped(frameIndex)` returns pointer for `memcpy`

**Why:** Avoids `vkMapMemory` per frame; coherent host memory visible to GPU after write.

**In Spark:** `VulkanSceneDescriptors::UniformMapped`.

### Descriptor set bound in opaque pass

**Concept:** Binding before draw

**Decision:** `VulkanSceneOpaquePass::Record` binds `sceneDescriptors.DescriptorSet(frameIndex)`

**Why:** All lit draws share frame UBO; per-draw model uses push constants after bind.

**In Spark:** `VulkanSceneOpaquePass.cpp`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/scene/VulkanSceneDescriptors.hpp`
- `shaders/scene_ubo.glsl`
- `src/spark/render/scene/VulkanSceneUniformWriter.cpp`
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp`

### Step 1 — Shader UBO block

```glsl
// shaders/scene_ubo.glsl (lines 4–23, abbreviated)
layout(std140, set = 0, binding = 0) uniform SceneUBO {
    mat4 viewProj;
    vec4 lightDir;
    vec4 cameraPos;
    vec4 lightColor;
    vec4 ambientColor;
    // shadow matrices, clusters, IBL — later chapters
} ubo;
```

Vertex shader multiplies: `gl_Position = ubo.viewProj * worldPos`.

### Step 2 — Descriptor class API

```cpp
// VulkanSceneDescriptors.hpp (abbreviated)
class VulkanSceneDescriptors {
public:
    void CreateSetLayout(VkDevice device);
    void CreateUniformBuffers(VkPhysicalDevice physicalDevice, VkDevice device, std::uint32_t framesInFlight);
    void CreatePoolAndSets(VkDevice device, std::uint32_t framesInFlight, const BindingSources& sources);

    [[nodiscard]] VkDescriptorSet DescriptorSet(std::uint32_t frameIndex) const;
    [[nodiscard]] void* UniformMapped(std::uint32_t frameIndex) const;
};
```

### Step 3 — Write uniforms each frame

After `WaitForCurrentFrameFence` in `DrawFrame`, Spark calls `WriteUniformBuffer(currentFrame)` which copies resolved lighting and `pendingScene.viewProjection` into the mapped buffer for that flight index.

**Build-along:** Find `WriteUniformBuffer` in `VulkanRenderer.cpp` and list which `SceneRenderParams` fields feed the UBO.

### Step 4 — Bind descriptor set in pass

```cpp
// VulkanSceneOpaquePass.cpp (after pipeline bind)
    vkCmdBindDescriptorSets(
            commandBuffer,
            VK_PIPELINE_BIND_POINT_GRAPHICS,
            ctx.pipelineLayout,
            0,
            1,
            &ctx.descriptorSet,
            0,
            nullptr);
```

### Step 5 — Minimal UBO for your triangle

For Chapter 7's triangle without transforms, you may use an **empty** pipeline layout with no descriptors. Once you add a `viewProj` matrix to move a mesh, add one UBO binding — do not skip straight to ten bindings.

## 5. How This Connects to the Rest of Your Engine

Part **III** deep-dives std140 and push constant layouts. Part **IV** adds binding 1 texture array to the same set. Part **IX** adds shadow map samplers at higher bindings.

Descriptors are how frame-level shader inputs scale without recompiling pipelines every draw.

## 6. Pitfalls and Debugging

- **Layout mismatch** — shader binding 0 UBO vs layout binding 1 COMBINED_IMAGE_SAMPLER.
- **Writing wrong flight buffer** — flickering lights; index with `currentFrame` only after fence wait.
- **std140 padding wrong** — matrix columns shifted; compare CPU/GPU layouts in RenderDoc.
- **Forgetting to bind set** — garbage or default zero buffers.

## 7. Exercises

1. Open RenderDoc, capture a frame, inspect descriptor set 0 contents at a draw call.
2. How many uniform buffers does Spark allocate for two frames in flight?
3. Why are push constants used for `model` but not for `viewProj`?
4. List bindings 0–3 in `VulkanSceneDescriptors` comments — which are Part II vs Part IV?

## Summary

Descriptor sets connect CPU-owned buffers to shader bindings. Spark's `SceneUBO` at set 0 binding 0 carries per-frame view and lighting; per-flight buffers prevent GPU races. Bind the descriptor set once per pass, then issue draws with push constants for per-object data.

---

Next: [Drawing Your First Triangle](07-first-triangle.md).
