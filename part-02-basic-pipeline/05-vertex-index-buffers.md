# Vertex and Index Buffers

> **Build-Along Milestone:** You understand staging-to-device upload, interleaved vertex layout, and indexed drawing with `vkCmdBindVertexBuffers` / `vkCmdBindIndexBuffer`.

**You will learn:**

- Pack vertex attributes into a GPU buffer with correct stride and usage flags.
- Upload CPU mesh data through a staging buffer.
- Share one vertex buffer across multiple meshes via index offsets.

## 1. The Problem We Are Solving

Shaders expect vertex attributes at `location` 0..N. Vulkan delivers them from **device-local memory** bound as vertex buffers. CPU-side `Mesh` vertices must be:

1. Converted to the **interleaved layout** the pipeline expects
2. Copied to GPU memory with `TRANSFER_DST` usage
3. Referenced by draws with correct **index ranges**

Spark packs cube, ground plane, and sprite quad into **one** static VB/IB pair in `CreateSceneGeometry` — efficient for small built-in geometry and the pattern you will reuse for imported meshes in Part VI.

## 1b. Concept Deep Dive — CreateSceneGeometry Packed Buffer

### One VB/IB, three submeshes — subrange drawing

Spark uploads cube, ground plane, and sprite quad into a **single** device-local vertex buffer and index buffer. Draw items select geometry by index range, not by rebinding buffers:

```text
Interleaved vertex buffer (device-local):
  [cube verts 0..7][plane verts 8..11][sprite verts 12..15]

Index buffer:
  [cube indices 0..35][plane indices 36..41][sprite indices 42..47]
   ↑ firstIndex=0      ↑ planeFirstIndex=36   ↑ spriteQuadFirstIndex=42
```

| Submesh | `firstIndex` | `indexCount` | `vertexOffset` |
|---------|--------------|--------------|----------------|
| Unit cube | 0 | `cubeIndexCount` | 0 |
| Ground plane | `planeFirstIndex` | 6 | 0 (absolute indices) |
| Sprite quad | `spriteQuadFirstIndex` | 6 | 0 |

### Building interleaved CPU data from Mesh::Vertex

