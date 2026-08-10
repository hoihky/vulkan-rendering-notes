# Part 9: Shadow Mapping

Build directional cascaded shadow maps and punctual light shadows: depth-only passes, atlas layout, PCF sampling in the lit pass, per-draw cast/receive flags, and bias tuning.

| # | Chapter |
|---|---------|
| 1 | [Shadow Mapping Theory](01-shadow-theory.md) |
| 2 | [Directional CSM](02-directional-csm.md) |
| 3 | [Shadow Depth Shader](03-shadow-depth-shader.md) |
| 4 | [PCF and Cascades](04-pcf-cascades.md) |
| 5 | [Punctual Shadows](05-punctual-shadows.md) |
| 6 | [Cast and Receive Flags](06-shadow-flags.md) |
| 7 | [Skinned Mesh Shadows](07-skinned-shadows.md) |
| 8 | [Bias and Artifacts](08-shadow-tuning.md) |

**Arc:** light-space depth pre-pass → four-tile CSM atlas (`VulkanDirectionalShadowPass`) → `shadow_depth.vert` → 5×5 rotated PCF + cascade blend in `scene.frag` → punctual spot/point atlases → `shadowFlags` cast/receive → skinned SSBO parity → bias and seam tuning.

Continue to [Part X — HDR and Post](../part-10-hdr-post/index.md).
