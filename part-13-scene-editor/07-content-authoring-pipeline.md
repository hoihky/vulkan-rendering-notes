# Content Authoring Pipeline

> **Build-Along Milestone:** Import a `.glb` through the editor, save a `.sparkscene` prefab, and instantiate it in a level scene with correct materials and transforms.

**Spark API note:** Placed props draw after **`world.Rendering().SubmitLit`** rebuilds `SceneRenderParams` from ECS. Pipeline diagrams may label **`FillStandardLitSceneFromWorld`** — that legacy step is **`LitSceneSubmitter`** inside `WorldRendering`.

**You will learn:**

- Trace the full authoring loop: DCC export → asset library → prefab → level scene → runtime expansion.
- Use `GltfPrefabImporter` to convert glTF files into reusable `.sparkscene` prefabs.
- Understand how `gltf_scene` components defer heavy import until scene Apply.
- Connect editor catalog, placement, and serialization to the glTF → mesh → submit → GPU path from Parts VI–VII.

## 1. The Problem We Are Solving

Game content is not authored in C++. Artists export from Blender, Maya, or Substance, then designers place props in levels. Your engine needs a **content authoring pipeline** that:

1. Accepts interchange files (glTF) into a stable asset library layout.
2. Produces reusable prefabs designers can drag into scenes.
3. Serializes only ECS data + relative paths — never GPU handles.
4. Rebuilds meshes and materials at load time through the same import path runtime spawning uses.

Spark's scene editor implements this with `GltfPrefabImporter`, `SceneEditorAssetCatalog`, and `gltf_scene` snapshot handlers. This chapter ties Part XIII serialization to the glTF and material pipelines from Parts VI–VII.

### Who does what in a real studio pipeline

| Role | Tool | Output | Spark equivalent |
|------|------|--------|------------------|
| Artist | Blender / Maya | `.glb` model | `assets/models/chair.glb` |
| Technical artist | Engine import | Prefab / material library | `GltfPrefabImporter` → `prefabs/chair.sparkscene` |
| Level designer | In-engine editor | Level scene file | `scenes/level01.sparkscene` |
| Programmer | Engine runtime | GPU frames | `SceneManager` + `VulkanRenderer` |

As a programmer learning Vulkan, you are building the **last row** — but your code only works if the rows above produce consistent file formats. This chapter explains how Spark connects artist exports to GPU draws without special-case editor rendering.

### The golden rule: never save GPU data

Beginners sometimes try to "save the rendered mesh" or serialize `VkBuffer` handles. That cannot work:

- GPU memory addresses change every run.
- Different GPUs have different memory layouts.
- A saved buffer does not include material or transform state.

Spark saves **references**:

```text
C gltf_scene "models/chair.glb"     ← path to source asset
C transform 1 1 1 0 0 0 ...       ← where to place it
C multi_material ...                ← material library keys (not pixels)
```

On load, the engine **re-imports** the glTF through the same code path as the first time you dragged the asset into the editor. If import breaks, both editor and runtime break the same way — which is good for debugging.

### Day-one workflow: import your first prop

Follow these steps literally when learning Spark's editor:

1. Export a cube or chair from Blender as `.glb` (Y-up, apply transforms).
2. Copy file to `assets/models/chair.glb` in the Spark project.
3. Run editor import (or `GltfPrefabImporter::ImportFromGltf`).
4. Confirm `prefabs/chair.sparkscene` exists with `gltf_scene` line.
5. Open a test scene, place the prefab, save `scenes/test.sparkscene`.
6. Reload the scene — chair should reappear with materials.
7. Capture RenderDoc — verify `vkCmdDrawIndexed` calls and texture layers.

If step 7 fails, walk backward: submit → components → import → file on disk. Do not debug tonemap if the mesh never submitted.

## 1b. Concept Deep Dive — GltfPrefabImporter Pipeline

### glTF → portable prefab without GPU handles

`GltfPrefabImporter::ImportFromGltf` validates the source file, computes bounds for uniform scale, and writes a `.sparkscene` prefab with a `gltf_scene` component — not mesh vertices:

```cpp
// include/spark/scene/prefab/GltfPrefabImporter.hpp
class GltfPrefabImporter final {
    [[nodiscard]] GltfPrefabImportResult ImportFromGltf(
            GameWorld& world,
            const char* gltfPath,
            const GltfPrefabImportOptions& options = {}) const;
};
```

Default `targetMaxExtent = 2.2F` scales oversized DCC exports to consistent world units; `groundYOffset` lifts the root so feet sit on Y=0.

### BuildPrefabDocument emits deferred import

The importer writes ECS lines only:

```cpp
// src/spark/scene/prefab/GltfPrefabImporter.cpp
const SceneDocument document = BuildPrefabDocument(
        baseName, options.assetsRoot, result.gltfAssetRel.CStr(),
        uniformScale, groundYOffset);
SceneSerializer serializer{};
serializer.WriteToFile(document, buildPrefabPath.CStr());
```

On level load, `gltf_scene` snapshot handler requests the glTF through `GameWorldAssetLoader` — same path as runtime `LoadGltf`. Editor and game share one import stack from Parts VI–VII.

