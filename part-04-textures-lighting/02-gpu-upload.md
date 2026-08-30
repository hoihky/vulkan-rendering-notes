# GPU Texture Upload

> **Build-Along Milestone:** `VulkanSceneTextureUploader` owns a 16×512² `R8G8B8A8_UNORM` array image; dirty CPU layers copy through staging each frame.

**You will learn:**

- Create a `VkImage` 2D array with transfer and sample usage.
- Stage CPU pixels through a host-visible buffer into device-local memory.
- Transition image layouts with pipeline barriers.
- Invalidate uploads using content fingerprints without re-uploading unchanged layers.

## 1. The Problem We Are Solving

Chapter 1 left pixels on the CPU. The GPU cannot sample host RAM directly — you need:

1. A **device-local** `VkImage` (fast random access for texture units)
2. A **staging buffer** (host-visible, for `vkCmdCopyBufferToImage`)
3. **Layout transitions** (UNDEFINED → TRANSFER_DST → SHADER_READ_ONLY)
4. A **sampler** + **image view** for shader binding

Spark wraps this in `VulkanSceneTextureUploader` — one 2D array atlas, 16 layers, 512×512 texels each, single mip level. This chapter walks creation, per-frame dirty tracking, and command buffer recording.

### Getting pixels onto the GPU (beginner)

The CPU has `Texture2D` bytes in RAM. The GPU texture unit reads from **VRAM** on the graphics card. You cannot `memcpy` directly — Vulkan requires an explicit upload path:

```text
1. Create VkImage (GPU memory, device-local)
2. Create staging VkBuffer (CPU-visible)
3. memcpy CPU pixels → staging buffer
4. vkCmdCopyBufferToImage (recorded in command buffer)
5. Image layout barrier: TRANSFER_DST → SHADER_READ_ONLY
6. Create VkImageView + VkSampler
7. vkUpdateDescriptorSets (binding 1)
```

**Image layouts** confuse beginners — they are Vulkan's way of saying "how will this memory be accessed next?"

| Layout | Meaning |
|--------|---------|
| `UNDEFINED` | Don't care about old contents (first upload) |
| `TRANSFER_DST_OPTIMAL` | Copy destination |
| `SHADER_READ_ONLY_OPTIMAL` | Fragment shader sampling |

A **barrier** (`vkCmdPipelineBarrier`) tells the driver to finish copies before shaders sample — skipping it causes validation errors or garbage texels.

Spark batches uploads in `deferredUploadBatch.Record` **before** scene draws each frame — only **dirty** layers (content fingerprint changed) copy again.

## 1b. Concept Deep Dive — VulkanSceneTextureUploader Lifecycle

### Fixed atlas dimensions

Spark packs all scene materials into one device-local 2D array:

```cpp
// include/spark/render/scene/VulkanSceneTextureUploader.hpp
class VulkanSceneTextureUploader {
public:
    static constexpr std::uint32_t kLayerSize = 1024;
    static constexpr std::uint32_t kLayerCount = SceneRenderParams::MaxSceneTextures;
    void PrepareUploads(const SceneRenderParams& scene, bool sceneParamsValid);
    void RecordUploads(VkCommandBuffer commandBuffer);
    [[nodiscard]] VkImageView ArrayView() const noexcept;
    [[nodiscard]] VkSampler Sampler() const noexcept;
};
```

Each layer is resampled to `kLayerSize` on the CPU during `PrepareUploads`, then copied through a mapped staging buffer in `RecordUploads`. Layout transitions (UNDEFINED → TRANSFER_DST → SHADER_READ_ONLY) happen inside the recorded command buffer.

### Two-phase upload avoids GPU stalls

| Phase | When | Work |
|-------|------|------|
| `PrepareUploads` | Before command buffer record | CPU resample, fingerprint compare, staging `memcpy` |
| `RecordUploads` | Inside command buffer | `vkCmdCopyBufferToImage`, mip blits, barriers |

`NeedsUpload` returns false when every layer fingerprint matches cache — the renderer skips both phases for that frame.

### Per-layer dirty cache

Private arrays track what was last uploaded:

```cpp
const Texture2D* lastUploadedTextures[kLayerCount]{};
std::uint64_t lastFingerprints[kLayerCount]{};
bool pendingLayerDirty[kLayerCount]{};
```

A layer re-uploads when the `Texture2D*` pointer changes **or** `GetContentFingerprint()` differs. Nearest-filtered pixel art sets `SetSceneUploadNearest(true)` — uploader skips mip generation for that layer.

### Dual samplers

