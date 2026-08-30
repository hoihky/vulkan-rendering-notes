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

### What is glTF? (start here if you are new to 3D assets)

**glTF** (GL Transmission Format) is a file format for 3D models — think "PNG for geometry." Artists export from Blender, Maya, or Substance Painter into `.gltf` (JSON + sidecar files) or `.glb` (single binary package).

A glTF file typically contains:

| Piece | What it stores | Why your engine cares |
|-------|----------------|----------------------|
| **Meshes** | Vertex positions, normals, UVs, indices | Becomes your `Mesh` type |
| **Materials** | Base color, metallic, roughness, normal maps | Becomes `MaterialComponent` |
| **Textures** | PNG/JPEG embedded or referenced | Becomes `Texture2D` on CPU |
| **Nodes** | Transform hierarchy (parent/child) | Becomes `GameObject` tree |
| **Scenes** | Which nodes are roots | Entry point for loading |

You almost never hand-author glTF in a text editor. You **import** it. Spark's job is to turn glTF bytes into engine-native structures the renderer already understands.

### The two import paths in Spark (which one to use)

**Path A — Quick prop (`Mesh::TryLoadFromGltf`):**

- Merges the whole scene into **one** `Mesh`.
- Bakes node transforms into vertex positions.
- Grabs the first base-color texture it finds.
- Good for: trees, rocks, simple props in demos.

**Path B — Full scene (`GltfSceneLoader`):**

- Preserves **node hierarchy** as separate objects.
- Keeps **one mesh with many submeshes** (one per glTF primitive/material).
- Loads **all materials** into a table.
- Good for: characters, furniture with wood/metal parts, entire levels.

If your asset has **multiple materials on one mesh** (a chair with fabric seat + metal legs), you **must** use Path B. Path A would merge geometry but lose per-primitive material assignment.

### Step-by-step: what happens when you load a .glb file

Follow one byte from disk to pixel:

**Step 1 — Parse file (CPU, no GPU yet)**

```cpp
cgltf_parse_file(&options, path, &data);   // read JSON/binary structure
cgltf_load_buffers(&options, data, path);  // load .bin + images into memory
```

cgltf builds an in-memory tree: meshes, materials, accessors pointing at raw byte buffers.

**Step 2 — For each triangle primitive, read attributes**

An **accessor** is glTF's way of saying "there are N vec3 positions starting at byte offset X with stride Y." Spark calls `cgltf_accessor_read_float` for each vertex:

```text
For vertex i:
  read POSITION  → Vector3 position (object space)
  read NORMAL    → Vector3 normal (optional but needed for lighting)
  read TEXCOORD_0 → Vector2 uv (needed for textures)
```

**Step 3 — Transform positions by node hierarchy**

glTF nodes have translation, rotation, scale. Spark multiplies vertices by the accumulated **world matrix** so rigid imports can collapse to one mesh in Path A, or keep per-node transforms in Path B.

**Step 4 — Build index buffer**

Triangles are triplets of vertex indices. Spark stores `uint32_t` indices for `vkCmdDrawIndexed`. If glTF winding does not match Vulkan's cull mode, Spark swaps two indices per triangle (`ShouldFlipGltfTriangleWinding`).

**Step 5 — Record submesh slices (Path B only)**

Each primitive appends a `MeshSubmesh` with `indexOffset`, `indexCount`, `materialIndex`. This tells the renderer "indices 0–599 use material 0, indices 600–1200 use material 1" without duplicating vertices.

**Step 6 — Decode materials (still CPU)**

`GltfMaterialLoader` reads metallic/roughness factors, loads PNGs into `Texture2D`, merges occlusion into ORM when needed. **No VkImage yet.**

**Step 7 — Attach to ECS**

`MeshComponent` holds `SharedPtr<Mesh>`. `MaterialComponent` or `MultiMaterialComponent` holds texture pointers and scalar PBR values.

**Step 8 — Scene submit (every frame)**

`FillStandardLitSceneFromWorld` walks entities, builds `SceneDrawItem` list, assigns texture **array layer indices**.

**Step 9 — GPU upload**

`VulkanCustomMeshPool` copies vertices/indices to `VkBuffer`. `VulkanSceneTextureUploader` copies texels to a `sampler2DArray`.

**Step 10 — Draw**

`VulkanSceneOpaquePass::Record` binds pipeline, descriptors, buffers, push constants, `vkCmdDrawIndexed`.

You will not implement all ten steps in one day. Parts VI–VII split them across chapters; this list is the map.

## 1b. Concept Deep Dive — Spark's Rigid glTF Pipeline

### Content classifier: rigid vs skinned routing

Spark probes every glTF path before choosing a loader. Skinned files must **not** go through `GltfRigidLoader` — they need `LoadSkinnedGltf`:

