# Component Snapshots

> **Build-Along Milestone:** Transform and MeshComponent round-trip through registry without data loss.

**You will learn:**

- Register Capture/Restore handler pairs per component type.
- Serialize stable entity IDs and parent links in three-pass Apply.
- Round-trip Transform and MeshComponent through the registry.
- Use SceneCaptureContext for editor-specific asset path mapping.

## 1. The Problem We Are Solving

A scene file is a list of entities with **typed component blobs**. Without a registry, every new component bloats a monolithic switch in the loader — unmaintainable by Part XIV. Spark's `ComponentSnapshotRegistry` is the extension point: handlers own payload format; core code owns entity graph and line I/O.

You will trace `TransformSnapshotHandler` as the minimal template, then follow editor save hooks that supply asset path context Capture needs for mesh and material restoration.

Milestone: round-trip an entity with Transform + MeshComponent through Capture → Apply without WriteToFile; translation matches within epsilon 1e-4.

## 1b. Concept Deep Dive — Registry Handlers in Spark

### One handler pair per component kind

`ComponentSnapshotRegistry` owns `IComponentSnapshotHandler` instances registered at startup. Core serializer code never switches on every component type — it asks the registry by kind on capture and by tag on restore:

```cpp
class TransformSnapshotHandler final : public IComponentSnapshotHandler {
public:
    [[nodiscard]] ComponentKind GetKind() const noexcept override { return ComponentKind::Transform; }
    [[nodiscard]] const char* GetKindTag() const noexcept override { return "transform"; }

    [[nodiscard]] bool TryCapture(
            const GameObject& owner,
            const SceneCaptureContext& /*ctx*/,
            ComponentRecord& out) const override {
        const TransformComponent* tr = owner.GetComponent<TransformComponent>();
        // ... snprintf scale + translation + rotation into out.payload ...
        out.kind = Utf8String(GetKindTag());
        return true;
    }

    [[nodiscard]] bool TryRestore(
            GameObject& owner,
            const ComponentRecord& record,
            GameWorld& /*world*/,
            const SceneApplyContext& /*ctx*/) const override {
        // ... sscanf payload → TransformComponent::SetLocalTransform ...
    }
};
```

The transform handler is the **minimal template**: fixed kind tag, space-separated numeric payload, no asset IO.

### SceneCaptureContext for editor path mapping

Editor save hooks supply `SceneCaptureContext` so mesh handlers write **project-relative** paths (`models/chair.glb`) instead of absolute temp paths from drag-import. Restore uses the same context via `SceneApplyContext::assetsRoot` + `assetLoader`.

### Extensibility without touching core I/O

Adding `SoundCueComponent` persistence means one new handler class + `RegisterHandler` call — not edits to line parser loops. Rendering handlers live in `ComponentSnapshotHandlersRendering.cpp`; gameplay handlers in `ComponentSnapshotHandlersGameplay.cpp`.

## 2. Core Concepts

### Registry pattern for extensibility

Each component kind registers an `IComponentSnapshotHandler` with `{TryCapture, TryRestore}` and a stable **kind tag** string. `ComponentSnapshotRegistry::Default()` owns all handlers at startup. Core serializer code never switches on every component type — it asks the registry by `ComponentKind` on capture and by tag on restore.

Adding `TerrainComponent` or `PostProcessVolumeComponent` means one new handler class plus `RegisterHandler<>` — not editing a central 200-case switch.

### Snapshot vs live component

A **snapshot** is plain data: space-separated floats, quoted paths, enum integers — whatever fits in `ComponentRecord::payload`. A **live component** is a C++ object on `GameObject` with pointers, cached GPU indices, and runtime flags.

Capture reads live → writes snapshot. Restore reads snapshot → constructs or updates live. Handlers must never serialize raw pointers or `SharedPtr` addresses. Mesh capture stores `"path/to/model.gltf"` plus slot tag; restore calls `world.RegisterMesh` or queues async load through `SceneApplyContext`.

### Transform handler as template

`TransformSnapshotHandler` is the simplest reference: ten floats (scale xyz, translation xyz, rotation quaternion xyzw) in payload, kind tag `transform`. Capture uses `snprintf`; restore uses `sscanf` then `SetLocalTransform`. No asset dependencies — synchronous Apply.

Every other handler copies this shape: validate kind tag, parse payload, add component if missing, set fields.

### Stable entity ids and parent linking

Entity ids in the file are `GameObject::GetId()` values at save time. Apply creates objects first (pass one), restores components (pass two), then links parents by id (pass three). Parent ids reference file ids, not live pointers.

`HashMap<uint64_t, GameObject*>` built during Apply enables editor selection restore and external references. Invalid parent id → Apply returns false.

### SceneCaptureContext and SceneApplyContext

Capture receives `SceneCaptureContext` with optional callbacks: `resolveMeshAssetPath`, `resolveTexturePath`, userData for editor-specific path mapping. Apply receives `SceneApplyContext` with `assetsRoot`, `sceneInstanceId`, `assetLoader`, and deferred-restore hooks.

