# Loading OBJ

> **Build-Along Milestone:** One OBJ prop renders on the ground via LoadMesh and SceneMeshSlot::Custom.

**You will learn:**

- Parse OBJ v/vn/vt/f lines into Mesh vertices and indices.
- Explain Spark's per-corner expansion vs welding strategies.
- Load OBJ through GameWorldAssetCache::LoadMesh.

## 1. The Problem We Are Solving

ASCII OBJ remains useful for quick props and debugging importers. Before glTF, you need a loader that turns
`f` corners into Spark's interleaved `Mesh::Vertex` stream suitable for indexed draw and the custom mesh pool.

Build along: load one `.obj` through the cache, assign `SceneMeshSlot::Custom`, and see it on the ground plane.

### OBJ format for beginners

**OBJ** is a text file artists export from Blender/Maya. Key lines:

| Line | Meaning |
|------|---------|
| `v x y z` | Vertex position |
| `vn x y z` | Vertex normal (optional) |
| `vt u v` | Texture coordinate (optional) |
| `f i j k` | Triangle corner indices (1-based!) |

Example face: `f 1/1/1 2/2/2 3/3/3` means three corners, each referencing position/texcoord/normal index.

**Why expansion happens:** one OBJ position can appear in multiple faces with **different UVs** — Spark expands each unique `(v, vt, vn)` triple into one GPU vertex (same strategy as many engines).

```text
OBJ file (indexed pools)     Spark Mesh (interleaved)
  v[0], v[1], v[2]      →    Vertex { pos, normal, uv }
  vt[0], vt[1]               one row per corner
  f 1/1 2/2 3/1
```

**Workflow:** place `rock.obj` in assets → `GameWorld::LoadMesh("rock.obj")` → cache returns shared `Mesh*` → assign to `MeshRendererComponent` with `SceneMeshSlot::Custom`.

OBJ lacks materials/PBR — use for graybox props; glTF (Chapter 3) for production assets.

## 1b. Concept Deep Dive — How Spark Parses OBJ

### Three separate pools, one interleaved output

Wavefront OBJ was designed for exporters, not GPUs. A single corner on a cube face might reference `position 5`, `texcoord 12`, `normal 3` — three different indices into three different arrays. Spark's loader reads each pool independently, then **expands** every face corner into one `Mesh::Vertex`:

```text
File pools (1-based indices):          Spark output (interleaved):
  positions[1..N]                        vertices[0] = { pos[5], normal[3], uv[12] }
  texCoords[1..M]                        vertices[1] = { pos[6], normal[3], uv[13] }
  normals[1..K]                          indices = [0, 1, 2, 0, 2, 3, ...]
```

This is why a cube with per-face UVs has 24 vertices, not 8 — each face corner is unique even when positions repeat.

### ParseFaceCorner handles all slash formats

Spark's helper parses `1`, `1/2`, `1//3`, and `1/2/3` corners, including **negative indices** (relative to end of pool — Blender sometimes exports these):

```cpp
// src/spark/scene/mesh/Mesh.cpp — ParseFaceCorner (concept)
// "12/5/8"  → vi=12, ti=5, ni=8
// "12//8"   → vi=12, ti=0 (no UV), ni=8
// "-1"      → last position in pool
```

### Fan triangulation for n-gons

Quads and n-gons are not GPU-native. Spark fans from corner 0:

```cpp
// src/spark/scene/mesh/Mesh.cpp (lines 477–523)
        } else if (tag == "f") {
            // ... collect faceV, faceT, faceN per corner ...
            const std::size_t n = faceV.GetSize();
            if (n < 3) {
                continue;
            }
            for (std::size_t k = 2; k < n; ++k) {
                outMesh.indices.PushBack(cornerIndices[0]);
                outMesh.indices.PushBack(cornerIndices[k - 1]);
                outMesh.indices.PushBack(cornerIndices[k]);
            }
        }
```

A quad `f 1 2 3 4` becomes triangles `(1,2,3)` and `(1,3,4)` — winding follows file order, which matters for back-face culling.

### Cache integration — one load, many instances

After parse, `GameWorldAssetCache::LoadMesh` stores the result by path key:

```cpp
// src/spark/scene/assets/GameWorldAssetCache.cpp (lines 154–171)
SharedPtr<Mesh> GameWorldAssetCache::LoadMesh(const char* path) {
    if (IsGltfPath(path)) {
        return LoadGltf(path).mesh;  // .gltf/.glb routes to glTF loader
    }
    const Utf8String key(path);
    if (const SharedPtr<Mesh>* cached = meshCache.Find(key)) {
        return *cached;  // cache hit — no disk I/O
    }
    auto mesh = MakeShared<Mesh>(Utf8String(path));
    if (!Mesh::TryLoadFromObj(path, *mesh)) {
        return SharedPtr<Mesh>();
    }
    meshCache.Add(key, mesh);
    return mesh;
}
```

