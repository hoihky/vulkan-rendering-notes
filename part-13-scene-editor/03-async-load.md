# Async Loading

> **Build-Along Milestone:** Large scene load shows no multi-frame main-thread stall; assets appear over subsequent frames.

**You will learn:**

- Parse scene files into SceneDocument without blocking PresentFrame.
- Apply SceneDocument on the main thread via SceneManager.
- Track in-flight loads with SceneInstanceId and Pump until ready.
- Handle deferred component restore for async assets.

## 1. The Problem We Are Solving

Chapter 1–2 save and load scenes synchronously — fine for ten entities, unacceptable for production levels with hundreds of meshes, skinned characters, and texture arrays. A single `LoadSceneFromFile` that blocks until every `TryRestore` succeeds will hitch the same thread responsible for input, physics (Part XIV), UI panel ticks, and `PresentFrame`.

Spark's **async loading** model separates three concerns: (1) parsing bytes into `SceneDocument`, (2) applying entity graph and synchronous components on the main thread, and (3) completing deferred restores as `GameWorldAssetLoader` finishes IO. `SceneManager` owns this lifecycle — you hold a `SceneInstanceId`, call `Pump()` each engine tick, and poll `IsSceneReady` before rebinding editor selection.

Milestone: load a scene with glTF references async; status line shows "Loading scene (async assets)…" then "Loaded N entities" without multi-frame stall.

## 1b. Concept Deep Dive — SceneManager Pump Lifecycle

### Three-phase load: parse, apply, finalize

Large scenes split work so `PresentFrame` never blocks on disk:

```text
BeginLoadScene(path)
  → parse file into SceneDocument (can be async)
  → ApplyDocumentInstance on main thread (entities + sync components)
  → defer mesh/gltf_scene restores until assets ready

Each frame: SceneManager::Pump()
  → GameWorldAssetLoader::Pump (worker → main register)
  → RetryPendingComponents for each in-flight instance
  → IsSceneReady(id) when pending queue empty
```

### Deferred restore callback

When `TryRestore` needs a glTF not yet cached, the handler returns false and the record lands in `pendingComponents`:

```cpp
// src/spark/scene/core/SceneManager.cpp
void SceneManager::RetryPendingComponents(LoadedSceneInstance& instance) {
    for (each pending record) {
        const IComponentSnapshotHandler* handler =
                ComponentSnapshotRegistry::Default().FindByTag(pending.record.kind.CStr());
        if (handler->TryRestore(*pending.object, pending.record, world, ctx)) {
            continue;  // succeeded — drop from queue
        }
        stillPending.PushBack(pending);  // retry next Pump
    }
}
```

`Apply` registers `onDeferredComponent` so async handlers queue work instead of blocking.

### Pump ties asset loader to scene readiness

```cpp
void SceneManager::Pump() {
    world.GetAssetLoader().Pump(world);
    for (each LoadedSceneInstance not ready) {
        RetryPendingComponents(instance);
        FinalizeInstanceReadyState(instance);
    }
}

bool SceneManager::IsSceneReady(const SceneInstanceId id) const noexcept {
    return instance->ready && !instance->failed;
}
```

Editor and demos call `Pump()` every Update tick — same hook Part XVI async-assets chapter extends with progress UI.

## 2. Core Concepts

### Why async loading matters

Large scenes parse thousands of lines and trigger mesh/texture IO. Doing that synchronously inside `LoadSceneFromFile` stalls the thread that also calls `PresentFrame` — visible hitches, input lag, and swapchain timeouts on macOS MoltenVK.

**Async loading** moves document parse and asset requests off the critical path while keeping ECS mutation and GPU resource creation on the main thread where Spark's rules require it. The user sees entities appear with transforms immediately; textures and skinned meshes refine over subsequent frames.

### SceneManager responsibilities

`SceneManager` owns loaded scene **instances**, each tagged with `SceneInstanceId`. It wraps `SceneDeserializer::Apply`, tracks root objects per instance, and coordinates `GameWorldAssetLoader` for pending component restores. `Pump()` advances asset loads and retries deferred handlers.

Games call `BeginLoadSceneAsync` → poll `IsSceneReady` → gameplay continues. Editor shows status "Loading scene (async assets)…" until `FinalizeInstanceReadyState` marks ready.

### Parse vs Apply threading

Spark's async entry reads the file into `SceneDocument` on the calling thread (often main), then `BeginLoadSceneInternal` runs Apply immediately but leaves components pending until assets arrive. Heavy file IO can move to workers in your engine; **Apply and GameObject creation stay main-thread**.

Never call `world.CreateGameObject` from a worker — races with Update, physics, and render snapshot.

