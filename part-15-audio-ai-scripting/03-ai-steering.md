# AI Steering

> **Build-Along Milestone:** Enemy seeks player with Arrive slowing in 2D or 3D steering demo.

**You will learn:**

- Compose Seek/Arrive/Flee behaviors via SteeringComposer.
- Configure AiSteeringPlane for 2D vs 3D agents.
- Feed blackboard targets from player transform each Update.

## 1. The Problem We Are Solving

Sliding enemies feel robotic. **Steering behaviors** compute direction from simple rules — seek the player, flee when threatened, arrive without overshoot — and sum into acceleration.

Spark packages behaviors behind `ISteeringBehavior` with shared `AiBlackboard` storage on `AiAgentComponent`. You will drive Transform or rigidbody velocity from composed acceleration **before** SceneSubmit, and optionally link speed to Part XI sprite FSM states.

Milestone: enemy in SteeringShowcase3DDemo seeks player with Arrive slowing — stops within 0.5 m without oscillating.

## 1b. Concept Deep Dive — SteeringComposer Weights

### Behaviors return acceleration, not velocity

Each `ISteeringBehavior` maps agent state + blackboard targets → desired **acceleration**. `SteeringComposer` sums weighted outputs:

```cpp
// include/spark/ai/steering/SteeringBehaviors.hpp
class SteeringComposer final {
    void AddBehavior(const ISteeringBehavior& behavior, const float weight);
    [[nodiscard]] Vector2 Compose(
            const Vector2& positionXZ,
            const Vector2& velocityXZ,
            AiBlackboard& board) const;
};
```

Seek pulls toward target; Arrive scales down inside slowing radius; Flee pushes away. Weights tune dominance — `(Seek, 1.0) + (Avoid, 2.0)` lets avoidance override pure pursuit near walls.

### AiBlackboard on AiAgentComponent

Shared per-agent storage holds target entity id, slowing radius, max speed — behaviors read/write through the board without tight coupling. Update feeds player transform into the board **before** compose, then applies resulting acceleration to velocity or `TransformComponent`.

### 3D variant in SteeringShowcase3DDemo

`SteeringComposer3D` mirrors the 2D API with full Vector3 positions. The demo composes Seek + Arrive for ground enemies — Arrive's slowing radius prevents overshoot oscillation around the player.

Run steering before SceneSubmit so submitted transforms reflect the latest AI motion same frame.

## 2. Core Concepts

### Steering as acceleration composition

**Steering behaviors** map agent state + world targets → desired **acceleration** (not velocity directly).

| Behavior | Output when... |
|----------|----------------|
| Seek | Accelerate toward target |
| Flee | Accelerate away from threat |
| Arrive | Seek with speed scaled inside slowing radius |
| Wander | Perturb heading for idle motion |

Spark's 2D behaviors implement `ISteeringBehavior::ComputeAcceleration(positionXZ, velocityXZ, board)`. A **SteeringComposer** sums weighted outputs — add new behaviors without editing Seek's code.

### AiBlackboard slot convention

Behaviors read targets from **AiBlackboard** float/int slots:

```text
Seek/Arrive: float(0)=targetX, float(1)=targetZ
Arrive:      float(2)=slowingRadius
Wander:      int(0)=RNG seed scratch
```

Gameplay or FSM writes slots each frame; steering reads them. Undeclared slot reuse causes "enemy runs to origin" bugs.

### AiSteeringPlane

`AiAgentComponent` selects how steering `Vector2` maps to motion:

| Plane | Maps to |
|-------|---------|
| `XzWorld` | world X and Z (3D character on ground plane) |
| `XyRigidbody2D` | `Rigidbody2DComponent` velocity X/Y |

Pick the plane matching physics from Part XIV — 2D platformer agents use `XyRigidbody2D`; Maze3D enemies use `XzWorld`.

### Arrive vs Seek stability

Raw Seek overshoots oscillating targets. **Arrive** scales desired speed by `dist / slowRadius` inside the radius:

