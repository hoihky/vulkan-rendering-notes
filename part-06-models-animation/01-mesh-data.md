# Mesh Data Structures

> **Build-Along Milestone:** Colored cube and ground plane render via MeshComponent with correct SceneMeshSlot.

**You will learn:**

- Define Spark's Mesh::Vertex layout and indexed triangle list.
- Choose SceneMeshSlot::UnitCube vs Custom for procedural vs imported geometry.
- Compute axis-aligned bounds for scaling and camera framing.

## 1. The Problem We Are Solving

Part V gave you transforms and draw gathering, but every prop still assumes built-in cubes and planes. Real content
needs a **CPU mesh type** — vertices, indices, optional bounds — shared across entities and independent of Vulkan.

Your build-along milestone: represent a colored cube and ground plane through `Mesh` + `MeshComponent`, understanding
which slots hit static GPU packs vs the dynamic custom path.

### What is a mesh? (complete beginner)

A **3D model** in a game is made of **triangles**. GPUs only rasterize triangles (or lines/points in special modes). A **mesh** is the engine's CPU-side container for:

- **Vertices** — corners of triangles, each with data (position, normal, UV, …)
- **Indices** — which three vertices form each triangle

Example: a quad (four corners) is drawn as **two** triangles → six indices, but you might store four unique vertices and reuse corners via indices.

```text
Vertices:  A B C D  (positions in 3D space)
Indices:   0,1,2  and  0,2,3  → two triangles ABC and ACD
```

`vkCmdDrawIndexed` says: "For each triangle, look up three indices, fetch three vertices, run shaders."

### Why separate mesh from GameObject

A **GameObject** is an entity in your world (a tree, the player, a bullet). A **Mesh** is shared geometry — one oak tree mesh can instantiate 500 trees with different transforms. Spark uses:

```text
SharedPtr<Mesh>  ← loaded once, read-only geometry
MeshComponent    ← per-entity: which mesh + which render slot + fallback color
TransformComponent ← per-entity: position, rotation, scale
```

Without this split, duplicating a forest would duplicate megabytes of vertex data.

### Static slot vs custom upload (the two GPU paths)

Spark optimizes the two most common cases:

| `SceneMeshSlot` | Geometry | GPU path | When |
|-----------------|----------|----------|------|
| `UnitCube` | Engine built-in cube | Pre-uploaded static `VkBuffer` | Debug, placeholders |
| `GroundPlane` | Engine built-in plane | Pre-uploaded static `VkBuffer` | Ground in demos |
| `Custom` | Any `SharedPtr<Mesh>` | `VulkanCustomMeshPool` uploads each frame or caches | glTF imports, procedural |

**Beginner rule:** If you loaded a file from disk, you almost always use `SceneMeshSlot::Custom`. Built-in slots exist so Part II can draw cubes before importers exist.

### Walkthrough: one vertex's journey to the screen

1. **Authoring** — Blender exports position `(0.5, 1.0, 0.2)`, normal `(0,1,0)`, UV `(0.3, 0.7)`.
2. **Import** — glTF loader writes `Mesh::Vertex` into `Array<Vertex>`.
3. **Gameplay** — `MeshComponent` references the mesh; `TransformComponent` places the object at `(10, 0, 5)`.
4. **Submit** — `SceneDrawItem` gets world matrix + mesh slot + material layers.
5. **Upload** — Custom pool copies vertex array to GPU buffer (or reuses cached upload).
6. **Record** — `vkCmdBindVertexBuffers`, `vkCmdBindIndexBuffer`, `vkCmdDrawIndexed`.
7. **Vertex shader** — Multiplies position by `model * viewProj` matrices.
8. **Fragment shader** — Samples texture at interpolated UV, applies lighting.
9. **HDR framebuffer** — Stores linear color (Part X).
10. **Tonemap** — Compresses to monitor range, present.

Every later chapter replaces one step with production detail. Part VI focuses on steps 1–3; Part II on 5–6; Part IV on 8; Part X on 9–10.

## 1b. Concept Deep Dive — Full Vertex Layout and MeshComponent

### Production Vertex record (beyond position/normal/UV)

Spark's rigid `Mesh::Vertex` carries fields glTF importers populate:

```cpp
// include/spark/scene/mesh/Mesh.hpp (lines 25–33)
struct Vertex {
    Vector3 position{Vector3::Zero};
    Vector3 normal{Vector3::UnitZ};
    Vector2 texCoord{Vector2::Zero};
    Vector2 texCoord1{Vector2::Zero};   // second UV set (lightmaps, etc.)
    Vector4 color{1, 1, 1, 1};          // vertex color multiplier
    Vector4 tangent{};                  // xyz = tangent, w = bitangent handedness
};
```