```cpp
// src/spark/scene/assets/GameWorldAssetCache.cpp (concept)
const GltfContentClassifier::ProbeResult probe = GltfContentClassifier::ProbeFile(path);
if (probe.parseOk && probe.kind == GltfContentKind::Skinned) {
    outcome.errorMessage = "glTF contains skinned mesh; use LoadSkinnedGltf";
    return outcome;
}
```

### Scene graph traversal bakes transforms, skips skinned nodes

`GltfRigidLoader` walks the node tree recursively. Each node's local TRS multiplies into a **world matrix** that is baked into vertex positions at import time. Nodes with `skin != nullptr` are skipped — those belong to the skinned path:

```cpp
// src/spark/scene/assets/gltf/GltfRigidLoader.cpp (lines 14–37)
bool VisitNode(cgltf_data* data, cgltf_node* node, const Matrix4& parentWorld, Mesh& outMesh, ...) {
    const Matrix4 local = GltfNodeTransforms::LocalTransform(node).ToMatrix4();
    const Matrix4 world = parentWorld * local;

    if (node->mesh != nullptr && node->skin == nullptr) {
        const cgltf_mesh* mesh = node->mesh;
        for (cgltf_size pi = 0; pi < mesh->primitives_count; ++pi) {
            GltfMeshBuilder::AppendPrimitive(data, &mesh->primitives[pi], world, outMesh);
        }
    }
    for (cgltf_size ci = 0; ci < node->children_count; ++ci) {
        VisitNode(data, node->children[ci], world, outMesh, ...);
    }
    return true;
}
```

**Why bake transforms?** Rigid props import as one `Mesh` in model space; the entity's `TransformComponent` places them in the world. Skinned characters keep vertices in bind pose and animate joints instead.

### GltfAsset — mesh + materials + variants in one cache entry

```cpp
// include/spark/scene/assets/GameWorldAssetCache.hpp
struct GltfAsset {
    SharedPtr<Mesh> mesh;
    Array<GltfMaterialDesc> materials;       // indexed by primitive material index
    Array<Utf8String> materialVariantNames; // KHR_materials_variants
};
```

`Mesh::TryLoadFromGltf` is a thin wrapper over `GltfRigidLoader` that copies mesh + primary material into caller out-params — demos often use `GameWorld::LoadGltf` instead for the full `GltfAsset`.

### Thin wrapper entry point

```cpp
// src/spark/scene/mesh/mesh_gltf.cpp
bool Mesh::TryLoadFromGltf(const char* path, Mesh& outMesh, ...) {
    GltfRigidLoadResult result{};
    if (!GltfRigidLoader{}.LoadFromFile(path, result) || !result.mesh) {
        return false;
    }
    outMesh = *result.mesh;
    // copy materials to outMaterials if requested
    return true;
}
```

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

### Full scene import vs single-mesh bake

Spark exposes **two** glTF entry points:

| Path | Use case | Output |
|------|----------|--------|
| `Mesh::TryLoadFromGltf` | Simple props (one merged mesh) | Single `Mesh` + optional base color texture |
| `GltfSceneLoader::TryLoadFromFile` | Full scenes with hierarchy | `GltfSceneDocument` — meshes, materials, nodes, skins |

For production assets with **multiple materials per mesh**, prefer the scene loader. `GltfMeshBuilder::BuildFromCgltfMesh` walks each primitive, appends geometry, and records a `MeshSubmesh` per primitive with its glTF `materialIndex`. Materials decode separately via `GltfMaterialLoader::LoadAll` into `GltfMaterialDesc` entries (base color, normal, ORM merge, emissive, KHR extensions).

### Multi-material ECS binding

When a mesh has more than one submesh, `GltfAssetBinder` attaches `MultiMaterialComponent` instead of a single `MaterialComponent`:

```cpp
// src/spark/scene/assets/gltf/GltfAssetBindings.cpp (lines 52–91)
[[nodiscard]] bool ShouldUseMultiMaterialComponent(const GltfAsset& asset) noexcept {
    return static_cast<bool>(asset.mesh) && asset.mesh->GetSubmeshes().GetSize() > 1U;
}

void BindMaterials(GameObject& owner, const GltfAsset& asset, const char* gltfLibraryKey) {
    // ...
    if (ShouldUseMultiMaterialComponent(asset)) {
        if (MultiMaterialComponent* multi = owner.AddComponent<MultiMaterialComponent>()) {
            multi->BindFromGltfAsset(world, libraryKey.CStr(), asset);
        }
        return;
    }
    // single MaterialComponent path for one-submesh assets ...
}
```

Each slot in `MultiMaterialComponent` maps to a glTF material index. Library keys follow `path#material/N` so `GameWorldAssetCache` can share decoded textures across instances.

### End-to-end: glTF file → draw call

