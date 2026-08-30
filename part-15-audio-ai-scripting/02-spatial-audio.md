# Spatial Audio

> **Build-Along Milestone:** Looped world cue audibly fades with distance; pickup one-shot plays centered.

**You will learn:**

- Resolve listener pose via AudioListenerComponent or camera fallback.
- Queue spatial and mono cues through SoundCueComponent flush.
- Tune distance attenuation and stereo pan with spatialBlend.

## 1. The Problem We Are Solving

A footstep at full volume when the enemy is across the map breaks immersion. **Spatial audio** positions sources in world space, pans stereo by listener orientation, and attenuates by distance.

Spark keeps Vulkan out of audio entirely. You will wire `ProcessAudioListeners` before `SoundCueComponent::FlushTo`, attach listeners to cameras, and place looped cues on world entities.

Milestone: torch loop fades with distance as you walk away; pickup cue plays centered (mono).

## 1b. Concept Deep Dive — Listener Resolution

### ProcessAudioListeners before FlushTo

Spatial pan and attenuation need a frame-local listener pose. Spark resolves it once per tick before cue flush:

```cpp
// src/spark/audio/SoundSubsystem.cpp
void ProcessSoundCues(GameWorld& world, IEngineContext& context) {
    ProcessAudioListeners(world);
    ProcessAmbientZones(world);
    SoundEngine* audio = context.TryGetSoundEngine();
    world.ForEachActiveGameObject([audio](GameObject* o) {
        SoundCueComponent* sc = o->GetComponent<SoundCueComponent>();
        if (sc != nullptr) {
            sc->FlushTo(audio);  // spatialBlend + distance → pan/volume
        }
    });
}
```

**Order matters:** listener pose must be current before any cue computes relative direction.

### Priority-based AudioListenerComponent

```cpp
// src/spark/audio/AudioSpatial.cpp
void ProcessAudioListeners(const GameWorld& world) noexcept {
    world.ForEachActiveGameObject([&](GameObject* o) {
        const AudioListenerComponent* listener = o->GetComponent<AudioListenerComponent>();
        if (listener == nullptr || !listener->IsEnabled()) return;
        if (!found || listener->GetPriority() > bestPriority) {
            gFrameListener.position = {worldM.m[12], worldM.m[13], worldM.m[14]};
            gFrameListener.right/up/forward from world matrix columns;
        }
    });
    // Fallback: main 3D camera pose when no explicit listener
}
```

Attach listener to fly camera; split-screen can add a higher-priority listener on the active player camera.

### spatialBlend mixes 2D and 3D

`spatialBlend = 0` → centered mono (UI pickup chime). `spatialBlend = 1` → full world attenuation and stereo pan (torch loop). Vulkan stays out of this path entirely.

## 2. Core Concepts

### Listener pose resolution

Spatial audio needs a **listener** — position and orthonormal basis (right, up, forward). Spark does **not** expose `SoundEngine::SetListener`. Instead:

1. `ProcessAudioListeners(world)` scans `AudioListenerComponent` — highest **priority** enabled listener wins.
2. Writes `AudioListenerPose` into frame storage.
3. If no listener component exists, falls back to **main 3D camera** pose via `TryResolveMainCamera`.

This runs inside `ProcessSoundCues` **before** cue flush — spatial one-shots always see current listener.

### Distance attenuation and stereo pan

`SoundMixer::AccumulateVoiceForFrame` applies spatial gain when `spatialBlend > 0`:

```text
falloff = 1 - clamp((dist - minDist) / (maxDist - minDist), 0, 1)
pan     = dot(delta, listener.right) / dist
spatialGain = falloff * spatialBlend + (1 - spatialBlend)
```

| spatialBlend | Behavior |
|--------------|----------|
| 0 | Mono centered (UI bleeps) |
| 1 | Full pan + distance falloff |
| 0.5 | Hybrid — common for "mostly 2D" games |

### SoundCueComponent queue pattern

Components **queue** requests; subsystem **flushes** once per frame:

```text
Gameplay: cue->Queue(clip, vol)  or  QueueAtWorld(...)
     ↓ (same frame, later in Update)
ProcessSoundCues → FlushTo(SoundEngine)
     ↓
SoundMixer::PlayOneShotSpatial
```

Queueing decouples "I want a sound" from "mixer is ready." Multiple cues on one entity append to `pending` array; flush clears after play.

### ProcessSoundCues ordering

