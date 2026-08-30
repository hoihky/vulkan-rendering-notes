# Heightfield Generation

> **Build-Along Milestone:** TerrainDemo shows lit procedural hills with correct shading normals.

**You will learn:**

- Generate a regular grid mesh from a 2D height sample array using TerrainMeshGenerator.
- Compute per-vertex normals from height differences.
- Route terrain geometry through SceneMeshSlot::Custom and the custom mesh pool.

## 1. The Problem We Are Solving

Open outdoor scenes need large ground surfaces. A **heightfield** extrudes an XZ grid using height samples — procedural fBM for demos, brushes or imported maps later. Spark implements generation in **`TerrainMeshGenerator`** as pure CPU code; GPU sees only the resulting custom mesh like any other prop.

Milestone: run `TerrainDemo` and verify lit hills with correct normals and visible winding under back-face cull.

### Terrain as a heightfield (beginner)

**Terrain** is ground that stretches far into the distance. Instead of modeling every hill as a separate mesh file, engines often use a **heightfield**:

- A 2D grid of height values `height[x][z]` (or a function like noise).
- Each grid point becomes a vertex; grid cells become two triangles.

```text
Top view (XZ grid):          Side view (heights):

  ●───●───●                  ●
  │ \ │ / │                 / \
  ●───●───●                ●───●───●  hills
  │ / │ \ │
  ●───●───●
```

Spark's `TerrainMeshGenerator` runs on **CPU** at load or when brushes edit height — same `Mesh` + `Custom` slot path as glTF props. The fragment shader does not know "this is terrain"; it only sees normals and albedo like any mesh.

**Normals from height:** for each vertex, sample neighboring heights to estimate slope — steeper slope → normal tilts → grass looks darker on hillsides under directional sun.

## 1b. Concept Deep Dive — CPU Heightfield to Custom Mesh Slot

### fBM fills height samples; BuildIntoFromHeights emits topology

Spark generates outdoor ground as a regular XZ grid. `FillProceduralHeights` evaluates layered smooth noise; `BuildIntoFromHeights` places vertices, indexes triangles, then derives normals from neighbor height differences:

```cpp
// src/spark/scene/mesh/TerrainMeshGenerator.cpp (lines 86–89, 137–156)
            const float h = (Fbm(nxw, nzw, settings) * 2.0F - 1.0F) * settings.heightScale;
            outHeights.PushBack(h);
// ...
            Vector3 t{R.x - L.x, R.y - L.y, R.z - L.z};
            Vector3 b{U.x - D.x, U.y - D.y, U.z - D.z};
            Vector3 n = Vector3::Cross(b, t);
            verts[i].normal = n.Normalized();
```

The GPU fragment shader does not know "this is terrain" — it sees positions, normals, and UVs like any lit mesh.

### Custom mesh slot, same opaque pass as glTF props

Generated geometry routes through `SceneMeshSlot::Custom` and `VulkanCustomMeshPool` — identical submit and record path to imported props. Planar UVs scale with world XZ via `worldUnitsPerTextureRepeat` for tiled grass/soil materials (Part VII).

### Winding matches engine convention

Triangle winding in `BuildIntoFromHeights` matches unit cube / Vulkan back-face culling — TerrainDemo hills should remain visible under default cull mode. Wrong winding shows invisible terrain with depth-only shadow silhouettes.

## 2. Core Concepts

### Heightfield grid topology

Outdoor ground is commonly a **heightfield**: a regular XZ grid where each vertex height comes from a scalar sample. Spark derives counts from `TerrainGeneratorSettings`:

```text
vertexCountX = max(2, subdivX) + 1
vertexCountZ = max(2, subdivZ) + 1
heightSamples = vertexCountX × vertexCountZ
world step X  = 2 × halfExtentX / (vertexCountX - 1)
```

Each quad spans two triangles between four neighboring height samples. Higher subdiv increases silhouette fidelity at linear GPU cost — choose subdiv to match shadow cascade texel density near the camera (Chapter 5 streams many patches instead of one giant grid).

### Procedural fBM height fill

`FillProceduralHeights` evaluates layered smooth noise with octaves, persistence, lacunarity, and seed:

```text
h = (Fbm(nx, nz, settings) * 2 - 1) * heightScale
```

Imported heightmaps can replace fBM by filling the same `Array<float>` buffer before `BuildIntoFromHeights`.

### Triangle winding and indices

```cpp
mesh.AddTriangle(i0, i1, i2);
mesh.AddTriangle(i1, i3, i2);
```

Wrong winding makes terrain **invisible** with back-face cull enabled.

### Planar UV tiling

```cpp
v.texCoord = { x / uvRepeat, z / uvRepeat };
```

`MaterialComponent` supplies albedo/normal/ORM on top of planar UVs from Part VII.

### Normal reconstruction pass

```text
n = normalize(cross(upNeighbor - downNeighbor, rightNeighbor - leftNeighbor))
```

Normals recompute after brush edits (Chapter 2) or slopes shade incorrectly.

