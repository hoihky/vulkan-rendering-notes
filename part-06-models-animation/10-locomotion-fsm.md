# Locomotion FSM

> **Build-Along Milestone:** Part VI complete — WASD drives idle/walk/run crossfades on skinned Khronos character.

**You will learn:**

- Drive Animator clip selection from horizontal speed and sprint input.
- Resolve idle/walk/run clips from skeleton name heuristics.
- Order Character3DAnimFsm before AnimatorComponent on the same entity.

## 1. The Problem We Are Solving

Raw animation playback without **locomotion logic** feels robotic: walk plays while standing still, run triggers
on tiny jitter. A finite-state machine maps gameplay measurements to discrete clips and crossfades.

Finish Part VI: wire Character3DAnimFsm to your skinned hero so WASD selects idle, walk, and run clips.

### Locomotion FSM for beginners

**Animator** plays clips. **FSM** decides **which** clip based on gameplay:

```text
Measure horizontal speed from velocity
  speed < 0.1  → Idle clip
  speed < 4.0  → Walk clip
  speed >= 4.0 → Run clip (if sprint held)
```

**Finite state machine** = discrete states + rules to switch:

```text
        ┌──────┐  speed up   ┌──────┐  sprint   ┌─────┐
        │ Idle │ ──────────► │ Walk │ ────────► │ Run │
        └──────┘ ◄────────── └──────┘ ◄──────── └─────┘
              speed down         speed down
```

**Crossfade on transition:** FSM calls `SetClipIndexWithCrossfade` — Animator blends poses over ~0.15s so feet don't pop.

**Clip name heuristics:** Spark searches clip names like `"Walk"`, `"Run"`, `"Idle"` on the skeleton — Khronos samples often match; custom rigs may need explicit indices.

**Component order:** `Character3DAnimFsmComponent` updates **before** `AnimatorComponent` on same entity — FSM sets desired clip, Animator advances time.

Part VI complete: mesh → glTF → skeleton → animator → GPU skinning → locomotion FSM.

## 1b. Concept Deep Dive — FSM Logic in Spark

### LocomotionClipSet — resolved from skeleton clip names

```cpp
// include/spark/animation/LocomotionClipSet.hpp
constexpr std::uint32_t kInvalidAnimClipIndex = 0xFFFFFFFFu;

struct LocomotionClipSet {
    std::uint32_t idle = 0;
    std::uint32_t walk = 0;
    std::uint32_t run = kInvalidAnimClipIndex;   // disabled when invalid
    std::uint32_t attack = kInvalidAnimClipIndex;
};
```

`ResolveLocomotionClipsFromSkeleton` searches clip names case-insensitively for `"idle"`, `"walk"`, `"run"`, `"attack"` substrings — Khronos sample assets usually match out of the box.

### Full OnUpdate decision tree

```cpp
// src/spark/ecs/components/animation/Character3DAnimFsmComponent.cpp (lines 123–179)
    const bool holdingAttack = attackOk && cur == attackClip && !anim->IsClipFinished();
    if (holdingAttack) {
        want = attackClip;
        wantLoop = AnimLoopMode::Once;
    } else if (pendingAttack && attackOk) {
        want = attackClip;
        wantLoop = AnimLoopMode::Once;
    } else if (manualClipActive && ClipValid_(*anim, manualClip)) {
        want = manualClip;
        wantLoop = manualLoop;
    } else if (!locomotionDrivingEnabled) {
        return;
    } else {
        const float speed = MeasureLocomotionSpeed_(owner, timing);  // XZ velocity or transform delta
        const bool moving = speed >= walkThresh;
        const bool wantSprint = runOk && speed >= runThresh;
        if (!moving)       want = idleClip;
        else if (wantSprint) want = runClip;
        else if (walkOk)   want = walkClip;
    }
    ApplyClip_(*anim, want, wantLoop);  // crossfade when clip changes
```

**Speed sources:** `Rigidbody3DComponent::GetVelocity()` XZ magnitude, or world-position delta when no physics body. `SetLocomotionInput(moving, sprint)` overrides for network prediction.

### Default thresholds (tunable per character)

```cpp
// Character3DAnimFsmComponent.hpp
float walkThresh = 0.35F;      // m/s — below = idle
float runThresh = 2.2F;        // m/s — above = run (if run clip valid)
float crossfadeDuration = 0.18F;
```