`MeshComponent` pairs a `SharedPtr<Mesh>` with render slot and fallback tint:

```cpp
// include/spark/ecs/components/rendering/MeshComponent.hpp (concept)
class MeshComponent {
    SharedPtr<Mesh> mesh;
    SceneMeshSlot slot = SceneMeshSlot::UnitCube;
    Vector3 albedo{1, 1, 1};  // used when no MaterialComponent
};
```

### Interleaved GPU stride matches VulkanSceneVertexLayout

When `VulkanCustomMeshPool` packs a rigid mesh, `VulkanGpuMeshInterleaved::AppendRigidVertexToInterleaved` writes floats in shader location order — positions 0–2 in `scene.vert` align with this packing. Skinned meshes append joint indices (bitcast float) and weights after color.

### TryComputeAxisAlignedBounds — used everywhere

```cpp
// Demos scale imported glTF to target height:
Vector3 bmin, bmax;
if (mesh->TryComputeAxisAlignedBounds(bmin, bmax)) {
    const float h = bmax.y - bmin.y;
    const float scale = targetHeight / h;
    transform->SetScale({scale, scale, scale});
}
```

CharacterCameraDemo and ThreeDDemo use bounds for camera framing and tree scaling — cheap O(n) after load.

## 2. Core Concepts

### CPU mesh as shared asset

A **mesh** is geometry on CPU: an array of vertices (position, normal, UV) plus an index buffer describing triangle corners. Spark keeps meshes **renderer-agnostic** — no `VkBuffer` inside `Mesh`. Multiple `MeshComponent` instances share one `SharedPtr<Mesh>` while transforms differ per entity.


### Indexed triangles

Each triangle is three indices into `vertices[]`. `vkCmdDrawIndexed` fetches corner attributes through the index buffer, letting shared corners reuse data. Spark uses **`uint32_t` indices** everywhere so large glTF assets never overflow 16-bit limits.


### Built-in slots vs custom geometry

`SceneMeshSlot::UnitCube` and `GroundPlane` map to static GPU packs created once at renderer init. `SceneMeshSlot::Custom` means “upload this CPU mesh into the dynamic pool this frame” — the path for imported glTF trees and props.


### Procedural authoring conventions

Spark procedural helpers (`CreateUnitCube`, `CreateGroundPlane`, …) duplicate vertices at face corners when UVs must differ — a cube uses **24 vertices / 36 indices**, not 8/36. Winding and normals are chosen for Vulkan back-face culling with Spark's perspective camera.


### Axis-aligned bounds

`TryComputeAxisAlignedBounds` scans positions for min/max — used by demos to scale trees to target height and frame debug cameras. Bounds live on CPU; culling chapters consume them later.

### MeshSubmesh: one buffer, many materials

glTF files often ship **one mesh with multiple primitives**, each referencing a different material. Spark does not split these into separate `Mesh` objects. Instead, a single `Mesh` holds one shared vertex/index buffer and an array of `MeshSubmesh` slices:

```cpp
// include/spark/scene/mesh/MeshSubmesh.hpp (lines 15–31)
/** Index range + glTF material index for one primitive within a shared vertex buffer. */
struct MeshSubmesh {
    std::uint32_t indexOffset = 0;
    std::uint32_t indexCount = 0;
    std::uint32_t materialIndex = 0;
    Array<MeshSubmeshVariantMapping> variantMappings;

    [[nodiscard]] std::uint32_t ResolveMaterialIndex(const std::uint32_t activeVariant) const noexcept {
        for (std::size_t i = 0; i < variantMappings.GetSize(); ++i) {
            if (variantMappings[i].variantIndex == activeVariant) {
                return variantMappings[i].materialIndex;
            }
        }
        return materialIndex;
    }
};
```

Each glTF **primitive** appends vertices/indices, then records a submesh entry with `indexOffset`, `indexCount`, and `materialIndex`:

```cpp
// src/spark/scene/assets/gltf/GltfMeshBuilder.cpp (lines 127–137)
    MeshSubmesh submesh{};
    submesh.indexOffset = indexOffset;
    submesh.indexCount = indexCount;
    submesh.materialIndex = MaterialIndex(data, prim->material);
    for (cgltf_size mi = 0; mi < prim->mappings_count; ++mi) {
        MeshSubmeshVariantMapping mapping{};
        mapping.variantIndex = static_cast<std::uint32_t>(prim->mappings[mi].variant);
        mapping.materialIndex = MaterialIndex(data, prim->mappings[mi].material);
        submesh.variantMappings.PushBack(mapping);
    }
    outMesh.GetSubmeshes().PushBack(submesh);
```