SceneEditor3DDemo maps placed entities to project-relative paths through capture callbacks — without them mesh handler would write empty paths for built-in slots.

### Failure policy on restore

`TryRestore` returning false aborts Apply for strict components. Spark special-cases `skinned_mesh`: missing skeleton assets defer to `SceneManager::RetryPendingComponents` instead of failing immediately. Document which components are hard-fail vs deferred so gameplay programmers know load UX.

Partial success leaves created entities in world — callers must UnloadScene or clear before retry.

### Three-pass Apply invariant

Apply must never SetParent before all entities exist:

```text
Pass 1: CreateGameObject → idToObject[id]
Pass 2: For each C line → FindByTag → TryRestore
Pass 3: SetParent(child, parent[id])
```

Log pass-two failures with entity name and kind tag. Pass-three failure means parentId typo or cyclic hierarchy rejected by `SetParent`.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### ComponentSnapshotRegistry::Default()

**Concept:** Registry pattern for extensibility

**Decision:** ComponentSnapshotRegistry::Default()

**Why:** Built-in handlers registered in RegisterBuiltInHandlers; extended modules add more.

**In Spark:** `include/spark/scene/serialization/ComponentSnapshotRegistry.hpp`

### TransformSnapshotHandler with ten-float payload

**Concept:** Transform handler as template

**Decision:** TransformSnapshotHandler with ten-float payload

**Why:** Simplest handler demonstrates Capture/Restore without asset IO.

**In Spark:** `ComponentSnapshotRegistry.cpp` lines 82–148

### ComponentRecord kind + payload strings

**Concept:** Snapshot vs live component

**Decision:** ComponentRecord kind + payload strings

**Why:** Line format stays handler-owned; core parser stays generic.

**In Spark:** `ParseComponentLine` in SceneSerializer.cpp

### Three-pass Apply in SceneDeserializer

**Concept:** Stable entity ids and parent linking

**Decision:** Three-pass Apply in SceneDeserializer

**Why:** Create all entities before parent links — avoids dangling parent pointers.

**In Spark:** `SceneDeserializer::Apply` lines 436–494

### Context structs passed to every handler

**Concept:** SceneCaptureContext and SceneApplyContext

**Decision:** Context structs passed to every handler

**Why:** Editor path resolution and async asset loader without global state.

**In Spark:** `SceneCaptureContext` / `SceneApplyContext` headers

### Return false on first hard failure

**Concept:** Failure policy on restore

**Decision:** Return false on first hard failure

**Why:** Skinned mesh defers; most components fail fast for clear error messages.

**In Spark:** Apply loop with skinned_mesh exception

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scene/serialization/ComponentSnapshotRegistry.hpp`
- `src/spark/scene/serialization/ComponentSnapshotRegistry.cpp`
- `src/spark/scene/serialization/SceneSerializer.cpp`
- `src/spark/demo/SceneEditor3DDemo.cpp`

### Step 1 — Read IComponentSnapshotHandler interface

Note GetKind, GetKindTag, TryCapture, TryRestore signatures.

*Handler interface contract*

```cpp
// include/spark/scene/serialization/IComponentSnapshotHandler.hpp (lines 1–35)
struct SceneCaptureContext {
    Utf8String (*resolveMeshAssetPath)(const GameObject& owner, void* userData) = nullptr;
    void* meshAssetUserData = nullptr;
    Utf8String (*resolveSkinnedAssetPath)(const GameObject& owner, void* userData) = nullptr;
    void* skinnedAssetUserData = nullptr;
    Utf8String (*resolveTexturePath)(const GameObject& owner, void* userData) = nullptr;
    void* textureUserData = nullptr;
};

struct SceneApplyContext {
    const char* assetsRoot = nullptr;
    SceneInstanceId sceneInstanceId = kInvalidSceneInstanceId;
    GameWorldAssetLoader* assetLoader = nullptr;
    void (*onEntityCreated)(GameObject* object, void* userData) = nullptr;
    void* entityUserData = nullptr;
};
```

### Step 2 — Study Transform Capture/Restore

Trace float formatting and sscanf restore — template for your own components.

*TransformSnapshotHandler complete*

```cpp
// src/spark/scene/serialization/ComponentSnapshotRegistry.cpp (lines 82–148)
class TransformSnapshotHandler final : public IComponentSnapshotHandler {
public:
    [[nodiscard]] const char* GetKindTag() const noexcept override { return "transform"; }

    [[nodiscard]] bool TryCapture(
            const GameObject& owner, const SceneCaptureContext& /*ctx*/, ComponentRecord& out) const override {
        const TransformComponent* tr = owner.GetComponent<TransformComponent>();
        if (tr == nullptr) {
            return false;
        }
        const Transform& L = tr->GetLocalTransform();
        char buf[256]{};
        std::snprintf(buf, sizeof(buf),
                "%.6f %.6f %.6f %.6f %.6f %.6f %.6f %.6f %.6f %.6f",
                L.scale.x, L.scale.y, L.scale.z,
                L.translation.x, L.translation.y, L.translation.z,
                L.rotation.x, L.rotation.y, L.rotation.z, L.rotation.w);
        out.kind = Utf8String(GetKindTag());
        out.payload = Utf8String(buf);
        return true;
    }

