# Loading glTF

> **Build-Along Milestone:** Textured rigid glTF prop renders with correct winding and scale on the ground.

**You will learn:**

- Trace cgltf parse → AppendPrimitive → Mesh for rigid glTF.
- Explain winding flip and baked node transforms on import.
- Load GltfAsset with mesh and baseColorTexture from GameWorldAssetCache.

## 1. The Problem We Are Solving

glTF is the interchange format for PBR props and environments. Your engine must decode accessors into the same
`Mesh` type procedural geometry uses — or you maintain parallel render paths forever.

Build along: import a rigid `.glb`, spawn it with `MeshComponent` + `MaterialComponent`, and scale from bounds.

## 2. Core Concepts

### cgltf parse pipeline

Spark uses **cgltf** to parse `.gltf`/`.glb`, load external buffers, then walk scene nodes. `TryLoadFromGltf` bakes node transforms into vertices — output is one world-space mesh suitable for rigid `MeshComponent` drawing.


### Accessor → vertex attributes

Each primitive reads POSITION (required), NORMAL, TEXCOORD_0 via `cgltf_accessor_read_float`. Indices come from the primitive's index accessor or generate sequential triplets when absent.


### Winding correction

glTF front faces are CCW; Spark built-ins use opposite winding for Vulkan culling. `ShouldFlipGltfTriangleWinding` mirrors triangle index order when the bake matrix determinant is non-negative.


### Base color texture extraction

First usable PBR `baseColorTexture` (PNG/JPEG from buffer or URI) decodes into `SharedPtr<Texture2D>`. Materials become ECS `MaterialComponent` data later — not GPU objects inside the loader.


### Y-up, meters, baked transforms

glTF is Y-up. Spark bakes node hierarchy into vertices for rigid import — document any extra root rotation you apply in gameplay so animation and static props stay aligned.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Mesh::TryLoadFromGltf with cgltf

**Concept:** cgltf parse pipeline

**Decision:** Mesh::TryLoadFromGltf with cgltf

**Why:** One entry point for .glb and .gltf; shares buffer load path with skinned importer.

**In Spark:** `src/spark/scene/mesh_gltf.cpp`

### AppendPrimitive per cgltf_primitive

**Concept:** Accessor → vertex attributes

**Decision:** AppendPrimitive per cgltf_primitive

**Why:** Triangular primitives only; skips lines/points/strips to keep index buffer uniform.

**In Spark:** `AppendPrimitive` — POSITION/NORMAL/TEXCOORD_0 loop

### Conditional index swap on import

**Concept:** Winding correction

**Decision:** Conditional index swap on import

**Why:** Prevents inside-out glTF props without forcing artists to re-export winding.

**In Spark:** `ShouldFlipGltfTriangleWinding` + AddTriangle index order

### TryLoadFirstBaseColorTexture side channel

**Concept:** Base color texture extraction

**Decision:** TryLoadFirstBaseColorTexture side channel

**Why:** GltfAsset bundles mesh + texture for one cache entry; CharacterCameraDemo trees use this path.

