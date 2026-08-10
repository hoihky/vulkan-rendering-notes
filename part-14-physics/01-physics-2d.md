# 2D Physics

> **Build-Along Milestone:** Platformer player falls, lands on tilemap floor, and sprite transform matches physics body same frame.

**You will learn:**

- Integrate 2D rigid-body simulation before render snapshot build.
- Trace Spark PhysicsWorld2D::Simulate pipeline stage by stage.
- Map physics transforms into sprite draw items via TransformComponent.
- Wire PhysicsSubsystem::Simulate2D from game Update.

## 1. The Problem We Are Solving

Part XI added 2D sprites and tilemaps with transforms driven by gameplay code. **Collision and response** need a dedicated physics step that updates positions before `SceneSubmit` gathers sprite matrices — otherwise characters float through floors, rest on invisible offsets, or appear one frame behind input.

Spark's `PhysicsWorld2D::Simulate` is deliberately **renderer-agnostic**: no Vulkan types, no `SceneRenderParams` mutation. It rebuilds collision geometry from components each frame, integrates rigid bodies, resolves contacts, and solves joints. The only render-facing output is updated `TransformComponent` values and gameplay signals like grounded state.

Milestone: run Platformer2DDemo — player falls, lands on tilemap, sprite Y matches physics body same frame.

## 2. Core Concepts

### PhysicsWorld2D role in the engine

2D physics integrates velocities, resolves collisions, and solves joints — it **never** binds textures, records Vulkan commands, or writes `SceneRenderParams` directly. Spark's `PhysicsWorld2D::Simulate` is a pure ECS mutation pass: read collider and rigidbody components, update internal collision arrays, write results back to `TransformComponent` and `Rigidbody2DComponent` velocity fields.

Signals like `Physics2DTriggerOverlap` emit during the simulate step — gameplay subscribes via `SignalId`, not via render callbacks.

### Simulate pipeline stages

Each frame `Simulate` runs a fixed pipeline:

```text
Rebuild colliders → Integrate dynamics → Static contacts → Dynamic pairs → Joints
```

`ColliderBakePipeline2D::Rebuild` walks ECS and fills `Array<Collider2D>` plus `SpatialHashGrid2D` broadphase. `RigidbodyIntegrator2D::Integrate` applies gravity scaled by `GetGravityScale()`, clamps fall speed, integrates velocity into translation on `TransformComponent`. `ContactResolver2D` resolves static and dynamic contacts. `JointSolver2D` runs last so constraints see post-contact velocities.

Skipping any stage produces visible bugs — integrate without resolve → tunneling through floors; resolve without rebuild → stale AABBs after inspector collider resize.

### Fixed timestep and variable render dt

Spark passes `FrameTiming::deltaTimeSeconds` directly to Simulate — variable dt per frame. For stable stacks and consistent jump height, production code often **accumulates** dt and steps at fixed `1/60` s with a cap (max 4 substeps) to avoid spiral-of-death after hitches.

Platformer2DDemo sets player velocity before Simulate; integrator applies gravity inside Simulate.

### Transform write-back

After integration and contact resolution, dynamic body positions live in `TransformComponent`. Render submit reads transform **after** Simulate in Update — one-frame lag if order is wrong.

Grounded state (`Rigidbody2DComponent::IsGrounded`) is updated during contact resolve — jump FSM queries it before setting upward velocity, as Platformer2DDemo demonstrates.

### PhysicsSubsystem facade

Games do not construct `PhysicsWorld2D` directly — they call `PhysicsSubsystem::Simulate2D(world, timing)`. The subsystem owns world2D settings, broadphase cell size, and shares cell size with `PhysicsQueryWorld2D` for consistent query grids.

Never call Simulate from `OnRender` — simulation belongs in Update before scene snapshot build.

### 2D vs 3D separation

An entity should not be driven by both `PhysicsWorld2D` and `PhysicsWorld3D` on the same Transform — double integration causes jitter. Tag gameplay entities: platformer player uses Rigidbody2D + box collider; ThreeDDemo props use 3D rigidbodies or remain static.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### PhysicsSubsystem::Simulate2D

**Concept:** PhysicsWorld2D role in the engine

**Decision:** PhysicsSubsystem::Simulate2D