Trace this pipeline when debugging a multi-material import:

```text
.glb file
  → cgltf_parse + cgltf_load_buffers (GltfDataLoader)
  → GltfSceneLoader::TryLoadFromFile
       ├── GltfMeshBuilder per cgltf_mesh → Mesh + MeshSubmesh[]
       └── GltfMaterialLoader → GltfMaterial[] (CPU Texture2D pixels)
  → GltfSceneImporter / GltfAssetBinder
       ├── MeshComponent(Custom, SharedPtr<Mesh>)
       └── MultiMaterialComponent OR MaterialComponent
  → FillStandardLitSceneFromWorld (SceneSubmit)
       └── PushRigidMeshDraws — one SceneDrawItem per submesh
  → VulkanCustomMeshPool::UpdateFromScene — pack VBO/IBO + submesh slices
  → VulkanSceneOpaquePass::Record — push constants + vkCmdDrawIndexed
  → scene.frag — sample sceneTextures[layer], ORM, normal map
```

At submit, `PushRigidMeshDraws` loops submeshes and resolves the material slot by `sm.ResolveMaterialIndex(activeVariant)`:

```cpp
// src/spark/scene/submit/SceneSubmit.cpp (lines 101–115)
    for (std::size_t si = 0; si < submeshes.GetSize(); ++si) {
        SceneDrawItem item = baseItem;
        item.submeshIndex = static_cast<std::uint32_t>(si);
        const MeshSubmesh& sm = submeshes[si];
        const std::uint32_t materialIndex = sm.ResolveMaterialIndex(multiMat->GetActiveVariantIndex());
        if (materialIndex < multiMat->GetSlotCount()) {
            const MultiMaterialComponent::Slot& slot = multiMat->GetSlot(materialIndex);
            ApplyMultiMaterialSlotToSceneDrawItem(item, slot, &params);
            ApplyAlbedoTexture(item, slot.baseColor, slot.tint, findOrAddTexture);
        }
        drawList.PushBack(item);
    }
```

On the GPU, `VulkanSceneOpaquePass` copies each draw's layer indices into push constants (`textureLayer`, `normalMapLayer`, `metallicRoughnessMapLayer`) before `vkCmdDrawIndexed`. The fragment shader samples `sceneTextures` and `sceneHdrTextures` by layer — no per-material descriptor sets.

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

**Line-by-line for beginners:**

```cpp
// src/spark/scene/mesh_gltf.cpp — AppendPrimitive (excerpt)
void AppendPrimitive(const cgltf_primitive* prim, const Matrix4& world, Mesh& outMesh) {
```

- `prim` — one glTF primitive (usually one material's triangles).
- `world` — accumulated transform from parent nodes (position/rotation/scale baked into this matrix).

```cpp
    if (prim == nullptr || prim->type != cgltf_primitive_type_triangles) {
        return;  // Spark only supports triangles, not lines or points
    }
```

```cpp
    // Find POSITION, NORMAL, TEXCOORD_0 accessors by scanning attributes
    for (cgltf_size ai = 0; ai < prim->attributes_count; ++ai) {
        const cgltf_attribute& a = prim->attributes[ai];
        if (a.type == cgltf_attribute_type_position) { pos = a.data; }
        // ...
    }
```

```cpp
    const std::uint32_t base = static_cast<std::uint32_t>(outMesh.GetVertices().GetSize());
```

`base` is the vertex index where **this primitive's** vertices start in the shared array. Earlier primitives already appended vertices; indices will offset by `base`.

```cpp
    for (cgltf_size vi = 0; vi < vcount; ++vi) {
        cgltf_accessor_read_float(pos, vi, p, 3);  // read local-space position
        const Vector3 pw = world.TransformPoint({p[0], p[1], p[2]});  // bake transform
```

After this loop, every vertex is in **world space** (for Path A) or mesh space with node transform applied (Path B).

Full reference implementation:

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
4. **Beginner:** Open a `.glb` in [gltf-viewer](https://github.khronos.org/glTF-Sample-Viewer-Release/) and count how many materials it has — predict whether Spark will use `MaterialComponent` or `MultiMaterialComponent`.
5. **Beginner:** Draw the data flow from `assets/models/tree.glb` to `SceneDrawItem` on paper — one box per step from the ten-step list in Section 1.
6. **Intermediate:** Load the same asset via Path A (`Mesh::TryLoadFromGltf`) and Path B (`GltfSceneLoader`) — compare submesh count and material assignment.
7. **Debug:** Intentionally flip winding (disable `ShouldFlipGltfTriangleWinding`) — describe what you see with back-face culling enabled.

## Summary

Rigid glTF import bakes nodes into Mesh via cgltf; GltfAsset pairs geometry and baseColor for cache + spawn.

---

Next: [Asset Caching](04-asset-cache.md).
