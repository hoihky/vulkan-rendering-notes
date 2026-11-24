# Mesh LOD and GPU Instancing

> **Build-Along Milestone:** A prop uses `MeshLodComponent` to switch meshes by distance; a separate entity uses `RigidMeshInstancedBatchComponent` to draw many transforms in one `vkCmdDrawIndexed` instance count.

**You will learn:**

- Resolve LOD tiers at submit time with hysteresis (`MeshLodComponent`).
- Collect rigid mesh instancing into `meshInstancedBatches` / `meshInstances`.
- Trace GPU path: `scene_instanced.vert`, SSBO binding 16, `RecordRigidInstancedBatches`.
- Know LOD vs instancing limits (batch owner cull, no per-instance LOD on GPU yet).

## 1. The Problem We Are Solving

Duplicating `MeshComponent` for every tree in a forest wastes CPU submit and GPU draw calls. **Instancing** shares one mesh/material template and many per-instance transforms.

Hero props need **distance LOD** — high-poly near camera, low-poly or culled far away — without spawning separate entities per tier. Spark applies LOD **during lit submit**, not as a GPU morph pass.

Build along: study `FoliageWindDemo` for `MeshLodComponent` + foliage instancing patterns; study `MeshInstancedSubmitTest` for rigid batch collection.

## 1b. Concept Deep Dive — Submit-Time LOD

### MeshLodComponent

```cpp
// include/spark/ecs/components/rendering/MeshLodComponent.hpp (concept)
// levels[]: MeshLodLevel { minDistance, meshSlot, rigidMesh, skinnedMesh, culled, paletteUpdateStride }
```

`MeshLodSelector::SelectIndex` picks active tier using **hysteresis** (default ~2 m) so LOD does not oscillate at band boundaries. Optional **horizontal-only** distance (XZ) ignores vertical offset for flying cameras.

### Resolve at collect

```cpp
// src/spark/scene/submit/SceneMeshSubmitCollectors.cpp (concept)
MeshLodResolvedDraw resolved = MeshLodResolveRigidDraw(mesh, lod, camera, objectWorld);
if (resolved.culled) return;  // skip draw
baseItem.mesh = resolved.slot;
baseItem.customMesh = resolved.rigidMesh;
```

Skinned path: `MeshLodResolveSkinnedDraw` may override `SkinnedMesh` and `paletteUpdateStride`.

**Not GPU LOD** — switching replaces which CPU mesh/slot reaches `SceneDrawItem`; pool repacks when mesh set changes.

### Mesh LOD limits

- Levels should be sorted by ascending `minDistance`; tier 0 at distance 0.
- `RigidMeshInstancedBatchComponent`: LOD on batch **owner** culls or keeps **entire batch** — no per-instance GPU LOD bucket yet (see Spark `docs/FOLIAGE_ROADMAP.md`).

## 1c. Concept Deep Dive — Rigid Mesh Instancing

### ECS component

`RigidMeshInstancedBatchComponent` — grid of instance transforms; pairs with `MeshComponent` + optional `MaterialComponent`.

### Snapshot arrays

```cpp
// include/spark/engine/SceneRenderParams.hpp (excerpt)
Array<SceneMeshInstancedBatch> meshInstancedBatches;
Array<MeshInstanceRecord> meshInstances;  // each: Matrix4 transform
static constexpr std::uint32_t MaxRigidMeshInstances = 4096;
```

`SceneMeshInstancedBatch` stores a **template** `SceneDrawItem` plus `[instanceBegin, instanceCount)` into `meshInstances`.

### Collector

`MeshInstancedSceneCollector::CollectInto` runs in `LitSceneSubmitter::FinalizeScene` after opaque list build — expands batches without duplicating N draw items in `draws`.

### GPU record

```text
UploadRigidInstances → SSBO binding 16 (VulkanSceneDescriptors)
scene_instanced.vert + scene.frag (same material path)
vkCmdDrawIndexed(..., instanceCount, ..., firstInstance)
push.instanceBase offsets into SSBO
```

**Opaque only** — transparent or transmission materials skip instanced batch record in `RecordRigidInstancedBatches`.

### Foliage instancing (separate path)

`FoliageInstancedMeshComponent` + `VulkanFoliageInstancedPass` use descriptor **binding 15** — different shader/pass from rigid crowds; do not merge with `meshInstancedBatches` mentally.

## 2. Core Concepts

### When to use which pattern

| Need | Pattern |
|------|---------|
| 10 identical rocks, different transforms | Shared `Mesh` + 10 `MeshComponent` OR one instanced batch |
| 10 000 grass blades | `RigidMeshInstancedBatchComponent` or foliage pass |
| Hero mesh + impostor far | `MeshLodComponent` with `culled` or alternate `SharedPtr<Mesh>` per level |
| Skinned crowd | Still per-entity skinned draws; instancing is rigid-focused today |

### Custom mesh pool interaction

Instanced templates referencing `SceneMeshSlot::Custom` still pack through `VulkanCustomMeshPool`; instance transforms do not duplicate vertex buffers.

## 3. Design Decisions

**LOD on CPU at submit** — keeps one shader path; trades streaming/morph complexity for asset tiers.

**Template + SSBO instances** — avoids N push constant updates for N rocks; one material push, many matrices in SSBO.

**Separate foliage pass** — wind animation and alpha cutout may need different pipeline state than opaque instanced props.

## 4. Implementation Walkthrough

### Spark files

- `include/spark/scene/lod/MeshLodLevel.hpp`, `MeshLodSubmit.hpp`
- `include/spark/ecs/components/rendering/MeshLodComponent.hpp`
- `include/spark/ecs/components/rendering/RigidMeshInstancedBatchComponent.hpp`
- `src/spark/scene/submit/MeshInstancedSceneCollector.cpp`
- `shaders/scene_instanced.vert`
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp` — `RecordRigidInstancedBatches`
- `tests/scene/MeshInstancedSubmitTest.cpp`, `MeshLodSubmitTest.cpp`

### Milestone check

RenderDoc: one instanced draw shows `Instance Count > 1`, same VB/IB as non-instanced prop. Move camera across LOD distance — watch `customMesh` pointer or slot change in submit log (dev-only), not mid-draw GPU swap.

## 5. How This Connects to the Rest of Your Engine

Part II Chapter 11 submit calls collectors. Part VII materials flow through template `SceneDrawItem`. Part XII distance ties to LOD `minDistance`. Part IX shadows may cast instanced batches via same mesh transforms (verify shadow pass support for your content).

## 6. Pitfalls and Debugging

- **Expecting instancing for skinned heroes** — use skinned draw path; instancing is rigid.
- **Transparent instanced batch** — silently skipped; split materials or use non-instanced draws.
- **LOD without hysteresis** — pop every frame at boundary; set `SetHysteresisMeters`.

## Summary

**MeshLodComponent** chooses mesh/slot/cull at submit; **RigidMeshInstancedBatchComponent** fills `meshInstancedBatches` + SSBO instanced draw path.

---

Previous: [Locomotion FSM](10-locomotion-fsm.md). Next: [Part VII — Sky and Materials](../part-07-sky-materials/index.md).