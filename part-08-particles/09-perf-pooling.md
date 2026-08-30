# Performance and Pooling

> **Build-Along Milestone:** Documented particle budget: max live count, bytes/frame, and FPS at stress settings.

**You will learn:**

- Explain `SimParticle` slot pooling and spawn reuse without allocation.
- Estimate per-frame vertex upload bytes from particle count.
- Tune `maxParticles` and emission rate to stay within GPU and CPU budget.

## 1. The Problem We Are Solving

Unbounded particles destroy frame time — thousands of spawns allocate heap memory, `memcpy` explodes, and additive overdraw saturates fill rate. You need **pooling, caps, and profiling discipline** before shipping VFX-heavy scenes.

Build along: profile `ParticleDemo` at max settings and document headroom against `MaxParticles` and vertex buffer size.

## 1b. Concept Deep Dive — Budget Constants and Pool Mechanics

### Hard caps in Spark

```cpp
// include/spark/engine/SceneRenderParams.hpp
static constexpr std::uint32_t MaxParticles = 8192;

// VulkanParticlePass.cpp
constexpr VkDeviceSize kParticleVertexBytes = 10 * 1024 * 1024;
constexpr std::uint32_t kParticleFloatsPerVertex = 10;
```

Submit stops collecting at 8192. Record further clamps to `vertexCapacityBytes / (40 * 4)` ≈ 65 536 verts / 6 ≈ 10 922 particles — the lower of the two limits wins in practice (8192).

### Per-frame upload cost estimate

```text
bytes/frame = particleCount × 6 verts × 10 floats × 4 bytes
8192 particles → 8192 × 240 = ~1.97 MB memcpy per frame
```

CPU cost is linear in alive count: `OnUpdate` walks all slots O(maxParticles) per emitter, `CollectInstances` walks alive slots, Record expands and memcpy's. Profile with four emitters at max settings before shipping VFX-heavy scenes.

### SetMaxParticles clears the pool

```cpp
// ParticleEmitterComponent.cpp (lines 16–19)
void ParticleEmitterComponent::SetMaxParticles(std::uint32_t n) noexcept {
    maxParticles = std::max(1u, n);
    slots.Clear();  // all particles die on resize — no stale alive flags
}
```

Resizing is destructive by design. Gameplay code should set `maxParticles` once at load, not every frame.

## 2. Core Concepts

### Slot pool vs allocation

`SimParticle` slots are preallocated to `maxParticles` per emitter — `SpawnOne` finds the first dead index without `new`/`delete`. `SetMaxParticles` clears and resizes the pool.

```text
SpawnOne: for i in 0..cap: if !slots[i].alive → reuse
OnUpdate: integrate all alive slots O(cap) per emitter
```

### Global cap economics

| Limit | Value | Where enforced |
|-------|-------|----------------|
| `SceneRenderParams::MaxParticles` | 8192 | Submit collection loop |
| Vertex buffer capacity | ~10 MB host-visible | `VulkanParticlePass::Record` clamp |
| Per-emitter `maxParticles` | Artist-tuned | `SpawnOne` cannot exceed pool |

Upper bound memcpy: `8192 × 6 verts × 10 floats × 4 bytes ≈ 1.9 MB` per frame — budget emitter counts accordingly.

### Host-visible tradeoff

Spark maps a host-coherent vertex buffer and `memcpy`s expanded quads each frame — simple for teaching, but ties expansion to the record thread. Production may switch to `DEVICE_LOCAL` + staging ring buffer.

### Overdraw and fill rate

Additive stacked quads burn fill rate — lower `maxParticles`, shorter lifetime, or smaller `sizeEnd` for dense effects. Profile GPU time when four emitters run at full rate in ParticleDemo.

### Submit walk order

When global cap is hit, emitters later in `ForEachActiveGameObject` order lose tail particles — document ordering if multiple high-count emitters compete.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Fixed Array<SimParticle>

**Concept:** Slot pool vs allocation

**Decision:** Pre-sized array per emitter

**Why:** Predictable worst-case CPU and memory; dead slots are O(1) reuse.