**In Spark:** `GameWorldAssetCache::LoadGltf`

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/mesh_gltf.cpp`
- `src/spark/scene/GameWorldAssetCache.cpp`
- `src/spark/demo/CharacterCameraDemo.cpp`

### Step 1 — Read TryLoadFromGltf entry

*Parse file, visit scene roots, accumulate vertices.*

```cpp
// src/spark/scene/mesh_gltf.cpp (lines 236–265)
bool Mesh::TryLoadFromGltf(const char* path, Mesh& outMesh, SharedPtr<Texture2D>* outBaseColor) {
    if (path == nullptr || path[0] == '\0') {
        return false;
    }
    cgltf_options options{};
    cgltf_data* data = nullptr;
    if (cgltf_parse_file(&options, path, &data) != cgltf_result_success || data == nullptr) {
        return false;
    }
    if (cgltf_load_buffers(&options, data, path) != cgltf_result_success) {
        cgltf_free(data);
        return false;
    }

    outMesh.Clear();
    outMesh.GetName() = Utf8String(path);

    cgltf_scene* primary = data->scene;
    if (primary == nullptr && data->scenes_count > 0) {
        primary = &data->scenes[0];
    }
    auto visitSceneRoots = [&](cgltf_scene* sc) {
        if (sc == nullptr) {
            return;
        }
        for (cgltf_size i = 0; i < sc->nodes_count; ++i) {
            VisitNode(sc->nodes[i], Matrix4::Identity, outMesh);
        }
    };
    visitSceneRoots(primary);
```

### Step 2 — Study AppendPrimitive attribute read

*POSITION/NORMAL/UV accessors → Mesh::Vertex in baked world space.*

```cpp
// src/spark/scene/mesh_gltf.cpp (lines 48–90)
void AppendPrimitive(const cgltf_primitive* prim, const Matrix4& world, Mesh& outMesh) {
    if (prim == nullptr || prim->type != cgltf_primitive_type_triangles) {
        return;
    }
    const cgltf_accessor* pos = nullptr;
    const cgltf_accessor* nrm = nullptr;
    const cgltf_accessor* uv = nullptr;
    for (cgltf_size ai = 0; ai < prim->attributes_count; ++ai) {
        const cgltf_attribute& a = prim->attributes[ai];
        if (a.type == cgltf_attribute_type_position) {
            pos = a.data;
        } else if (a.type == cgltf_attribute_type_normal) {
            nrm = a.data;
        } else if (a.type == cgltf_attribute_type_texcoord && a.index == 0) {
            uv = a.data;
        }
    }
    if (pos == nullptr || pos->type != cgltf_type_vec3) {
        return;
    }

    const cgltf_size vcount = pos->count;
    const std::uint32_t base = static_cast<std::uint32_t>(outMesh.GetVertices().GetSize());
    const bool flipWinding = ShouldFlipGltfTriangleWinding(world);

    for (cgltf_size vi = 0; vi < vcount; ++vi) {
        float p[3]{};
        cgltf_accessor_read_float(pos, vi, p, 3);
        const Vector3 pw = world.TransformPoint({p[0], p[1], p[2]});
        Vector3 nw{0.0F, 1.0F, 0.0F};
        if (nrm != nullptr && nrm->type == cgltf_type_vec3) {
            float n[3]{};
            cgltf_accessor_read_float(nrm, vi, n, 3);
            nw = world.TransformVector({n[0], n[1], n[2]}).Normalized();
        }
        Vector2 tc{0.0F, 0.0F};
        if (uv != nullptr && uv->type == cgltf_type_vec2) {
            float t[2]{};
            cgltf_accessor_read_float(uv, vi, t, 2);
            tc = {t[0], t[1]};
        }
        outMesh.AddVertex({pw, nw, tc});
    }
```

### Step 3 — Follow index triangulation + winding flip

*Indexed triangles with optional winding mirror.*

```cpp
// src/spark/scene/mesh_gltf.cpp (lines 92–107)
    if (prim->indices != nullptr) {
        const cgltf_accessor* idx = prim->indices;
        const cgltf_size icount = idx->count;
        if (icount % 3 != 0) {
            return;
        }
        for (cgltf_size ti = 0; ti < icount; ti += 3) {
            const std::uint32_t i0 = static_cast<std::uint32_t>(cgltf_accessor_read_index(idx, ti + 0));
            const std::uint32_t i1 = static_cast<std::uint32_t>(cgltf_accessor_read_index(idx, ti + 1));
            const std::uint32_t i2 = static_cast<std::uint32_t>(cgltf_accessor_read_index(idx, ti + 2));
            if (flipWinding) {
                outMesh.AddTriangle(base + i0, base + i2, base + i1);
            } else {
                outMesh.AddTriangle(base + i0, base + i1, base + i2);
            }
        }
```

### Step 4 — Load via cache as GltfAsset

*LoadGltf — mesh + baseColorTexture cached by path.*

```cpp
// src/spark/scene/GameWorldAssetCache.cpp (lines 66–84)
GltfAsset GameWorldAssetCache::LoadGltf(const char* path) {
    GltfAsset empty{};
    if (path == nullptr || path[0] == '\0') {
        return empty;
    }
    const Utf8String key(path);
    if (const GltfAsset* cached = gltfCache.Find(key)) {
        return *cached;
    }
    auto mesh = MakeShared<Mesh>(Utf8String(path));
    SharedPtr<Texture2D> tex;
    if (!Mesh::TryLoadFromGltf(path, *mesh, &tex)) {
        return empty;
    }
    GltfAsset asset{};
    asset.mesh = mesh;
    asset.baseColorTexture = tex;
    gltfCache.Add(key, asset);
    return asset;
```

### Step 5 — Spawn a glTF tree like CharacterCameraDemo

*Transform + MeshComponent Custom slot + MaterialComponent with baseColor.*

```cpp
// src/spark/demo/CharacterCameraDemo.cpp (lines 91–103)
    Spark::GameObject* tree = w.CreateGameObject();
    tree->GetName() = Spark::Utf8String(name);
    Spark::TransformComponent* tr = tree->AddComponent<Spark::TransformComponent>();
    tr->SetUniformScale(uniformScale);
    tr->SetTranslation({
            placement.x,
            kCharGroundPlaneY - kTreeGroundAnchorLocalY * uniformScale,
            placement.z});
    tr->SetRotation(Spark::Quaternion::FromAxisAngle(Spark::Vector3::UnitY, placement.yawRadians));
    const Spark::Vector3 meshAlbedo =
            asset.baseColorTexture ? Spark::Vector3::One : Spark::Vector3{0.42F, 0.68F, 0.34F};
    tree->AddComponent<Spark::MeshComponent>(asset.mesh, Spark::SceneMeshSlot::Custom, meshAlbedo);
    if (asset.baseColorTexture) {
```

### Step 6 — Scale using bounds

```cpp
Vector3 bmin, bmax;
if (asset.mesh->TryComputeAxisAlignedBounds(bmin, bmax)) {
    const float h = bmax.y - bmin.y;
    tr->SetUniformScale(targetHeight / h);
}
```

### Step 7 — Compare .glb vs .gltf

Load the same asset in both containers; vertex/index counts should match. External .bin + images exercise URI resolution in cgltf_load_buffers.

### Step 8 — Milestone check — textured glTF prop

One rigid glTF (e.g. tree) on the ground with PBR albedo from import. Custom slot upload visible in RenderDoc after submit.

## 5. How This Connects to the Rest of Your Engine

**Asset Caching** stores `GltfAsset` by path. **Skinned glTF** uses a parallel importer with JOINTS_0/WEIGHTS_0.
Part VII **MaterialComponent** maps metallic/roughness from glTF materials at submit time.

## 6. Pitfalls and Debugging

- **Ignoring accessor byteStride** — interleaved glTF buffers misalign attributes; cgltf_read handles stride.
- **Double axis conversion** — baking + extra root rotation offsets animation; measure once in bind pose.
- **Draco compression** — unsupported without extension; fail gracefully with loader message.
- **Wrong material albedo** — multi-material files need per-primitive texture pick (skinned path does this).

## 7. Exercises

1. Import a glTF cube; compare bounds to CreateUnitCube.
2. List accessor types for one Tree glTF primitive in RenderDoc or cgltf inspect tool.
3. Disable baseColor texture; confirm flat albedo tint still draws.

## Summary

Rigid glTF import bakes nodes into Mesh via cgltf; GltfAsset pairs geometry and baseColor for cache + spawn.

---

Next: [Asset Caching](04-asset-cache.md).
