# Binding Textures

> **Build-Along Milestone:** Scene descriptor set binding 1 points at the uploader's `sampler2DArray`; `scene.frag` samples with `texture(sceneTextures, vec3(uv, layer))`.

**You will learn:**

- Declare combined image samplers in GLSL matching `VkDescriptorSetLayout`.
- Allocate descriptor pools and per-frame sets for scene rendering.
- Write image + sampler info into binding 1 each frame.
- Connect push constant `textureLayer` to array layer selection.

## 1. The Problem We Are Solving

A `VkImage` on the GPU is invisible to shaders until you bind:

1. **VkImageView** — typed view (2D array layer range)
2. **VkSampler** — filtering, addressing, LOD clamps
3. **VkDescriptorSet** — ties (image, sampler, layout) to binding index
4. **vkCmdBindDescriptorSets** — attaches set to pipeline layout at draw time

Spark's scene pass uses **set 0** with ten bindings — binding 1 is the material texture array. This chapter focuses on that binding and how it fits the wider layout shaders in Part III already declared.

### What is a texture? (beginner)

A **texture** is a 2D image stored on the GPU. Instead of painting every pixel of a brick wall by hand in code, you:

1. Load a photo of bricks into memory (`Texture2D` on CPU in Spark).
2. Assign **UV coordinates** to each mesh vertex (Part IV Chapter 4).
3. In the fragment shader, look up the photo at the interpolated UV: `texture(sampler, uv)`.

The GPU's texture units are optimized for this lookup — millions of pixels per frame sample textures with filtering (blending neighboring texels for smooth minification).

### Why Vulkan needs descriptors

The fragment shader declares:

```glsl
layout(set = 0, binding = 1) uniform sampler2DArray sceneTextures;
```

That is only a **declaration**. At runtime you must tell Vulkan:

- **Which VkImage** holds the texels
- **Which VkImageView** interprets the memory as a 2D array
- **Which VkSampler** defines filtering (linear vs nearest) and wrapping (repeat vs clamp)

A **descriptor set** packages these handles. `vkCmdBindDescriptorSets` attaches the package to the pipeline before drawing.

**Beginner analogy:** The shader is a function signature. The descriptor set is the argument you pass when calling the function.

### Why Spark uses a texture *array* instead of one texture per material

Naive approach: bind a different texture before every draw call. Problems:

- Descriptor binding is expensive when done thousands of times per frame.
- Vulkan has limits on descriptor updates per frame.

Spark's approach:

1. Upload **all** scene textures into layers of one `sampler2DArray` (up to `MaxSceneTextures` layers).
2. Bind the array **once** per frame (descriptor set 0, binding 1).
3. Per draw, pass a **layer index** in push constants (`textureLayer`, `normalMapLayer`, etc.).

```text
Draw chair seat:  push.textureLayer = 2, push.metallicRoughnessMapLayer = 5
Draw chair legs:  push.textureLayer = 3, push.metallicRoughnessMapLayer = 6
Same bound descriptor set for both draws — only push constants change.
```

This is the core of Spark's **material pipeline**: CPU assigns layers during submit; GPU samples by index during shade.

## 1b. Concept Deep Dive — Descriptor Binding and Layer Indices

### One array, many materials

Scene descriptor set 0 binding 1 is a combined `sampler2DArray`. The uploader exposes handles the writer binds once per frame:

```text
VulkanSceneTextureUploader::ArrayView()  →  VkDescriptorImageInfo.imageView
VulkanSceneTextureUploader::Sampler()    →  VkDescriptorImageInfo.sampler
layout = SHADER_READ_ONLY_OPTIMAL
```

Every opaque draw shares this binding; per-draw variation comes from push constants, not rebinding textures.

### Layer index flow from MaterialComponent

`SceneSubmitMaterial.cpp` resolves each map through `FindOrAddSceneTexture`:

