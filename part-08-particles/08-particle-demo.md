# Particle Demo

> **Build-Along Milestone:** ParticleDemo running with four distinct effects and live slider tuning.

**You will learn:**

- Run Spark ParticleDemo with four preset emitter archetypes.
- Build SceneRenderParams with manual particle collection and mesh draws.
- Tune emitter parameters live via demo UI.

## 1. The Problem We Are Solving

Individual chapters cover emitters, submit, pass, and shaders — you need a **vertical slice demo** proving the
full Part VIII pipeline in one scene before integrating VFX into a game level.

Build along: run ParticleDemo, inspect Load/OnRender, and verify all four presets against the checklist.

## 1b. Concept Deep Dive — ParticleDemo Vertical Slice

### Four preset archetypes in ApplyParticlePreset

```cpp
// include/spark/demo/ParticleDemoDetail.hpp
inline void ApplyParticlePreset(const int presetIndex, ParticleEmitterComponent& pe) {
    switch (presetIndex) {
    case 0: // Fire — rate 110, upward cone, warm→red fade, low gravity
        pe.SetMaxParticles(800);
        pe.SetEmissionRate(110.0F);
        pe.SetStartEndColor({1, 0.55, 0.12, 1}, {0.85, 0.05, 0, 0});
        break;
    case 1: // Snow — rate 240, downward drift, white fade
        pe.SetMaxParticles(1400);
        break;
    case 2: // Smoke — slow rise, gray expand (sizeEnd > sizeStart)
        pe.SetMaxParticles(500);
        break;
    default: // Magic — purple spark trail
        pe.SetMaxParticles(900);
        break;
    }
}
```

Each preset tunes rate, lifetime, size curve, color fade, gravity, spread, and speed independently — the same `ParticleEmitterComponent` API your game code uses.

### Manual params build mirrors SceneSubmit

```cpp
// src/spark/demo/ParticleDemo.cpp (OnRender, concept)
params.particles.Clear();
for (each emitter entity) {
    pe.CollectInstances(chunk);
    // merge with MaxParticles cap — same loop as SceneSubmit.cpp
}
```

ParticleDemo skips `FillStandardLitSceneFromWorld` for particles and collects manually so the demo runs before full submit integration is wired. Ground plane + ref cube provide depth occlusion for blend/depth verification.

### Live UI tuning

ImGui sliders bind directly to `ParticleEmitterComponent` fields via `MakeFloatBinding`. `OnResetPresetClicked` calls `ApplyParticlePreset` to restore archetype defaults after manual edits.

## 2. Core Concepts

### Four preset emitters

ParticleDemo spawns Fire, Snow, Smoke, and Magic emitters at fixed world positions with ApplyParticlePreset tuning each style.


### Manual params build

Demo constructs SceneRenderParams directly — particles collected via ForEachParticleEmitter before mesh draws fill params.draws.


### Retained UI tuning

Sliders bind to selected emitter fields; Reset Preset reapplies ApplyParticlePreset for the effect index.


### Reference scene props

Ground plane and unit cube provide depth occlusion and lighting context for judging particle blend.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### ApplyParticlePreset table

**Concept:** Four preset emitters

**Decision:** ApplyParticlePreset table

**Why:** Teaching comparison in one scene — rate/gravity/spread differ per effect archetype.

**In Spark:** `ParticleDemoDetail.hpp` switch cases

### Not using SubmitStandardLitScene for particles section

**Concept:** Manual params build

**Decision:** Not using SubmitStandardLitScene for particles section

**Why:** Explicit collection teaches submit internals; draws still filled via ForEachDrawable.

**In Spark:** `ParticleDemo.cpp` OnRender params block

### ImGui-style sliders on emitter fields

**Concept:** Retained UI tuning

**Decision:** ImGui-style sliders on emitter fields

**Why:** Live tweak validates parameter chapter without recompile.

**In Spark:** `ParticleDemo::BuildRetainedUi`

### GroundPlane + UnitCube meshes

**Concept:** Reference scene props

**Decision:** GroundPlane + UnitCube meshes

**Why:** Occlusion and scale reference — particles pass depth test against cube.

