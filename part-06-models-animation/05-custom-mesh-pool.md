# Custom Mesh Pool

> **Build-Along Milestone:** Two glTF props draw from one pooled vertex/index buffer with correct subranges.

**You will learn:**

- Explain VulkanCustomMeshPool fingerprint and pack/upload flow.
- Connect SceneDrawItem.customMesh to GPU slice offsets.
- Use Prepare/Record split without per-mesh VkBuffer creation.

## 1. The Problem We Are Solving

Imported glTF props cannot each own a permanent device buffer if scenes churn at edit time. Spark packs every
`SceneMeshSlot::Custom` mesh into **shared dynamic buffers**, uploading when the set of unique meshes changes.

Build along: spawn two glTF trees on Custom slots and confirm one pooled vertex/index buffer in RenderDoc.

### Mesh pool for beginners

Each `VkBuffer` creation costs driver time. If every imported prop got its own vertex buffer, loading a city would allocate thousands of buffers.

**Pool = one big buffer, many sub-ranges:**

```text
VulkanCustomMeshPool
  vertexBuffer: [ meshA verts | meshB verts | meshC verts ... ]
  indexBuffer:  [ meshA idx  | meshB idx  | meshC idx  ... ]

Draw mesh B:
  vkCmdBindVertexBuffers(pool.vertexBuffer)
  vkCmdDrawIndexed(indexCount, ..., firstIndex = B.offset, ...)
```

**Fingerprint:** when scene's set of custom meshes changes (add/remove prop), pool **repacks** and re-uploads. Unchanged scenes skip upload.

**SceneMeshSlot::Custom** tells submit "vertices live in pool slice X" — not the built-in cube/sphere primitives.

**RenderDoc check:** two different props should show **same** `VkBuffer` handle, different `firstIndex` / vertex offsets.

## 1b. Concept Deep Dive — Interleaved Pack and Slice Tables

### CustomMeshGpuSlice — draw-time lookup

```cpp
// include/spark/render/scene/VulkanCustomMeshPool.hpp (concept)
struct CustomMeshGpuSlice {
    std::uint32_t firstIndex = 0;
    std::uint32_t indexCount = 0;
    std::uint32_t vertexOffset = 0;
};
HashMap<const Mesh*, CustomMeshGpuSlice> rigidSlices;
HashMap<const SkinnedMesh*, CustomMeshGpuSlice> skinnedSlices;
```

At record time, `VulkanSceneOpaquePass` resolves `d.customMesh.Get()` → slice → `vkCmdDrawIndexed(indexCount, 1, firstIndex, vertexOffset, 0)`.

### Skinned interleave — joints as float bit pattern

```cpp
// src/spark/render/gpu/VulkanGpuMeshInterleaved.cpp (lines 42–70)
void AppendSkinnedVertexToInterleaved(const SkinnedMesh::Vertex& v, Array<float>& out) {
    // position, normal, texCoord0/1, tangent, color ...
    out.PushBack(std::bit_cast<float>(v.joints[0]));
    out.PushBack(std::bit_cast<float>(v.joints[1]));
    out.PushBack(std::bit_cast<float>(v.joints[2]));
    out.PushBack(std::bit_cast<float>(v.joints[3]));
    out.PushBack(v.weights[0]);
    out.PushBack(v.weights[1]);
    out.PushBack(v.weights[2]);
    out.PushBack(v.weights[3]);
}
```

Rigid meshes write zero joint indices and weight 1.0 on bone 0 — same vertex stride, `push.useSkinning = 0` skips SSBO read.

### ClearKnownMeshes on demo teardown

When leaving a scene, call `customMeshPool.ClearKnownMeshes()` so destroyed CPU `Mesh*` pointers are not hashed in the next session's fingerprint — prevents use-after-free in slice maps.

### Frame timeline

```text
UpdateFromScene(scene)     → fingerprint changed? pack + memcpy staging
RecordUploads(cmd)         → vkCmdCopyBuffer + barriers
OpaquePass draw loop       → bind pool VB/IB, draw per slice
ReleaseRetiredBuffers()    → after fence wait, free old device buffers
```

