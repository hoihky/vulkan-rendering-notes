# Sync Transforms

> **Build-Along Milestone:** Player sprite and debug collider align same frame with no visible physics lag.

**You will learn:**

- Enforce physics-before-submit update order in engine tick.
- Distinguish kinematic authoring from dynamic simulation.
- Diagnose and fix one-frame transform lag between physics and rendering.
- Gate simulation in editor vs play mode appropriately.

## 1. The Problem We Are Solving

Physics and rendering share **TransformComponent** as the bridge. If Simulate runs after snapshot build, players see one-frame lag. If gameplay and physics both write transforms without rules, entities jitter. Spark writes transforms inside Simulate integrators — your responsibility is **Update ordering** and kinematic/dynamic policy across 2D, 3D, and editor modes.

Milestone: log transform Y immediately after Simulate2D and at SceneSubmit entry — values must match same frame.

## 2. Core Concepts

### Canonical Update order

Production frame tick for physics-driven games:

```text
Input → Gameplay pre-physics → Physics Simulate → Gameplay post-physics → Animation → SceneSubmit → Present
```

Platformer2DDemo: read input → set player velocity → `Simulate2D` → combat/bullets read post-physics position → later render gather fills sprites from Transform.

PhysicsBallThrow3DDemo: camera input → throw velocity → `Simulate3D` → audio bounce check on velocity delta → render uses updated ball transform.

| Phase | Platformer2D | BallThrow3D | Editor edit |
|-------|--------------|-------------|-------------|
| Input | velocity | throw | UI + camera |
| Physics | Simulate2D | Simulate3D | skip |
| Submit | sprite gather | FillStandardLit | FillStandardLit |

### Spark's implicit sync model

Some engines expose explicit SyncToPhysics / SyncFromPhysics. Spark integrates sync **inside** Simulate: integrators and resolvers write `TransformComponent` directly after updating body state. There is no separate sync pass if you respect Update ordering — Transform is authoritative for rendering that frame once Simulate returns.

Rigidbody2DComponent holds velocity and grounded state; TransformComponent holds pose for rendering. Gameplay reads velocity from Rigidbody for animation; render reads Transform for matrices.

### Kinematic vs dynamic bodies

**Dynamic** bodies: physics drives transform (gravity, contacts). **Kinematic** bodies: gameplay sets transform each frame; physics moves collider with body for interaction but does not integrate velocity. Moving platforms in platformers are often kinematic — script sets path; player rides contact resolver.

Mark body type correctly on `Rigidbody2DComponent` / `Rigidbody3DComponent` — wrong type causes platform to fall or player to ignore platform motion.

### One-frame lag diagnosis

If visuals trail physics by exactly one frame, audit call order: Simulate likely runs **after** FillSceneFromWorld / SetSceneRenderParams. Move Simulate earlier in Update or remove duplicate transform write elsewhere.

Log transform Y in same frame immediately after Simulate and at start of submit — values must match.

### Render interpolation (advanced)

Fixed physics timestep with variable render rate can stutter without **interpolation**: store previous and current physics transforms, blend for render alpha between steps. Spark baseline uses latest transform — acceptable at 60fps variable dt; add interp if you adopt aggressive fixed substeps.

### Editor and paused simulation

Part XIII editor may skip Simulate while authoring — inspector edits Transform directly, submit shows edits immediately. Never run Simulate and direct inspector transform writes on same dynamic entity same frame — race on position.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Engine tick owns ordering contract

**Concept:** Canonical Update order

**Decision:** Engine tick owns ordering contract

**Why:** Demos document order in OnUpdate; custom games copy Platformer2DDemo pattern.

**In Spark:** `Platformer2DDemo` Update before submit

### Transform write inside integrator/resolver

**Concept:** Spark's implicit sync model

**Decision:** Transform write inside integrator/resolver

**Why:** No separate sync API — reduces duplicate state bugs.

**In Spark:** `RigidbodyIntegrator2D` SetTranslation

### RigidbodyBodyType enum on components

**Concept:** Kinematic vs dynamic bodies

**Decision:** RigidbodyBodyType enum on components

**Why:** Moving platforms kinematic; player dynamic.

**In Spark:** `Rigidbody2DComponent` body type

### Simulate before FillStandardLitSceneFromWorld

**Concept:** One-frame lag diagnosis

**Decision:** Simulate before FillStandardLitSceneFromWorld

**Why:** EditorApplication and demos gather scene after physics when simulate enabled.

**In Spark:** Compare Platformer2DDemo vs misordered stub