### Deferred component restore

Skinned meshes and large assets may not be resident when Apply first runs. Handlers queue `PendingComponentRestore` records; `RetryPendingComponents` attempts restore each Pump until success or hard failure.

This pattern generalizes to Part **XVI** streaming — entities appear with transform first, meshes pop in when glTF decode completes. `skinned_mesh` soft-fail prevents entire level failing when one hero asset is missing.

### SceneInstanceId and unload

Each load allocates a unique instance id; entities tag `SetSceneInstanceId` for bulk unload. `UnloadScene(id)` destroys root subtrees without touching other instances. Opening a new file with `additive = false` calls `UnloadAllScenes` first.

If user opens file B while A is pending, discard stale instance or cancel asset requests to avoid duplicate sun/ground entities.

### Progress and Pump integration

Central engine tick should call `sceneManager.Pump()` alongside asset loader pump. Without Pump, pending components never resolve and `IsSceneReady` stays false forever.

```text
BeginLoad → ApplyDocumentInstance → Pump loops → IsSceneReady
```

| State | pendingComponents | User-visible |
|-------|-------------------|--------------|
| Loading | >0 | status line pending count |
| Ready | 0 | finalize editor arrays |
| Failed | any | error + UnloadScene |

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### BeginLoadSceneAsync + IsSceneReady polling

**Concept:** Why async loading matters

**Decision:** BeginLoadSceneAsync + IsSceneReady polling

**Why:** Main thread stays responsive; assets stream through existing loader.

**In Spark:** `src/spark/scene/SceneManager.cpp`

### LoadedSceneInstance table with pendingComponents

**Concept:** SceneManager responsibilities

**Decision:** LoadedSceneInstance table with pendingComponents

**Why:** Encapsulates per-file load state; demos hold SceneInstanceId handle.

**In Spark:** `include/spark/scene/SceneManager.hpp`

### ApplyDocumentInstance on main thread only

**Concept:** Parse vs Apply threading

**Decision:** ApplyDocumentInstance on main thread only

**Why:** Same Apply path as sync load; only scheduling differs.

**In Spark:** `SceneManager::ApplyDocumentInstance`

### RetryPendingComponents each Pump

**Concept:** Deferred component restore

**Decision:** RetryPendingComponents each Pump

**Why:** Skinned_mesh and async assets resolve without blocking first frame.

**In Spark:** `SceneManager::RetryPendingComponents`

### TagSubtreeSceneInstance on roots

**Concept:** SceneInstanceId and unload

**Decision:** TagSubtreeSceneInstance on roots

**Why:** Bulk destroy loaded content; supports additive multi-scene later.

**In Spark:** `SceneManager::UnloadScene`

### Pump merges assetLoader + instance finalize

**Concept:** Progress and Pump integration

**Decision:** Pump merges assetLoader + instance finalize

**Why:** Single call site in engine tick for all async scene work.

**In Spark:** `SceneManager::Pump`

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/SceneManager.cpp`
- `include/spark/scene/SceneManager.hpp`
- `src/spark/demo/SceneEditor3DDemo.cpp`

### Step 1 — Read BeginLoadSceneAsync entry

File path variant reads document then delegates to document overload.

*BeginLoadSceneAsync overloads*

```cpp
// src/spark/scene/SceneManager.cpp (lines 133–203)
SceneInstanceId SceneManager::BeginLoadSceneAsync(const char* path, const SceneLoadOptions& options) {
    SceneDocument document;
    SceneDeserializer deserializer;
    if (!deserializer.ReadFromFile(path, document)) {
        return kInvalidSceneInstanceId;
    }
    return BeginLoadSceneAsync(document, path, options);
}

