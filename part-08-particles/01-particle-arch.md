# Particle Architecture

> **Build-Along Milestone:** Architecture diagram: Emitter → CollectInstances → `particles[]` → `VulkanParticlePass`.

**You will learn:**

- Separate particle simulation on CPU from billboard rendering on GPU.
- Describe `SceneParticleInstance` fields and why velocity stays off the snapshot.
- Place `VulkanParticlePass` after the lit scene pass in the HDR subpass.

## 1. The Problem We Are Solving

Fire, sparks, snow, and hit effects need hundreds of **short-lived quads** facing the camera. Treating each as a full mesh draw with material push constants wastes CPU time, complicates sorting, and bloats `SceneDrawItem` arrays.

A dedicated **particle pipeline** simulates on CPU, flattens to a compact instance array, and draws in one pass with additive blending. Build along: trace Spark's path from `ParticleEmitterComponent` through `SceneRenderParams::particles` to `VulkanParticlePass::Record`.

## 2. Core Concepts

### Simulation vs rendering split

Spark simulates particles on CPU inside `ParticleEmitterComponent::OnUpdate`. Rendering never reads live ECS state — submit copies a flat `Array<SceneParticleInstance>` into `SceneRenderParams` each frame, immutable for the GPU slot.

```text
┌─────────────────────┐     OnUpdate      ┌──────────────────┐
│ ParticleEmitter     │ ────────────────► │ SimParticle[]    │
│ + Transform         │                   │ (velocity, age)  │
└─────────────────────┘                   └────────┬─────────┘
                                                   │ CollectInstances
                                                   ▼
┌─────────────────────┐   particles[]     ┌──────────────────┐
│ SceneRenderParams   │ ◄──────────────── │ Submit / Demo    │
└─────────┬───────────┘                   └──────────────────┘
          │ Record
          ▼
┌─────────────────────┐
│ VulkanParticlePass  │  → HDR color + depth read
└─────────────────────┘
```

### SceneParticleInstance record

Each living particle becomes `{ position, size, color }` — no velocity, no mesh slot, no texture layer. Size and color are **already interpolated** by age on CPU; the GPU only billboards and applies radial alpha.

| Field | CPU source | GPU usage |
|-------|------------|-----------|
| `position` | Integrated world pos | Billboard center |
| `size` | Lerp `size0` → `size1` | Quad half-extent |
| `color` | Lerp `color0` → `color1` | Vertex color × disc alpha |

### Camera-facing billboards

Quad corners are reconstructed in the vertex shader from `particleCameraRight` and `particleCameraUp` in scene params — CPU stores world center only. One camera basis per frame is shared by all particles.

### Additive HDR pass

Particles draw into the same HDR color attachment as the lit scene pass, after opaque meshes and sprites, with `SRC_ALPHA` / `ONE` additive blending for fire and glow. Tonemap (Part X) compresses accumulated energy later.

### Why not scene.frag?

Particles skip the lit mesh pipeline entirely — no BRDF, no textures, no skinning, no push constant bloat. A separate `particle.vert` / `particle.frag` pair keeps `scene.frag` maintainable.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### ParticleEmitterComponent owns sim

**Concept:** Simulation vs rendering split

**Decision:** `ComponentKind::ParticleEmitter` on `GameObject`

**Why:** Gameplay attaches emitters to entities; renderer stays agnostic — same pattern as `MeshComponent` → submit.

**In Spark:** `include/spark/ecs/components/rendering/ParticleEmitterComponent.hpp`

### Minimal snapshot struct

**Concept:** `SceneParticleInstance` record

**Decision:** Three fields only on snapshot

**Why:** Interpolate size/color on CPU during `CollectInstances`; GPU only billboards and soft-disc alpha.

**In Spark:** `SceneParticleInstance` in `SceneRenderParams.hpp`

### Basis vectors in SceneRenderParams

**Concept:** Camera-facing billboards

**Decision:** `particleCameraRight`, `particleCameraUp` per frame

**Why:** One normalized basis shared by all particles — avoids storing rotation per instance.

**In Spark:** Filled from active camera in submit or demo

