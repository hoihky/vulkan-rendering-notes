# GPU Memory Strategy

> **Build-Along Milestone:** Clean shutdown with zero validation memory leaks after extended play and resize.

**You will learn:**

- Select memory types via `FindMemoryType` for device-local vs staging.
- Place uploads in `DeferredUploadBatch` before the frame graph.
- Size frames-in-flight rings and budget shadow/HDR VRAM.

## 1. The Problem We Are Solving

Vulkan makes memory **visible**. Wrong flags mean slow UMA paths or allocation failure at load time. **Production strategy** centralizes buffer/image creation, batches uploads once per frame, rings per-flight resources, and budgets VRAM for the shadow/HDR/post stacks Spark built in Parts IX–X.

You will trace Spark helpers — not adopt a proprietary allocator yet — and produce a numeric budget for your target hardware.

Milestone: play ThreeDDemo for 60 s with two resizes, exit with validation object tracking enabled — zero leaked `VkDeviceMemory`.

## 2. Core Concepts

### Explicit memory types

Vulkan exposes memory **heaps** and **types** via `vkGetPhysicalDeviceMemoryProperties`. You choose types matching required **property flags**:

| Need | Typical flags | Example |
|------|---------------|---------|
| GPU-only vertex/index | `DEVICE_LOCAL` | Mesh VB |
| CPU write each frame | `HOST_VISIBLE`, often `HOST_COHERENT` | Uniform ring |
| Staging upload | `HOST_VISIBLE` → copy → `DEVICE_LOCAL` | Texture upload |

Never assume type index 0 works — always `FindMemoryType(typeFilter, properties)` as in Spark's `VulkanGpuBufferImage`.

### Staging upload pattern

Static geometry and textures follow:

```text
Create HOST_VISIBLE staging buffer → memcpy → cmdCopyBuffer/Image → DEVICE_LOCAL resource
```

Spark batches per-frame texture and UI uploads in **DeferredUploadBatch** at the **start** of scene command buffer recording — before the shadow pass consumes updated descriptors.

One-time mesh upload uses `RunOneTimeCommands` helper in `VulkanRendererGpu.hpp` — submit, wait idle, free temp command buffer.

### Frames in flight rings

With `kMaxFramesInFlight = 2`, CPU frame *N* may write uniform slice while GPU reads slice *N-1*. Spark duplicates:

- Scene uniform buffers per frame index
- Skin SSBO per frame index
- Sprite instance SSBO
- HDR depth/color flight targets

**Never** map and overwrite the uniform buffer the GPU is reading — `frameSync.WaitForCurrentFrameFence` gates CPU writes.

```text
frameIndex:     [0]     [1]     [0]     [1]
CPU writes:      W       .       W       .
GPU reads:       .       R       .       R
```

Every per-frame GPU write must use the same `frameIndex` as the descriptor set bound during record.

### VRAM budgeting

Production teams sum major consumers at target resolution (1080p baseline):

```text
CSM shadow atlas + punctual shadow maps
RGBA16F HDR color + depth
SSAO scratch + noise
Scene texture array layers (16 max)
Mesh + skinning GPU pools
```

Spark uses explicit `vkAllocateMemory` per resource for teaching clarity — shipping engines may adopt VMA or bucket allocators, but **budget math** stays the same. Tie shadow resolution and SSAO to quality presets (Chapter 5).

| Resource | Formula | 1080p High (approx.) |
|----------|---------|----------------------|
| CSM atlas | cascades × W × H × 4 (D32) | 4 × 2048² × 4 ≈ 64 MB |
| HDR color | W × H × 8 (RGBA16F) | ~16 MB |
| HDR depth | W × H × 4 (D32) | ~8 MB |
| SSAO buffer | W × H × 4 | ~8 MB |

Document sum and compare to heap size × 0.8 safety margin.

### Retired buffer pools

