# 3D Physics

> **Build-Along Milestone:** Dynamic sphere collides with ground and renders at physics-updated position same frame.

**You will learn:**

- Integrate PhysicsWorld3D through PhysicsSubsystem with substep configuration.
- Run PhysicsBallThrow3DDemo as 3D physics baseline.
- Order character controller and trigger simulation after base 3D step.
- Keep physics mutation out of render and Vulkan code paths.

## 1. The Problem We Are Solving

3D games need rigid bodies, controllable characters, and trigger volumes. Spark splits these across simulators united by **PhysicsSubsystem** — you call `Simulate3D` or `SimulateAll3D` from Update, then submit the scene with updated transforms. Substepping trades CPU for contact stability on stacks and fast projectiles.

Rendering never imports physics headers beyond gameplay systems — the bridge is always `TransformComponent`.

Milestone: throw ball in PhysicsBallThrow3DDemo — bounces on ground, mesh position matches physics same frame.

## 2. Core Concepts

### PhysicsWorld3D scope

3D simulation handles rigid bodies (dynamic, static, kinematic), box/sphere/capsule colliders, and joints. Spark splits **character controllers** and **trigger volumes** into separate simulators called after base rigidbody step — controllers override raw integration for player capsules; triggers emit overlap events without blocking motion.

Swept CCD in the integrator path reduces tunneling for fast movers — enable when velocity × dt exceeds collider thickness.

### Substepped 3D integration

Unlike 2D's single-pass dt, `PhysicsWorld3D::Simulate` divides frame dt by `settings.substeps` and runs integrate → static impulses → penetration resolve → dynamic pair velocities → joints **per substep**. Collider rebuild happens **once** per frame before the substep loop.

More substeps improve stack stability and reduce tunneling at higher CPU cost. `resolveIterations` and `jointIterations` add inner solver quality separate from substep count — PhysicsBallThrow3DDemo exposes all three plus `baumgarteContactBias` (often zero for clean bouncing balls).

### SimulateAll3D ordering

`PhysicsSubsystem::SimulateAll3D` runs three calls in sequence:

```text
world3D.Simulate → characterController3D → triggerVolumes3D
```

Base dynamics move props and loose spheres; controllers adjust player transforms after impulses; triggers detect overlaps for gameplay callbacks without render coupling.

### Transform and collider sync in 3D

Integrator applies translation delta to both `TransformComponent` and internal `DynamicBody3D` collider via `ApplyTranslationDeltaToBody`. Rotation integrates quaternion on transform for rolling spheres. Render submit uses same transform matrices as 2D.

### PhysicsBallThrow3DDemo vertical slice

Demo spawns static ground with box collider + physics material, dynamic ball with sphere mesh/collider, optional cube obstacles with varying restitution. Throw applies velocity impulse on click; R resets pose. Validates 3D bake pipeline, contact audio cues on bounce, and GUI tuning of solver parameters.

### No render coupling

3D physics headers stay out of VulkanRenderer. Gameplay systems call Simulate; `FillStandardLitSceneFromWorld` reads transforms unchanged. Shadow casters (Part IX) use post-physics transforms automatically when Update order is correct (Chapter 4).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### PhysicsSubsystem facade with separate simulators

**Concept:** PhysicsWorld3D scope

**Decision:** PhysicsSubsystem facade with separate simulators

**Why:** Simulate3D, SimulateCharacterControllers3D, SimulateTriggerVolumes3D exposed individually.

**In Spark:** `include/spark/physics/PhysicsSubsystem.hpp`

### PhysicsWorld3DSettings.substeps

**Concept:** Substepped 3D integration

**Decision:** PhysicsWorld3DSettings.substeps

**Why:** Configurable quality vs performance; demo exposes in GUI.

**In Spark:** `PhysicsWorld3D::Simulate` substep loop

### Controllers after rigidbody base

**Concept:** SimulateAll3D ordering

**Decision:** Controllers after rigidbody base

