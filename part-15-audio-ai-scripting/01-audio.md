# Sound Engine

> **Build-Along Milestone:** Preloaded footstep WAV plays on key press with no frame hitch; BGM loops under SFX.

**You will learn:**

- Initialize Spark SoundEngine separately from VulkanRenderer.
- Load SoundClip assets at level load and play one-shots without blocking PresentFrame.
- Trace Startup → PumpFrame → Shutdown in Engine.cpp.

## 1. The Problem We Are Solving

Parts I–XIV built a Vulkan renderer and simulation stack — but silent games feel broken. **Audio** uses OS drivers and PCM buffers, not command buffers. Spark wraps decode, mixing, and output in **SoundEngine**, a facade parallel to `IFramePresenter`.

You will wire the same lifecycle you already know from the main loop: init after attach, pump after update, shutdown before exit. Audio is **not** a render pass — it neither reads nor writes `SceneRenderParams`.

Milestone: preload footstep WAV at level load; key press plays one-shot while BGM loops underneath without frame hitch.

## 2. Core Concepts

### SoundEngine as a subsystem facade

| Responsibility | Owner | Vulkan? |
|----------------|-------|---------|
| Decode WAV/OGG/MP3 → PCM | `TryLoadSoundClipFromFile` | No |
| Voice mixing | `SoundMixer` | No |
| Device output | `ISoundOutput` (Win32 / Apple / Null) | No |
| Per-frame pump | `SoundEngine::PumpFrame` | No |

The engine loop calls `audio.Startup()` after `OnAttach`, `audio.PumpFrame(deltaTime)` after `OnUpdate`, and `audio.Shutdown()` after `OnDetach` — mirroring how `IFramePresenter` wraps GPU work without exposing handles to gameplay.

### Assets vs voices

A **SoundClip** is immutable decoded PCM: stereo interleaved `float` samples at a tagged sample rate. Loading is expensive; playback is cheap.

A **voice** is ephemeral state in the mixer pool: read cursor, volume, optional world position, loop flag.

```text
TryLoadSoundClipFromBundledAsset("assets/sfx/footstep.wav")
        → SharedPtr<SoundClip>  (asset, shared)
PlayOneShot(clip, 0.8f)       → borrows Voice slot until clip ends
```

Spark pre-reserves **48 voices** at startup. When the pool is full, `PlayOneShot` steals the oldest active voice — better a dropped footstep than silence forever.

### PumpFrame timing model

```text
OnUpdate(gameplay) → PumpFrame(dt) → OnRender → PresentFrame
```

`PumpFrame` converts frame delta to a sample frame count (clamped 1–4096 at 48 kHz), zero-fills a scratch buffer, calls `mixer.MixAdd`, then `output->SubmitInterleavedFloat`. Never call `PumpFrame` from inside Vulkan record or present.

### Platform output abstraction

`CreatePlatformSoundOutput()` returns Win32, CoreAudio, or a stub. If factory returns null or `Start(48000, 2)` fails, Spark falls back to **NullSoundOutput** — CI headless runs and broken drivers keep running without `#ifdef` in every demo.

### Background music vs one-shots

| API | Voice | Evicted by PlayOneShot? |
|-----|-------|-------------------------|
| `SetBackgroundMusic(clip, vol, loop)` | Dedicated `music` voice | No |
| `PlayOneShot` / spatial variants | Pool voices | Yes (oldest steal) |

Platformer2DDemo loads looping BGM while gunshots use one-shots. `ClearBackgroundMusic` stops the bed without touching SFX voices.

### Load path and bundled assets

`TryLoadSoundClipFromBundledAsset` resolves `assets/...` relative to `SPARK_ASSETS_DIR` then `SPARK_BUILD_ASSETS_DIR`, matching texture and scene path rules from Part XIII. Load at **level transition**, not on first key press — synchronous decode on the main thread hitches `PresentFrame`.

If a file is missing, prefer `SoundClip::CreateToneBlip` for UI feedback rather than crashing.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Engine owns SoundEngine

**Concept:** SoundEngine as a subsystem facade