## 2. Core Concepts

### Why a pool exists

Creating/destroying `VkBuffer` per imported prop each frame causes allocation churn. `VulkanCustomMeshPool` packs **all Custom-slot meshes** into shared vertex/index buffers, uploading only when the scene fingerprint changes.


### Fingerprint-driven rebuild

`ComputeFingerprint` hashes pointers and vertex/index counts of every Custom draw (rigid + skinned). Same scene composition → skip upload; new mesh pointer → repack and stage copy.


### Prepare vs record split

`UpdateFromScene` memcpy interleaved data into host-mapped staging. `RecordUploads` records `vkCmdCopyBuffer` + barriers on the command buffer — mirrors texture upload discipline from Part IV.


### Deferred buffer retirement

When capacity grows, old device buffers enter a **retired queue** freed after N frames in flight — no `vkDeviceWaitIdle` on rebuild.


### CustomMeshGpuSlice table

Each unique `Mesh*` or `SkinnedMesh*` maps to vertex/index subranges in the big buffers. Draw recording uses slices — not per-mesh VkBuffer handles.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanCustomMeshPool class

**Concept:** Why a pool exists

**Decision:** VulkanCustomMeshPool class

**Why:** One dynamic buffer pair serves arbitrary imported content; built-ins bypass the pool entirely.

**In Spark:** `include/spark/render/scene/VulkanCustomMeshPool.hpp`

### Fnv64 over Custom draw meshes

**Concept:** Fingerprint-driven rebuild

**Decision:** Fnv64 over Custom draw meshes

**Why:** Cheap early-out when static scene; repack when spawn/despawn changes unique mesh set.

**In Spark:** `ComputeFingerprint` — VulkanCustomMeshPool.cpp

### UpdateFromScene + RecordUploads

**Concept:** Prepare vs record split

**Decision:** UpdateFromScene + RecordUploads

**Why:** CPU pack can run before command buffer; GPU copy happens inside render pass setup.

**In Spark:** `VulkanCustomMeshPool` public methods

### rigidSlices / skinnedSlices HashMaps

**Concept:** CustomMeshGpuSlice table

**Decision:** rigidSlices / skinnedSlices HashMaps

**Why:** Submit sets customMesh or skinnedMesh pointer; pool resolves byte offsets at pack time.

**In Spark:** `PackSceneGeometry` — collects unique Mesh* and SkinnedMesh*

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/scene/VulkanCustomMeshPool.hpp`
- `src/spark/render/scene/VulkanCustomMeshPool.cpp`
- `src/spark/scene/SceneSubmit.cpp`

### Step 1 — Read pool responsibilities

*Dynamic rigid/skinned geometry — CPU pack, async GPU upload.*

```cpp
// include/spark/render/scene/VulkanCustomMeshPool.hpp (lines 17–50)
/**
 * Dynamic custom rigid/skinned mesh geometry: CPU pack, async GPU upload via command buffer copies,
 * deferred buffer retirement (no vkDeviceWaitIdle on rebuild).
 */
class VulkanCustomMeshPool {
public:
    struct Bindings {
        VkBuffer vertexBuffer = VK_NULL_HANDLE;
        VkBuffer indexBuffer = VK_NULL_HANDLE;
    };

    void CreateResources(VkPhysicalDevice physicalDevice, VkDevice device);
    void DestroyResources(VkDevice device);

    /** Call after the in-flight fence for the current frame slot has been waited on. */
    void ReleaseRetiredBuffers(std::uint64_t frameCounter);

    /**
     * Rebuild CPU geometry when the scene fingerprint changes; memcpy into host staging.
     * Device buffer (re)allocation retires the previous set without blocking the GPU.
     */
    void UpdateFromScene(
            const SceneRenderParams& scene,
            std::uint64_t frameCounter,
            std::uint32_t maxFramesInFlight);

    /** Records staging→device copies + vertex/index barriers before custom mesh draws. */
    void RecordUploads(VkCommandBuffer commandBuffer);