```text
if dist < slowRadius: speed = dist / slowRadius
desired = normalize(offset) * speed
accel = (desired - velocity) * weight
```

Tune `slowRadius` relative to agent max speed — Spark defaults `br > 0.05` guard in `SteeringArrive`.

### AiAgentComponent as AI hub

Beyond steering, `AiAgentComponent` optionally hosts FSM, GOAP library, fuzzy module, path polyline — each gated by enable flags. For this chapter focus on:

```text
blackboard + steeringComposer + maxSpeed + steeringPlane
```

`GameAiSubsystem::SimulateGameAi` ticks each enabled agent each frame.

### Linking motion to animation

Part XI sprite FSM selects clips from **speed** and blackboard combat int:

```text
steering → velocity magnitude → FSM "run" vs "idle"
blackboard combat int → hurt/attack overlay clip
```

Ai subsystem writes velocity; animation reads state — do not drive steering from animation timers.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### ISteeringBehavior + SteeringComposer

**Concept:** Steering as acceleration composition

**Decision:** ISteeringBehavior + SteeringComposer

**Why:** Weighted sum extensibility — add Flee without modifying Seek source.

**In Spark:** `include/spark/ai/steering/SteeringBehaviors.hpp`.

### Float slots 0–2 for 2D seek/arrive

**Concept:** AiBlackboard slot convention

**Decision:** Float slots 0–2 for 2D seek/arrive

**Why:** Keeps behavior implementations stateless and reusable across demos.

**In Spark:** `src/spark/ai/steering/SteeringBehaviors.cpp`.

### Enum on AiAgentComponent

**Concept:** AiSteeringPlane

**Decision:** Enum on AiAgentComponent

**Why:** Single agent code path serves 2D physics and 3D XZ walkers.

**In Spark:** `include/spark/ecs/components/ai/AiAgentComponent.hpp`.

### SteeringArrive uses slowing radius slot

**Concept:** Arrive vs Seek stability

**Decision:** SteeringArrive uses slowing radius slot

**Why:** Teaches stable chasing without hand-tuned velocity clamps in gameplay.

**In Spark:** SteeringBehaviors.cpp lines 34–51.

### kAiBlackboardIntSprite2DCombatCommand

**Concept:** Linking motion to animation

**Decision:** kAiBlackboardIntSprite2DCombatCommand

**Why:** Documents cross-system slot for FSM without hard includes between AI and animation modules.

**In Spark:** `include/spark/ai/AiBlackboardSprite2DAnim.hpp`.

### GameAiSubsystem::SimulateGameAi

**Concept:** AiAgentComponent as AI hub

**Decision:** GameAiSubsystem::SimulateGameAi

**Why:** Central tick for nav, perception, and agent subsystem — same entry point for C++ and interop.

**In Spark:** `src/spark/ai/GameAiSubsystem.cpp`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/ai/steering/SteeringBehaviors.cpp`
- `include/spark/ai/steering/SteeringBehaviors.hpp`
- `include/spark/ecs/components/ai/AiAgentComponent.hpp`
- `src/spark/ai/GameAiSubsystem.cpp`

### Step 1 — Browse steering headers

*Seek, Arrive, SteeringComposer*

```cpp
// include/spark/ai/steering/SteeringBehaviors.hpp (lines 8–80)
class SteeringSeek final : public ISteeringBehavior {
public:
    Vector2 ComputeAcceleration(
            const Vector2& positionXZ, const Vector2& velocityXZ, AiBlackboard& board) const override;
};

class SteeringArrive final : public ISteeringBehavior {
public:
    Vector2 ComputeAcceleration(
            const Vector2& positionXZ, const Vector2& velocityXZ, AiBlackboard& board) const override;
};

