# Async Assets

> **Build-Along Milestone:** Level load shows placeholders then final textures without multi-frame main-thread stall.

**You will learn:**

- Decode assets on worker threads; create Vulkan resources on the main thread.
- Integrate `SceneManager` async `Pump` with `GameWorldAssetLoader`.
- Use placeholders and generation ids to avoid stale uploads.

## 1. The Problem We Are Solving

Synchronous glTF/PNG load **hitches** `PresentFrame` — users feel stutter even if average FPS looks fine. **Async assets** decode off-thread, finalize into existing `Texture2D` / `Mesh` CPU types on the main thread, then upload through `DeferredUploadBatch`.

Spark's `GameWorldAssetLoader` worker thread and `SceneManager::Pump` (Part XIII) combine into the production loading pipeline you will extend in this chapter.

Milestone: async-load a large scene with an ImGui progress bar — max frame ms stays under 2× normal gameplay frame budget.

## 1b. Concept Deep Dive — Two-Thread Asset Pump

### Worker decodes; main thread registers + uploads

```cpp
// include/spark/scene/assets/GameWorldAssetLoader.hpp
class GameWorldAssetLoader {
    void RequestGltf(const char* path);
    void OnGltfReady(const char* path, AssetLoadCallback callback);
    /** Call Pump from the main thread each frame; caches updated only in Pump. */
    void Pump(GameWorld& world);
};
```

Worker thread parses glTF/PNG off the hot path; `Pump` moves decoded CPU data into `GameWorldAssetCache` and fires ready callbacks on the main thread — safe for `AddComponent` and Vulkan resource creation.

### SceneManager::Pump chains scene finalize

Part XIII async load and Part XVI production loading share one pump hook:

```cpp
void SceneManager::Pump() {
    world.GetAssetLoader().Pump(world);
    for (each LoadedSceneInstance not ready) {
        RetryPendingComponents(instance);
        FinalizeInstanceReadyState(instance);
    }
}
```

Deferred `gltf_scene` / `skinned_mesh` restores retry each frame until assets register — no multi-frame main-thread stall parsing hundreds of files synchronously.

### Placeholders and generation ids

Show low-res placeholder textures while high-res loads; bump generation id on finalize so stale worker completions cannot overwrite a newer request. Max frame ms should stay under 2× normal gameplay budget during load — if not, reduce per-frame Pump work or increase worker parallelism.

## 2. Core Concepts

### Two-stage asset pipeline

**Async assets** split work across threads:

```text
Worker:  disk read → decode (PNG/glTF/WAV) → CPU-ready blob
Main:    Pump → register in GameWorld cache → queue GPU upload → mark ready
```

Vulkan object creation stays on the **main thread** in Spark's default model — workers never call `vkCreateImage`. Part XVI teaches decode off-thread; GPU finalize reuses Part IV fingerprint gating.

### GameWorldAssetLoader

Spark's `GameWorldAssetLoader` owns a single worker thread:

| API | Thread | Action |
|-----|--------|--------|
| `RequestGltf` / `RequestTexture` | Main | Enqueue job |
| `WorkerLoop` | Worker | Decode file |
| `Pump(world)` | Main | Commit completed jobs into `GameWorld` caches |

`SceneManager::Pump` calls `assetLoader.Pump(world)` first, then advances scene instance readiness.

### SceneManager async vs sync

`LoadSceneFromFile` parses the document synchronously then pumps until ready — simple but hitches on large scenes.

`BeginLoadSceneAsync` returns immediately; caller polls `SceneManager::Pump()` each frame. Same instance id tracking applies — **async = spread Pump work**, not magic background Vulkan.

```text
Pending → LoadingDocument → ApplyingEntities → UploadingGpu → Ready
                                              ↘ Failed
```

### Placeholder materials

While a GPU texture layer is not ready, draw with **checkerboard** or a pink placeholder layer index. Spark's scene texture uploader skips bind until **fingerprint** matches uploaded content (Part IV). Descriptor writes happen when `DeferredUploadBatch` completes the copy — not when PNG decode finishes on the worker.

### Generation id cancel

Open-world streaming (Part XII) assigns **generation** to load requests. When the player turns a corner, bump generation — completed worker results with a stale id are discarded before GPU upload.