### Custom mesh slot upload path

`TerrainComponent::RegenerateMesh` assigns **`SceneMeshSlot::Custom`** — same dynamic pool path as glTF props (Part VI).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### TerrainMeshGenerator static API

**Concept:** Heightfield grid topology

**Decision:** TerrainMeshGenerator static API

**Why:** Pure CPU functions without Vulkan — testable unit for height → mesh conversion.

**In Spark:** `include/spark/scene/TerrainMeshGenerator.hpp`.

### TerrainGeneratorSettings tunables

**Concept:** Procedural fBM height fill

**Decision:** TerrainGeneratorSettings tunables

**Why:** Octaves/persistence/seed live on component — designers iterate in demo without code edits.

**In Spark:** `include/spark/scene/TerrainGeneratorSettings.hpp`.

### Document winding in generator comment

**Concept:** Triangle winding and indices

**Decision:** Document winding in generator comment

**Why:** Future importers copy same order — single convention across procedural and authored meshes.

**In Spark:** `TerrainMeshGenerator.cpp` lines 130–132.

### worldUnitsPerTextureRepeat on settings

**Concept:** Planar UV tiling

**Decision:** worldUnitsPerTextureRepeat on settings

**Why:** UV scale is authoring param, not shader magic number.

**In Spark:** `TerrainMeshGenerator.cpp` lines 116–117.

### Second pass over vertex grid

**Concept:** Normal reconstruction pass

**Decision:** Second pass over vertex grid

**Why:** Keeps BuildIntoFromHeights usable for imported heights and procedural fills alike.

**In Spark:** `TerrainMeshGenerator.cpp` lines 136–158.

### MeshComponent Custom slot

**Concept:** Custom mesh slot upload path

**Decision:** MeshComponent Custom slot

**Why:** Terrain participates in standard opaque scene pass without special terrain renderer.

**In Spark:** `TerrainComponent::RegenerateMesh` in component cpp.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/TerrainMeshGenerator.cpp`
- `include/spark/scene/TerrainMeshGenerator.hpp`
- `include/spark/demo/TerrainDemo.hpp`

### Step 1 — Read grid count helpers

Vertex and height sample counts.

*grid counts*

```cpp
// src/spark/scene/TerrainMeshGenerator.cpp (lines 58–70)
std::int32_t TerrainMeshGenerator::GridVertexCountX(const TerrainGeneratorSettings& settings) noexcept {
    return (std::max)(std::int32_t{2}, settings.subdivX) + 1;
}

std::int32_t TerrainMeshGenerator::GridVertexCountZ(const TerrainGeneratorSettings& settings) noexcept {
    return (std::max)(std::int32_t{2}, settings.subdivZ) + 1;
}

std::size_t TerrainMeshGenerator::HeightSampleCount(const TerrainGeneratorSettings& settings) noexcept {
    const std::int32_t nx = GridVertexCountX(settings);
    const std::int32_t nz = GridVertexCountZ(settings);
    return static_cast<std::size_t>(nx) * static_cast<std::size_t>(nz);
}
```

### Step 2 — Study FillProceduralHeights

fBM height fill loop.

*FillProceduralHeights*

```cpp
// src/spark/scene/TerrainMeshGenerator.cpp (lines 72–92)
void TerrainMeshGenerator::FillProceduralHeights(const TerrainGeneratorSettings& settings, Array<float>& outHeights) {
    const std::int32_t nx = GridVertexCountX(settings);
    const std::int32_t nz = GridVertexCountZ(settings);
    const std::size_t need = static_cast<std::size_t>(nx) * static_cast<std::size_t>(nz);
    outHeights.Clear();
    outHeights.Reserve(need);
    const float hx = settings.halfExtentX;
    const float hz = settings.halfExtentZ;
    for (std::int32_t iz = 0; iz < nz; ++iz) {
        const float tz = static_cast<float>(iz) / static_cast<float>(nz - 1);
        const float z = -hz + tz * (2.0F * hz);
        for (std::int32_t ix = 0; ix < nx; ++ix) {
            const float tx = static_cast<float>(ix) / static_cast<float>(nx - 1);
            const float x = -hx + tx * (2.0F * hx);
            const float nxw = x * settings.noiseScale;
            const float nzw = z * settings.noiseScale;
            const float h = (Fbm(nxw, nzw, settings) * 2.0F - 1.0F) * settings.heightScale;
            outHeights.PushBack(h);
        }
    }
}
```

### Step 3 — Walk vertex and index build

Positions, UVs, indices.

*mesh build*

```cpp
// src/spark/scene/TerrainMeshGenerator.cpp (lines 106–134)
    mesh.GetVertices().Reserve(need);
    for (std::int32_t iz = 0; iz < nz; ++iz) {
        const float tz = static_cast<float>(iz) / static_cast<float>(nz - 1);
        const float z = -hz + tz * (2.0F * hz);
        for (std::int32_t ix = 0; ix < nx; ++ix) {
            const float tx = static_cast<float>(ix) / static_cast<float>(nx - 1);
            const float x = -hx + tx * (2.0F * hx);
            const std::size_t idx = static_cast<std::size_t>(iz * nx + ix);
            Mesh::Vertex v{};
            v.position = {x, heights[idx], z};
            const float uvRepeat = (std::max)(settings.worldUnitsPerTextureRepeat, 1.0e-3F);
            v.texCoord = {x / uvRepeat, z / uvRepeat};
            mesh.AddVertex(v);
        }
    }

    auto& idxBuf = mesh.GetIndices();
    idxBuf.Reserve(static_cast<std::size_t>(nx - 1) * static_cast<std::size_t>(nz - 1) * 6U);
    for (std::int32_t iz = 0; iz < nz - 1; ++iz) {
        for (std::int32_t ix = 0; ix < nx - 1; ++ix) {
            const std::uint32_t i0 = static_cast<std::uint32_t>(iz * nx + ix);
            const std::uint32_t i1 = i0 + 1U;
            const std::uint32_t i2 = static_cast<std::uint32_t>((iz + 1) * nx + ix);
            const std::uint32_t i3 = i2 + 1U;
            // Winding matches unit cube / Vulkan PerspectiveVulkan back-face culling (not glTF CCW).
            mesh.AddTriangle(i0, i1, i2);
            mesh.AddTriangle(i1, i3, i2);
        }
    }
