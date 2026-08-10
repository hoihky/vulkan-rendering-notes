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
