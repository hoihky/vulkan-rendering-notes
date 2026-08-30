# Emitter Parameters

> **Build-Along Milestone:** Fire emitter entity spawning particles with visible color fade and upward cone.

**You will learn:**

- Configure emission rate, lifetime, size, color, gravity, direction, and spread on ParticleEmitterComponent.
- Explain spawnDebt fractional emission at variable frame rates.
- Describe slot-pool spawn and kill without heap allocation.

## 1. The Problem We Are Solving

An emitter without tunable parameters produces identical dots forever. Artists need **rate, lifetime, color fade,
size shrink, gravity, and cone spread** — all adjustable without recompiling shaders.

Build along: spawn a fire-style emitter with upward direction, warm start color, and transparent end color.

## 1b. Concept Deep Dive — SimParticle Fields and spawnDebt

### Per-slot state vs exported instance

Each alive slot stores simulation-only fields the renderer never sees:

```cpp
// include/spark/ecs/components/rendering/ParticleEmitterComponent.hpp (lines 60–70)
struct SimParticle {
    bool alive = false;
    Vector3 position{};
    Vector3 velocity{};
    float age = 0.0F;
    float maxAge = 1.0F;
    float size0 = 0.1F;
    float size1 = 0.05F;
    Vector4 color0{1, 1, 1, 1};
    Vector4 color1{1, 1, 1, 0};
};
```

At spawn, `size0`/`size1` and `color0`/`color1` are copied from emitter parameters (`sizeStart`, `colorStart`, etc.). `CollectInstances` lerps by `age / maxAge` before export — the GPU receives final size and color, not endpoints.

### spawnDebt — fractional emission at any frame rate

```cpp
// src/spark/ecs/components/rendering/ParticleEmitterComponent.cpp (lines 108–112)
spawnDebt += emissionRate * dt;
while (spawnDebt >= 1.0F) {
    spawnDebt -= 1.0F;
    SpawnOne(origin);
}
```

At 30 FPS with `emissionRate = 48`, each frame accrues ~1.6 debt → one spawn most frames, two every third frame. At 144 FPS debt accrues ~0.33 per frame → still ~48 spawns/sec without floating-point drift. `SpawnOne` walks the slot pool for the first dead index — no heap allocation.

### Cone spread via RandomUnitSphere jitter

```cpp
// SpawnOne (lines 80–89)
const Vector3 basis = emissionDir;
const Vector3 jitter = RandomUnitSphere() * spreadRadians;
Vector3 dir = basis + jitter;
// ... normalize, then velocity = dir * (speedMin + Random01() * (speedMax - speedMin))
```

`spreadRadians` scales a random unit-sphere offset added to the emission direction. Fire presets use `0.55F` (~31° half-angle); snow uses `1.25F` for wide drift.

## 2. Core Concepts

### Slot pool simulation

Each emitter owns a fixed `Array<SimParticle>` sized to `maxParticles`. Spawn finds the first dead slot; no heap allocation per frame.


### Emission rate debt

`spawnDebt += emissionRate * dt` accumulates fractional spawns. While debt ≥ 1, spawn one particle and subtract 1 — smooth rate at any frame time.


### Cone emission

`emissionDir` is the cone axis; `spreadRadians` scales a random unit-sphere jitter added to the axis before normalization and speed assignment.


### Lifetime interpolation

Each SimParticle stores start/end size and color. CollectInstances lerps by `age / maxAge` when building SceneParticleInstance.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Fixed SimParticle slots

**Concept:** Slot pool simulation

**Decision:** Fixed SimParticle slots

**Why:** O(maxParticles) update regardless of live count; predictable memory for mobile and editor tuning.

**In Spark:** `ParticleEmitterComponent::slots` private array

### Fractional spawn accumulator

**Concept:** Emission rate debt

**Decision:** Fractional spawn accumulator

**Why:** 48 particles/sec at 144 Hz spawns evenly — no burst-on-frame-boundary artifacts.

