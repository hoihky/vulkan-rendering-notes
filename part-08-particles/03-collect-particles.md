# Collecting Particles

> **Build-Along Milestone:** params.particles populated after submit with correct count and camera right/up.

**You will learn:**

- Implement CollectInstances to export living particles with interpolated size and color.
- Wire particle collection into FillStandardLitSceneFromWorld with enableParticles and camera basis.
- Respect MaxParticles cap when merging multiple emitters.

## 1. The Problem We Are Solving

Simulation updated emitters in OnUpdate, but the GPU reads **SceneRenderParams** — a snapshot taken once per frame.
You need a **collection step** that flattens all enabled emitters into `particles[]` with a shared camera basis for billboarding.

Build along: after UpdateGameObjects, fill `params.particles` from every active ParticleEmitterComponent.

## 1b. Concept Deep Dive — CollectInstances and SceneSubmit Wiring

### Export loop — age interpolation on CPU

```cpp
// src/spark/ecs/components/rendering/ParticleEmitterComponent.cpp (lines 130–149)
void ParticleEmitterComponent::CollectInstances(Array<SceneParticleInstance>& out) const {
    for (std::uint32_t i = 0; i < n; ++i) {
        const SimParticle& p = slots[i];
        if (!p.alive) continue;
        const float t = p.maxAge > 1.0e-6F ? (p.age / p.maxAge) : 1.0F;
        const float u = std::clamp(t, 0.0F, 1.0F);
        const float sz = p.size0 + (p.size1 - p.size0) * u;
        const Vector4 c = p.color0 + (p.color1 - p.color0) * u;
        SceneParticleInstance inst{};
        inst.position = p.position;
        inst.size = sz;
        inst.color = c;
        out.PushBack(inst);
    }
}
```

No ECS pointers leave the emitter — only plain structs suitable for `SceneRenderParams`.

### FillStandardLitSceneFromWorld — global cap and camera basis

```cpp
// src/spark/scene/submit/SceneSubmit.cpp (lines 524–547)
params.particles.Clear();
if (enableParticles) {
    params.particleCameraRight = particleCameraRight.Normalized();
    params.particleCameraUp = particleCameraUp.Normalized();
    world.ForEachActiveGameObject([&params](GameObject* o) {
        if (params.particles.GetSize() >= SceneRenderParams::MaxParticles) return;
        const ParticleEmitterComponent* pe = o->GetComponent<ParticleEmitterComponent>();
        if (pe == nullptr || !pe->IsEmitterEnabled()) return;
        Array<SceneParticleInstance> chunk;
        pe->CollectInstances(chunk);
        for (std::size_t ci = 0; ci < chunk.GetSize(); ++ci) {
            if (params.particles.GetSize() >= SceneRenderParams::MaxParticles) return;
            params.particles.PushBack(chunk[ci]);
        }
    });
}
```

`MaxParticles` is 8192 globally — later emitters are silently truncated once the cap is hit. Camera right/up come from the main camera view matrix (not particle-specific) so all billboards share one facing basis per frame.

## 2. Core Concepts

### CollectInstances export

`CollectInstances` walks alive slots and pushes SceneParticleInstance records — no ECS or GameObject data in the output.


### Submit integration

`FillStandardLitSceneFromWorld` clears `params.particles`, optionally sets camera basis, then walks active objects with ParticleEmitterComponent.


### Global MaxParticles cap

`SceneRenderParams::MaxParticles` (8192) limits total instances across all emitters — submit stops pushing when full.


### enableParticles gate

2D demos pass `enableParticles = false` to skip collection overhead; 3D particle demos pass true with camera right/up from the active camera.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Const method, append to caller array

**Concept:** CollectInstances export

**Decision:** Const method, append to caller array

**Why:** Submit owns the global array; emitter only exports its chunk — easy to test in isolation.

**In Spark:** `ParticleEmitterComponent::CollectInstances`

### ForEachActiveGameObject walk

**Concept:** Submit integration

**Decision:** ForEachActiveGameObject walk

**Why:** Same active-hierarchy rule as mesh submit — disabled parent hides child emitters.

**In Spark:** `SceneSubmit.cpp` particles block

### Hard cap at submit and Record

**Concept:** Global MaxParticles cap

**Decision:** Hard cap at submit and Record

**Why:** Protects GPU vertex buffer overflow and frame time; oldest emitters may lose tail particles.

**In Spark:** `SceneRenderParams::MaxParticles`

### Bool parameter on FillStandardLitSceneFromWorld

**Concept:** enableParticles gate

**Decision:** Bool parameter on FillStandardLitSceneFromWorld

**Why:** Platformer2D skips particles; ParticleDemo and Maze3D enable them explicitly.

**In Spark:** `SceneSubmit.hpp` enableParticles param

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/ecs/components/rendering/ParticleEmitterComponent.cpp`
- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/demo/ParticleDemo.cpp`

### Step 1 — Read CollectInstances

*Lerp size/color by normalized age; push SceneParticleInstance.*