    [[nodiscard]] bool TryRestore(
            GameObject& owner, const ComponentRecord& record,
            GameWorld& /*world*/, const SceneApplyContext& /*ctx*/) const override {
        Transform L{};
        if (std::sscanf(record.payload.CStr(), "%f %f %f %f %f %f %f %f %f %f",
                &L.scale.x, &L.scale.y, &L.scale.z,
                &L.translation.x, &L.translation.y, &L.translation.z,
                &L.rotation.x, &L.rotation.y, &L.rotation.z, &L.rotation.w) != 10) {
            return false;
        }
        TransformComponent* tr = owner.GetComponent<TransformComponent>();
        if (tr == nullptr) {
            tr = owner.AddComponent<TransformComponent>();
        }
        tr->SetLocalTransform(L);
        return true;
    }
};
```

### Step 3 — Find registration site

See RegisterBuiltInHandlers order — Transform and Mesh register before lights and physics kinds.

*RegisterBuiltInHandlers*

```cpp
// src/spark/scene/serialization/ComponentSnapshotRegistry.cpp (lines 660–674)
void RegisterBuiltInHandlers(ComponentSnapshotRegistry& registry) {
    RegisterHandler<TransformSnapshotHandler>(registry);
    RegisterHandler<MeshSnapshotHandler>(registry);
    RegisterHandler<MaterialSnapshotHandler>(registry);
    RegisterHandler<PointLightSnapshotHandler>(registry);
    RegisterHandler<CameraSnapshotHandler>(registry);
    RegisterHandler<SkinnedMeshSnapshotHandler>(registry);
    RegisterHandler<AnimatorSnapshotHandler>(registry);
    RegisterExtendedSnapshotHandlers(registry);
    RegisterMoreSnapshotHandlers(registry);
    RegisterRenderingSnapshotHandlers(registry);
}
```

### Step 4 — Trace Apply three-pass linking

Create entities → restore components → SetParent by id map.

*SceneDeserializer::Apply*

```cpp
// src/spark/scene/serialization/SceneSerializer.cpp (lines 436–437)
    // Pass 1: CreateGameObject for each EntityRecord → idToObject
    // Pass 2: TryRestore each ComponentRecord via FindByTag
    // Pass 3: SetParent(child, parent) using record.parentId
```

### Step 5 — Editor capture callbacks

SceneEditor3DDemo resolves mesh/texture paths for placed props via SceneCaptureContext lambdas.

*Capture context path resolvers*

```cpp
// src/spark/demo/SceneEditor3DDemo.cpp (lines 870–906)
ctx.resolveMeshAssetPath = [](const GameObject& owner, void* userData) -> Utf8String {
    SceneEditor3DDemo* self = static_cast<CaptureCtx*>(userData)->self;
    for (std::size_t i = 0; i < self->placed.GetSize(); ++i) {
        if (self->placed[i] == &owner) {
            return self->placedRel[i];
        }
    }
    return Utf8String{};
};
```

### Step 6 — Round-trip Transform unit test

Entity with known local transform → Capture → Apply into empty world → compare matrix epsilon 1e-4.

### Step 7 — Save from editor menu

File → Save triggers Capture with includeEntity filter, WriteToFile.

### Step 8 — Milestone — Transform + Mesh round-trip

Confirm placed cube mesh + transform lines in scene.txt and viewport position after reload.

## 5. How This Connects to the Rest of Your Engine

Part **XII** terrain height arrays may use custom snapshot (large payloads). Part **XI** tilemaps may reference external TMX paths. Part **XIV** box/sphere collider handlers appear in extended registration. Part **XVI** async asset loader completes deferred skinned_mesh restores.

Cross-reference inspector editable fields with handler payloads so designers edit the same floats Capture writes. Prefer new kind tags over breaking existing payloads when evolving format.

## 6. Pitfalls and Debugging

- **Pointer serialization** — always ids and asset paths, never addresses.
- **Forgetting to register new component** — silent omission on save; entity loads without expected behavior.
- **Apply order** — parents linked only after all entities exist.
- **Empty resolveMeshAssetPath** — saves built-in slots with blank paths; restore falls back to slot tags.
- **Handler tag typo** — `transform` vs `Transform` breaks restore silently (handler not found).

**Debugging workflow:** edit transform, Save, grep scene.txt — must match inspector within float format precision. One-frame lag means panel tick after OnRender.

## 7. Exercises

1. Register PostProcessVolumeComponent snapshot handler.
2. Round-trip entity with Transform + MeshComponent; verify tint and slot.
3. Load scene with missing asset path — observe loader error or deferred pending state.
4. Capture parent/child chain of depth 3; verify parentId fields in file.
5. Construct ComponentRecord for transform in code; TryRestore and assert translation without disk IO.

## Summary

Component snapshots use a registry of TryCapture/TryRestore handlers per ECS type. Spark stores kind-tagged records in SceneDocument with stable entity ids and three-pass parent linking.

---

Next: [Async Loading](03-async-load.md).