Dynamic custom meshes and font atlases defer destruction until the GPU finishes — `customMeshPool.ReleaseRetiredBuffers(submittedFrameCounter)` in DrawFrame. Same pattern as Part VIII particle retire queues.

Leaks show at exit via validation object tracking — zero `VkDeviceMemory` remaining after `Shutdown`.

### DeferredUploadBatch barrier ordering

Upload batch is the **first GPU work** in the scene command buffer because shadow and scene passes sample the scene texture array:

```text
DrawFrame
  Prepare(deferredUploadBatch)     ← CPU fills staging, marks dirty layers
  RecordSceneCommandBuffer
    deferredUploadBatch.Record     ← barriers + copies FIRST
    RecordShadowMapPass            ← may sample updated albedo
    HDR scene pass
```

Missing final layout barrier → validation `VUID-*-imageLayout*` or garbled albedo until next frame.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanGpuBufferImage helpers

**Concept:** Explicit memory types

**Decision:** Centralize `CreateBuffer` / `CreateImage` in one module.

**Why:** Demos call wrappers — one place to fix memory selection bugs across the engine.

**In Spark:** `VulkanGpuBufferImage.cpp` and `VulkanRendererGpu.hpp`.

### DeferredUploadBatch at record start

**Concept:** Staging upload pattern

**Decision:** `deferredUploadBatch.Record` before shadow pass.

**Why:** Groups barriers and copies before shadow/scene passes need fresh textures.

**In Spark:** `VulkanRenderer::RecordSceneCommandBuffer` line 211.

### VulkanFrameSync::kMaxFramesInFlight

**Concept:** Frames in flight rings

**Decision:** `kMaxFramesInFlight = 2` drives uniform and descriptor indexing.

**Why:** Descriptor sets and uniform buffers indexed by `frameIndex` — matches fence ring.

**In Spark:** `VulkanFrameSync.hpp`.

### customMeshPool + font atlas retire

**Concept:** Retired buffer pools

**Decision:** Release retired buffers before acquire each frame.

**Why:** Destroying VB while a command buffer references it → validation error or GPU fault.

**In Spark:** `VulkanRenderer::DrawFrame` retire calls.

### VRAM spreadsheet per quality preset

**Concept:** VRAM budgeting

**Decision:** Document Low/Medium/High caps in `docs/vram-budget.md`.

**Why:** Numeric budget prevents OOM on min-spec — ties to Chapter 5 presets.

**In Spark:** Quality preset shadow map sizes.

### HOST_COHERENT for uniform rings

**Concept:** Explicit memory types

**Decision:** `HOST_VISIBLE | HOST_COHERENT` for per-frame uniform maps.

**Why:** No `vkFlushMappedMemoryRanges` on coherent heaps; re-audit on non-coherent mobile heaps when porting.

**In Spark:** `FindMemoryType` call sites in `VulkanGpuBufferImage`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/gpu/VulkanGpuBufferImage.cpp`
- `include/spark/render/core/VulkanRendererGpu.hpp`
- `src/spark/render/core/VulkanRenderer.cpp`
- `include/spark/render/core/VulkanFrameSync.hpp`

### Step 1 — Read FindMemoryType and CreateBuffer

Understand the type filter loop — mirror when adding your own particle SSBO.

*FindMemoryType and CreateBuffer*

```cpp
// src/spark/render/gpu/VulkanGpuBufferImage.cpp (lines 9–51)
std::uint32_t VulkanGpuBufferImage::FindMemoryType(
        VkPhysicalDevice physicalDevice,
        const std::uint32_t typeFilter,
        const VkMemoryPropertyFlags properties) {
    VkPhysicalDeviceMemoryProperties memProperties{};
    vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProperties);
    for (std::uint32_t i = 0; i < memProperties.memoryTypeCount; ++i) {
        if ((typeFilter & (1U << i)) != 0U &&
            (memProperties.memoryTypes[i].propertyFlags & properties) == properties) {
            return i;
        }
    }
    throw std::runtime_error("FindMemoryType failed");
}