```text
ProcessAudioListeners(world)
ProcessAmbientZones(world)
ForEach SoundCueComponent → FlushTo(audio)
```

Games call this from `OnUpdate` via `IEngineContext`. **Never** flush from `OnRender` — render should stay free of audio side effects.

### Ambient zones and listener priority

`AmbientZoneComponent` scales mix based on listener proximity to volumes. Listener priority resolves split-screen ambiguity: attach `AudioListenerComponent` to player1 camera with priority 10, spectator camera priority 0.

For single-player ThreeDDemo, explicit listener on camera is optional — camera fallback fills pose when no component present.

### 2D vs 3D spatial authoring

Side-scrollers often use **XY** gameplay with `spatialBlend < 1`. 3D demos use full `QueueAtWorld` with world position from `TransformComponent`. Sound engine stays **dimension-agnostic** — it only sees `Vector3` positions.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### AudioListenerComponent + ProcessAudioListeners

**Concept:** Listener pose resolution

**Decision:** AudioListenerComponent + ProcessAudioListeners

**Why:** Multiple cameras exist in editor and split-screen prototypes — priority beats hardcoding listener to main camera only.

**In Spark:** `src/spark/audio/AudioSpatial.cpp`.

### Queue on component, FlushTo on subsystem

**Concept:** SoundCueComponent queue pattern

**Decision:** Queue on component, FlushTo on subsystem

**Why:** Gameplay systems should not hunt for SoundEngine — they mutate components; one subsystem knows flush order.

**In Spark:** `include/spark/ecs/components/audio/SoundCueComponent.hpp`.

### Per-request min/max on SoundPlayRequest

**Concept:** Distance attenuation and stereo pan

**Decision:** Per-request min/max on SoundPlayRequest

**Why:** Designer-tunable curves without recompiling mixer code.

**In Spark:** `SoundCueComponent::QueueAtWorld` parameters; mixer lines 113–133.

### Listeners before cues

**Concept:** ProcessSoundCues ordering

**Decision:** Listeners before cues

**Why:** Spatial math requires current pose — ordering bug sounds like listener one frame behind camera.

**In Spark:** `src/spark/audio/SoundSubsystem.cpp`.

### spatialBlend field on SoundPlayRequest

**Concept:** 2D vs 3D spatial authoring

**Decision:** spatialBlend field on SoundPlayRequest

**Why:** One code path serves platformer UI blips and 3D torch loops.