Spawn ten rocks with the same path → ten `MeshComponent`s, one `SharedPtr<Mesh>`, one pool slice at upload time.

## 2. Core Concepts

### OBJ as separate index pools

Wavefront OBJ stores **positions**, **normals**, and **UVs** in independent arrays. Face lines reference triples `v/vt/vn` (1-based in file). The loader must expand each unique corner combination into one interleaved GPU vertex.


### Face formats and triangulation

Spark's `TryLoadFromObj` accepts `v`, `v/vt`, `v//vn`, and `v/vt/vn` corners. n-gons fan-triangulate from corner 0: triangles `(0,1,2)`, `(0,2,3)`, … preserving winding from the face loop.


### No welding in Spark OBJ path

Unlike textbook welders, Spark **appends one vertex per face corner** during OBJ parse — simpler code, more vertices, correct hard edges when normals differ per corner. Acceptable for classroom props.


### Fallback attributes

Missing normals default to `UnitZ`; missing UVs stay zero. Malformed face corners are skipped rather than aborting the whole file — partial meshes beat a hard crash in demo tooling.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### TryLoadFromObj static method on Mesh

**Concept:** OBJ as separate index pools

**Decision:** TryLoadFromObj static method on Mesh

**Why:** Pure function: path → Mesh. No ECS or Vulkan. GameWorldAssetCache calls it for .obj paths.

**In Spark:** `Mesh::TryLoadFromObj` — Mesh.hpp lines 68–72

### ParseFaceCorner helper

**Concept:** Face formats and triangulation

**Decision:** ParseFaceCorner helper

**Why:** Handles slash-separated indices and negative indices (relative to end of pool).

**In Spark:** `src/spark/scene/mesh/Mesh.cpp` — face loop inside TryLoadFromObj

### PushBack per corner, not hash map

**Concept:** No welding in Spark OBJ path

**Decision:** PushBack per corner, not hash map

**Why:** Trade memory for implementation clarity; glTF is the shipping path for production assets.

**In Spark:** `TryLoadFromObj` — cornerIndices.PushBack per face corner

### UnitZ normal when vn missing

**Concept:** Fallback attributes

**Decision:** UnitZ normal when vn missing

**Why:** Loader succeeds on position-only OBJ; artist fixes normals in DCC or recomputes later.

**In Spark:** `TryLoadFromObj` — vert.normal = Vector3::UnitZ branch

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scene/mesh/Mesh.hpp`
- `src/spark/scene/mesh/Mesh.cpp`
- `src/spark/scene/GameWorldAssetCache.cpp`

### Step 1 — Read the public OBJ API

*Minimal OBJ subset — v, vn, vt, f.*

```cpp
// include/spark/scene/mesh/Mesh.hpp (lines 68–72)
    /**
     * Loads positions, normals, UVs and triangulated faces from a minimal OBJ subset.
     * Supports `v`, `vn`, `vt`, `f` (v, v/vt, v//vn, v/vt/vn). Returns false on I/O or parse failure.
     */
    [[nodiscard]] static bool TryLoadFromObj(const char* path, Mesh& outMesh);
