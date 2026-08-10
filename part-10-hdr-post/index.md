# Part 10: HDR and Post-Processing

Render the scene into float HDR targets, tonemap with ACES for display, add SSAO and artist-driven exposure/fog volumes, and capture HDR frames for debugging.

| # | Chapter |
|---|---------|
| 1 | [HDR Render Targets](01-hdr-targets.md) |
| 2 | [ACES Tonemapping](02-aces-tonemap.md) |
| 3 | [SSAO](03-ssao.md) |
| 4 | [Exposure Volumes](04-exposure-volumes.md) |
| 5 | [Fog Volumes](05-fog.md) |
| 6 | [Capture Tools](06-capture.md) |

**Arc:** `RGBA16F` + D32 HDR subpass (scene/sprites/particles) → optional SSAO composite → `RecordTonemap` (ACES + exposure) → screen UI on swapchain → `VulkanFrameCapture` readback.

Continue to [Part XI — 2D Rendering](../part-11-2d-rendering/index.md).