At submit time, each submesh becomes a separate `SceneDrawItem` with `submeshIndex = si`. The GPU issues one `vkCmdDrawIndexed` per submesh, slicing the shared index buffer with `firstIndex` and `indexCount` from the packed custom mesh pool. This is how a single imported `.glb` chair with wood/metal/fabric materials renders without duplicating vertex data.

**Design rule:** if `GetSubmeshes().GetSize() > 1`, pair the mesh with `MultiMaterialComponent` (Part VI Chapter 3, Part VII Chapter 5). A lone `MaterialComponent` only affects the first submesh or the full buffer when no submesh table exists.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Mesh class with Vertex + indices

**Concept:** CPU mesh as shared asset

**Decision:** Mesh class with Vertex + indices

**Why:** One type feeds procedural primitives, OBJ, and baked glTF. GPU upload stays in VulkanCustomMeshPool.

**In Spark:** `include/spark/scene/Mesh.hpp`

### uint32_t index arrays

**Concept:** Indexed triangles

**Decision:** uint32_t index arrays

**Why:** Matches glTF UNSIGNED_INT/SHORT accessors after promotion; avoids silent truncation on hero assets.

**In Spark:** `Mesh::GetIndices()` — `Array<std::uint32_t>`

### SceneMeshSlot enum on draw items

**Concept:** Built-in slots vs custom geometry

**Decision:** SceneMeshSlot enum on draw items

**Why:** Renderer binds static geometry by slot index; custom path packs arbitrary SharedPtr<Mesh> each frame.

**In Spark:** `include/spark/engine/SceneRenderParams.hpp` — `SceneMeshSlot`

### Static factory methods on Mesh

**Concept:** Procedural authoring conventions

**Decision:** Static factory methods on Mesh

**Why:** Demos run without asset files; same Vertex layout as imported content for one shader path.

**In Spark:** `src/spark/scene/Mesh.cpp` — `CreateUnitCube`, `CreateGroundPlane`

### TryComputeAxisAlignedBounds on Mesh

**Concept:** Axis-aligned bounds

**Decision:** TryComputeAxisAlignedBounds on Mesh

**Why:** Cheap O(n) pass after load; CharacterCameraDemo scales glTF trees using bounds height.

**In Spark:** `Mesh::TryComputeAxisAlignedBounds` in Mesh.hpp

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scene/Mesh.hpp`
- `src/spark/scene/Mesh.cpp`
- `include/spark/engine/SceneRenderParams.hpp`
- `shaders/scene.vert`

### Step 1 — Read the Mesh vertex record

*CPU-side mesh — SharedPtr-friendly, no Vulkan types.*

```cpp
// include/spark/scene/Mesh.hpp (lines 15–44)
/**
 * CPU-side mesh (vertices + indices). Shared across GameObjects / MeshComponents via SharedPtr.
 * GPU upload is renderer-specific and stays outside this type.
 */
class Mesh {
public:
    struct Vertex {
        Vector3 position{Vector3::Zero};
        Vector3 normal{Vector3::UnitZ};
        Vector2 texCoord{Vector2::Zero};
    };

    Mesh() = default;
    explicit Mesh(Utf8String meshName);

    [[nodiscard]] Utf8String& GetName() noexcept { return name; }
    [[nodiscard]] const Utf8String& GetName() const noexcept { return name; }

    [[nodiscard]] Array<Vertex>& GetVertices() noexcept { return vertices; }
    [[nodiscard]] const Array<Vertex>& GetVertices() const noexcept { return vertices; }

    [[nodiscard]] Array<std::uint32_t>& GetIndices() noexcept { return indices; }
    [[nodiscard]] const Array<std::uint32_t>& GetIndices() const noexcept { return indices; }

    void Clear() noexcept;

    void AddVertex(const Vertex& v);
    void AddTriangle(std::uint32_t i0, std::uint32_t i1, std::uint32_t i2);