**Why:** Engine-owned subsystem; games call from Update only.

**In Spark:** `src/spark/physics/PhysicsSubsystem.cpp`

### PhysicsWorld2D::Simulate ordered calls

**Concept:** Simulate pipeline stages

**Decision:** PhysicsWorld2D::Simulate ordered calls

**Why:** Rebuild → integrate → resolve → joints — explicit stages aid profiling.

**In Spark:** `src/spark/physics/PhysicsWorld2D.cpp`

### FrameTiming.deltaTimeSeconds input

**Concept:** Fixed timestep and variable render dt

**Decision:** FrameTiming.deltaTimeSeconds input

**Why:** Spark baseline uses variable dt; games may wrap with accumulator.

**In Spark:** `PhysicsWorld2D::Simulate` dt guard

### RigidbodyIntegrator2D mutates TransformComponent

**Concept:** Transform write-back

**Decision:** RigidbodyIntegrator2D mutates TransformComponent

**Why:** Single source for render matrices post-physics.

**In Spark:** `src/spark/physics/simulation/RigidbodyIntegrator2D.cpp`

### Simulate2D syncs query grid cell size

**Concept:** PhysicsSubsystem facade

**Decision:** Simulate2D syncs query grid cell size

**Why:** Queries and simulation share broadphase scale.

**In Spark:** `PhysicsSubsystem::SetBroadPhaseCellSize2D`

### Separate component sets per entity

**Concept:** 2D vs 3D separation

**Decision:** Separate component sets per entity

**Why:** No automatic coupling — gameplay chooses 2D or 3D stack.

**In Spark:** Platformer2DDemo vs PhysicsBallThrow3DDemo

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/physics/PhysicsWorld2D.cpp`
- `include/spark/physics/PhysicsWorld2D.hpp`
- `src/spark/physics/PhysicsSubsystem.cpp`
- `src/spark/demo/Platformer2DDemo.cpp`

### Step 1 — Read PhysicsWorld2D::Simulate

Follow full pipeline from collider rebuild through joint solve. Note `dt <= 0` early return for pause menus.

*2D simulation pipeline*

```cpp
// src/spark/physics/PhysicsWorld2D.cpp (lines 15–35)
void PhysicsWorld2D::Simulate(GameWorld& world, const FrameTiming& timing) {
    const float dt = timing.deltaTimeSeconds;
    if (dt <= 0.0F) {
        return;
    }

    Array<Collider2D> colliders;
    SpatialHashGrid2D broadPhase;
    ColliderBakePipeline2D::GetDefault().Rebuild(world, broadPhaseCellSize, colliders, broadPhase);

    RigidbodyIntegrator2D::Integrate(world, dt, settings);
    ContactResolver2D::ResolveAllDynamicsAgainstStatics(world, colliders, broadPhase);

    Array<DynamicBody2D> dynamics;
    SpatialHashGrid2D dynBroad;
    Array<std::uint32_t> dynPairScratch;
    CollectDynamicBodies2D(world, dynamics);
    ContactResolver2D::ResolveDynamicDynamicPairs(dynamics, settings, broadPhaseCellSize, dynBroad, dynPairScratch);

    JointSolver2D::Solve(world, settings);
}
```

### Step 2 — Trace integrator write-back

Gravity, velocity clamp, translation update on TransformComponent.

*Integrate dynamic bodies*

```cpp
// src/spark/physics/simulation/RigidbodyIntegrator2D.cpp (lines 15–50)
void RigidbodyIntegrator2D::Integrate(
        GameWorld& world, const float deltaTimeSeconds, const PhysicsWorld2DSettings& settings) noexcept {
    world.ForEachActiveGameObject([&](GameObject* o) {
        Rigidbody2DComponent* rb = o->GetComponent<Rigidbody2DComponent>();
        TransformComponent* tr = o->GetComponent<TransformComponent>();
        if (rb == nullptr || tr == nullptr || rb->GetBodyType() != RigidbodyBodyType2D::Dynamic) {
            return;
        }
        // requires box or circle collider
        Vector2 v = rb->GetVelocity();
        v.y += settings.gravityY * rb->GetGravityScale() * deltaTimeSeconds;
        if (v.y < -settings.maxFallSpeed) {
            v.y = -settings.maxFallSpeed;
        }
        rb->SetVelocity(v);

        Vector3 pos = tr->GetLocalTransform().translation;
        pos.x += v.x * deltaTimeSeconds;
        pos.y += v.y * deltaTimeSeconds;
        tr->SetTranslation(pos);
    });
}
```

### Step 3 — Wire PhysicsSubsystem in demo

```cpp
void OnUpdate(FrameTiming t) {
    physics.Simulate2D(world, t);
    // then build scene snapshot
}
```

### Step 4 — Run Platformer2DDemo

Player capsule + tilemap static colliders — verify fall, floor stop, jump gated on IsGrounded.

*Velocity setup before Simulate2D*

```cpp
// src/spark/demo/Platformer2DDemo.cpp (lines 372–383)
        Spark::Vector2 v = playerRb->GetVelocity();
        v.x = run * 10.0F;
        const bool jumpPressed = playerRb->IsGrounded() && in.IsKeyPressedThisFrame(GLFW_KEY_SPACE);
        if (jumpPressed) {
            v.y = 12.8F;
        }
        playerRb->SetVelocity(v);

        physics.Simulate2D(world, timing);