**Why:** Player movement overrides raw rigidbody result same frame.

**In Spark:** `PhysicsSubsystem::SimulateAll3D`

### ApplyTranslationDeltaToBody helper

**Concept:** Transform and collider sync in 3D

**Decision:** ApplyTranslationDeltaToBody helper

**Why:** Keeps narrow-phase geometry aligned with rendered mesh origin.

**In Spark:** `RigidbodyIntegrator3D.cpp`

### Throw impulse + material tuning

**Concept:** PhysicsBallThrow3DDemo vertical slice

**Decision:** Throw impulse + material tuning

**Why:** Minimal 3D physics regression scene alongside ThreeDDemo.

**In Spark:** `src/spark/demo/PhysicsBallThrow3DDemo.cpp`

### Same SceneSubmit as ThreeDDemo

**Concept:** No render coupling

**Decision:** Same SceneSubmit as ThreeDDemo

**Why:** Physics only mutates ECS transforms and rigidbody state.

**In Spark:** Standard lit gather after simulate

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/physics/PhysicsSubsystem.cpp`
- `src/spark/physics/PhysicsWorld3D.cpp`
- `include/spark/demo/PhysicsBallThrow3DDemo.hpp`
- `src/spark/demo/PhysicsBallThrow3DDemo.cpp`

### Step 1 — Read SimulateAll3D ordering

Three simulate calls — know when to use full bundle vs individual passes.

*SimulateAll3D sequence*

```cpp
// src/spark/physics/PhysicsSubsystem.cpp (lines 12–33)
void PhysicsSubsystem::Simulate2D(GameWorld& world, const FrameTiming& timing) {
    queries2D.SetCellWorldSize(world2D.GetBroadPhaseCellSize());
    world2D.Simulate(world, timing);
}

void PhysicsSubsystem::Simulate3D(GameWorld& world, const FrameTiming& timing) {
    world3D.Simulate(world, timing);
}

void PhysicsSubsystem::SimulateAll3D(GameWorld& world, const FrameTiming& timing) {
    world3D.Simulate(world, timing);
    characterController3D.Simulate(world, timing);
    triggerVolumes3D.Simulate(world, timing);
}
```

### Step 2 — Study 3D substep loop

Rebuild colliders once per frame; integrate/resolve repeat per substep.

*PhysicsWorld3D::Simulate*

```cpp
// src/spark/physics/PhysicsWorld3D.cpp (lines 17–41)
void PhysicsWorld3D::Simulate(GameWorld& world, const FrameTiming& timing) {
    const float dt = timing.deltaTimeSeconds;
    if (dt <= 0.0F) {
        return;
    }

    const int substeps = std::max(1, settings.substeps);
    const float h = dt / static_cast<float>(substeps);

    Array<Collider3D> colliders;
    SpatialHashGrid3D broadPhase;
    ColliderBakePipeline3D::GetDefault().Rebuild(world, broadPhaseCellSize, colliders, broadPhase);

    Array<std::uint32_t> queryScratch;
    Array<DynamicBody3D> bodies;
    CollectDynamicBodies3D(world, bodies);

    for (int s = 0; s < substeps; ++s) {
        RigidbodyIntegrator3D::IntegrateSubstep(bodies, h, settings, colliders, broadPhase, queryScratch);
        ContactResolver3D::ApplyStaticNormalImpulses(bodies, colliders, broadPhase, queryScratch, h, settings);
        ContactResolver3D::ResolvePenetrations(bodies, colliders, broadPhase, queryScratch, settings);
        ContactResolver3D::ResolveDynamicPairVelocities(bodies, h, settings);
        JointSolver3D::SolveSubstep(world, h, settings);
    }
}
```

### Step 3 — Run PhysicsBallThrow3DDemo

Launch demo — ground static collider, throwable dynamic ball, cube obstacles. Tune substeps slider effect on ball stack.

### Step 4 — Trace throw and Simulate call

*Throw input and Simulate3D*

```cpp
// src/spark/demo/PhysicsBallThrow3DDemo.cpp (lines 198–230)
        if (in.IsMouseButtonPressedThisFrame(0)) {
            const Spark::Vector3 dir = camera.Forward().Normalized();
            ballTr->SetTranslation(camera.position + dir * 0.55F);
            ballRb->SetAngularVelocity(Spark::Vector3::Zero);
            ballRb->SetVelocity(dir * guiThrow + Spark::Vector3::UnitY * upKick);
        }
        if (in.IsKeyPressedThisFrame(GLFW_KEY_R)) {
            ballTr->SetTranslation({0.0F, 1.25F, 2.0F});
            ballRb->SetVelocity(Spark::Vector3::Zero);
            ballRb->SetAngularVelocity(Spark::Vector3::Zero);
        }

        PhysicsWorld3DSettings& phys = physics.GetWorld3D().GetSettings();
        phys.gravityY = guiGravityY;
        phys.substeps = 3;
        phys.resolveIterations = 10;
        phys.baumgarteContactBias = 0.0F;
        physics.Simulate3D(world, timing);