### SceneEditorAssetCatalog scans prefabs and scenes

`SceneEditorAssetCatalog::Refresh` walks `prefabs/` and `scenes/` under the build assets root. The project panel lists entries; placement tools instantiate prefabs into the live world through `PrefabInstantiator`, which expands `gltf_scene` into mesh + material components at apply time.

## 2. Core Concepts

### Authoring loop overview

```text
┌─────────────┐    ┌──────────────────┐    ┌─────────────────┐
│ DCC export  │───►│ assets/models/   │───►│ GltfPrefabImporter │
│ (.glb)      │    │   chair.glb      │    │ → prefabs/chair.sparkscene
└─────────────┘    └──────────────────┘    └────────┬────────┘
                                                    │
┌─────────────┐    ┌──────────────────┐             ▼
│ Level design│◄───│ scenes/level01   │◄─── ScenePlacementActions
│ in editor   │    │   .sparkscene    │     (instantiate prefab)
└─────────────┘    └────────┬─────────┘
                            │ Apply at runtime
                            ▼
                   GltfSceneSourceComponent expands
                   → GltfAssetBinder → Mesh + MultiMaterial
                   → SceneSubmit → GPU draw
```

The critical rule: **prefabs store references, not baked geometry**. A `gltf_scene` component holds a relative path; `GltfAssetBinder::BindFromPathAsChild` performs the full import when the scene loads.

### glTF → prefab document

`GltfPrefabImporter::ImportFromGltf` stages the glTF into the asset library, optionally loads bounds for scale/ground offset, then writes a minimal `SceneDocument` with two components: `transform` and `gltf_scene`.

```cpp
// src/spark/scene/prefab/GltfPrefabImporter.cpp (lines 102–145)
SceneDocument BuildPrefabDocument(
        const Utf8String& displayName,
        const char* assetsRoot,
        const char* gltfAssetRel,
        const float uniformScale,
        const float groundYOffset) noexcept {
    SceneDocument document{};
    // ...
    ComponentRecord gltfScene{};
    gltfScene.kind = Utf8String("gltf_scene");
    char gltfPayload[512]{};
    std::snprintf(gltfPayload, sizeof(gltfPayload), "\"%s\"", gltfAssetRel);
    gltfScene.payload = Utf8String(gltfPayload);
    entity.components.PushBack(MoveTemp(transform));
    entity.components.PushBack(MoveTemp(gltfScene));
    document.entities.PushBack(MoveTemp(entity));
    return document;
}
```

The prefab file on disk looks like:

```text
spark_scene_v4
1
H name "Chair"
E 1 -1 "Chair"
C transform 1.0 1.0 1.0 0.0 0.0 0.0 0.0 0.0 0.0 1.0
C gltf_scene "models/chair.glb"
```

No mesh vertices appear in the text file — only the path.

### Runtime expansion: gltf_scene → live hierarchy

When `SceneDeserializer::Apply` restores a `gltf_scene` component, the snapshot handler calls `GltfAssetBinder::BindFromPathAsChild`, which:

1. Resolves the path relative to `assetsRoot`.
2. Loads `GltfSceneDocument` from cache (`GameWorldAssetCache::TryLoadGltfScene`).
3. Walks the glTF node tree, spawning child `GameObject`s with transforms.
4. Attaches `MeshComponent` / `SkinnedMeshComponent` per node.
5. Binds `MaterialComponent` or `MultiMaterialComponent` via `GltfAssetBindings`.

Multi-material chairs get `MultiMaterialComponent` automatically when `GetSubmeshes().GetSize() > 1` (Part VI Chapter 3).

### Editor asset catalog

`SceneEditorAssetCatalog` scans project folders at editor startup:

```cpp
// src/spark/scene/editor/SceneEditorAssetCatalog.cpp (concept)
ScanDirectory(..., "prefabs", SceneEditorAssetKind::Prefab);
ScanDirectory(..., "scenes", SceneEditorAssetKind::Scene);
```

The asset browser lists `.sparkscene` files. `GltfImportService` wraps `GltfPrefabImporter` for "Import glTF" menu actions — artists drop a `.glb` into the library and receive a prefab without writing serialization code by hand.

### Material library keys across authoring

Imported glTF materials register in `GameWorldAssetCache` under keys like:

```text
models/chair.glb#material/0
models/chair.glb#material/1
```

`multi_material` snapshot handlers serialize these keys so round-trip save/load preserves slot assignments. Editing a material in the inspector updates the slot; Capture writes the library key back to disk.

## 3. Design Decisions

### Prefab stores gltf_scene reference, not mesh bytes

**Concept:** Authoritative ECS, derived GPU state

**Decision:** `gltf_scene` component with relative path payload

**Why:** Re-importing an updated `.glb` automatically picks up new geometry on next load without re-saving every level.

**In Spark:** `GltfPrefabImporter::BuildPrefabDocument`, `ComponentSnapshotHandlersRendering.cpp`

### GltfAssetPathResolver normalizes library paths

**Concept:** Portable asset paths

**Decision:** Copy glTF + sidecar files into `assets/` and store relative paths only