Without cancel, late arrivals overwrite active textures — "pop-in from wrong district" bugs.

### Load screen UX contract

| Element | Shipping expectation |
|---------|---------------------|
| Progress | `Pump()` advances ready/total — progress bar 0–100% |
| Cancel | Bump generation id — back button aborts without crash |
| Placeholder | Checkerboard visible under 3 s for normal assets |
| Error | `instance->failed` flag — message + return to menu |
| Main thread budget | Cap `Pump` work to ~2 ms per frame during load screen |

Measure **max frame ms** during load, not average — averages hide single 200 ms spikes that trigger "game frozen" reports.

### Worker/main handoff contract

Worker threads **never** touch `GameWorld`, `VkDevice`, or `PumpFrame`. Handoff is always:

```text
Worker                          Main thread
  Read file                       SceneManager::Pump()
  Decode PNG/glTF                 assetLoader.Pump(world)
  Push to completed queue         Mark needsGpuUpload
                                  deferredUploadBatch.Prepare (DrawFrame)
                                  GPU copy in Record (DrawFrame)
```

No mutex on `GameWorld` — single consumer on main.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Worker decode, main upload

**Concept:** Two-stage asset pipeline

**Decision:** `GameWorldAssetLoader` worker produces CPU blobs; main commits caches.

**Why:** Keeps command buffer recording and Vulkan allocation on one thread — fewer locks.

**In Spark:** `GameWorldAssetLoader.hpp` class comment + `Pump`.

### BeginLoadSceneAsync + Pump

**Concept:** SceneManager async vs sync

**Decision:** Async entry returns id; caller pumps each frame.

**Why:** Same document path as sync — reduces duplicate loader code.

**In Spark:** `SceneManager.cpp` `BeginLoadSceneAsync` and `Pump`.

### Fingerprint gating in uploader

**Concept:** Placeholder materials

**Decision:** Defer descriptor bind until upload completes.

**Why:** Avoids binding half-uploaded images — validation error or garbage pixels.

**In Spark:** Part IV `SceneTextureUploader` pattern.

### Generation id for streaming

**Concept:** Generation id cancel

**Decision:** Document pattern for terrain tile loads.

**Why:** Spark terrain streaming (Part XII) reuses the same stale-result discard model.

**In Spark:** Extension point on tile load requests.

### Max frame ms metric

**Concept:** Load screen UX contract

**Decision:** Acceptance criteria use max ms, not average.

**Why:** One hitch frame fails shipping even if mean FPS looks fine.

**In Spark:** Compare sync `LoadSceneFromFile` pump loop vs async spread.

### assetLoader.Pump before scene finalize

**Concept:** GameWorldAssetLoader

**Decision:** `SceneManager::Pump` calls `assetLoader.Pump` first.

**Why:** Scene components referencing textures need caches populated before `FinalizeInstanceReadyState`.

**In Spark:** `SceneManager::Pump` line 206.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/GameWorldAssetLoader.cpp`
- `include/spark/scene/GameWorldAssetLoader.hpp`
- `src/spark/scene/SceneManager.cpp`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Read GameWorldAssetLoader header

Note the contract: call `Pump` from main thread; caches updated only in `Pump`.

*Background I/O + CPU decode*

```cpp
// include/spark/scene/GameWorldAssetLoader.hpp (lines 34–61)
/**
 * Background I/O + CPU decode for GameWorld asset caches.
 * Call Pump from the main thread each frame; GameWorld caches are updated only in Pump.
 */
