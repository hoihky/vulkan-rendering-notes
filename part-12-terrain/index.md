# Part 12: Terrain and Outdoor Scenes

Generate heightfield meshes on the CPU, wire them through ECS and the lit scene pass, cull large outdoor draw lists, and plan streaming for open worlds.

| # | Chapter |
|---|---------|
| 1 | [Heightfield Generation](01-heightfield.md) |
| 2 | [TerrainComponent](02-terrain-component.md) |
| 3 | [Terrain in Scene Pass](03-terrain-lighting.md) |
| 4 | [Frustum Culling](04-frustum-cull.md) |
| 5 | [Streaming Roadmap](05-open-world.md) |

**Arc:** `TerrainMeshGenerator` fBM → `TerrainComponent` brush/regenerate → opaque lit pass + materials/shadows → `DispatchDrawableFrustumCull` → open-world chunk/LOD roadmap.

Continue to [Part XIII — Scene and Editor](../part-13-scene-editor/index.md).