**Decision:** Engine owns SoundEngine; gameplay uses IEngineContext::TryGetSoundEngine

**Why:** Centralizing audio in `Engine::Impl` guarantees startup order: window exists, game attached, then `audio.Startup()`.

**In Spark:** `src/Engine.cpp` — `SoundEngine audio{}` member.

### CreatePlatformSoundOutput + NullSoundOutput fallback

**Concept:** Platform output abstraction

**Decision:** CreatePlatformSoundOutput + NullSoundOutput fallback

**Why:** Hardware init failure must not abort the engine — students run Spark in varied environments.

**In Spark:** `src/spark/audio/SoundEngine.cpp` lines 43–52.

### SharedPtr<SoundClip> + fixed voice pool

**Concept:** Assets vs voices

**Decision:** SharedPtr<SoundClip> + fixed voice pool

**Why:** Shared ownership lets BGM and SFX reference one decode. Pool cap bounds worst-case CPU mix cost per PumpFrame.

**In Spark:** `include/spark/audio/SoundMixer.hpp`.

### Pull model tied to engine deltaTime

**Concept:** PumpFrame timing model

**Decision:** Pull model tied to engine deltaTime

**Why:** Keeps audio phase-locked to simulation without a second thread mutex on ECS.

**In Spark:** `SoundEngine::PumpFrame` in SoundEngine.cpp.

### TryLoadSoundClipFromBundledAsset

**Concept:** Load path and bundled assets

**Decision:** TryLoadSoundClipFromBundledAsset

**Why:** Same mental model as `assets/textures/...` — relative paths survive packaging.

**In Spark:** `src/spark/audio/SoundFileLoader.cpp`.

### Dedicated music voice outside steal pool

**Concept:** Background music vs one-shots

**Decision:** Dedicated music voice outside steal pool

**Why:** BGM must not be evicted when SFX pool fills.

**In Spark:** `SoundEngine::SetBackgroundMusic` in SoundEngine.cpp.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/audio/SoundEngine.cpp`
- `include/spark/audio/SoundEngine.hpp`
- `src/spark/audio/SoundFileLoader.cpp`
- `src/Engine.cpp`

### Step 1 — Read SoundEngine::Startup and fallback

Note **two** fallback layers: null factory, then failed `Start`.

*Startup — platform output, null fallback, mixer configure*

```cpp
// src/spark/audio/SoundEngine.cpp (lines 39–58)
bool SoundEngine::Startup() {
    if (running) {
        return true;
    }
    output = CreatePlatformSoundOutput();
    if (!output) {
        output.Reset(MakeUnique<NullSoundOutput>().Release());
    }
    if (!output->Start(kSampleRate, kChannels)) {
        output.Reset();
        output.Reset(MakeUnique<NullSoundOutput>().Release());
        if (!output->Start(kSampleRate, kChannels)) {
            return false;
        }
    }
    mixer.Configure(kSampleRate, 48);
    scratch.Clear();
    scratch.Reserve(8192);
    running = true;
    return true;
}
```

### Step 2 — Trace Engine loop integration

Audio pumps **after** `OnUpdate` so gameplay can queue cues first.

*PumpFrame after OnUpdate; Shutdown after OnDetach*

```cpp
// src/Engine.cpp (lines 188–205)
            game->OnUpdate(timing, *engineContext);
            audio.PumpFrame(timing.deltaTimeSeconds);
            game->OnRender(renderFrame, *engineContext);
            renderer->PresentFrame();
        }
        game->OnDetach();
        audio.Shutdown();