    [[nodiscard]] Bindings GetBindings() const noexcept;
    void FillCustomDrawPacked(
            const SceneRenderParams& scene,
            Array<CustomMeshGpuSlice>& outOpaquePacked,
            Array<CustomMeshGpuSlice>& outTransparentPacked) const;
```

### Step 2 — Trace submit → customMesh pointer

*MeshComponent Custom slot copies SharedPtr into SceneDrawItem.*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 92–97)
        SceneDrawItem item{};
        item.model = worldM;
        item.mesh = mc.GetSlot();
        if (mc.GetSlot() == SceneMeshSlot::Custom) {
            item.customMesh = mc.GetMesh();
        }
```

### Step 3 — Inspect fingerprint hash

*Mix mesh pointers + vertex/index counts from all Custom draws.*

```cpp
// src/spark/render/scene/VulkanCustomMeshPool.cpp (lines 193–224)
std::uint64_t VulkanCustomMeshPool::ComputeFingerprint(const SceneRenderParams& scene) const {
    std::uint64_t h = Fnv64Begin();
    auto mixDraw = [&](const SceneDrawItem& d) {
        if (d.mesh != SceneMeshSlot::Custom) {
            return;
        }
        if (d.skinnedMesh) {
            const SkinnedMesh* p = d.skinnedMesh.Get();
            Fnv64Mix(h, reinterpret_cast<std::uintptr_t>(p));
            if (p != nullptr) {
                Fnv64Mix(h, static_cast<std::uint64_t>(p->GetVertices().GetSize()));
                Fnv64Mix(h, static_cast<std::uint64_t>(p->GetIndices().GetSize()));
            }
            return;
        }
        if (!d.customMesh) {
            return;
        }
        const Mesh* p = d.customMesh.Get();
        Fnv64Mix(h, reinterpret_cast<std::uintptr_t>(p));
        if (p != nullptr) {
            Fnv64Mix(h, static_cast<std::uint64_t>(p->GetVertices().GetSize()));
            Fnv64Mix(h, static_cast<std::uint64_t>(p->GetIndices().GetSize()));
        }
    };
    for (std::size_t i = 0; i < scene.draws.GetSize(); ++i) {
        mixDraw(scene.draws[i]);
    }
    for (std::size_t i = 0; i < scene.transparentDraws.GetSize(); ++i) {
        mixDraw(scene.transparentDraws[i]);
    }
    return h;
```

### Step 4 — Follow UpdateFromScene pack + staging

*PackSceneGeometry → memcpy staging → EnsureDeviceCapacity.*

```cpp
// src/spark/render/scene/VulkanCustomMeshPool.cpp (lines 368–400)
    const std::uint64_t fp = ComputeFingerprint(scene);
    if (fp == lastFingerprint && active.vertexBuffer != VK_NULL_HANDLE && !uploadPending) {
        return;
    }
    lastFingerprint = fp;

    Array<float> interleaved;
    Array<std::uint32_t> meshIndices;
    PackSceneGeometry(scene, interleaved, meshIndices);

    const VkDeviceSize vbSize = sizeof(float) * static_cast<std::size_t>(interleaved.GetSize());
    const VkDeviceSize ibSize = sizeof(std::uint32_t) * static_cast<std::size_t>(meshIndices.GetSize());
    if (vbSize == 0 || ibSize == 0) {
        if (active.vertexBuffer != VK_NULL_HANDLE || active.indexBuffer != VK_NULL_HANDLE) {
            QueueRetire(MoveTemp(active), frameCounter + static_cast<std::uint64_t>(maxFramesInFlight));
        }
        rigidSlices.Clear();
        skinnedSlices.Clear();
        lastFingerprint = 0;
        uploadPending = false;
        pendingVertexBytes = 0;
        pendingIndexBytes = 0;
        return;
    }

    EnsureStagingCapacity(vbSize, ibSize);
    std::memcpy(stagingVertexMapped, interleaved.GetData(), static_cast<std::size_t>(vbSize));
    std::memcpy(stagingIndexMapped, meshIndices.GetData(), static_cast<std::size_t>(ibSize));
    pendingVertexBytes = vbSize;
    pendingIndexBytes = ibSize;

    EnsureDeviceCapacity(vbSize, ibSize, frameCounter, maxFramesInFlight);
    uploadPending = true;
```

### Step 5 — Record GPU copies

*vkCmdCopyBuffer + vertex/index input barriers.*

```cpp
// src/spark/render/scene/VulkanCustomMeshPool.cpp (lines 403–450)
void VulkanCustomMeshPool::RecordUploads(const VkCommandBuffer commandBuffer) {
    if (!uploadPending || device == VK_NULL_HANDLE || commandBuffer == VK_NULL_HANDLE) {
        return;
    }
    if (active.vertexBuffer == VK_NULL_HANDLE || active.indexBuffer == VK_NULL_HANDLE ||
        stagingVertex == VK_NULL_HANDLE || stagingIndex == VK_NULL_HANDLE) {
        uploadPending = false;
        return;
    }

    VkBufferCopy vtxCopy{};
    vtxCopy.size = pendingVertexBytes;
    vkCmdCopyBuffer(commandBuffer, stagingVertex, active.vertexBuffer, 1, &vtxCopy);

    VkBufferCopy idxCopy{};
    idxCopy.size = pendingIndexBytes;
    vkCmdCopyBuffer(commandBuffer, stagingIndex, active.indexBuffer, 1, &idxCopy);

    VkBufferMemoryBarrier barriers[2]{};
    barriers[0].sType = VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER;
    barriers[0].srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barriers[0].dstAccessMask = VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT;
    barriers[0].srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barriers[0].dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barriers[0].buffer = active.vertexBuffer;
    barriers[0].offset = 0;
    barriers[0].size = pendingVertexBytes;

    barriers[1].sType = VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER;
    barriers[1].srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barriers[1].dstAccessMask = VK_ACCESS_INDEX_READ_BIT;
    barriers[1].srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barriers[1].dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barriers[1].buffer = active.indexBuffer;
    barriers[1].offset = 0;
    barriers[1].size = pendingIndexBytes;

    vkCmdPipelineBarrier(
            commandBuffer,
            VK_PIPELINE_STAGE_TRANSFER_BIT,
            VK_PIPELINE_STAGE_VERTEX_INPUT_BIT,
            0,
            0,
            nullptr,
            2,
            barriers,
            0,
            nullptr);
```

### Step 6 — Spawn two different glTF props

Place Tree_1 and Tree_3 from Spark assets — RenderDoc should show one vertex buffer with two index ranges via CustomMeshGpuSlice offsets.

### Step 7 — Despawn and verify retire path

Remove one prop; fingerprint changes → pool repacks without blocking GPU. Watch retired buffer queue free after maxFramesInFlight.

### Step 8 — Milestone check — two glTF meshes, one upload pair

Two distinct Custom meshes render correctly; toggling visibility triggers at most one rebuild per unique set change.

## 5. How This Connects to the Rest of Your Engine

**Loading glTF** produces CPU meshes referenced by draw items. **GPU Skinning** uses the same pool with
skinned interleaved layout and joint attributes. Built-in UnitCube/GroundPlane skip the pool entirely.

## 6. Pitfalls and Debugging

- **Slot reuse without upload** — stale fingerprint skips copy; ensure draw item mesh pointer updates invalidate hash.
- **Index count mismatch** — slice past buffer end → validation error; verify PackSceneGeometry counts.
- **Creating VkBuffer per prop** — defeats the pool; always route Custom through the pool in Spark's path.
- **Forgetting barriers** — TRANSFER_DST → VERTEX_INPUT required before draw.

## 7. Exercises

1. Log fingerprint before/after spawning a second Custom mesh.
2. Capture RenderDoc; note single VkBuffer bound for two glTF draws.
3. Remove all Custom draws; confirm pool clears active buffers.

## Summary

VulkanCustomMeshPool packs Custom-slot CPU meshes into shared GPU buffers keyed by scene fingerprint.

---

Next: [Skeleton and Bones](06-skeleton.md).