**In Spark:** `SoundCueComponent.cpp` FlushTo branch on spatialBlend.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/audio/AudioSpatial.cpp`
- `include/spark/ecs/components/audio/SoundCueComponent.hpp`
- `src/spark/audio/SoundSubsystem.cpp`
- `src/spark/audio/SoundMixer.cpp`

### Step 1 — Read listener resolution

Trace component priority scan and camera fallback.

*ProcessAudioListeners — component priority + camera fallback*

```cpp
// src/spark/audio/AudioSpatial.cpp (lines 37–79)
void ProcessAudioListeners(const GameWorld& world) noexcept {
    gFrameListener.valid = false;
    bool found = false;
    std::int32_t bestPriority = 0;
    world.ForEachActiveGameObject([&](GameObject* o) {
        const AudioListenerComponent* listener = o->GetComponent<AudioListenerComponent>();
        if (listener == nullptr || !listener->IsEnabled()) {
            return;
        }
        if (!found || listener->GetPriority() > bestPriority) {
            found = true;
            bestPriority = listener->GetPriority();
            const Matrix4& worldM = o->GetWorldMatrix();
            gFrameListener.position = {worldM.m[12], worldM.m[13], worldM.m[14]};
            gFrameListener.right = NormalizeOr({worldM.m[0], worldM.m[1], worldM.m[2]}, Vector3::UnitX);
            gFrameListener.up = NormalizeOr({worldM.m[4], worldM.m[5], worldM.m[6]}, Vector3::UnitY);
            gFrameListener.forward = NormalizeOr(
                    {-worldM.m[8], -worldM.m[9], -worldM.m[10]},
                    Vector3{0.0F, 0.0F, -1.0F});
            gFrameListener.valid = true;
        }
    });
    // Fallback: main 3D camera pose when no explicit listener is present.
}
```

### Step 2 — Attach AudioListenerComponent

```cpp
cameraGo->AddComponent<AudioListenerComponent>()->SetPriority(10);
```

### Step 3 — Study SoundCueComponent flush

*FlushTo — spatial vs mono paths*

```cpp
// src/spark/ecs/components/audio/SoundCueComponent.cpp (lines 38–58)
void SoundCueComponent::FlushTo(SoundEngine* engine) noexcept {
    if (engine == nullptr || !engine->IsRunning()) {
        pending.Clear();
        return;
    }
    SoundMixer& mx = engine->GetMixer();
    for (std::size_t i = 0; i < pending.GetSize(); ++i) {
        const SoundPlayRequest& req = pending[i];
        if (req.spatialBlend > 1.0e-4F) {
            mx.PlayOneShotSpatial(req.clip, req.volume, req.worldPosition,
                    req.spatialBlend, req.minDistance, req.maxDistance);
        } else {
            mx.PlayOneShot(req.clip, req.volume);
        }
    }
    pending.Clear();
}
```

### Step 4 — Wire ProcessSoundCues in OnUpdate

*ProcessSoundCues — listeners, ambient, cue flush*

```cpp
// src/spark/audio/SoundSubsystem.cpp (lines 13–28)
void ProcessSoundCues(GameWorld& world, IEngineContext& context) {
    ProcessAudioListeners(world);
    ProcessAmbientZones(world);
    SoundEngine* audio = context.TryGetSoundEngine();
    if (audio == nullptr || !audio->IsRunning()) {
        return;
    }
    world.ForEachActiveGameObject([audio](GameObject* o) {
        if (SoundCueComponent* sc = o->GetComponent<SoundCueComponent>()) {
            sc->FlushTo(audio);
        }
    });
}
```

### Step 5 — Place looped torch cue

```cpp
cue->QueueAtWorld(torchLoop, 0.6f, transform.GetWorldPosition(), 1.0f, 1.0f, 24.0f);
```

Walk away — volume should fall to near silence at maxDistance.

### Step 6 — Inspect spatial mix math

*Spatial gain — falloff, pan, accumulate*

```cpp
// src/spark/audio/SoundMixer.cpp (lines 109–136)
    if (v.spatialBlend > 1.0e-4F) {
        const AudioListenerPose& listener = GetFrameAudioListenerPoseConst();
        if (listener.valid) {
            const Vector3 delta = v.worldPosition - listener.position;
            const float dist = std::sqrt(delta.LengthSquared());
            const float falloff = 1.0F - std::clamp(
                    (dist - v.minDistance) / (v.maxDistance - v.minDistance), 0.0F, 1.0F);
            const float pan = std::clamp(
                    Vector3::Dot(delta, listener.right) / std::max(dist, 1.0e-4F), -1.0F, 1.0F);
            // gainL/gainR from pan; multiply by spatialGain
        }
    }
```

### Step 7 — One-shot pickup cue

On coin collect (edge triggered): `cue->Queue(pickupClip, 1.0f);` — spatialBlend 0.

### Step 8 — Audit include boundaries

```bash
rg 'vulkan.h' include/spark/audio include/spark/ecs/components/audio
```

Expect **zero** hits.

## 5. How This Connects to the Rest of Your Engine

Part **V** camera resolution feeds listener fallback when no `AudioListenerComponent`. Part **XIII** editor can expose cue fields in inspector. Chapter 4 exposes `spark_world_process_sound_cues` to C# — same native order.

Part **XIV** physics events should call `Queue` on edge — not from inside `PhysicsWorld2D::Simulate`.

## 6. Pitfalls and Debugging

- **Skipping ProcessAudioListeners** — pan uses stale or invalid pose; sounds center incorrectly.
- **QueueAtWorld every frame** — retriggers voice steal; use looping clip or long one-shot.
- **Forward vector zero on custom camera** — fallback NormalizeOr prevents NaN; fix transform.
- **maxDistance <= minDistance** — clamp behavior undefined perceptually; keep max > min.
- **Flushing cues before listener update** — one-frame lag behind camera; follow subsystem order.

## 7. Exercises

1. Build falloff spreadsheet for torch cue at 1 m steps; attach CSV.
2. Record 10 s regression WAV during circle walk around spatial source.
3. Document listener priority tie-break behavior for split-screen stub.
4. Compare mono `Queue` vs spatial `QueueAtWorld` on same entity — describe pan difference.
5. Log `(dist, falloff, pan)` once per second while circling torch — verify maxDistance tuning.

## Summary

Spatial audio resolves AudioListenerPose each frame, then flushes SoundCueComponent queues into SoundMixer spatial one-shots. Spark separates listener components, cue queueing, and PCM mix.

---

Next: [AI Steering](03-ai-steering.md).