```

### Step 3 — Study PumpFrame sample generation

dt → frame count → scratch buffer → MixAdd → Submit.

*PumpFrame — dt to samples, mix, output*

```cpp
// src/spark/audio/SoundEngine.cpp (lines 74–96)
void SoundEngine::PumpFrame(const float deltaTimeSeconds) noexcept {
    if (!running || !output) {
        return;
    }
    const float dt = deltaTimeSeconds > 0.0F ? deltaTimeSeconds : (1.0F / 60.0F);
    std::size_t frames = static_cast<std::size_t>(static_cast<double>(kSampleRate) * static_cast<double>(dt) + 0.5);
    if (frames < 1U) { frames = 1U; }
    if (frames > 4096U) { frames = 4096U; }
    scratch.Clear();
    for (std::size_t i = 0; i < frames * 2U; ++i) {
        scratch.PushBack(0.0F);
    }
    mixer.MixAdd(scratch.GetData(), frames);
    output->SubmitInterleavedFloat(scratch.GetData(), frames);
}
```

### Step 4 — Load a WAV at level load

```cpp
SharedPtr<SoundClip> footstep = TryLoadSoundClipFromBundledAsset("assets/sfx/footstep.wav");
if (footstep && SoundEngine* audio = context.TryGetSoundEngine()) {
    audio->GetMixer().PlayOneShot(footstep, 0.85f);
}
```

Call `PlayOneShot` on key edge, not every frame.

### Step 5 — Wire looping background music

```cpp
if (SharedPtr<SoundClip> bgm = LoadPlatformerBgm()) {
    audio.SetBackgroundMusic(bgm, 0.32f, true);
}
```

### Step 6 — Inspect SoundFileLoader dispatch

*Extension dispatch — wav, ogg, mp3*

```cpp
// src/spark/audio/SoundFileLoader.cpp (lines 105–123)
SharedPtr<SoundClip> TryLoadSoundClipFromFile(const char* utf8Path) {
    const char* ext = FileExtensionDot(utf8Path);
    if (ExtEqualsIgnoreCase(ext, ".wav") || ExtEqualsIgnoreCase(ext, ".wave")) {
        return TryLoadSoundClipFromWavFile(utf8Path);
    }
    if (ExtEqualsIgnoreCase(ext, ".ogg")) {
        return TryLoadSoundClipFromOggFile(utf8Path);
    }
    if (ExtEqualsIgnoreCase(ext, ".mp3")) {
        return TryLoadSoundClipFromMp3File(utf8Path);
    }
    return SharedPtr<SoundClip>();
}
```

### Step 7 — Add debug master volume

ImGui slider on `PlayOneShot` volume confirms you are hitting `GetMixer()` — not Vulkan.

### Step 8 — Milestone — footstep + BGM

Preloaded footstep on key press; BGM loops under 10 rapid one-shots without eviction.

## 5. How This Connects to the Rest of Your Engine

Chapter 2 adds **spatial panning** via `AudioListenerComponent` and `SoundCueComponent`. Part **XIII** editor can attach cues to entities. Part **XIV** physics runs before audio cue flush — footstep triggers on landing event after simulate, not inside collision solver.

Part **XVI** async loading may decode SFX on worker threads while main thread uploads GPU assets — same "decode off hot path" instinct.

## 6. Pitfalls and Debugging

- **Decode on first Play** — multi-ms hitch; preload in `Load()` or async worker.
- **PlayOneShot every frame** — exhausts voice pool; trigger on input edge.
- **Assuming audio failed = crash** — check `IsRunning()`; NullSoundOutput is valid.
- **Calling PumpFrame before OnUpdate** — cues queued this frame would lag one frame.
- **Wrong sample rate clip** — resample at load or accept pitch shift; mixer assumes 48 kHz output.

## 7. Exercises

1. Log voice pool usage: count active voices after 10 rapid one-shots.
2. Replace missing footstep WAV with `SoundClip::CreateToneBlip(880, 0.08, 0.5)`.
3. Compare PumpFrame ms with BGM on vs off — confirm mix cost scales with active voices.
4. Trace `Engine.cpp` shutdown order — document when `audio.Shutdown` runs relative to window destroy.
5. List three reasons NullSoundOutput is preferable to aborting engine startup on audio failure.

## Summary

SoundEngine owns mixer + platform output, decodes clips via SoundFileLoader, and pumps interleaved stereo each frame after Update. Spark falls back to NullSoundOutput when hardware init fails.

---

Next: [Spatial Audio](02-spatial-audio.md).