class GameWorldAssetLoader {
public:
    void RequestGltf(const char* path);
    void RequestTexture(const char* path);
    void Pump(GameWorld& world);
    // ...
};
```

### Step 2 — Trace WorkerLoop decode path

Worker reads jobs, decodes off-thread, pushes completed arrays under mutex:

*Worker decodes glTF on background thread*

```cpp
// src/spark/scene/GameWorldAssetLoader.cpp (lines 270–283)
        switch (job.key.kind) {
            case AssetLoadJobKind::Gltf: {
                CompletedGltf result{};
                result.path = job.key.path;
                auto mesh = MakeShared<Mesh>(job.key.path);
                SharedPtr<Texture2D> tex;
                result.ok = Mesh::TryLoadFromGltf(job.key.path.CStr(), *mesh, &tex);
                if (result.ok) {
                    result.asset.mesh = mesh;
                    result.asset.baseColorTexture = tex;
                }
                std::lock_guard<std::mutex> lock(mutex);
                completedGltf.PushBack(MoveTemp(result));
                break;
            }
```

### Step 3 — Read Pump commit on main thread

`Pump` moves completed batches out of the mutex, then registers into `GameWorld`:

*Main thread commits worker results*

```cpp
// src/spark/scene/GameWorldAssetLoader.cpp (lines 186–210)
void GameWorldAssetLoader::Pump(GameWorld& world) {
    Array<CompletedGltf> gltfBatch;
    // ... move completed* arrays under lock ...

    for (std::size_t i = 0; i < gltfBatch.GetSize(); ++i) {
        const CompletedGltf& c = gltfBatch[i];
        const JobKey key{c.path, AssetLoadJobKind::Gltf};
        if (c.ok && c.asset.mesh) {
            world.RegisterGltf(c.asset, c.path.CStr());
            std::lock_guard<std::mutex> lock(mutex);
            states.Add(key, AssetLoadState::Ready);
        } else {
            std::lock_guard<std::mutex> lock(mutex);
            states.Add(key, AssetLoadState::Failed);
        }
    }
```

### Step 4 — Contrast sync vs async scene load

Sync path blocks in a pump loop — acceptable for editor, not for shipped level transitions:

*LoadSceneFromFile synchronous pump until ready*

```cpp
// src/spark/scene/SceneManager.cpp (lines 169–172)
    constexpr int kMaxPumpIterations = 100000;
    for (int i = 0; i < kMaxPumpIterations && !instance->ready && !instance->failed; ++i) {
        Pump();
    }
```

Async path returns immediately:

```cpp
// src/spark/scene/SceneManager.cpp (lines 205–214)
void SceneManager::Pump() {
    assetLoader.Pump(world);
    for (std::size_t i = 0; i < instances.GetSize(); ++i) {
        LoadedSceneInstance& instance = instances[i];
        if (instance.ready || instance.failed) {
            continue;
        }
        RetryPendingComponents(instance);
        FinalizeInstanceReadyState(instance);
    }
}
```

### Step 5 — Hook Pump in game loop during load screen

```cpp
sceneManager.Pump();  // each frame while loading
```

Spinner UI runs while instances transition to `ready`.

### Step 6 — Guard superseded loads

When starting a new async load, increment `loadGeneration` — worker completion compares before touching `GameWorld`.

### Step 7 — Log slow loads

If placeholder visible over 5 s, LOG error with asset path — silent failure kills trust in streaming.

### Step 8 — Compare hitch histogram

Load a large glTF sync vs async — chart max ms per frame; target max under 2× normal frame budget.

## 5. How This Connects to the Rest of Your Engine

Part **XIII** scene async API. Part **XII** terrain streaming adds tile IO concurrency. Part **XV** audio decode can share the worker pool pattern. Part **XVI** Chapter 4 profiling validates hitch reduction numerically.

Part **IV** fingerprint gating is a prerequisite — async without fingerprint causes binding garbage pixels.

## 6. Pitfalls and Debugging

- **vkCreateImage on worker** — default forbidden unless transfer queue ownership is documented.
- **Applying superseded load** — guard with generation id.
- **Placeholder forever** — failed load must LOG loudly.
- **Locking GameWorld on worker** — pass immutable blobs across queue, mutate main only.
- **Progress bar without Pump call** — stuck at 0% while work completes incorrectly.
- **Sync LoadSceneFromFile on level transition** — acceptable editor only.

## 7. Exercises

1. Implement a load screen with a 2 ms Pump cap; log max frame ms during a large glTF load.
2. Cancel load mid-flight; confirm generation id discards stale texture.
3. Document worker/main handoff struct fields in `docs/async-assets.md`.
4. Compare `GetPendingJobCount()` during load vs progress bar percentage — they should correlate.
5. Trace one texture from `RequestTexture` through `Pump`, `Prepare`, and `Record` — list each thread.

## Summary

Async assets decode off-thread via `GameWorldAssetLoader` and upload on main through `DeferredUploadBatch` and fingerprint gating. `SceneManager::Pump` spreads scene finalize work across frames instead of one blocking hitch.

---

Next: [Profiling](04-profiling.md).