### Optional prev/current transform buffers

**Concept:** Render interpolation (advanced)

**Decision:** Optional prev/current transform buffers

**Why:** Spark baseline: latest transform only; document extension point.

**In Spark:** Future PhysicsInterpolationComponent pattern

### Gate Simulate when !editorSimulate

**Concept:** Editor and paused simulation

**Decision:** Gate Simulate when !editorSimulate

**Why:** Author transforms without physics fighting inspector.

**In Spark:** SceneEditor3DDemo simulate flag

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/physics/PhysicsSubsystem.cpp`
- `src/spark/demo/Platformer2DDemo.cpp`
- `src/spark/demo/PhysicsBallThrow3DDemo.cpp`
- `src/spark/scene/SceneSubmit.cpp`

### Step 1 — Audit Platformer2DDemo Update order

Input and velocity setup, then Simulate2D, then gameplay/combat — physics before render gather.

*Simulate2D after input velocity*

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

### Step 2 — Confirm PhysicsSubsystem entry

Simulate2D forwards to world2D — call site must precede scene fill in your engine loop.

*Simulate2D entry point*

```cpp
// src/spark/physics/PhysicsSubsystem.cpp (lines 12–15)
void PhysicsSubsystem::Simulate2D(GameWorld& world, const FrameTiming& timing) {
    queries2D.SetCellWorldSize(world2D.GetBroadPhaseCellSize());
    world2D.Simulate(world, timing);
}
```

### Step 3 — Detect one-frame lag experiment

Intentionally move Simulate after submit one run — screenshot visible lag; revert immediately.

### Step 4 — Kinematic platform test

Move platform transform from script each frame; player dynamic body should ride without platform falling.

### Step 5 — 3D throw demo ordering

PhysicsBallThrow3DDemo::Simulate runs before render in demo tick — same contract as 2D.

*Simulate3D in demo Simulate hook*

```cpp
// src/spark/demo/PhysicsBallThrow3DDemo.cpp (lines 222–230)
        PhysicsWorld3DSettings& phys = physics.GetWorld3D().GetSettings();
        phys.gravityY = guiGravityY;
        phys.substeps = 3;
        phys.resolveIterations = 10;
        phys.baumgarteContactBias = 0.0F;
        physics.Simulate3D(world, timing);
```

### Step 6 — Skinned mesh vs physics capsule

Part VI animation may offset visual mesh while physics capsule drives entity root — document split for characters.

### Step 7 — Editor paused physics

When editor simulate disabled, skip PhysicsSubsystem — inspector Transform edits submit directly.

### Step 8 — Document team contract

Add comment block in engine tick listing canonical order:

```text
// Input → Gameplay → physics.Simulate2D/3D → Animation → FillScene → Present
```

## 5. How This Connects to the Rest of Your Engine

Part **XI** anim FSM reads physics velocity after Simulate. Part **V** camera follow targets post-physics transform. Part **XIII** editor may disable simulate while authoring. Part **XVI** profilers split physics CPU from submit CPU in frame timing overlays.

SceneSubmit caching between frames without invalidating after Simulate reproduces lag even when tick order looks correct — always rebuild snapshot after physics.

## 6. Pitfalls and Debugging

- **Double simulate per frame** — 2× gravity if called from Update and fixed tick both.
- **Editing transform during simulate** — race; edit when paused or entity is kinematic.
- **3D and 2D both on one entity** — conflicting transform writes.
- **Simulate after FillStandardLitSceneFromWorld** — most common lag bug in student engines.
- **Camera follow before simulate** — one frame camera lag separate from sprite lag.

**Debugging workflow:** log transform Y after Simulate and at SceneSubmit — must match. Binary search call order: comment out subsystems until lag disappears.

## 7. Exercises

1. Log transform Y immediately after Simulate2D and at SceneSubmit entry — values must match.
2. Move Simulate after submit one frame; capture screenshot showing sprite inside floor mesh.
3. Draw engine tick order diagram comparing Platformer2DDemo and PhysicsBallThrow3DDemo call graphs.
4. Toggle editor simulate off; move prop in inspector; confirm no physics snap back on release.
5. Add scoped timer labels around Simulate and Submit; paste frame graph screenshot in lab notes.

## Summary

Physics must complete before SceneSubmit each frame. Spark updates TransformComponent during Simulate — enforce ordering in the engine loop and respect kinematic vs dynamic roles.

---

Continue to [Part XV — Audio, AI, Scripting](../part-15-audio-ai-scripting/index.md).