```cpp
item.textureLayer = FindOrAddSceneTexture(params, mat->GetBaseColorTexture(), ...);
item.normalMapLayer = FindOrAddSceneTexture(params, mat->GetNormalTexture(), ...);
item.metallicRoughnessMapLayer = FindOrAddSceneTexture(params, mat->GetMetallicRoughnessTexture(), ...);
```

`-1` means "no texture" — the shader branches to flat tint or default ORM values. HDR emissive layers index `sceneHdrTextures` with a separate push field (`textureIsHdrLinear` on the draw item).

### GLSL sampling contract

```glsl
layout(set = 0, binding = 1) uniform sampler2DArray sceneTextures;
// push.textureLayer selects the array slice:
vec4 albedo = texture(sceneTextures, vec3(sparkMapUv(0), float(push.textureLayer)));
```

`sparkMapUv` applies atlas UV scale/offset from the draw item — the same layer index works for differently tiled materials when submit stored per-draw UV transforms.

### Why push constants instead of per-material descriptors

Vulkan limits descriptor updates per frame. Spark binds set 0 once, then changes only push constants (`textureLayer`, `normalMapLayer`, `metallicRoughnessMapLayer`, …) between draws — thousands of meshes, one texture array binding.

## 2. Core Concepts

### Combined image sampler

Vulkan merges image and sampler into one descriptor type:

```glsl
layout(set = 0, binding = 1) uniform sampler2DArray sceneTextures;
```

```cpp
bindings[1].binding = 1;
bindings[1].descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
bindings[1].descriptorCount = 1;
bindings[1].stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;
```

The fragment shader calls GLSL `texture(sampler, vec3(u, v, layer))` — third component selects array layer.

### Scene descriptor set layout (set 0 overview)

| Binding | Type | Stages | Resource |
|---------|------|--------|----------|
| 0 | Uniform buffer | VS + FS | Scene UBO |
| **1** | **Combined image sampler** | **FS** | **Texture 2D array** |
| 2 | Storage buffer | VS | Skin palette |
| 3 | Combined image sampler | FS | Directional shadow atlas |
| 4 | Storage buffer | FS | Cluster lights list |
| 5 | Storage buffer | FS | Cluster grid indices |
| 6 | Storage buffer | FS | Punctual shadow metadata |
| 7 | Combined image sampler | FS | Spot shadow map |
| 8 | Combined image sampler | FS | Point shadow map |
| 9 | Storage buffer | VS + FS | Sprite instances |

Binding 1 is the **material atlas** from `VulkanSceneTextureUploader`. Shadow and IBL textures use other bindings — do not conflate them in descriptor writes.

### Descriptor pool sizing

Pools pre-allocate descriptor counts per type × frames in flight:

```text
COMBINED_IMAGE_SAMPLER count ≥ framesInFlight × (bindings 1,3,7,8)
UNIFORM_BUFFER count ≥ framesInFlight × 1
STORAGE_BUFFER count ≥ framesInFlight × (skin + cluster + shadow SSBOs)
```

Undersized pools fail `vkAllocateDescriptorSets` at init — not at draw time.

### Writing binding 1

After upload, image layout must be `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`:

```cpp
VkDescriptorImageInfo imageInfo{};
imageInfo.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
imageInfo.imageView = sources.sceneTextureUploader.ArrayView();
imageInfo.sampler = sources.sceneTextureUploader.Sampler();

VkWriteDescriptorSet texWrite{};
texWrite.dstSet = descriptorSets[frameIndex];
texWrite.dstBinding = 1;
texWrite.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
texWrite.descriptorCount = 1;
texWrite.pImageInfo = &imageInfo;
vkUpdateDescriptorSets(device, 1, &texWrite, 0, nullptr);
```

Image view and sampler handles are **stable** across frames — only pixel content changes via upload. Descriptor write can happen once at init if the view never recreates; Spark refreshes each frame or on resize for consistency with other bindings.

### Sampler state

`CreateTextureSampler` typically sets:

