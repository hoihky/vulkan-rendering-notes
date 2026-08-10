# Animation Clips

> **Build-Along Milestone:** Skinned character loops walk animation with smooth clip crossfade on manual switch.

**You will learn:**

- Advance clip time with loop modes on AnimatorComponent.
- Crossfade between clips via SetClipIndexWithCrossfade.
- Produce joint palettes through ComputeJointPalette at submit.

## 1. The Problem We Are Solving

Skeletal **animation clips** are time-varying TRS keys on joints. Gameplay needs a component that advances time,
samples keys, blends states, and feeds the skeleton — without embedding glTF details in the renderer.

Build along: add AnimatorComponent to your skinned character and play the walk clip in a loop.

## 2. Core Concepts

### Clip playback on OnUpdate

`AnimatorComponent` advances `timeSeconds` each frame using `AnimLoopMode` (Loop, Once, Hold). Update priority `AnimatorPlayback` runs before render gather so palettes are fresh.


### Crossfade between clips

`SetClipIndexWithCrossfade` keeps prior clip/time in `CrossfadeState` and blends palettes via `Skeleton::ComputeBlendedPalette` — smooth locomotion transitions without popping.


### ComputeJointPalette pipeline

Evaluate sample time → optional crossfade blend → `SampleClipPose` → `BuildPaletteFromPose`. Submit calls this; component does not store palette internally.


### RetargetSkeleton

Swapping character models replaces SharedPtr<Skeleton> and resets playback — used when CharacterCameraDemo switches Fox vs CesiumMan avatars.


### Clip lookup by name

Skeleton exposes exact and case-insensitive name search — FSM and tooling resolve idle/walk/run without hard-coded indices when clip tables differ per asset.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### AnimatorComponent with speed + loop mode

**Concept:** Clip playback on OnUpdate

**Decision:** AnimatorComponent with speed + loop mode