**In Spark:** `ParticleDemo::Load`

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/demo/ParticleDemo.cpp`
- `include/spark/demo/ParticleDemoDetail.hpp`
- `include/spark/demo/ParticleDemo.hpp`

### Step 1 — Load demo scene

*Ground, ref cube, four emitters with presets and transforms.*

```cpp
// src/spark/demo/ParticleDemo.cpp (lines 96–146)
void ParticleDemo::Load(Spark::GameWorld& w, Spark::IEngineContext& context)
{
        roots.Clear();
        groundAsset = Spark::MakeShared<Spark::Mesh>(Spark::Utf8String("ParticleDemoGround"));
        *groundAsset = Spark::Mesh::CreateGroundPlane(Spark::kSceneGroundHalfExtent);
        groundObject = w.CreateGameObject();
        groundObject->GetName() = Spark::Utf8String("Ground");
        groundObject->AddComponent<Spark::TransformComponent>();
        groundObject->AddComponent<Spark::MeshComponent>(
                groundAsset, Spark::SceneMeshSlot::GroundPlane, Spark::Vector3{0.35F, 0.4F, 0.36F});
        roots.PushBack(groundObject);

        unitCubeAsset = Spark::MakeShared<Spark::Mesh>(Spark::Utf8String("ParticleDemoCube"));
        *unitCubeAsset = Spark::Mesh::CreateUnitCube();
        cubeObject = w.CreateGameObject();
        cubeObject->GetName() = Spark::Utf8String("RefCube");
        {
            Spark::TransformComponent* tr = cubeObject->AddComponent<Spark::TransformComponent>();
            tr->SetTranslation({1.8F, 0.85F, -1.2F});
            tr->SetUniformScale(0.9F);
        }
        cubeObject->AddComponent<Spark::MeshComponent>(
                unitCubeAsset, Spark::SceneMeshSlot::UnitCube, Spark::Vector3{0.75F, 0.5F, 0.22F});
        roots.PushBack(cubeObject);

        static constexpr float kFxPos[Detail::kParticleDemoEffectCount][3] = {
                {0.0F, 0.16F, 0.0F},
                {0.0F, 5.6F, 0.0F},
                {-2.35F, 0.22F, 0.95F},
                {2.45F, 0.28F, -1.25F},
        };
        static constexpr const char* kFxNames[Detail::kParticleDemoEffectCount] = {
                "FireEmitter",
                "SnowEmitter",
                "SmokeEmitter",
                "MagicEmitter",
        };
        for (int i = 0; i < Detail::kParticleDemoEffectCount; ++i) {
            Spark::GameObject* go = w.CreateGameObject();
            go->GetName() = Spark::Utf8String(kFxNames[static_cast<std::size_t>(i)]);
            Spark::TransformComponent* tr = go->AddComponent<Spark::TransformComponent>();
            tr->SetTranslation(
                    {kFxPos[static_cast<std::size_t>(i)][0],
                     kFxPos[static_cast<std::size_t>(i)][1],
                     kFxPos[static_cast<std::size_t>(i)][2]});
            Spark::ParticleEmitterComponent* pe = go->AddComponent<Spark::ParticleEmitterComponent>();
            Detail::ApplyParticlePreset(i, *pe);
            effectObjects[static_cast<std::size_t>(i)] = go;
            effectEmitters[static_cast<std::size_t>(i)] = pe;
            roots.PushBack(go);
        }
```

### Step 2 — Build params each frame

*viewProj, lighting, camera basis, particle collection.*

```cpp
// src/spark/demo/ParticleDemo.cpp (lines 232–262)
        Spark::SceneRenderParams params{};
        params.viewProjection = viewProj;
        params.cameraPositionWorld = camera.position;
        params.lightDirectionWorld = Spark::Vector3{0.4F, 0.85F, 0.2F}.Normalized();
        params.lightColor = {1.0F, 0.98F, 0.92F};
        params.lightIntensity = 0.92F;
        params.ambientColor = {0.10F, 0.11F, 0.14F};

        const Spark::Vector3 f = camera.Forward();
        Spark::Vector3 r = Spark::Vector3::Cross(Spark::Vector3::UnitY, f);
        if (r.LengthSquared() < 1.0e-10F) {
            r = Spark::Vector3::UnitX;
        } else {
            r = r.Normalized();
        }
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
        });
```

### Step 3 — Fill mesh draws

*ForEachDrawable → SceneDrawItem for ground and cube.*

```cpp
// src/spark/demo/ParticleDemo.cpp (lines 278–296)
        scene.ForEachDrawable([&](Spark::GameObject* obj, const Spark::MeshComponent& mc,
                                     const Spark::MaterialComponent* mat, const Spark::Matrix4& world) {
            if (obj != nullptr && obj->GetComponent<Spark::SkyComponent>() != nullptr) {
                return;
            }
            Spark::SceneDrawItem item{};
            item.model = world;
            item.mesh = mc.GetSlot();
            if (mc.GetSlot() == Spark::SceneMeshSlot::Custom) {
                item.customMesh = mc.GetMesh();
            }
            Spark::Vector3 alb = mc.GetAlbedo();
            item.textureLayer = -1;
            if (mat != nullptr) {
                ApplyMaterialComponentToSceneDrawItem(item, mat, &params);
            }
            item.albedo = alb;
            params.draws.PushBack(item);
        });
```

### Step 4 — Run Spark ParticleDemo

```bash
# From Spark build directory
./spark_particle_demo   # or launch ParticleDemo from engine menu
```

Fly camera — compare four effect styles side by side.

### Step 5 — Select emitter in UI panel

Change emission rate slider — confirm live particle density change.

### Step 6 — Checklist

(1) Fire rises, (2) Snow falls, (3) Smoke expands, (4) Magic bursts, (5) Particles occluded by cube, (6) FPS stable with all four enabled.

## 5. How This Connects to the Rest of Your Engine

**Emitter Parameters** presets match ApplyParticlePreset cases. **Collecting Particles** mirrors ForEachParticleEmitter
block. **Frame Graph** order visible when particles glow over ground. **Maze3DDemo** reuses collection for gem sparkles.

## 6. Pitfalls and Debugging

- **Load in OnRender** — recreate emitters every frame; load once in OnAttach/Load.
- **Missing camera basis** — copy right/up computation from demo when wiring your camera.
- **Forgetting SetSceneRenderParams** — GPU reads stale empty particles.",
- **All emitters at origin** — presets overlap; use spaced transforms like demo kFxPos.",

## 7. Exercises

1. Duplicate Magic preset at new transform — two independent emitters.
2. Disable Snow via `SetEmitterEnabled(false)` — count drops in log.
3. Screenshot golden frame for regression after shader edits.
4. Map keys 1–4 to preset names and primary tuning differences.
5. Switch to `SubmitStandardLitSceneFromWorld` with `enableParticles=true` — compare particle count.

## Summary

`ParticleDemo` is the Part VIII capstone — four presets, manual collection, mesh context, and live UI tuning over the full CPU particle pipeline.

---

Continue to [Part IX — Shadows](../part-09-shadows/index.md).

ParticleDemo loads four preset emitters, builds params with explicit particle collection, and provides UI tuning — Part VIII regression harness.

---

Next: [Performance and Pooling](09-perf-pooling.md).
