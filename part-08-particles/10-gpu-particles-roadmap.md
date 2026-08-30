# GPU Particles Roadmap

> **Build-Along Milestone:** Written roadmap: SSBO upload milestone, then compute sim + indirect draw.

**You will learn:**

- Contrast Spark CPU slot sim with GPU compute simulation architecture.
- Explain indirect draw and procedural VS as replacements for `scratchVertices` expansion.
- Plan incremental migration from current `ParticleEmitterComponent` without breaking demos.

## 1. The Problem We Are Solving

CPU particles cap around a few thousand with acceptable frame time on mid-range hardware. Shipping titles need **GPU sim, indirect draw, and texture flipbooks** — plan the migration now while Part VIII CPU path remains the learning baseline.

Build along: document a two-milestone roadmap from current Spark code to GPU-driven particles.

## 1b. Concept Deep Dive — What the CPU Path Does Today

### Current bottleneck: scratchVertices expansion on CPU

```cpp
// VulkanParticlePass::Record — every frame, every particle, six verts
scratchVertices.Clear();
for (std::size_t pi = 0; pi < nPart; ++pi) {
    for (int t = 0; t < 6; ++t) {
        scratchVertices.PushBack(/* 10 floats per corner */);
    }
}
std::memcpy(vertexMapped, scratchVertices.GetData(), vbBytes);
vkCmdDraw(commandBuffer, vertCount, 1, 0, 0);
```

At 8192 particles this is ~49k vertices rebuilt on CPU. GPU sim would store `{pos, vel, age, ...}` in an SSBO and let the vertex shader (or a compute prepass) emit corners procedurally — eliminating the inner expansion loop.

### Migration anchor points in existing code

| Current hook | GPU replacement |
|--------------|-----------------|
| `ParticleEmitterComponent::OnUpdate` | Compute dispatch: integrate SSBO |
| `CollectInstances` | GPU compaction → indirect draw count |
| `scratchVertices` + `vkCmdDraw` | `vkCmdDrawIndirect` with alive count |
| `ParticleUniformGpu` | Same UBO — camera basis unchanged |

Keep `SceneParticleInstance` and `CollectInstances` during transition so CPU demos keep working while GPU path is feature-flagged.

### ParticleEmitterComponent API stays stable

Demos and scripts call `SetEmissionRate`, `SetLifetime`, etc. A GPU backend would read those constants once per frame into a small UBO/SSBO header; gameplay code would not change. Only the internal `OnUpdate` body swaps from slot-loop to dispatch.

## 2. Core Concepts

### GPU simulation

Move integrate/spawn/kill to a compute shader writing an SSBO — CPU only sets emitter constants once per frame. Scales to 100k+ particles when ALU-bound on GPU instead of CPU slot loops.

### Indirect draw

Compute appends alive count to a `VkDrawIndirectCommand` buffer — GPU culls dead particles without walking `CollectInstances` on CPU. Avoids CPU-GPU sync on particle count each frame.

### Procedural vertex shader

One draw with `gl_VertexIndex` quad expansion or `gl_InstanceIndex` into SSBO — eliminates the `scratchVertices` loop and host `memcpy` in `Record`. Billboard math from Chapter 5 transfers directly to VS reading SSBO.

### Texture sheet animation

Add `frameIndex` / `uvOffset` to `SceneParticleInstance` (or GPU struct) and sample a flipbook from `sceneTextures` — sparks and smoke reuse the Part VII texture array.

### Migration strategy

| Milestone | CPU | GPU | Risk |
|-----------|-----|-----|------|
| **0 (today)** | Sim + collect + expand | Draw billboards | Baseline |
| **1** | Sim + collect | SSBO upload, VS expands quad | Low — drop memcpy |
| **2** | Emitter constants only | Compute sim + indirect draw | Medium — determinism |

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Compute pass before graphics

**Concept:** GPU simulation

**Decision:** `ParticleComputePass` before HDR subpass

**Why:** Scales particle count; CPU path stays teaching baseline.

**In Spark:** Future addition to `VulkanRenderer`

### vkCmdDrawIndirect

**Concept:** Indirect draw

**Decision:** Alive count from GPU atomic

**Why:** No readback of particle count to CPU each frame.

**In Spark:** Future: indirect buffer from compute

### SSBO SceneParticleInstance[]

**Concept:** Procedural vertex shader

**Decision:** Record binds buffer once; VS reads by instance

**Why:** Drop `scratchVertices` and host VB `memcpy`.

**In Spark:** Evolution of `VulkanParticlePass::Record`

### Extend snapshot struct optionally

**Concept:** Texture sheet animation

**Decision:** Add fields without breaking CPU path

**Why:** Frag samples `sceneTextures` like sprites when `frameIndex >= 0`.

**In Spark:** Extend `SceneParticleInstance`

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `src/spark/render/sprites2d/VulkanParticlePass.cpp`
- `include/spark/ecs/components/rendering/ParticleEmitterComponent.hpp`
- `include/spark/engine/SceneRenderParams.hpp`

### Step 1 — Audit current CPU hot paths

List: `OnUpdate` slot loop, `CollectInstances` lerp, `Record` `scratchVertices` expansion, `memcpy` VB.

### Step 2 — Sketch GPU pipeline

```text
Emitter constants UBO → Compute (spawn/sim) → Barrier → Graphics (procedural billboard VS)
→ Same additive blend/depth as today
```

### Step 3 — Identify what stays on CPU

Gameplay still owns emitter placement via `TransformComponent`; only sim moves GPU. `enableParticles` gate and camera basis remain on snapshot.

### Step 4 — Compare with Spark sprite SSBO path

Review how 2D sprite pass batches draws — particle GPU roadmap can mirror descriptor patterns and texture array binding.

### Step 5 — Migration milestone 1

Move `CollectInstances` data to SSBO without compute — CPU fills SSBO, VS expands quad from `gl_InstanceIndex`. Validate ParticleDemo parity.

### Step 6 — Migration milestone 2

Add compute spawn/kill; CPU sets `emissionRate` only; indirect draw count from GPU. Profile vs milestone 1.

## 5. How This Connects to the Rest of Your Engine

**Performance and Pooling** identifies CPU/`memcpy` limits motivating GPU move. **Particle Shaders** billboard math transfers to procedural VS. **Scene texture array** from Part VII enables flipbook frag sampling. **Frame Graph** adds compute pass before existing particle graphics record.

## 6. Pitfalls and Debugging

- **Big-bang rewrite** — ship SSBO upload first, compute second.
- **Non-deterministic GPU sim** — replays and netcode need fixed seeds or CPU authority.
- **Breaking `SceneParticleInstance`** — add optional fields; keep CPU path until parity.
- **Ignoring mobile compute limits** — test threadgroup size and SSBO size on target hardware.

## 7. Exercises

1. Write struct layout for GPU `SceneParticleInstance` with `age` and `frameIndex`.
2. Estimate bandwidth: CPU `memcpy` today vs SSBO upload at 50k particles.
3. List which ParticleDemo presets benefit most from GPU sim (snow storm vs single burst).
4. Draw frame graph with compute pass inserted — label barriers.
5. Identify which `VulkanParticlePass` resources become obsolete in milestone 2.

## Summary

GPU roadmap: compute sim + SSBO + procedural VS + indirect draw; Spark CPU emitter remains the teaching path; migrate in two milestones without breaking existing demos.

---

Continue to [Part IX — Shadows](../part-09-shadows/index.md).