SceneInstanceId SceneManager::BeginLoadSceneAsync(
        const SceneDocument& document, const char* path, const SceneLoadOptions& options) {
    if (!options.additive) {
        UnloadAllScenes();
    }
    return BeginLoadSceneInternal(document, path, options);
}
```

Sync `LoadSceneFromFile` pumps up to `kMaxPumpIterations` until ready — compare frame hitch profile against async spread.

### Step 2 — Trace ApplyDocumentInstance

Apply with deferred hook, collect roots, retry pending, finalize ready state.

*ApplyDocumentInstance pipeline*

```cpp
// src/spark/scene/SceneManager.cpp (lines 57–131)
void SceneManager::ApplyDocumentInstance(
        const SceneDocument& document, LoadedSceneInstance& instance, const SceneLoadOptions& options) {
    SceneApplyContext ctx{};
    ctx.assetsRoot = options.assetsRoot;
    ctx.sceneInstanceId = instance.id;
    ctx.assetLoader = &assetLoader;
    ctx.onDeferredComponent = OnDeferredComponentStatic;
    ctx.deferredUserData = &instance;

    HashMap<std::uint64_t, GameObject*> idMap;
    if (!deserializer.Apply(document, world, ctx, &idMap)) {
        instance.failed = true;
        return;
    }
    // Collect root objects (parentId < 0), TagSubtreeSceneInstance
    RetryPendingComponents(instance);
    FinalizeInstanceReadyState(instance);
}
```

### Step 3 — Study RetryPendingComponents

Deferred handler retry loop — skinned_mesh soft-fail vs hard-fail for other kinds.

*Pending component retry*

```cpp
// src/spark/scene/SceneManager.cpp (lines 57–87)
void SceneManager::RetryPendingComponents(LoadedSceneInstance& instance) {
    for (std::size_t i = 0; i < instance.pendingComponents.GetSize(); ++i) {
        PendingComponentRestore pending = instance.pendingComponents[i];
        const IComponentSnapshotHandler* handler =
                ComponentSnapshotRegistry::Default().FindByTag(pending.record.kind.CStr());
        if (handler->TryRestore(*pending.object, pending.record, world, ctx)) {
            continue;  // success — drop from pending
        }
        if (std::strcmp(pending.record.kind.CStr(), "skinned_mesh") != 0) {
            instance.failed = true;
        }
        stillPending.PushBack(MoveTemp(pending));
    }
    instance.pendingComponents = MoveTemp(stillPending);
}
```

### Step 4 — Wire Pump into engine tick

Call order: gameplay Update → `sceneManager.Pump()` → render OnRender.

### Step 5 — Editor LoadSceneFromFile async path

Clears placed entities, stores document, BeginLoadSceneAsync with assetsRoot.

*Async load from editor menu*

```cpp
// src/spark/demo/SceneEditor3DDemo.cpp (lines 1053–1084)
ClearPlaced(w);
ClearUserLights(w);
pendingLoadDocument = document;
Spark::SceneLoadOptions options{};
options.assetsRoot = SPARK_ASSETS_DIR;
options.additive = true;
loadedSceneId = sceneManager->BeginLoadSceneAsync(document, path, options);
sceneLoadInProgress = true;
if (sceneManager->IsSceneReady(loadedSceneId)) {
    FinalizeAsyncSceneLoad(w);
    sceneLoadInProgress = false;
} else {
    SetStatusMessage(Spark::Utf8String("Loading scene (async assets)…"));
}
```

### Step 6 — Poll IsSceneReady in Update

```cpp
if (sceneLoadInProgress && sceneManager->IsSceneReady(loadedSceneId)) {
    FinalizeAsyncSceneLoad(world);
    sceneLoadInProgress = false;
}
```

### Step 7 — Cancel superseded load

UnloadScene(oldId) when user opens second file mid-load.

### Step 8 — Milestone — async load without stall

Profile max frame ms during async vs sync internal pump — no multi-frame stall during entity creation.

## 5. How This Connects to the Rest of Your Engine

Part **XVI** generalizes worker pools for asset decode. Editor Chapters 4–5 show load status in status line. Part **VI** glTF import completes through same asset loader pending queue. Sync `LoadSceneFromFile` with internal pump loop remains for tooling scripts.

Editor finalize should set `needsTreeRebuild` when async load completes so hierarchy shows new entities without restart.

## 6. Pitfalls and Debugging

- **Apply from worker thread** — data races on GameWorld; main thread only.
- **Forgetting Pump** — loads never complete; IsSceneReady always false.
- **Double apply same document** — duplicate entities; guard with load state flag.
- **Loading while sim running** — pause physics or load into staging world first.
- **additive=false without clear** — orphaned entities from previous edit session.

**Debugging workflow:** log `pendingComponents.GetSize()` each Pump — unchanged for 60+ frames means wrong asset path or missing Pump.

## 7. Exercises

1. Load 100-entity scene async; log main-thread frame ms during first 60 frames.
2. Cancel in-flight load by starting second file before first completes.
3. Defer skinned_mesh: verify entity exists with transform while mesh pending.
4. Compare sync LoadSceneFromFile internal pump vs async spread — total wall time vs max single-frame dt.
5. Implement progress status showing pendingComponents.GetSize() in editor status line.

## Summary

Async loading applies SceneDocument immediately then resolves assets via Pump and deferred handlers. Spark exposes BeginLoadSceneAsync and SceneInstanceId through SceneManager. Without Pump each frame, pendingComponents never drain and IsSceneReady stays false indefinitely.

---

Next: [Editor Application](04-editor-shell.md).