| Parameter | Typical Spark value | Effect |
|-----------|---------------------|--------|
| `magFilter` | LINEAR | Smooth magnification |
| `minFilter` | LINEAR | Smooth minification (single mip) |
| `mipmapMode` | NEAREST | No mip chain |
| `addressModeU/V` | REPEAT | Tiling textures |
| `maxLod` | 0 | Clamp to mip 0 only |

Anisotropic filtering (`maxAnisotropy > 1`) is a one-line quality win on desktop when enabled in device features.

### Per-draw layer selection

Descriptor binds the **whole array**. Per-draw material selection is **not** a descriptor change — it is push constants:

```glsl
// Vertex: pass layer as flat int
vTextureLayer = push.textureLayer;

// Fragment:
if (vTextureLayer >= 0) {
    int layer = clamp(vTextureLayer, 0, 15);
    vec4 tex = texture(sceneTextures, vec3(vTexCoord, float(layer)));
}
```

Additional map types use separate push fields:

| Push field | Map type |
|------------|----------|
| `textureLayer` | Albedo/base color |
| `normalMapLayer` | Tangent-space normal |
| `metallicRoughnessMapLayer` | ORM packed |
| `emissiveMapLayer` | Emissive |

All index the **same** `sceneTextures` array — different CPU textures occupy different layers via `FindOrAddSceneTexture`.

### Bind at draw time

```cpp
vkCmdBindDescriptorSets(
    commandBuffer,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout,
    0,              // firstSet
    1,
    &descriptorSet, // set 0
    0, nullptr);    // dynamic offsets
```

Must occur after pipeline bind, before draws using `scene.frag`. Wrong set → validation error or wrong texture.

### How binding 1 connects to the material pipeline

Texture binding is not an isolated Vulkan exercise — it is the **GPU side** of Spark's material pipeline. The CPU path looks like this:

```text
MaterialComponent / MultiMaterialComponent
  → SceneSubmit::FindOrAddSceneTexture(SharedPtr<Texture2D>)
       returns layer index (0 … MaxSceneTextures-1)
  → SceneDrawItem.textureLayer / normalMapLayer / metallicRoughnessMapLayer
  → VulkanSceneTextureUploader::PrepareUploads (staging → GPU array layer)
  → VulkanSceneDescriptors writes binding 1 (LDR) and binding 11 (HDR float)
  → VulkanSceneOpaquePass push constants → scene.frag samples by layer
```

`FindOrAddSceneTexture` deduplicates by `SharedPtr` identity — ten draws sharing one brick albedo map reuse the same layer index. Different maps (albedo vs normal vs ORM) occupy **different layers in the same array**, selected by separate push constant fields.

glTF import decodes PNG/JPEG into CPU `Texture2D` pixels in `GltfMaterialLoader::TryLoadTexturesFromMaterial`. GPU upload happens later at frame record via `deferredUploadBatch.Record` — the ECS never holds `VkImage` handles.

For HDR emissive maps (`RGBA32F`), `ApplyAlbedoTexture` sets `item.textureIsHdrLinear` and the uploader routes pixels to `sceneHdrTextures` (binding 11). The fragment shader branches: LDR albedo uses `sparkSrgbToLinear` on `sceneTextures`; HDR layers skip sRGB decode.

When building your engine, implement **layer assignment on the CPU** during scene gather, not descriptor rebinding per draw. Vulkan rewards one array + push constant layers over hundreds of descriptor sets.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### One array, many material maps

**Concept:** Combined image sampler

**Decision:** Single binding 1 `sampler2DArray` for all PBR maps

**Why:** Minimize descriptor updates; layer index in push constants scales to hundreds of draws.

**In Spark:** `scene.frag` binding 1; multiple push layer fields.

### Layer in push, not dynamic descriptors

**Concept:** Per-draw layer selection

**Decision:** `textureLayer` in 192-byte push block

**Why:** Push constants cheaper than `vkUpdateDescriptorSets` per material.

**In Spark:** `ModelPushConstants::textureLayer`.