```cpp
// src/spark/ecs/components/rendering/ParticleEmitterComponent.cpp (lines 130–149)
void ParticleEmitterComponent::CollectInstances(Array<SceneParticleInstance>& out) const {
    if (slots.IsEmpty()) {
        return;
    }
    const std::uint32_t n = static_cast<std::uint32_t>(slots.GetSize());
    for (std::uint32_t i = 0; i < n; ++i) {
        const SimParticle& p = slots[i];
        if (!p.alive) {
            continue;
        }
        const float t = p.maxAge > 1.0e-6F ? (p.age / p.maxAge) : 1.0F;
        const float u = std::clamp(t, 0.0F, 1.0F);
        const float sz = p.size0 + (p.size1 - p.size0) * u;
        const Vector4 c = p.color0 + (p.color1 - p.color0) * u;
        SceneParticleInstance inst{};
        inst.position = p.position;
        inst.size = sz;
        inst.color = c;
        out.PushBack(inst);
    }
```

### Step 2 — Trace submit collection loop

*Clear particles, set camera basis, walk emitters, cap at MaxParticles.*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 373–396)
    params.particles.Clear();
    if (enableParticles) {
        params.particleCameraRight = particleCameraRight.Normalized();
        params.particleCameraUp = particleCameraUp.Normalized();
        world.ForEachActiveGameObject([&params](GameObject* o) {
            if (o == nullptr) {
                return;
            }
            if (params.particles.GetSize() >= SceneRenderParams::MaxParticles) {
                return;
            }
            const ParticleEmitterComponent* pe = o->GetComponent<ParticleEmitterComponent>();
            if (pe == nullptr || !pe->IsEmitterEnabled()) {
                return;
            }
            Array<SceneParticleInstance> chunk;
            pe->CollectInstances(chunk);
            for (std::size_t ci = 0; ci < chunk.GetSize(); ++ci) {
                if (params.particles.GetSize() >= SceneRenderParams::MaxParticles) {
                    return;
                }
                params.particles.PushBack(chunk[ci]);
            }
        });
```

### Step 3 — Read Scene helper

*ForEachParticleEmitter — use when building params manually outside FillStandardLitScene.*

```cpp
// include/spark/scene/Scene.hpp (lines 254–269)
    /**
     * ParticleEmitterComponent on enabled objects. Use when building SceneRenderParams::particles
     * (after GameWorld::UpdateGameObjects so simulation is current).
     */
    template<typename Fn>
    void ForEachParticleEmitter(Fn&& fn) const {
        world.ForEachActiveGameObject([&fn](GameObject* o) {
            if (o == nullptr) {
                return;
            }
            const ParticleEmitterComponent* pe = o->GetComponent<ParticleEmitterComponent>();
            if (pe == nullptr || !pe->IsEmitterEnabled()) {
                return;
            }
            fn(*pe, o->GetWorldMatrix());
        });
```

### Step 4 — Manual collection (ParticleDemo pattern)

*Compute camera right/up, clear particles, ForEachParticleEmitter + CollectInstances.*

```cpp
// src/spark/demo/ParticleDemo.cpp (lines 247–261)
        const Spark::Vector3 u = Spark::Vector3::Cross(f, r).Normalized();
        params.particleCameraRight = r;
        params.particleCameraUp = u;

        params.particles.Clear();
        scene.ForEachParticleEmitter([&params](const Spark::ParticleEmitterComponent& pe,
                                                    const Spark::Matrix4& /*world*/) {
            Spark::Array<Spark::SceneParticleInstance> chunk;
            pe.CollectInstances(chunk);
            for (std::size_t ci = 0; ci < chunk.GetSize(); ++ci) {
                if (params.particles.GetSize() >= Spark::SceneRenderParams::MaxParticles) {
                    return;
                }
                params.particles.PushBack(chunk[ci]);
            }
```

### Step 5 — Enable via standard submit

```cpp
Spark::SubmitStandardLitSceneFromWorld(
    world, context, viewProj, camPos,
    lightDir, lightColor, lightIntensity, ambient,
    /*enableParticles=*/ true,
    camRight, camUp, sceneTime);
```

### Step 6 — Log particle count

After submit, log `params.particles.GetSize()` — should rise with emission rate and cap at MaxParticles.

## 5. How This Connects to the Rest of Your Engine

**ECS Overview** — update before submit. **Vulkan Particle Pass** consumes `params.particles`. **Frame Graph** —
params must be set before RecordSceneCommandBuffer. **Particle Demo** builds params manually for teaching clarity.

## 6. Pitfalls and Debugging

- **Collect before OnUpdate** — one-frame lag; always update emitters first.
- **Forgetting camera basis** — billboards misaligned when using FillStandardLitScene without right/up args.
- **Inactive emitters still collecting** — check IsEmitterEnabled and active hierarchy.
- **Exceeding cap silently** — later emitters in walk order lose particles; document ordering if it matters.

## 7. Exercises

1. Spawn three emitters; verify total count respects `MaxParticles`.
2. Disable parent `GameObject` — confirm its emitter stops contributing.
3. Compare particle count with `enableParticles` true vs false.
4. Call `CollectInstances` before `OnUpdate` — observe one-frame lag.
5. Log `particleCameraRight` magnitude — should be ~1.0 after submit normalize.

## Summary

Submit clears and fills particles[] via CollectInstances per emitter, sets billboard camera axes, and enforces MaxParticles before GPU record.

---

Next: [Vulkan Particle Pass](04-particle-pass.md).