**Why:** Separates **time advancement** from **clip selection** (FSM's job).

**In Spark:** `include/spark/ecs/components/animation/AnimatorComponent.hpp`

### SetClipIndexWithCrossfade

**Concept:** Crossfade between clips

**Decision:** SetClipIndexWithCrossfade

**Why:** Default 0.2s blend; FSM sets crossfade duration per character weight.

**In Spark:** `AnimatorComponent.cpp` — CrossfadeState + ComputeBlendedPalette

### Public palette API for submit

**Concept:** ComputeJointPalette pipeline

**Decision:** Public palette API for submit

**Why:** Scene code stays thin — one call fills SceneDrawItem.jointPalette array.

**In Spark:** `AnimatorComponent::ComputeJointPalette`

### Model swap without respawning entity

**Concept:** RetargetSkeleton

**Decision:** Model swap without respawning entity

**Why:** Preserves GameObject hierarchy; updates skeleton pointer and clip index.

**In Spark:** `CharacterCameraDemo::ApplyAvatarModel` — RetargetSkeleton call

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/ecs/components/animation/AnimatorComponent.hpp`
- `src/spark/ecs/components/animation/AnimatorComponent.cpp`
- `include/spark/animation/Skeleton.hpp`

### Step 1 — Read AnimatorComponent API

*OnUpdate, crossfade, ComputeJointPalette, clip queries.*

```cpp
// include/spark/ecs/components/animation/AnimatorComponent.hpp (lines 18–55)
class AnimatorComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::Animator;

    [[nodiscard]] ComponentKind Kind() const noexcept override { return TypeKind; }

    [[nodiscard]] int UpdatePriority() const noexcept override {
        return ComponentUpdatePriority::AnimatorPlayback;
    }

    AnimatorComponent(SharedPtr<Skeleton> inSkeleton, std::uint32_t clipIndex, float speed = 1.0F);

    void OnUpdate(const FrameTiming& timing, GameObject& owner, IEngineContext& context) override;

    [[nodiscard]] const SharedPtr<Skeleton>& GetSkeleton() const noexcept { return skeleton; }
    [[nodiscard]] std::uint32_t GetClipIndex() const noexcept { return clipIndex; }
    [[nodiscard]] float GetTimeSeconds() const noexcept { return timeSeconds; }
    [[nodiscard]] float GetSpeed() const noexcept { return speed; }
    [[nodiscard]] AnimLoopMode GetLoopMode() const noexcept { return loopMode; }
    [[nodiscard]] bool IsClipFinished() const noexcept { return clipFinished; }
    [[nodiscard]] bool IsCrossfading() const noexcept { return crossfade.active; }

    [[nodiscard]] std::uint32_t GetClipCount() const noexcept;
    [[nodiscard]] const Utf8String& GetClipName(std::uint32_t clipIndex) const;
    [[nodiscard]] std::int32_t FindClipIndexByName(const char* name) const;

    void SetClipIndex(std::uint32_t c);
    void SetClipIndexWithCrossfade(std::uint32_t c, float crossfadeDurationSec);
    void SetSpeed(float s);
    void SetTimeSeconds(float t);
    void SetLoopMode(AnimLoopMode mode) noexcept;
    /** Resets playback to t=0 on the current clip (clears crossfade). */
    void RestartCurrentClip() noexcept;
    /** Swaps skeleton/clip (e.g. character model switch); resets playback state. */
    void RetargetSkeleton(SharedPtr<Skeleton> newSkeleton, std::uint32_t clipIndex, float newSpeed = 1.0F);

    /** Fills skin joint palette using loop mode, optional crossfade, and evaluated sample times. */
    void ComputeJointPalette(Matrix4* outPalette, std::uint32_t paletteMax) const;
```

### Step 2 — Trace time advancement

*OnUpdate — AdvanceCrossfade + AdvancePrimaryTime.*

```cpp
// src/spark/ecs/components/animation/AnimatorComponent.cpp (lines 13–35)
void AnimatorComponent::OnUpdate(const FrameTiming& timing, GameObject& /*owner*/, IEngineContext& /*context*/) {
    if (!skeleton || skeleton->GetClipCount() == 0) {
        return;
    }
    AdvanceCrossfade_(timing.deltaTimeSeconds);
    AdvancePrimaryTime_(timing.deltaTimeSeconds);
}

void AnimatorComponent::AdvancePrimaryTime_(const float deltaSeconds) {
    const std::uint32_t c = clipIndex < skeleton->GetClipCount() ? clipIndex : 0;
    const float dur = skeleton->GetClipDuration(c);
    AdvanceAnimClipTime(deltaSeconds, speed, dur, loopMode, timeSeconds, clipFinished);
}

void AnimatorComponent::AdvanceCrossfade_(const float deltaSeconds) {
    if (!crossfade.active) {
        return;
    }
    crossfade.elapsed += deltaSeconds;
    if (crossfade.duration <= 1.0e-4F || crossfade.elapsed >= crossfade.duration) {
        crossfade.active = false;
    }
}
```

### Step 3 — Study crossfade request

*SetClipIndexWithCrossfade — captures fromClip/fromTime.*

```cpp
// src/spark/ecs/components/animation/AnimatorComponent.cpp (lines 66–81)
void AnimatorComponent::SetClipIndexWithCrossfade(const std::uint32_t c, const float crossfadeDurationSec) {
    if (!skeleton || c >= skeleton->GetClipCount()) {
        SetClipIndex(c);
        return;
    }
    if (c == clipIndex && !crossfade.active) {
        return;
    }
    crossfade.active = true;
    crossfade.fromClip = clipIndex < skeleton->GetClipCount() ? clipIndex : 0;
    crossfade.fromTime = timeSeconds;
    crossfade.duration = (crossfadeDurationSec > 1.0e-4F) ? crossfadeDurationSec : 0.2F;
    crossfade.elapsed = 0.0F;
    clipIndex = c;
    timeSeconds = 0.0F;
    clipFinished = false;
```

### Step 4 — Follow ComputeJointPalette

*Crossfade branch vs SampleClipPose + BuildPaletteFromPose.*

```cpp
// src/spark/ecs/components/animation/AnimatorComponent.cpp (lines 120–146)
void AnimatorComponent::ComputeJointPalette(Matrix4* outPalette, const std::uint32_t paletteMax) const {
    if (outPalette == nullptr || !skeleton || skeleton->GetJointCount() == 0) {
        return;
    }
    const std::uint32_t c = clipIndex < skeleton->GetClipCount() ? clipIndex : 0;
    const float dur = skeleton->GetClipDuration(c);
    const float sampleT = EvaluateAnimSampleTime(timeSeconds, dur, loopMode);

    if (crossfade.active && crossfade.duration > 1.0e-4F) {
        const float blend = crossfade.elapsed / crossfade.duration;
        const float fromDur = skeleton->GetClipDuration(crossfade.fromClip);
        const float fromSampleT = EvaluateAnimSampleTime(crossfade.fromTime, fromDur, AnimLoopMode::Hold);
        skeleton->ComputeBlendedPalette(
                crossfade.fromClip,
                fromSampleT,
                c,
                sampleT,
                blend,
                outPalette,
                paletteMax);
        return;
    }

    Array<Transform> pose;
    skeleton->SampleClipPose(c, sampleT, pose);
    skeleton->BuildPaletteFromPose(pose, outPalette, paletteMax);
}
```

### Step 5 — Add Animator to your character

```cpp
auto& anim = *go->AddComponent<AnimatorComponent>(asset.skeleton, asset.walkClipIndex, 1.f);
anim.SetLoopMode(AnimLoopMode::Loop);
```

Ensure OnUpdate runs before FillStandardLitSceneFromWorld.

### Step 6 — Scrub time in debug UI

Call `SetTimeSeconds` while paused — mesh should hold mid-stride pose when submit runs.

### Step 7 — Crossfade walk → run manually

```cpp
anim.SetClipIndexWithCrossfade(runClipIndex, 0.25f);
```

Observe 0.25s blend instead of limb pop.

### Step 8 — Milestone check — looping walk cycle

Character plays walk clip continuously; root and limbs deform; no T-pose flash after spawn.

## 5. How This Connects to the Rest of Your Engine

**Locomotion FSM** selects clip indices; Animator executes playback. **GPU Skinning** reads palettes uploaded
from jointPalette each draw. **Skeleton** owns clip data imported from glTF.

## 6. Pitfalls and Debugging

- **Euler interpolation** — Spark slerps quaternions on rotation channels; do not lerp Euler keys manually.
- **Instant clip switches** — use crossfade for locomotion changes.
- **Sampling before clip assigned** — invalid clip index → bind pose flash; validate clip count on spawn.
- **Tick after submit** — palette one frame stale; respect UpdatePriority ordering.

## 7. Exercises

1. Log timeSeconds each frame during walk loop; verify wrap at duration.
2. Blend two clips at 50% crossfade; screenshot mid-blend pose.
3. Call RetargetSkeleton to swap Fox/CesiumMan without deleting GameObject.

## Summary

AnimatorComponent advances clip time, crossfades, and builds joint palettes via Skeleton each frame before submit.

---

Next: [GPU Skinning](09-gpu-skinning.md).