### SHADER_READ_ONLY layout in descriptor

**Concept:** Writing binding 1

**Decision:** `imageLayout` must match actual image layout at sample time

**Why:** Validation VUID if layout is TRANSFER_DST during draw.

**In Spark:** Uploader ends in `SHADER_READ_ONLY_OPTIMAL`.

### Ten-binding unified scene layout

**Concept:** Scene descriptor set layout

**Decision:** Lights, shadows, skin in same set 0

**Why:** One `vkCmdBindDescriptorSets` per mesh pass batch; forward renderer simplicity.

**In Spark:** `VulkanSceneDescriptors::CreateSetLayout`.

### Stable image view across frames

**Concept:** Sampler state

**Decision:** Recreate view only when `CreateResources` runs (init/resize)

**Why:** Descriptor writes reference long-lived handles; upload mutates image memory in place.

**In Spark:** `ArrayView()` accessor on uploader.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/render/scene/VulkanSceneDescriptors.hpp`
- `src/spark/render/scene/VulkanSceneDescriptors.cpp`
- `shaders/scene.frag` (binding 1)
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp` (bind + push)

### Step 1 — Verify layout creation

Log all ten bindings from `CreateSetLayout`. Confirm binding 1 is `COMBINED_IMAGE_SAMPLER` fragment-only.

### Step 2 — Allocate sets

`CreatePoolAndSets(device, framesInFlight, bindingSources)` — one set per swapchain frame slot.

### Step 3 — Write binding 1

After `CreateResources` on uploader, update descriptor with `ArrayView()` and `Sampler()`. Do not pass swapchain image view by mistake.

### Step 4 — Shader sample test

Temporarily in `scene.frag`:

```glsl
outColor = texture(sceneTextures, vec3(vTexCoord, 0.0));
```

Checkerboard on layer 0 should appear on meshes with `textureLayer = 0`.

### Step 5 — Multi-layer test

Register two textures → layers 0 and 1. Two draws with different `textureLayer` push values — different patterns, one descriptor set.

### Step 6 — RenderDoc verification

Pipeline State → Resource List → `sceneTextures`. Confirm image type 2D Array, correct dimensions 512×512×16.

## 5. How This Connects to the Rest of Your Engine

Chapter **2** produces the image and layout binding 1 requires. Chapter **4** supplies `vTexCoord` for the `vec3` UV argument. Chapter **5** processes returned texels linearly. Chapter **9** shares set 0 with light SSBOs — binding numbers must stay stable.

Descriptor design is **API contract** between C++ and every scene-stage shader.

## 6. Pitfalls and Debugging

- **Vulkan magenta / null** — descriptor not written, or image view null.
- **Wrong layout in `VkDescriptorImageInfo`** — must be shader read optimal at draw.
- **Layer out of range** — clamp in shader; CPU must not assign layer ≥ 16.
- **Sampler not created** — combined descriptor requires valid `VkSampler`.
- **Set 0 bound but binding 1 stale** — uploaded new image without updating descriptor (if view recreated).

## 7. Exercises

1. Document byte size of one descriptor set's worth of bindings (not uniform buffer contents).
2. Sample layer 0 vs layer 1 side-by-side in split screen using `gl_FragCoord.x`.
3. Change sampler to `NEAREST` — screenshot pixel crispness on checkerboard.
4. List all shader files that reference `set = 0, binding = 1` (should be scene/sprite family only).
5. What happens if `textureLayer = -1`? Trace `scene.frag` albedo fallback path.

## Summary

Scene textures bind to descriptor set 0 binding 1 as a combined `sampler2DArray` image sampler. The uploader provides stable `VkImageView` and `VkSampler` after upload completes in `SHADER_READ_ONLY_OPTIMAL` layout. Per-draw material layers come from push constants, not per-material descriptor sets — efficient for forward rendering at Spark's draw counts.

---

Next: [UV Mapping](04-uv-mapping.md).