`Sampler()` is trilinear for PBR materials; `SpriteSampler()` is nearest for crisp 2D/UI layers bound through the same array view. Descriptor binding 1 uses the PBR sampler; sprite passes may override per draw.

## 2. Core Concepts

### Why a 2D texture array

Instead of 16 separate `VkImage` objects and 16 descriptor bindings, Spark packs materials into **one array image**:

```text
sceneTextures CPU[0..15]  →  GPU array layers 0..15
                              same width, height, format
```

Benefits:

- One combined image sampler in the fragment shader (`sampler2DArray`)
- One descriptor write per frame
- Per-draw `textureLayer` push constant selects layer

Tradeoff: all layers share 512² resolution (CPU nearest resample).

### Image creation parameters

```cpp
constexpr VkFormat kTexFormat = VK_FORMAT_R8G8B8A8_UNORM;
CreateImage2DArray(
    physicalDevice, device,
    kLayerSize, kLayerSize,  // 512 x 512
    kLayerCount,             // 16 layers
    kTexFormat,
    VK_IMAGE_TILING_OPTIMAL,
    VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT,
    VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
    arrayImage, arrayMemory);
```

| Parameter | Value | Why |
|-----------|-------|-----|
| Format | `R8G8B8A8_UNORM` | Matches CPU RGBA8 |
| Tiling | OPTIMAL | GPU texture cache |
| Usage | TRANSFER_DST + SAMPLED | Upload then shader read |
| Memory | DEVICE_LOCAL | Not mappable from CPU |
| Mip levels | **1** | No GPU mip chain (see below) |

### Staging buffer layout

One staging buffer holds **all 16 layers** contiguously:

```text
layerPitch = 512 * 512 * 4 = 1,048,576 bytes
stagingSize = layerPitch * 16 = 16 MB
```

`PrepareUploads` memcpy each resampled layer to `stagingMapped + i * layerPitch`. `RecordUploads` issues one `VkBufferImageCopy` per layer with matching `bufferOffset`.

### Layout transitions (barriers)

Vulkan images have a **layout** state machine. Sampling requires `SHADER_READ_ONLY_OPTIMAL`; copies require `TRANSFER_DST_OPTIMAL`.

Spark uses `VulkanRendererGpu::SceneTexBarrier` — a helper emitting `vkCmdPipelineBarrier` with correct `VkImageMemoryBarrier` for all array layers:

```text
Initial create:  UNDEFINED → TRANSFER_DST  (first white fill)
Per-frame upload: SHADER_READ_ONLY → TRANSFER_DST → SHADER_READ_ONLY
```

Wrong layout → validation error or corrupted sampling.

### No GPU mipmaps

Spark does **not** call `vkCmdBlitImage` or generate mips. Sampler uses `minLod = maxLod = 0`. Distance shimmering is mitigated by:

- 512² atlas resolution (adequate for mid-distance props)
- Linear filtering on the single mip
- Future: CPU mip chain or anisotropic filtering (production topic)

Document this explicitly so readers do not search for mip code that does not exist.

### Fingerprint-based dirty tracking

```cpp
bool NeedsUpload(const SceneRenderParams& scene, bool sceneParamsValid) {
    if (lastUploadedCount != currentCount) return true;
    for each layer i:
        if (scene.sceneTextures[i]->GetContentFingerprint() != lastFingerprints[i])
            return true;
    return false;
}
```

`PrepareUploads` resamples **all 16 layers** when any layer is dirty (simple, not optimal — acceptable at 16×512²). `RecordUploads` copies all layers when `uploadPending`.

Optimization path for later: per-layer dirty flags and partial `vkCmdCopyBufferToImage`.

### Frame integration

```text
CPU (before record):
  NeedsUpload? → PrepareUploads (resample + staging memcpy)

GPU (command buffer):
  RecordUploads (barriers + copies)  // before scene draws

Draw:
  Fragment shader samples SHADER_READ_ONLY layout image
```

`VulkanDeferredUploadBatch` calls `PrepareUploads` early, `RecordUploads` inside the frame's command buffer before render pass.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Fixed 512² atlas layers

**Concept:** 2D texture array

**Decision:** `kLayerSize = 512`, `kLayerCount = 16`

**Why:** Predictable VRAM (16 MB color + overhead); simple shader `clamp(layer, 0, 15)`; matches mobile budget teaching.

**In Spark:** `VulkanSceneTextureUploader.hpp` constants.

### UNORM not SRGB format

**Concept:** Image creation parameters