### Dedicated VulkanParticlePass

**Concept:** Additive HDR pass

**Decision:** Separate pipeline, blend, vertex layout

**Why:** No special cases in `scene.frag` or opaque pass record loop.

**In Spark:** `include/spark/render/sprites2d/VulkanParticlePass.hpp`

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/ecs/components/rendering/ParticleEmitterComponent.hpp`
- `include/spark/engine/SceneRenderParams.hpp`
- `include/spark/render/sprites2d/VulkanParticlePass.hpp`

### Step 1 — Read the data-flow diagram

```text
GameObject + TransformComponent + ParticleEmitterComponent
  → OnUpdate (spawn, integrate, kill)
  → CollectInstances → SceneRenderParams.particles[]
  → VulkanParticlePass::Record → HDR framebuffer
```

### Step 2 — Inspect SceneParticleInstance

```cpp
// include/spark/engine/SceneRenderParams.hpp
/** One billboard particle instance (CPU sim → Vulkan particle pass). */
struct SceneParticleInstance {
    Vector3 position{};
    float size = 0.1F;
    Vector4 color{1.0F, 1.0F, 1.0F, 1.0F};
};
```

### Step 3 — Read emitter component header

```cpp
// include/spark/ecs/components/rendering/ParticleEmitterComponent.hpp
/**
 * CPU particle emitter: spawns, simulates, and exposes instances for SceneRenderParams::particles.
 * Expects a TransformComponent on the same GameObject (emission origin = world translation).
 */
class ParticleEmitterComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::ParticleEmitter;
    void OnUpdate(const FrameTiming& timing, GameObject& owner, IEngineContext& context) override;
    void CollectInstances(Array<SceneParticleInstance>& out) const;
```

### Step 4 — Locate particles array in scene params

```cpp
// include/spark/engine/SceneRenderParams.hpp
    /** Billboard particles (additive pass). Filled from ParticleEmitterComponent. */
    static constexpr std::uint32_t MaxParticles = 8192;
    Array<SceneParticleInstance> particles;
    /** Camera-facing billboard axes in world space (normalized). */
    Vector3 particleCameraRight{1.0F, 0.0F, 0.0F};
    Vector3 particleCameraUp{0.0F, 1.0F, 0.0F};
```

### Step 5 — Sketch your engine boundary

Confirm no gameplay `.cpp` includes `<vulkan/vulkan.h>`. Particles follow the same ECS → snapshot → GPU path as mesh draws from Part VII.

### Step 6 — Milestone — architecture on paper

Draw three boxes: Emitter (CPU sim), Submit (flatten), ParticlePass (GPU billboards). Label the struct crossing each boundary.

## 5. How This Connects to the Rest of Your Engine

**Part V** camera supplies right/up for billboards. **Part VII** scene opaque pass writes depth particles read. **ECS** (Part VI) composes emitters on `GameObject`s. **Frame Graph** (Chapter 7) slots the pass after sprites. **Particle Demo** wires four preset emitters end-to-end.

## 6. Pitfalls and Debugging

- **Simulating inside Record** — ties GPU thread to gameplay; always update in `OnUpdate` before submit.
- **Storing mesh per particle** — thousands of draw items; use instanced billboards instead.
- **Skipping camera basis** — billboards collapse to lines if right/up are zero or stale.
- **Mixing particle and mesh submit** — particles are not `SceneDrawItem`; do not push them into `draws[]`.

## 7. Exercises

1. List three fields in `SceneParticleInstance` and what each means at draw time.
2. Contrast particle data flow with `MeshComponent` → `SceneDrawItem` flow.
3. Sketch where `MaxParticles` cap is enforced in submit vs Record.
4. Count Vulkan pipelines involved in a frame with particles vs without.
5. Explain why velocity is not in the snapshot struct.

## Summary

Particles simulate in `ParticleEmitterComponent`, flatten to `SceneParticleInstance` array in submit, and render via `VulkanParticlePass` as camera-facing additive billboards in the HDR subpass.

---

Next: [Emitter Parameters](02-emitter-params.md).
