# Part 14: Physics Integration

Simulate rigid bodies on CPU solvers, attach colliders through ECS components, and sync transforms into SceneRenderParams after the physics step.

| # | Chapter |
|---|---------|
| 1 | [2D Physics](01-physics-2d.md) |
| 2 | [3D Physics](02-physics-3d.md) |
| 3 | [Colliders](03-colliders.md) |
| 4 | [Sync Transforms](04-physics-render.md) |

**Arc:** `PhysicsWorld2D` platformer simulate → `PhysicsWorld3D` substepped rigid bodies → collider bake + queries → physics-before-submit tick order.

Continue to [Part XV — Audio, AI, Scripting](../part-15-audio-ai-scripting/index.md).