```

### Step 5 — Spawn ground + static collider

Ground box `hy` offset aligns collider top with visual y=0 plane.

*Ground entity with box collider and material*

```cpp
// src/spark/demo/PhysicsBallThrow3DDemo.cpp (lines 40–52)
        const float hx = Spark::kSceneGroundHalfExtent;
        const float hy = 0.18F;
        ground->AddComponent<Spark::BoxCollider3DComponent>(Spark::Vector3{hx, hy, hx}, Spark::Vector3{0.0F, hy, 0.0F});
        ground->AddComponent<Spark::Rigidbody3DComponent>(Spark::RigidbodyBodyType3D::Static, 1.0F);
        ground->AddComponent<Spark::PhysicsMaterial3DComponent>(0.62F, 0.48F, 0.58F);
```

### Step 6 — Tune PhysicsMaterial3D

Restitution and friction on ball vs floor — compare bounce height with GUI restitution slider.

### Step 7 — Character controller optional

If using `CharacterController3DComponent`, call `SimulateCharacterControllers3D` after `Simulate3D`.

### Step 8 — Trigger volume overlap

`SimulateTriggerVolumes3D` for pickup zones — gameplay listens to signals, not renderer.

## 5. How This Connects to the Rest of Your Engine

Part **V** camera follow targets post-physics transform. Part **IX** shadow casters use same transforms. Part **XIII** serializes PhysicsMaterial3D and collider components. Chapter 4 documents Update order with 2D platformer and 3D throw demo together.

Character controller and trigger volumes require `SimulateAll3D` ordering — never call controller before base rigidbody step.

## 6. Pitfalls and Debugging

- **Simulate after render snapshot** — visible lag on moving meshes.
- **Static mesh marked dynamic** — unnecessary solver work; use Static body type for level geo.
- **Unit scale mismatch** — physics meters vs art cm; document world unit convention.
- **Collider smaller than mesh** — visible interpenetration on landing.
- **Angular velocity unset on reset** — ball spins after R key if angular not cleared.

**Debugging workflow:** log linear velocity magnitude before and after first ground contact each throw — restitution changes should alter post-bounce velocity predictably.

## 7. Exercises

1. Throw 10 balls with substeps 1 vs 4; screenshot stack height after 5 seconds.
2. Log Simulate3D ms vs scene record ms same frame in BallThrow demo.
3. Press R reset; confirm linear and angular velocity zero before next throw.
4. Remove ground collider; ball falls through mesh; confirm no solver crash.
5. Tune GUI baumgarte to non-zero; observe spongy contacts vs clean bounce at zero.

## Summary

3D physics runs through PhysicsSubsystem with substepped integration and ordered controller/trigger passes. Spark demonstrates via PhysicsBallThrow3DDemo with tunable solver settings.

---

Next: [Colliders](03-colliders.md).
