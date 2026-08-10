# Part 8: Particles

Simulate CPU particles from emitters, collect them into `SceneRenderParams`, and render camera-facing billboard quads in a dedicated Vulkan pass with additive HDR blending, depth occlusion, and correct frame-graph ordering.

| # | Chapter |
|---|---------|
| 1 | [Particle Architecture](01-particle-arch.md) |
| 2 | [Emitter Parameters](02-emitter-params.md) |
| 3 | [Collecting Particles](03-collect-particles.md) |
| 4 | [Vulkan Particle Pass](04-particle-pass.md) |
| 5 | [Particle Shaders](05-particle-shaders.md) |
| 6 | [Blend and Depth State](06-blend-depth.md) |
| 7 | [Frame Graph Ordering](07-frame-graph.md) |
| 8 | [Particle Demo](08-particle-demo.md) |
| 9 | [Performance and Pooling](09-perf-pooling.md) |
| 10 | [GPU Particles Roadmap](10-gpu-particles-roadmap.md) |

**Arc:** `ParticleEmitterComponent` slot sim → `CollectInstances` → `particles[]` + camera basis → `VulkanParticlePass::Record` quad expansion → `particle.vert`/`particle.frag` soft discs → additive HDR after sprites → tonemap.

Continue to [Part IX — Shadows](../part-09-shadows/index.md).