`CreateSceneGeometry` converts ergonomic CPU vertices to the float layout `VulkanSceneVertexLayout` expects, then appends plane and sprite corners:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 742–762)
void VulkanRenderer::CreateSceneGeometry() {
    Mesh cubeMesh = Mesh::CreateUnitCube();
    Array<float> interleaved;
    interleaved.Reserve((cubeMesh.GetVertices().GetSize() + 8) * VulkanSceneVertexLayout::kFloatsPerVertex);
    for (std::size_t i = 0; i < cubeMesh.GetVertices().GetSize(); ++i) {
        VulkanRendererGpu::AppendRigidMeshVertexToInterleaved(cubeMesh.GetVertices()[i], interleaved);
    }

    const std::uint32_t planeVertexBase = static_cast<std::uint32_t>(cubeMesh.GetVertices().GetSize());
    const float groundExtent = kSceneGroundHalfExtent;
    // Four XZ corners at y=0 with +Y normals and tiled UVs (see full source)
    const Mesh::Vertex planeVerts[4] = { /* -groundExtent corners */ };
```

The same helper path is reused when glTF meshes upload in Part VI — one interleaved format everywhere.

### Index packing — CCW winding for +Y ground normal

Plane triangles use **absolute** vertex indices (`planeVertexBase + corner`). Spark explicitly sets `planeVertexOffset = 0` because adding the base again would double-offset and read past the buffer:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 780–802)
    planeFirstIndex = static_cast<std::uint32_t>(indices.GetSize());
    planeIndexCount = 6;
    indices.PushBack(planeVertexBase);
    indices.PushBack(planeVertexBase + 1);
    indices.PushBack(planeVertexBase + 2);
    indices.PushBack(planeVertexBase);
    indices.PushBack(planeVertexBase + 3);
    indices.PushBack(planeVertexBase + 2);

    cubeVertexOffset = 0;
    // Indices already store absolute vertex indices (planeVertexBase..); vertexOffset must be 0.
    planeVertexOffset = 0;
```

Wrong winding here makes back-face culling hide the entire ground plane.

### Staging upload — host-visible → device-local

Final buffers use `TRANSFER_DST | VERTEX_BUFFER_BIT` (or `INDEX_BUFFER_BIT`) with `DEVICE_LOCAL` memory. Spark maps staging, memcpy's, records one copy command, then destroys staging:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 807–848)
    VulkanRendererGpu::CreateBuffer(
            physicalDevice(), device(), vbSize,
            VK_BUFFER_USAGE_TRANSFER_SRC_BIT,
            VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
            stagingVertex, stagingVertexMemory);
    std::memcpy(vtxData, interleaved.GetData(), static_cast<std::size_t>(vbSize));

    VulkanRendererGpu::CreateBuffer(
            physicalDevice(), device(), vbSize,
            VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT,
            VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
            vertexBuffer, vertexBufferMemory);
```

Static geometry uploads once at init; dynamic custom meshes use `VulkanCustomMeshPool` with the same staging pattern each frame.

## 2. Core Concepts

### Buffer usage flags

| Flag | Purpose |
|------|---------|
| `VK_BUFFER_USAGE_VERTEX_BUFFER_BIT` | Bound as VBO |
| `VK_BUFFER_USAGE_INDEX_BUFFER_BIT` | Bound as IBO |
| `VK_BUFFER_USAGE_TRANSFER_DST_BIT` | Destination of `vkCmdCopyBuffer` |
| `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` | Fast GPU access |

Staging buffers add `HOST_VISIBLE | HOST_COHERENT` for CPU `memcpy`, then copy to device-local.

### Interleaved vs separate buffers

**Interleaved** (Spark default): position, normal, UV, ... per vertex in one struct — one binding, one stride.

```text
[V0: pos|n|uv|...][V1: pos|n|uv|...][V2: ...]
```

Separate buffers (SoA) can help SIMD uploads but complicate vertex input with multiple bindings.

### Indexed drawing

Triangles reference vertices by index:

```text
indices: [0,1,2, 0,2,3, ...]
vkCmdDrawIndexed(count, instanceCount, firstIndex, vertexOffset, firstInstance)
```

Reuse corners of a cube (8 vertices, 36 indices) instead of 36 full vertices.

### Submesh ranges in one buffer

Spark stores:

| Range | `firstIndex` | `indexCount` | `vertexOffset` |
|-------|--------------|--------------|----------------|
| Unit cube | 0 | `cubeIndexCount` | `cubeVertexOffset` |
| Ground plane | `planeFirstIndex` | `planeIndexCount` | `planeVertexOffset` |

`SceneDrawItem` with `mesh = UnitCube` tells the recorder which range to use — no per-draw buffer creation.

### Staging upload pattern

```text
1. Create staging buffer (host visible)
2. memcpy vertices/indices
3. Command buffer: vkCmdCopyBuffer staging → device
4. Barrier: transfer write → vertex attribute read
5. Destroy or reuse staging
```

Spark does this once at init for static geometry; dynamic meshes use `VulkanCustomMeshPool` per frame (Part VI).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Single static VB/IB for builtins

**Concept:** Submesh ranges

**Decision:** `CreateSceneGeometry` appends cube, plane, sprite quad into shared buffers

**Why:** One bind per pass for all builtin draws; only push constants and index ranges change.

**In Spark:** `VulkanRenderer::CreateSceneGeometry`.

### AppendRigidMeshVertexToInterleaved helper

**Concept:** Interleaved layout

**Decision:** `VulkanRendererGpu::AppendRigidMeshVertexToInterleaved` converts `Mesh::Vertex` to floats

**Why:** CPU `Mesh` type stays ergonomic; GPU layout stays identical for pipeline and importers.

**In Spark:** `VulkanGpuMeshInterleaved.hpp` / `.cpp`.

### DEVICE_LOCAL vertex/index buffers

**Concept:** Staging upload

**Decision:** Final buffers use device-local memory after staging copy

**Why:** Vertex fetch every frame should hit VRAM, not host-visible memory.

**In Spark:** `CreateSceneGeometry` — `CreateBuffer` with `DEVICE_LOCAL`.

### Index type UINT32

**Concept:** Indexed drawing

**Decision:** `VkIndexType_UINT32` for scene index buffer

**Why:** glTF and large meshes exceed 65k vertices; consistent type avoids ushort overflows.

**In Spark:** `vkCmdBindIndexBuffer(..., VK_INDEX_TYPE_UINT32)`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/core/VulkanRenderer.cpp` — `CreateSceneGeometry`
- `include/spark/render/scene/VulkanSceneVertexLayout.hpp`
- `include/spark/scene/mesh/Mesh.hpp`
- `src/spark/render/gpu/VulkanGpuBufferImage.cpp` — buffer creation helpers

### Step 1 — Build interleaved CPU array

```cpp
// VulkanRenderer.cpp (lines 670–690, abbreviated)
void VulkanRenderer::CreateSceneGeometry() {
    Mesh cubeMesh = Mesh::CreateUnitCube();
    Array<float> interleaved;
    interleaved.Reserve((cubeMesh.GetVertices().GetSize() + 8) * VulkanSceneVertexLayout::kFloatsPerVertex);
    for (std::size_t i = 0; i < cubeMesh.GetVertices().GetSize(); ++i) {
        VulkanRendererGpu::AppendRigidMeshVertexToInterleaved(cubeMesh.GetVertices()[i], interleaved);
    }

    const std::uint32_t planeVertexBase = static_cast<std::uint32_t>(cubeMesh.GetVertices().GetSize());
    // append 4 plane vertices ...
```

### Step 2 — Append indices for cube and plane

```cpp
    cubeIndexCount = static_cast<std::uint32_t>(cubeMesh.GetIndices().GetSize());
    for (std::size_t i = 0; i < cubeMesh.GetIndices().GetSize(); ++i) {
        indices.PushBack(cubeMesh.GetIndices()[i]);
    }
    planeFirstIndex = static_cast<std::uint32_t>(indices.GetSize());
    planeIndexCount = 6;
    indices.PushBack(planeVertexBase);
    indices.PushBack(planeVertexBase + 1);
    // ... two CCW triangles for +Y normal ground plane
```

Note comment in Spark: indices use **absolute** vertex indices; `vertexOffset` in draw must be 0 when using this scheme.

### Step 3 — Staging copy to device buffers

```cpp
    const VkDeviceSize vbSize = sizeof(float) * interleaved.GetSize();
    const VkDeviceSize ibSize = sizeof(std::uint32_t) * indices.GetSize();

    // Create staging + device buffers, map staging, memcpy, record copy command, submit once
    VulkanRendererGpu::CreateBuffer(..., VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT,
                                    VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);
```

Exact helper names are in `VulkanGpuBufferImage` — follow the same pattern for your engine.

### Step 4 — Bind at draw time

```cpp
// VulkanSceneOpaquePass.cpp (conceptual)
    vkCmdBindVertexBuffers(commandBuffer, 0, 1, &staticVertexBuffer, &offset);
    vkCmdBindIndexBuffer(commandBuffer, staticIndexBuffer, 0, VK_INDEX_TYPE_UINT32);
    vkCmdDrawIndexed(commandBuffer, range.indexCount, 1, range.firstIndex, range.vertexOffset, 0);
```

### Step 5 — Map mesh slot to range

When `SceneDrawItem::mesh == SceneMeshSlot::UnitCube`, the opaque pass selects `cubeIndexCount` and `cubeVertexOffset`. Ground plane uses `planeFirstIndex` / `planeIndexCount`.

## 5. How This Connects to the Rest of Your Engine

Chapter **8** draws cube and plane using these ranges. Part **VI** adds `Custom` meshes via `VulkanCustomMeshPool` with the same vertex layout. Part **III** documents shader attribute locations matching `VulkanSceneVertexLayout`.

Buffers are the bridge between `SceneDrawItem` and `vkCmdDrawIndexed`.

## 6. Pitfalls and Debugging

- **Wrong stride** — vertices look scrambled; verify `kStrideBytes`.
- **Index order winding** — back-face culling hides mesh; flip triangle order or cull mode.
- **vertexOffset double-applied** — Spark comment warns plane indices already absolute.
- **Forgot transfer barrier** — GPU reads staging garbage; validation may catch layout.

## 7. Exercises

1. How many floats per vertex in `VulkanSceneVertexLayout`?
2. Count cube vertices vs indices — what is the compression ratio?
3. Draw only the ground plane — which `firstIndex` and `indexCount`?
4. Trace `Mesh::CreateUnitCube` — where do normals come from?

## Summary

Vertex and index buffers store mesh geometry in device-local memory. Spark uploads interleaved cube and plane data once, shares one VB/IB, and selects subranges per `SceneMeshSlot`. Staging copy and correct usage flags are the pattern for all future mesh loading.

---

Next: [Descriptor Sets and Uniform Buffers](06-descriptors-uniforms.md).
