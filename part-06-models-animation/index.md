# Part 6: Models and Animation

Load static and skinned glTF assets, cache them on the CPU, upload dynamic geometry each frame, sample skeletal animation, and drive locomotion with a finite-state machine — the content pipeline from disk file to deforming mesh on screen.

| # | Chapter |
|---|---------|
| 1 | [Mesh Data Structures](01-mesh-data.md) |
| 2 | [Loading OBJ](02-load-obj.md) |
| 3 | [Loading glTF](03-load-gltf.md) |
| 4 | [Asset Caching](04-asset-cache.md) |
| 5 | [Custom Mesh Pool](05-custom-mesh-pool.md) |
| 6 | [Skeleton and Bones](06-skeleton.md) |
| 7 | [Skinned glTF](07-skinned-gltf.md) |
| 8 | [Animation Clips](08-animator.md) |
| 9 | [GPU Skinning](09-gpu-skinning.md) |
| 10 | [Locomotion FSM](10-locomotion-fsm.md) |

**Arc:** CPU `Mesh` / `SkinnedMesh` → path-keyed `GameWorldAssetCache` → per-frame `VulkanCustomMeshPool` pack → `AnimatorComponent` joint palette → `SkinSSBO` in `scene.vert` → `Character3DAnimFsm` selects walk/run clips.

Continue to [Part VII — Sky and Materials](../part-07-sky-materials/index.md).