```

### Step 2 — Trace the parse loop

*Stream lines — fill position/normal/UV pools, begin face expansion.*

```cpp
// src/spark/scene/mesh/Mesh.cpp (lines 359–413)
bool Mesh::TryLoadFromObj(const char* path, Mesh& outMesh) {
    if (path == nullptr) {
        return false;
    }
    std::ifstream file(path);
    if (!file) {
        return false;
    }

    Array<Vector3> positions;
    Array<Vector3> normals;
    Array<Vector2> texCoords;
    outMesh.Clear();

    std::string line;
    while (std::getline(file, line)) {
        if (line.empty()) {
            continue;
        }
        std::istringstream iss(line);
        std::string tag;
        iss >> tag;
        if (tag == "v") {
            float x = 0.0F;
            float y = 0.0F;
            float z = 0.0F;
            iss >> x >> y >> z;
            positions.PushBack({x, y, z});
        } else if (tag == "vn") {
            float x = 0.0F;
            float y = 0.0F;
            float z = 0.0F;
            iss >> x >> y >> z;
            normals.PushBack({x, y, z});
        } else if (tag == "vt") {
            float u = 0.0F;
            float v = 0.0F;
            iss >> u >> v;
            texCoords.PushBack({u, v});
        } else if (tag == "f") {
            Array<int> faceV;
            Array<int> faceT;
            Array<int> faceN;
            std::string corner;
            while (iss >> corner) {
                int vi = 0;
                int ti = 0;
                int ni = 0;
                if (!ParseFaceCorner(corner.c_str(), vi, ti, ni)) {
                    continue;
                }
                faceV.PushBack(vi);
                faceT.PushBack(ti);
                faceN.PushBack(ni);
            }
```

### Step 3 — Follow corner → vertex expansion

*Each face corner becomes one Mesh::Vertex + index.*

```cpp
// src/spark/scene/mesh/Mesh.cpp (lines 420–438)
            for (std::size_t idx = 0; idx < n; ++idx) {
                const int pi = faceV[idx] - 1;
                const int ti = faceT[idx] - 1;
                const int ni = faceN[idx] - 1;
                Vertex vert{};
                if (pi >= 0 && static_cast<std::size_t>(pi) < positions.GetSize()) {
                    vert.position = positions[static_cast<std::size_t>(pi)];
                }
                if (ni >= 0 && static_cast<std::size_t>(ni) < normals.GetSize()) {
                    vert.normal = normals[static_cast<std::size_t>(ni)];
                } else {
                    vert.normal = Vector3::UnitZ;
                }
                if (ti >= 0 && static_cast<std::size_t>(ti) < texCoords.GetSize()) {
                    vert.texCoord = texCoords[static_cast<std::size_t>(ti)];
                }
                const std::uint32_t vIndex = static_cast<std::uint32_t>(outMesh.vertices.GetSize());
                outMesh.vertices.PushBack(vert);
                cornerIndices.PushBack(vIndex);
```

### Step 4 — Load through GameWorldAssetCache

*LoadMesh — cache miss calls TryLoadFromObj for non-glTF paths.*

```cpp
// src/spark/scene/GameWorldAssetCache.cpp (lines 47–63)
SharedPtr<Mesh> GameWorldAssetCache::LoadMesh(const char* path) {
    if (path == nullptr || path[0] == '\0') {
        return SharedPtr<Mesh>();
    }
    if (IsGltfPath(path)) {
        return LoadGltf(path).mesh;
    }
    const Utf8String key(path);
    if (const SharedPtr<Mesh>* cached = meshCache.Find(key)) {
        return *cached;
    }
    auto mesh = MakeShared<Mesh>(Utf8String(path));
    if (!Mesh::TryLoadFromObj(path, *mesh)) {
        return SharedPtr<Mesh>();
    }
    meshCache.Add(key, mesh);
    return mesh;
```

### Step 5 — Spawn an OBJ prop in your engine

```cpp
SharedPtr<Mesh> rock = cache.LoadMesh("assets/props/rock.obj");
if (rock) {
    auto* o = world.CreateGameObject();
    o->AddComponent<MeshComponent>(rock, SceneMeshSlot::Custom, Vector3::One);
}
```

Custom slot triggers VulkanCustomMeshPool pack on submit.

### Step 6 — Validate winding and culling

If the mesh renders inside-out, compare triangle order to Spark's glTF winding flip logic — OBJ files from Blender may need a reverse-Z or cull-mode check in your mini-engine.

### Step 7 — Milestone check — textured OBJ on ground

One OBJ prop at `SceneMeshSlot::Custom` with albedo tint or baseColor texture. Bounds should frame the object without manual scale guessing.

## 5. How This Connects to the Rest of Your Engine

**Loading glTF** replaces OBJ for PBR and skinned characters. **Asset Caching** deduplicates the loaded Mesh
by path. **Custom Mesh Pool** uploads the result when submit marks `customMesh` on the draw item.

## 6. Pitfalls and Debugging

- **Forgetting 1-based OBJ indices** — off-by-one corrupts geometry; Spark subtracts 1 when indexing pools.
- **Expecting welding** — vertex count may exceed unique positions; normal hard edges need separate corners.
- **Huge ASCII files** — prefer glTF for production; OBJ is a teaching stepping stone.
- **Missing file** — TryLoadFromObj returns false; check LoadMesh empty SharedPtr before spawning.

## 7. Exercises

1. Load a cube OBJ; compare vertex count to CreateUnitCube.
2. Break one face line; confirm loader skips bad corners without crashing.
3. Load the same path twice via cache; assert SharedPtr address equality.

## Summary

OBJ loading expands face corners into Mesh vertices with triangulation; cache and Custom slot integrate with the renderer pool.

---

Next: [Loading glTF](03-load-gltf.md).