**In Spark:** `EnsureSlotCapacity` + `SpawnOne`

### MaxParticles at submit and Record

**Concept:** Global cap economics

**Decision:** Hard cap at two boundaries

**Why:** Fail gracefully under load — drop tail particles instead of validation errors or buffer overrun.

**In Spark:** `SceneRenderParams::MaxParticles`, `Record` clamp

### 10 MB mapped VB

**Concept:** Host-visible tradeoff

**Decision:** `kParticleVertexBytes` host-coherent allocation

**Why:** Good teaching scale; GPU roadmap moves to SSBO + procedural VS.

**In Spark:** `VulkanParticlePass::CreateGpuResources`

### Art-side limits via presets

**Concept:** Overdraw and fill rate

**Decision:** `ApplyParticlePreset` tunes rate and max per effect

**Why:** Designers tune preset tables, not C++, for ship builds.

**In Spark:** `ParticleDemoDetail.hpp`

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `src/spark/ecs/components/rendering/ParticleEmitterComponent.cpp`
- `src/spark/render/sprites2d/VulkanParticlePass.cpp`
- `include/spark/engine/SceneRenderParams.hpp`

### Step 1 — Profile live slot usage

```cpp
std::uint32_t live = 0;
for (const auto& s : slots) { if (s.alive) ++live; }
```

Compare live to `maxParticles` — headroom for burst effects.

### Step 2 — Estimate vertex bytes per frame

```cpp
const std::size_t n = params.particles.GetSize();
const std::size_t bytes = n * 6 * 10 * sizeof(float);
```

Log at peak — compare to `vertexCapacityBytes`.

### Step 3 — Read Record clamp

```cpp
// src/spark/render/sprites2d/VulkanParticlePass.cpp
    const std::size_t maxByBuffer =
            static_cast<std::size_t>(vertexCapacityBytes / (sizeof(float) * kParticleFloatsPerVertex)) / 4U;
    std::size_t nPart = scene.particles.GetSize();
    if (nPart > static_cast<std::size_t>(SceneRenderParams::MaxParticles)) {
        nPart = static_cast<std::size_t>(SceneRenderParams::MaxParticles);
    }
    if (nPart > maxByBuffer) {
        nPart = maxByBuffer;
    }
```

### Step 4 — Stress test

Set all four ParticleDemo presets to max emission — observe FPS and cap behavior.

### Step 5 — Reduce cost

Halve `maxParticles` and `emissionRate` on smoke preset — measure frame time delta.

### Step 6 — Pooling checklist

(1) Slots not heap, (2) Global cap enforced, (3) VB capacity clamp, (4) No sim work when disabled, (5) `CollectInstances` skips dead slots early.

## 5. How This Connects to the Rest of Your Engine

**Emitter Parameters** `maxParticles` sizes the pool. **Collecting Particles** enforces global cap. **Vulkan Particle Pass** clamps by buffer bytes. **GPU Particles Roadmap** moves sim/upload hot paths to GPU.

## 6. Pitfalls and Debugging

- **maxParticles >> live need** — wasted slot iteration in `OnUpdate`.
- **Many emitters each at 8192** — global cap starves later emitters in walk order.
- **Huge sizeEnd** — few particles still cover whole screen — fill bound.
- **CollectInstances on disabled emitter** — submit already skips; check `IsEmitterEnabled`.

## 7. Exercises

1. Graph frame time vs `particles.GetSize()`.
2. Compute theoretical max particles from 10 MB vertex buffer (`10 floats/vert × 6 verts/particle`).
3. List three art-side knobs to reduce overdraw without code changes.
4. Log which emitter loses particles first when global cap is hit.
5. Compare CPU time of `OnUpdate` vs `Record` scratch expansion at 4k particles.

## Summary

Spark pools `SimParticle` slots, caps at `MaxParticles`, clamps GPU upload by VB size — profile emission rate and size to control CPU `memcpy` and fill rate.

---

Next: [GPU Particles Roadmap](10-gpu-particles-roadmap.md).