```

### Step 5 — Study collider rebuild

Each Simulate rebakes colliders from ECS — inspector resize takes effect next frame.

*Rebuild from active GameObjects*

```cpp
// src/spark/physics/colliders/ColliderBakePipeline2D.cpp (lines 21–41)
void ColliderBakePipeline2D::Rebuild(
        GameWorld& world, const float cellWorldSize,
        Array<Collider2D>& outColliders, SpatialHashGrid2D& outGrid) {
    outColliders.Clear();
    outGrid.Clear();
    outGrid.SetCellSize(cellWorldSize);
    ColliderBakeContext2D context{outColliders, outGrid};
    world.ForEachActiveGameObject([&](GameObject* object) {
        for (std::size_t i = 0; i < strategies.GetSize(); ++i) {
            const IColliderBakeStrategy2D& strategy = *strategies[i];
            if (strategy.Contributes(*object)) {
                strategy.Bake(*object, context);
            }
        }
    });
}
```

### Step 6 — Debug draw AABBs (optional)

Line overlay for collider bounds — verify alignment with sprite feet.

### Step 7 — Cap substeps after hitch

If implementing fixed timestep, clamp accumulator steps to 4 — prevents death spiral after breakpoint pause.

### Step 8 — Milestone — player lands on tilemap

Log player Y before/after Simulate2D; sprite transform matches physics body after submit same frame.

## 5. How This Connects to the Rest of Your Engine

Part **XI** sprite draws read Transform after simulate. Part **XIII** serializes 2D collider components. Chapter 3 deepens collider authoring; Chapter 4 enforces Update ordering with 3D. BroadPhase2DDemo isolates spatial hash tuning without full platformer gameplay.

Combat and bullets must read post-physics transforms — any hit test sampling Transform must run after Simulate2D in the same Update.

## 6. Pitfalls and Debugging

- **Stepping physics after snapshot** — one-frame lag jitter on sprites.
- **Mixing 2D physics on 3D-only entities** — double transform writes if both worlds run.
- **Huge substep count after hitch** — cap substeps (max 4) on fixed timestep wrappers.
- **Skipping collider rebuild** — stale bounds after tilemap edit in editor.
- **Simulate in OnRender** — breaks determinism and input ordering.

**Debugging workflow:** disable Simulate2D — sprite hangs while velocity code runs, confirming submit reads Transform. Log collider count after Rebuild — empty count means bake strategies not registered.

## 7. Exercises

1. Log player Y before/after Simulate2D during free fall; confirm delta matches v*dt within epsilon.
2. Disable Simulate2D call; confirm sprite hangs in air while velocity code still runs.
3. Profile Simulate2D ms on full platformer tilemap vs empty scene.
4. Resize player box collider in inspector; land on ground next frame with updated contact.
5. Run BroadPhase2DDemo; tune cell size; note pair count vs platformer snag behavior.

## Summary

2D physics simulates in Update via PhysicsWorld2D, writing transforms back to ECS before SceneSubmit. Spark rebuilds colliders each step from components through ColliderBakePipeline2D.

---

Next: [3D Physics](02-physics-3d.md).