**Decision:** `VK_FORMAT_R8G8B8A8_UNORM` storage; sRGB decode in shader

**Why:** Same format for albedo, normal, ORM data textures; only albedo needs gamma decode (Chapter 5).

**In Spark:** Comment in uploader header; `sparkSrgbToLinear` in fragment shader.

### Single staging buffer, all layers

**Concept:** Staging buffer layout

**Decision:** One 16 MB mapped buffer for entire atlas

**Why:** One allocation; sequential copies in one `RecordUploads` pass.

**In Spark:** `CreateResources` in `VulkanSceneTextureUploader.cpp`.

### Nearest CPU resample

**Concept:** No GPU mipmaps

**Decision:** `Texture2D::ResampleNearest(512, 512)` in `PrepareUploads`

**Why:** Fast; sharp pixels for stylized art; bilinear available if switched.

**In Spark:** `PrepareUploads` loop.

### White initial fill

**Concept:** Layout transitions

**Decision:** memset staging 255, upload all layers at `CreateResources`

**Why:** Unassigned layers sample white (neutral multiply) instead of black or garbage.

**In Spark:** `CreateResources` one-time command buffer.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/render/scene/VulkanSceneTextureUploader.hpp`
- `src/spark/render/scene/VulkanSceneTextureUploader.cpp`
- `include/spark/render/gpu/VulkanGpuBufferImage.hpp` (helpers)
- `src/spark/render/scene/VulkanDeferredUploadBatch.cpp`

### Step 1 — Create resources at init

After `VkDevice` is ready:

```cpp
sceneTextureUploader.CreateResources(physicalDevice, device, commandPool, graphicsQueue);
```

Verify with RenderDoc: one `VkImage` type 2D array, 512×512×16, format 37 (`R8G8B8A8_UNORM`).

### Step 2 — Trace PrepareUploads

Set a breakpoint when `NeedsUpload` returns true. Watch:

1. Each CPU texture resampled to 512²
2. `memcpy` into `stagingMapped`
3. `pendingFingerprints` cached

### Step 3 — Trace RecordUploads

In a capture, find barriers bracketing `vkCmdCopyBufferToImage` × 16. Confirm final layout `SHADER_READ_ONLY_OPTIMAL`.

### Step 4 — Edit texture at runtime

Modify pixels via `SetPixels`, run frame. Fingerprint changes → upload runs. Second frame with no edit → `NeedsUpload` false, `RecordUploads` no-op.

### Step 5 — Layer count change

Add a second texture to `sceneTextures`. `lastUploadedCount` mismatch forces upload even if fingerprints match — array slot assignment changed.

### Step 6 — Destroy on teardown

`DestroyResources` frees image, view, sampler, staging. Order: wait idle → destroy view → image → memory.

## 5. How This Connects to the Rest of Your Engine

Chapter **3** exposes `ArrayView()` and `Sampler()` to descriptor writes. Chapter **4** uses UVs to address texels inside a layer. Chapter **5** applies gamma decode after `texture()` returns UNORM values.

Upload is **asynchronous** with respect to CPU — staging memcpy happens before submit; GPU copy runs in the queue timeline you record.

## 6. Pitfalls and Debugging

- **Black textures** — image still `UNDEFINED` layout; missing final barrier to `SHADER_READ_ONLY`.
- **Validation: image layout mismatch** — drew before `RecordUploads` completed.
- **Garbled colors** — row stride wrong in `VkBufferImageCopy` (must match tightly packed RGBA8).
- **No upload after edit** — fingerprint not refreshed; or `sceneParamsValid == false`.
- **16 MB staging every frame** — all layers re-uploaded on any dirty layer; profile if needed.

## 7. Exercises

1. Calculate exact VRAM for the array image (512×512×16×4 bytes) plus staging (host).
2. Change one layer's pixels — confirm all 16 layers copy (current Spark behavior).
3. Add logging in `NeedsUpload` listing which layer index changed fingerprint.
4. Sketch barrier `srcStageMask`/`dstStageMask` for `SceneTexBarrier` (transfer vs fragment shader).
5. Compare visual quality: nearest vs bilinear resample to 512² on a high-res photo.

## Summary

`VulkanSceneTextureUploader` creates a device-local 16-layer RGBA8 UNORM array, fills it through a host-visible staging buffer with layout barriers, and re-uploads when CPU fingerprints or layer count change. There is no GPU mip chain — one 512² mip per layer. Master `PrepareUploads` (CPU) vs `RecordUploads` (GPU) split before binding textures to descriptors.

---

Next: [Binding Textures](03-binding-textures.md).