class SteeringComposer final {
public:
    void AddBehavior(const ISteeringBehavior& behavior, const float weight);
    [[nodiscard]] Vector2 Compose(
            const Vector2& positionXZ, const Vector2& velocityXZ, AiBlackboard& board) const;
};
```

### Step 2 — Read Seek and Arrive implementation

*Seek and Arrive acceleration*

```cpp
// src/spark/ai/steering/SteeringBehaviors.cpp (lines 9–51)
Vector2 SteeringSeek::ComputeAcceleration(
        const Vector2& positionXZ, const Vector2& /*velocityXZ*/, AiBlackboard& board) const {
    const Vector2 target{board.GetFloat(0), board.GetFloat(1)};
    Vector2 desired = target - positionXZ;
    if (desired.LengthSquared() < 1.0e-8F) {
        return Vector2::Zero;
    }
    return desired.Normalized() * weight;
}

Vector2 SteeringArrive::ComputeAcceleration(
        const Vector2& positionXZ, const Vector2& velocityXZ, AiBlackboard& board) const {
    const Vector2 target{board.GetFloat(0), board.GetFloat(1)};
    const float slowRadius = board.GetFloat(2) > 0.05F ? board.GetFloat(2) : 0.05F;
    Vector2 offset = target - positionXZ;
    const float dist = offset.Length();
    if (dist < 1.0e-5F) {
        return velocityXZ * (-1.0F) * weight;
    }
    float speed = dist < slowRadius ? dist / slowRadius : 1.0F;
    return (offset.Normalized() * speed - velocityXZ) * weight;
}
```

### Step 3 — Spawn AiAgentComponent entity

```cpp
AiAgentComponent* agent = enemy->AddComponent<AiAgentComponent>();
agent->SetSteeringPlane(AiSteeringPlane::XzWorld);
agent->SetMaxSpeed(4.5f);
```

Write player world XZ into `blackboard.SetFloat(0/1, ...)` each Update.

### Step 4 — Compose Seek behavior

```cpp
SteeringComposer composer;
composer.AddBehavior(seek, 1.0f);
Vector2 accel = composer.Compose(posXZ, velXZ, agent->GetBlackboard());
velocity += accel * dt;
```

Clamp velocity to `maxSpeed` after integration.

### Step 5 — Switch to Arrive near player

Set float slot 2 = slowing radius (e.g. 2.0f). Swap Seek for Arrive when distance < threshold.

### Step 6 — Run SteeringShowcase3DDemo

Launch shell option **14 - 3D steering**. Observe multiple agents — grep demo for blackboard write pattern.

### Step 7 — Connect sprite FSM speed

On 2D enemy: if `velocity.Length() > 0.1f` FSM run else idle.

### Step 8 — Milestone — Arrive without oscillation

Enemy stops within 0.5 m of player without reversing direction for 3 s.

## 5. How This Connects to the Rest of Your Engine

Part **XI** sprite FSM reads velocity and blackboard combat int. Part **XIV** physics may carry agent velocity on `Rigidbody2DComponent`. Chapter 4 exposes `spark_world_simulate_game_ai` for managed parity.

Demo **14** validates 3D steering; Platformer enemies may use simpler chase without full agent stack — scale complexity to game genre.

## 6. Pitfalls and Debugging

- **Steering after SceneSubmit** — one-frame visual lag; run in OnUpdate early.
- **Forgetting to write blackboard targets** — enemy steers toward (0,0).
- **maxSpeed = 0** — zero displacement; guard integration.
- **Xz steering on Xy rigidbody** — motion on wrong plane; set AiSteeringPlane explicitly.
- **Composing Seek+Flee without weights** — jitter near equilibrium; tune weights or switch behaviors by state.

## 7. Exercises

1. Fill steering tuning table for your enemy; document maxSpeed and slowingRadius.
2. Run QA checklist: Seek only (overshoot), Arrive only (decelerate), zero target (no NaN).
3. Add debug line from agent to blackboard target in ImGui or log.
4. Compare `XzWorld` vs `XyRigidbody2D` on same entity — describe wrong-plane symptom.
5. Log blackboard floats 0–1 every frame while player moves — confirm no stale values.

## Summary

AI steering composes ISteeringBehavior accelerations on AiBlackboard data with AiSteeringPlane selecting 2D vs XZ worlds. Spark separates behavior math from ECS motion integration and animation FSM.

---

Next: [C# Scripting](04-scripting.md).
