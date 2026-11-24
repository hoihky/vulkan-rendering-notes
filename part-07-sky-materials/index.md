# Part 7: Sky and Materials

Render infinite HDR skies, image-based lighting, and per-entity PBR or toon materials — from `SkyComponent` and equirect sampling through `MaterialComponent` texture layers to the standard lit submit path.

| # | Chapter |
|---|---------|
| 1 | [Sky Rendering Modes](01-sky-modes.md) |
| 2 | [Sky Geometry](02-sky-meshes.md) |
| 3 | [Sky Pipeline Branch](03-sky-pipeline.md) |
| 4 | [IBL Basics](04-ibl-basics.md) |
| 5 | [MaterialComponent](05-material-component.md) |
| 6 | [Shading Models](06-shading-models.md) |
| 7 | [Normal Mapping](07-normal-mapping.md) |
| 8 | [ORM Textures](08-orm-textures.md) |
| 9 | [Material Submit](09-material-submit.md) |
| 10 | [Material Showcase Demo](10-material-showcase.md) |

**Arc:** `SceneSkyMode` + view-ray equirect → `pipelineSky` sort/bind → `iblEnvironmentLayer` from sky → `MaterialComponent` maps → `scene.frag` PBR vs toon branch → `world.Rendering().SubmitLit` (`LitSceneSubmitter` gather; legacy `FillStandardLitSceneFromWorld`) → MaterialShowcase regression.

Continue to [Part VIII — Particles](../part-08-particles/index.md).