void VulkanGpuBufferImage::CreateBuffer(
        VkPhysicalDevice physicalDevice,
        VkDevice vkDevice,
        const VkDeviceSize size,
        const VkBufferUsageFlags usage,
        const VkMemoryPropertyFlags properties,
        VkBuffer& buffer,
        VkDeviceMemory& bufferMemory) {
    // ... vkCreateBuffer, vkAllocateMemory, vkBindBufferMemory
}
```

### Step 2 — Trace deferredUploadBatch.Record

Upload batch is first work in scene command buffer — note ordering before shadow pass.

*Deferred upload batch before frame graph*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 201–213)
void VulkanRenderer::RecordSceneCommandBuffer(
        VkCommandBuffer commandBuffer,
        std::uint32_t imageIndex,
        std::uint32_t frameIndex) {
    // ... vkBeginCommandBuffer ...

    deferredUploadBatch.Record(commandBuffer, device(), sceneTextureUploader, screenUi);
    customMeshPool.RecordUploads(commandBuffer);
    RecordShadowMapPass(commandBuffer, frameIndex);
```

### Step 3 — Grep kMaxFramesInFlight consumers

List uniform, skin SSBO, HDR flight, and descriptor set indexing — draw the ring diagram for your repo wiki.

### Step 4 — Budget shadow + HDR at 1080p

Calculate approximate MB:

```text
4× CSM 2048² D32 + punctual atlas + RGBA16F color + D32 depth + SSAO
```

Compare Low preset (1024 CSM, no SSAO) vs High — justify defaults.

### Step 5 — Verify Prepare vs Record split

Read `deferredUploadBatch.Prepare` in DrawFrame — CPU-side staging fills before command buffer records copies. Async texture loading extends this boundary.

### Step 6 — Shutdown leak audit

Run with validation object tracking enabled. Play 60 s, resize twice, exit — confirm zero leaked memory objects in log.

### Step 7 — RunOneTimeCommands pattern

For one-off uploads in your fork, use helper from `VulkanRendererGpu.hpp` — do not leave permanent command buffers for staging copies.

### Step 8 — Document ring buffer rule

Add comment above your uniform write:

```cpp
// Only write uniformBuffersMapped[frameIndex] after WaitForCurrentFrameFence
```

Code review checklist item for all new per-frame GPU writes.

## 5. How This Connects to the Rest of Your Engine

Part **VIII** particle pools extend the retire pattern. Part **IX** shadow atlases dominate VRAM. Part **XII** terrain tiles add streaming pressure. Appendix **lifecycle** lists destroy order for memory objects at shutdown.

Chapter 1 swapchain recreate rebuilds HDR flight targets at new extent — same memory objects, new allocation sizes.

## 6. Pitfalls and Debugging

- **Single uniform buffer without ring** — race → flickering transforms.
- **HOST_VISIBLE for static VB** — works on some iGPUs, wastes bandwidth on dGPU.
- **Forgetting image layout transitions** — copy to undefined layout → validation error.
- **No retire queue for dynamic VB** — crash on mesh reload during play.
- **Spreadsheet using wrong format size** — D32 = 4 bytes per texel, RGBA16F = 8 bytes per pixel.
- **Retire queue skipped on font reload** — same validation as custom mesh pool.

## 7. Exercises

1. Fill the VRAM spreadsheet at 1080p and 1440p for High preset.
2. Draw the ring buffer fence diagram for your fork; grep all `frameIndex` consumers.
3. Intentionally skip `ReleaseRetiredBuffers` once — save the validation VUID.
4. Print physical device memory heaps at startup on dev machine and min-spec laptop.
5. Profile `deferredUploadBatch.Prepare` vs `Record` during a texture-heavy scene load.

## Summary

GPU memory strategy uses `VulkanGpuBufferImage` helpers, `DeferredUploadBatch` uploads, and frames-in-flight duplication. Spark retires dynamic GPU buffers after safe frame completion.

---

Next: [Async Assets](03-async-assets.md).