**In Spark:** `spawnDebt` in OnUpdate

### Direction + spread angle

**Concept:** Cone emission

**Decision:** Direction + spread angle

**Why:** Fire rises (+Y), snow falls (-Y), magic bursts omnidirectional — one API covers all presets.

**In Spark:** `SetEmissionDirection`, `SetSpreadAngleRadians`

### CPU-side color/size lerp

**Concept:** Lifetime interpolation

**Decision:** CPU-side color/size lerp

**Why:** GPU receives final per-frame values; shader stays simple disc + alpha.

**In Spark:** `CollectInstances` t = age / maxAge

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/ecs/components/rendering/ParticleEmitterComponent.hpp`
- `src/spark/ecs/components/rendering/ParticleEmitterComponent.cpp`
- `include/spark/demo/ParticleDemoDetail.hpp`

### Step 1 — Read public tuning API

*Rate, lifetime, size, color, gravity, direction, spread, speed setters.*

```cpp
// include/spark/ecs/components/rendering/ParticleEmitterComponent.hpp (lines 24–43)
    [[nodiscard]] bool IsEmitterEnabled() const noexcept { return enabled; }
    void SetEmitterEnabled(bool e) noexcept { enabled = e; }

    void SetMaxParticles(std::uint32_t n) noexcept;
    [[nodiscard]] std::uint32_t GetMaxParticles() const noexcept { return maxParticles; }

    /** Particles spawned per second (fractional accumulation). */
    void SetEmissionRate(float rate) noexcept { emissionRate = rate; }
    [[nodiscard]] float GetEmissionRate() const noexcept { return emissionRate; }

    void SetLifetime(float minSec, float maxSec) noexcept;
    void SetStartEndSize(float start, float end) noexcept;
    void SetStartEndColor(const Vector4& start, const Vector4& end) noexcept;
    void SetGravity(const Vector3& g) noexcept { gravity = g; }
    /** Average emission direction in world space (normalized internally). */
    void SetEmissionDirection(const Vector3& dir) noexcept;
    [[nodiscard]] const Vector3& GetEmissionDirection() const noexcept { return emissionDir; }
    /** Half-angle cone around emission direction (radians). */
    void SetSpreadAngleRadians(float rad) noexcept { spreadRadians = rad; }
    void SetSpeedRange(float minS, float maxS) noexcept;