**Why:** Absolute paths break on other machines; relative paths survive version control and CI.

**In Spark:** `GltfAssetPathResolver::EnsureInAssetLibrary`

### SceneManager async instantiate

**Concept:** Non-blocking level load

**Decision:** `InstantiatePrefab` queues async parse; `Pump` applies on main thread

**Why:** Large glTF scenes parse on worker threads; ECS mutation stays main-thread only.

**In Spark:** `SceneManager::InstantiatePrefab`, Chapter 3 async loading

### Inspector edits components, not SceneRenderParams

**Concept:** Editor mutates ECS

**Decision:** `MaterialInspectorWidget` calls `MaterialComponent` setters → `MeshDirty` signal

**Why:** Submit rebuilds draw lists next frame; no special "editor render path."

**In Spark:** `src/spark/editor/inspector/widgets/MaterialInspectorWidget.cpp`

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `src/spark/scene/prefab/GltfPrefabImporter.cpp`
- `src/spark/scene/editor/GltfImportService.cpp`
- `src/spark/scene/editor/SceneEditorAssetCatalog.cpp`
- `src/spark/scene/editor/ScenePlacementActions.cpp`
- `src/spark/scene/serialization/ComponentSnapshotHandlersRendering.cpp`
- `src/spark/scene/assets/gltf/GltfAssetBindings.cpp`

### Step 1 — Import glTF to prefab

Run editor import (or call `GltfPrefabImporter::ImportFromGltf`). Verify:

- `.glb` copied under `assets/models/`.
- `prefabs/<name>.sparkscene` created with `gltf_scene` line.
- Bounds-based uniform scale and ground Y offset applied to transform payload.

### Step 2 — Place prefab in level scene

Use `ScenePlacementActions` or drag from asset browser. Editor calls `SceneManager::InstantiatePrefab`, which async-loads the prefab document and Apply-spawns entities.

### Step 3 — Verify runtime expansion

After Pump completes, inspect the hierarchy:

- Child objects match glTF node names.
- Meshes use `SceneMeshSlot::Custom`.
- Multi-material assets have `MultiMaterialComponent` with one slot per glTF material.

### Step 4 — Save level and round-trip

Save `scenes/level01.sparkscene`. Reload — prefab instances should restore transforms and `gltf_scene` paths. GPU meshes rebuild through the same import path as first placement.

### Step 5 — Trace one draw to GPU

Pick one placed prop. Follow the pipeline from Part VI Chapter 3:

```text
gltf_scene Apply → GltfAssetBinder → MultiMaterialComponent
  → world.Rendering().SubmitLit (LitSceneSubmitter / legacy FillStandardLitSceneFromWorld) → PushRigidMeshDraws
  → VulkanCustomMeshPool → VulkanSceneOpaquePass → HDR → tonemap
```

Confirm in RenderDoc: N draw calls for N submeshes, each with distinct `textureLayer` push values.

### Step 6 — Edit material in inspector

Change roughness on slot 1. `MeshDirty` fires. Next frame's submit updates `SceneDrawItem` without re-importing glTF.

### Step 7 — Re-export glTF from DCC

Replace `assets/models/chair.glb` with an updated export. Reload level — geometry and materials refresh from disk cache invalidation.

## 5. How This Connects to the Rest of Your Engine

| Part | Role in authoring |
|------|-------------------|
| VI | glTF → Mesh + MultiMaterial |
| VII | Material submit + PBR push constants |
| IV | Texture array upload (binding 1) |
| X | HDR scene pass receives authored content |
| XIII Ch 1–3 | Scene format, snapshots, async load |
| XIII Ch 4–6 | Editor shell and panels |

The authoring pipeline is complete when **every visual property** an artist can edit in the inspector round-trips through Capture → disk → Apply → submit → GPU without a separate "editor renderer."

## 6. Pitfalls and Debugging

- **Absolute glTF paths in prefab** — break on other machines; always run through `GltfAssetPathResolver`.
- **Baking mesh into scene file** — bloated saves, stale geometry; use `gltf_scene` references.
- **Missing multi_material handler** — multi-submesh props load with default gray material on slots 1+.
- **Import on render thread** — hitch; use async `TryLoadGltfScene` + Pump pattern from Chapter 3.
- **Stale asset cache** — updated `.glb` not visible until cache key invalidated or process restart.

## 7. Exercises

1. Import a multi-material `.glb`; confirm prefab has only `gltf_scene`, not mesh `C` lines.
2. Place three instances at different transforms; verify one cached `GltfSceneDocument` shared by all.
3. Edit slot 0 roughness in inspector; capture frame before/after in RenderDoc push constants.
4. List files touched when importing `tree.glb` (model, prefab, cache entries).
5. Document your team's folder convention: `assets/`, `prefabs/`, `scenes/`.

## Summary

Spark's content authoring pipeline imports glTF into the asset library, writes lightweight `.sparkscene` prefabs with `gltf_scene` references, and expands them at runtime through the same glTF → ECS → submit → GPU path used everywhere else. The editor catalogs prefabs and scenes, places instances, and serializes component state — never GPU resources.

---

Next: [Part XIV — Physics](../part-14-physics/index.md).