### Configure from imported asset

```cpp
fsm->ConfigureLocomotionFromSkeleton(*asset.skeleton, asset.walkClipIndex);
```

Uses `walkClipIndex` from `SkinnedGltfAsset` as fallback when name heuristics fail.

## 2. Core Concepts

### FSM vs Animator responsibilities

`Character3DAnimFsmComponent` decides **which clip** (idle/walk/run/attack). `AnimatorComponent` decides **how time advances** and crossfades — keep selection out of render code.


### Speed thresholds on XZ plane

Locomotion measures horizontal speed from rigidbody velocity or finite-difference world position. Default walk threshold 0.35 m/s, run 2.2 m/s — tunable per character scale.


### ResolveLocomotionClipsFromSkeleton

Matches clip names case-insensitively: idle/survey/rest, walk, run/jog/sprint, attack/punch — fallback walk index from glTF loader when no walk name exists.


### Update priority ordering

FSM uses `ComponentUpdatePriority::AnimationDriver`; Animator uses `AnimatorPlayback` — FSM must run **first** on the same GameObject to set clip before time advances.


### Input override path

`SetLocomotionInput(moving, sprint)` lets gameplay drive gait directly (WASD demos) instead of inferring speed from transform delta — reduces foot sliding when physics moves the root.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Character3DAnimFsmComponent

**Concept:** FSM vs Animator responsibilities

**Decision:** Character3DAnimFsmComponent

**Why:** ApplyClip_ calls SetClipIndexWithCrossfade on sibling Animator — no duplicate playback state.

**In Spark:** `include/spark/ecs/components/animation/Character3DAnimFsmComponent.hpp`

### MeasureLocomotionSpeed_

**Concept:** Speed thresholds on XZ plane

**Decision:** MeasureLocomotionSpeed_

**Why:** Uses Rigidbody3D velocity when present; else differentiates transform translation.

**In Spark:** `Character3DAnimFsmComponent.cpp` — MeasureLocomotionSpeed_

### LocomotionClipSet helper

**Concept:** ResolveLocomotionClipsFromSkeleton

**Decision:** LocomotionClipSet helper

**Why:** Centralizes name heuristics so demos and games share the same clip resolution rules.

**In Spark:** `src/spark/animation/LocomotionClipSet.cpp`

### AnimationDriver before AnimatorPlayback

**Concept:** Update priority ordering

**Decision:** AnimationDriver before AnimatorPlayback

**Why:** Documented in component header — add FSM before Animator on the GameObject.

**In Spark:** `Character3DAnimFsmComponent.hpp` comment lines 20–21

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/ecs/components/animation/Character3DAnimFsmComponent.hpp`
- `src/spark/ecs/components/animation/Character3DAnimFsmComponent.cpp`
- `src/spark/animation/LocomotionClipSet.cpp`
- `src/spark/demo/CharacterCameraDemo.cpp`

### Step 1 — Read FSM public API

*ConfigureLocomotionFromSkeleton, thresholds, SetLocomotionInput.*

```cpp
// include/spark/ecs/components/animation/Character3DAnimFsmComponent.hpp (lines 16–48)
/**
 * Selects <c>AnimatorComponent</c> locomotion clips from horizontal speed (XZ plane).
 * Optional one-shot attack clip uses <c>AnimLoopMode::Once</c>. Locomotion changes crossfade.
 *
 * Add **before** <c>AnimatorComponent</c> on the same <c>GameObject</c> so <c>OnUpdate</c> runs first.
 */
class Character3DAnimFsmComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::Character3DAnimFsm;

    [[nodiscard]] ComponentKind Kind() const noexcept override { return TypeKind; }

    [[nodiscard]] int UpdatePriority() const noexcept override {
        return ComponentUpdatePriority::AnimationDriver;
    }

    Character3DAnimFsmComponent() noexcept = default;

    void OnUpdate(const FrameTiming& timing, GameObject& owner, IEngineContext& context) override;

    void SetLocomotionClips(std::uint32_t idleClipIndex, std::uint32_t walkClipIndex, std::uint32_t runClipIndex) noexcept;
    /**
     * Resolves idle/walk/run/attack clips from <c>skeleton</c> clip names and applies them on this FSM.
     * <c>walkClipFallback</c> is used when no walk-named clip exists.
     */
    void ConfigureLocomotionFromSkeleton(const Skeleton& skeleton, std::uint32_t walkClipFallback = 0) noexcept;
    /** Pass index >= clip count to disable attack overlay. */
    void SetAttackClip(std::uint32_t attackClipIndex) noexcept;

    void SetWalkSpeedThreshold(float metersPerSecond) noexcept;
    void SetRunSpeedThreshold(float metersPerSecond) noexcept;
    void SetCrossfadeDuration(float seconds) noexcept;