```

### Step 4 — Read normal pass

Neighbor cross product normals.

*normals*

```cpp
// src/spark/scene/TerrainMeshGenerator.cpp (lines 136–158)
    auto& verts = mesh.GetVertices();
    for (std::int32_t iz = 0; iz < nz; ++iz) {
        for (std::int32_t ix = 0; ix < nx; ++ix) {
            const std::size_t i = static_cast<std::size_t>(iz * nx + ix);
            const std::int32_t ixL = (std::max)(ix - 1, 0);
            const std::int32_t ixR = (std::min)(ix + 1, nx - 1);
            const std::int32_t izD = (std::max)(iz - 1, 0);
            const std::int32_t izU = (std::min)(iz + 1, nz - 1);
            const Vector3& L = verts[static_cast<std::size_t>(iz * nx + ixL)].position;
            const Vector3& R = verts[static_cast<std::size_t>(iz * nx + ixR)].position;
            const Vector3& D = verts[static_cast<std::size_t>(izD * nx + ix)].position;
            const Vector3& U = verts[static_cast<std::size_t>(izU * nx + ix)].position;
            Vector3 t{R.x - L.x, R.y - L.y, R.z - L.z};
            Vector3 b{U.x - D.x, U.y - D.y, U.z - D.z};
            Vector3 n = Vector3::Cross(b, t);
            if (n.LengthSquared() > 1.0e-12F) {
                n = n.Normalized();
            } else {
                n = Vector3::UnitY;
            }
            verts[i].normal = n;
        }
    }
```

### Step 5 — Run TerrainDemo

Outdoor baseline with CSM and fog.

### Step 6 — Log mesh statistics

Vertex and triangle counts for budget.

### Step 7 — Verify winding visually

Fix cull if terrain invisible.

### Step 8 — Milestone — lit procedural hills

Correct shading normals on slopes.

## 5. How This Connects to the Rest of Your Engine

Part **VI** custom mesh pool uploads dynamic geometry. Part **VII** materials texture terrain via planar UVs. Part **IX** CSM shadows need cast/receive flags on terrain draws. Chapter 2 **`TerrainComponent`** owns regeneration lifecycle.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

Heightfield output feeds TerrainComponent (Chapter 2) and opaque pass (Chapter 3). Same Mesh upload path as Part V static geometry. Noise functions here are CPU-only — GPU terrain displacement is a future extension.

## 6. Pitfalls and Debugging

- **Wrong winding** — entire terrain invisible with back-face cull ON.
- **Normals not recomputed after height edit** — lighting looks wrong on edited slopes.
- **Height buffer size mismatch** — `BuildIntoFromHeights` returns early if count wrong.
- **subdiv too high for real-time full regenerate** — brush path rebuilds whole mesh; budget subdiv for demo (Chapter 5 streaming roadmap).
- **UV repeat too small** — grass stretches; increase `worldUnitsPerTextureRepeat`.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

Inverted winding culls entire terrain — verify with known light direction. Height count mismatch in BuildIntoFromHeights returns silently — assert sizes in debug during brush iteration.

## 7. Exercises

1. Generate 64×64 heightfield; log vertex and triangle counts.
2. Visualize normals as color via debug shader override.
3. Compare flat plane vs fBM terrain in TerrainDemo side by side.
4. Change seed in settings — confirm different hill shapes with same subdiv.
5. Verify triangle count equals `subdivX * subdivZ * 2`.

## Summary

Heightfields extrude a regular XZ grid from height samples with computed normals and planar UVs. Spark implements generation in TerrainMeshGenerator and uploads via Custom mesh slot.

---

Next: [TerrainComponent](02-terrain-component.md).