```

### Step 2 — Study SpawnOne

*Find dead slot, random lifetime, cone direction, assign velocity.*

```cpp
// src/spark/ecs/components/rendering/ParticleEmitterComponent.cpp (lines 66–92)
void ParticleEmitterComponent::SpawnOne(const Vector3& origin) {
    EnsureSlotCapacity();
    const std::uint32_t cap = static_cast<std::uint32_t>(slots.GetSize());
    for (std::uint32_t i = 0; i < cap; ++i) {
        if (!slots[i].alive) {
            SimParticle& p = slots[i];
            p.alive = true;
            p.position = origin;
            p.age = 0.0F;
            p.maxAge = lifeMin + Random01() * (lifeMax - lifeMin);
            p.size0 = sizeStart;
            p.size1 = sizeEnd;
            p.color0 = colorStart;
            p.color1 = colorEnd;
            const Vector3 basis = emissionDir;
            const Vector3 jitter = RandomUnitSphere() * spreadRadians;
            Vector3 dir = basis + jitter;
            if (dir.LengthSquared() < 1.0e-8F) {
                dir = basis;
            } else {
                dir = dir.Normalized();
            }
            const float sp = speedMin + Random01() * (speedMax - speedMin);
            p.velocity = dir * sp;
            break;
        }
    }
```

### Step 3 — Trace OnUpdate integration

*Emission debt loop, gravity integration, age kill.*

```cpp
// src/spark/ecs/components/rendering/ParticleEmitterComponent.cpp (lines 95–127)
void ParticleEmitterComponent::OnUpdate(const FrameTiming& timing, GameObject& owner, IEngineContext&) {
    if (!enabled) {
        return;
    }
    const float dt = timing.deltaTimeSeconds;
    if (dt <= 0.0F) {
        return;
    }

    EnsureSlotCapacity();
    const Matrix4 wm = owner.GetWorldMatrix();
    const Vector3 origin{wm.m[12], wm.m[13], wm.m[14]};

    spawnDebt += emissionRate * dt;
    while (spawnDebt >= 1.0F) {
        spawnDebt -= 1.0F;
        SpawnOne(origin);
    }

    const std::uint32_t cap = static_cast<std::uint32_t>(slots.GetSize());
    for (std::uint32_t i = 0; i < cap; ++i) {
        SimParticle& p = slots[i];
        if (!p.alive) {
            continue;
        }
        p.age += dt;
        if (p.age >= p.maxAge) {
            p.alive = false;
            continue;
        }
        p.velocity += gravity * dt;
        p.position += p.velocity * dt;
    }
```

### Step 4 — Attach emitter to entity

```cpp
GameObject* fx = world.CreateGameObject();
fx->AddComponent<TransformComponent>()->SetTranslation({0, 0.2F, 0});
ParticleEmitterComponent* pe = fx->AddComponent<ParticleEmitterComponent>();
pe->SetEmissionRate(80.0F);
pe->SetLifetime(0.4F, 1.2F);
pe->SetStartEndColor({1, 0.6F, 0.1F, 1}, {0.8F, 0.05F, 0, 0});
```

### Step 5 — Apply a demo preset

*Fire preset — rate, gravity, spread tuned for upward burst.*

```cpp
// include/spark/demo/ParticleDemoDetail.hpp (lines 12–25)
inline void ApplyParticlePreset(const int presetIndex, ParticleEmitterComponent& pe) {
    pe.SetEmitterEnabled(true);
    switch (presetIndex) {
    case 0:
        pe.SetMaxParticles(800);
        pe.SetEmissionRate(110.0F);
        pe.SetLifetime(0.22F, 0.55F);
        pe.SetStartEndSize(0.24F, 0.03F);
        pe.SetStartEndColor(Vector4{1.0F, 0.55F, 0.12F, 1.0F}, Vector4{0.85F, 0.05F, 0.0F, 0.0F});
        pe.SetGravity({0.0F, 0.35F, 0.0F});
        pe.SetEmissionDirection({0.0F, 1.0F, 0.0F});
        pe.SetSpreadAngleRadians(0.55F);
        pe.SetSpeedRange(1.6F, 4.2F);
        break;
```

### Step 6 — Verify origin follows transform

Move the entity in editor — emission origin must track world translation from `owner.GetWorldMatrix()`.

## 5. How This Connects to the Rest of Your Engine

**TransformComponent** supplies world origin for spawn. **Collecting Particles** reads living slots after OnUpdate.
**Particle Demo** exposes sliders bound to the same setters. **Performance and Pooling** revisits slot count vs rate.

## 6. Pitfalls and Debugging

- **SetMaxParticles clears slots** — resizing pool kills all live particles instantly.
- **spreadRadians too large** — direction jitter dominates axis; emission becomes a sphere.
- **lifeMax < lifeMin** — SetLifetime clamps max to min; verify both args.
- **Missing TransformComponent** — origin still reads matrix translation but entity may not be where you expect.

## 7. Exercises

1. Tune snow: negative gravity, wide spread, slow speed — compare to fire preset.
2. Log live particle count each frame by counting alive slots.
3. Set `emissionRate` to 0 — confirm pool drains via lifetime only.
4. Set `lifeMin == lifeMax` — verify uniform lifetime per particle.
5. Trace `spawnDebt` across three frames at 144 Hz vs 30 Hz.

## Summary

ParticleEmitterComponent simulates fixed slots with fractional spawn debt, cone velocity, gravity integration, and CPU-side size/color interpolation over lifetime.

---

Next: [Collecting Particles](03-collect-particles.md).