```

### Step 2 — Resolve clips by name

*idle/walk/run/attack name heuristics.*

```cpp
// src/spark/animation/LocomotionClipSet.cpp (lines 25–51)
LocomotionClipSet ResolveLocomotionClipsFromSkeleton(
        const Skeleton& skeleton,
        const std::uint32_t walkFallback) noexcept {
    LocomotionClipSet out{};
    out.walk = walkFallback;

    static constexpr const char* kIdleNames[] = {"idle", "survey", "rest", "stand"};
    if (const std::int32_t idleIdx = FindFirstLocomotionClip(skeleton, kIdleNames, 4); idleIdx >= 0) {
        out.idle = static_cast<std::uint32_t>(idleIdx);
    }

    static constexpr const char* kWalkNames[] = {"walk"};
    if (const std::int32_t walkIdx = FindFirstLocomotionClip(skeleton, kWalkNames, 1); walkIdx >= 0) {
        out.walk = static_cast<std::uint32_t>(walkIdx);
    }

    static constexpr const char* kRunNames[] = {"run", "jog", "sprint"};
    if (const std::int32_t runIdx = FindFirstLocomotionClip(skeleton, kRunNames, 3); runIdx >= 0) {
        out.run = static_cast<std::uint32_t>(runIdx);
    }

    static constexpr const char* kAttackNames[] = {"attack", "punch", "kick", "shoot"};
    if (const std::int32_t attackIdx = FindFirstLocomotionClip(skeleton, kAttackNames, 4); attackIdx >= 0) {
        out.attack = static_cast<std::uint32_t>(attackIdx);
    }

    return out;
```

### Step 3 — Configure on spawn

*ConfigureLocomotionFromSkeleton + walk/run thresholds.*

```cpp
// src/spark/demo/CharacterCameraDemo.cpp (lines 976–981)
    if (charAnimFsm != nullptr) {
        charAnimFsm->ClearManualClip();
        charAnimFsm->ConfigureLocomotionFromSkeleton(*asset.skeleton, asset.walkClipIndex);
        charAnimFsm->SetWalkSpeedThreshold(0.35F);
        charAnimFsm->SetRunSpeedThreshold(2.5F);
    }
```

### Step 4 — Trace OnUpdate clip selection

*Attack overlay, manual clip, speed/input → want clip → ApplyClip_.*

```cpp
// src/spark/ecs/components/animation/Character3DAnimFsmComponent.cpp (lines 114–179)
void Character3DAnimFsmComponent::OnUpdate(
        const FrameTiming& timing,
        GameObject& owner,
        IEngineContext& /*context*/) {
    AnimatorComponent* anim = owner.GetComponent<AnimatorComponent>();
    if (anim == nullptr || !anim->GetSkeleton()) {
        return;
    }

    const bool attackOk = ClipValid_(*anim, attackClip);
    const std::uint32_t cur = anim->GetClipIndex();
    const bool holdingAttack = attackOk && cur == attackClip && !anim->IsClipFinished();

    std::uint32_t want = idleClip;
    AnimLoopMode wantLoop = AnimLoopMode::Loop;

    if (holdingAttack) {
        want = attackClip;
        wantLoop = AnimLoopMode::Once;
    } else if (pendingAttack && attackOk) {
        want = attackClip;
        wantLoop = AnimLoopMode::Once;
    } else if (manualClipActive && ClipValid_(*anim, manualClip)) {
        want = manualClip;
        wantLoop = manualLoop;
    } else if (!locomotionDrivingEnabled) {
        pendingAttack = false;
        return;
    } else {
        const bool runOk = ClipValid_(*anim, runClip);
        const bool walkOk = ClipValid_(*anim, walkClip);

        bool moving = false;
        bool wantSprint = false;
        if (hasLocomotionInput) {
            moving = inputMoving;
            wantSprint = inputSprint;
        } else {
            const float speed = MeasureLocomotionSpeed_(owner, timing);
            moving = speed >= walkThresh;
            wantSprint = runOk && speed >= runThresh;
        }
        hasLocomotionInput = false;

        if (!moving) {
            if (ClipValid_(*anim, idleClip)) {
                want = idleClip;
            } else if (walkOk) {
                want = walkClip;
            }
        } else if (wantSprint && runOk) {
            want = runClip;
        } else if (walkOk) {
            want = walkClip;
        } else if (ClipValid_(*anim, idleClip)) {
            want = idleClip;
        }
    }
    pendingAttack = false;

    if (!ClipValid_(*anim, want) && ClipValid_(*anim, idleClip)) {
        want = idleClip;
        wantLoop = AnimLoopMode::Loop;
    }

    ApplyClip_(*anim, want, wantLoop);
```

### Step 5 — Measure speed from physics or transform

*Rigidbody3D velocity XZ length or position delta.*

```cpp
// src/spark/ecs/components/animation/Character3DAnimFsmComponent.cpp (lines 93–111)
float Character3DAnimFsmComponent::MeasureLocomotionSpeed_(
        const GameObject& owner,
        const FrameTiming& timing) noexcept {
    if (const Rigidbody3DComponent* rb = owner.GetComponent<Rigidbody3DComponent>()) {
        const Vector3 v = rb->GetVelocity();
        return std::sqrt(v.x * v.x + v.z * v.z);
    }
    const Matrix4 world = owner.GetWorldMatrix();
    const Vector3 pos{world.m[12], world.m[13], world.m[14]};
    if (!hasLastWorldPos || timing.deltaTimeSeconds <= 1.0e-6F) {
        lastWorldPos = pos;
        hasLastWorldPos = true;
        return 0.0F;
    }
    const float invDt = 1.0F / timing.deltaTimeSeconds;
    const float dx = pos.x - lastWorldPos.x;
    const float dz = pos.z - lastWorldPos.z;
    lastWorldPos = pos;
    return std::sqrt(dx * dx + dz * dz) * invDt;
```

### Step 6 — Wire WASD input each frame

```cpp
const bool moving = input.IsKeyDown(KEY_W) || input.IsKeyDown(KEY_S) || ...;
const bool sprint = input.IsKeyDown(KEY_SHIFT);
fsm->SetLocomotionInput(moving, sprint);
```

Call before FSM OnUpdate from your character controller.

### Step 7 — Tune thresholds against clip root motion

Adjust SetWalkSpeedThreshold / SetRunSpeedThreshold so walk does not flicker when tapping movement keys.

### Step 8 — Milestone check — idle → walk → run

CharacterCameraDemo shell: stand still (idle), hold W (walk), hold Shift+W (run) with crossfades — log state transitions in your fork.

## 5. How This Connects to the Rest of Your Engine

Part V **character camera** reads the same entity transform. Part XIV physics can feed Rigidbody3D velocity
into MeasureLocomotionSpeed_. **Animator** and **GPU Skinning** consume FSM clip requests downstream.

## 6. Pitfalls and Debugging

- **FSM after Animator in component list** — one-frame wrong clip; respect update priority / add order.
- **No hysteresis** — borderline speed flickers walk/run; tune thresholds apart or use input path.
- **Locomotion in render code** — clip changes lag; keep in OnUpdate only.
- **Missing run clip** — runOk false falls back to walk; verify ResolveLocomotionClipsFromSkeleton results.

## 7. Exercises

1. Log clip index transitions while accelerating from idle to walk to run.
2. Tune walk threshold so a tap on W does not trigger walk — only sustained movement.
3. Call `RequestAttack()` during run — observe `AnimLoopMode::Once` overlay, then return to locomotion.
4. Disable `SetLocomotionInput` and rely on speed inference only — compare foot sliding on a physics-driven capsule.
5. List resolved clip indices from `ResolveLocomotionClipsFromSkeleton` for Fox vs CesiumMan assets.

## Summary

`Character3DAnimFsmComponent` selects idle, walk, run, and attack clips from speed thresholds or explicit locomotion input, applying crossfades through `AnimatorComponent`. Add the FSM **before** the animator on the same `GameObject` so update priorities run selection before time advancement. Part VI ends with a fully skinned, animated, locomoting character — continue to Part VII for sky and advanced materials.

---

Continue to [Part VII — Sky and Materials](../part-07-sky-materials/index.md).
