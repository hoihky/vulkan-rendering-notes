# Part 11: 2D Rendering

Draw world sprites and tilemaps into the HDR scene pass, layer and sort 2D content, apply optional 2D lighting, animate sprite clips, and composite screen UI after tonemap.

| # | Chapter |
|---|---------|
| 1 | [World Sprites](01-sprites.md) |
| 2 | [2D Lighting](02-sprite-lighting.md) |
| 3 | [Sprite Animation](03-sprite-anim.md) |
| 4 | [Tilemaps](04-tilemaps.md) |
| 5 | [Render Layers](05-render-layers.md) |
| 6 | [Screen UI](06-screen-ui.md) |
| 7 | [Dear ImGui](07-imgui.md) |
| 8 | [2D Game Pipeline](08-platformer-2d.md) |

**Arc:** `SceneSpriteDraw` + SSBO instancing → `VulkanSpritePass` in HDR subpass → sprite lighting modes → clip UV animation → tilemap cull/submit → `RenderLayerRegistry` sort → `VulkanScreenUiPass` after tonemap → ImGui debug → `Platformer2DDemo` integration.

Continue to [Part XII — Terrain](../part-12-terrain/index.md).