    /** Axis-aligned quad in XY at z=0, w x h, two triangles; normals +Z. */
```


Note the header comment: GPU upload is renderer-specific and stays outside `Mesh`.

### Step 2 — Study procedural cube topology

*Per-face vertex duplication — 4 verts per face, 2 triangles each.*

```cpp
// src/spark/scene/Mesh.cpp (lines 130–147)
    for (int f = 0; f < 6; ++f) {
        const std::uint32_t base = static_cast<std::uint32_t>(m.vertices.GetSize());
        for (int k = 0; k < 4; ++k) {
            Vertex v;
            v.position = p[faces[f][k]];
            v.normal = n[f];
            v.texCoord = faceUv[k];
            m.vertices.PushBack(v);
        }
        m.indices.PushBack(base);
        m.indices.PushBack(base + 1);
        m.indices.PushBack(base + 2);
        m.indices.PushBack(base);
        m.indices.PushBack(base + 2);
        m.indices.PushBack(base + 3);
    }
    return m;
}
```


Compare index count (36) to a naive 8-corner cube — UV seams require duplication.

### Step 3 — Inspect SceneMeshSlot on draw items

*Built-in slots: UnitCube, GroundPlane, Custom.*

```cpp
// include/spark/engine/SceneRenderParams.hpp (lines 42–48)
/** Built-in mesh slots matching the Vulkan scene vertex/index pack (see VulkanRenderer::CreateSceneGeometry). */
enum class SceneMeshSlot : std::uint8_t {
    UnitCube = 0,
    GroundPlane = 1,
    /** Arbitrary CPU mesh uploaded each frame into the renderer's dynamic scene buffers (see customMesh). */
    Custom = 2,
};
```

### Step 4 — Wire MeshComponent in your build-along

```cpp
auto mesh = MakeShared<Mesh>(Mesh::CreateUnitCube());
auto* prop = world.CreateGameObject();
prop->AddComponent<TransformComponent>()->SetTranslation({2, 0.5f, 0});
prop->AddComponent<MeshComponent>(mesh, SceneMeshSlot::UnitCube, Vector3{0.8f, 0.2f, 0.2f});
```

Built-in slot draws skip the custom pool — good for early milestones before glTF import.

### Step 5 — Compute bounds after building geometry

```cpp
Vector3 bmin, bmax;
if (mesh->TryComputeAxisAlignedBounds(bmin, bmax)) {
    const float height = bmax.y - bmin.y;
    // scale / camera framing
}
```

### Step 6 — Match vertex locations to scene.vert

*Vertex inputs — position, normal, UV, tangent; joints/weights for skinned path.*

```glsl
// shaders/scene.vert (lines 4–10)
layout(location = 0) in vec3 inPosition;
layout(location = 1) in vec3 inNormal;
layout(location = 2) in vec2 inTexCoord;
layout(location = 3) in vec4 inTangent;
// Packed joint indices (bit pattern of uint stored in float) — avoids integer vertex fetch issues on some MVK paths.
layout(location = 4) in vec4 inJointsPacked;
layout(location = 5) in vec4 inWeights;
```


Rigid `Mesh::Vertex` supplies locations 0–2. Skinned meshes add tangent + joint streams later.

### Step 7 — Milestone check — one cube + one ground plane

Run your scene with `SceneMeshSlot::UnitCube` on a prop and `GroundPlane` under the camera. RenderDoc should show two distinct static vertex buffers — no custom pool uploads yet.

## 5. How This Connects to the Rest of Your Engine

**Loading OBJ** and **Loading glTF** fill the same `Mesh` struct from files. **Custom Mesh Pool** uploads
`SceneMeshSlot::Custom` geometry each frame. **Skinned glTF** extends the vertex layout with joints and weights
while keeping the same index buffer pattern.

## 6. Pitfalls and Debugging

- **Sharing one UV at cube corners** — texture smears; duplicate verts per face like Spark's `CreateUnitCube`.
- **16-bit indices on large assets** — Spark uses uint32_t; do not downcast glTF indices silently.
- **Putting VkBuffer inside Mesh** — breaks sharing and hot reload; keep GPU in the renderer.
- **Wrong mesh slot** — glTF on UnitCube slot draws wrong static buffer; use Custom for imports.

## 7. Exercises

1. Print vertex and index counts for `CreateUnitCube` vs `CreateGroundPlane`.
2. Log byte stride of `Mesh::Vertex` and compare to GLSL location offsets for rigid draws (locations 0–2).
3. Spawn two `MeshComponent`s sharing one `SharedPtr<Mesh>`; verify one CPU copy and two draw items.
4. Build a procedural pyramid with `AddTriangle` — compute bounds and center it on the origin.
5. Draw with wrong `SceneMeshSlot::UnitCube` while passing a custom mesh pointer — describe the visual bug.

## Summary

Mesh = CPU vertices + uint32 indices, renderer-agnostic; SceneMeshSlot selects static vs custom GPU upload.

---

Next: [Loading OBJ](02-load-obj.md).
