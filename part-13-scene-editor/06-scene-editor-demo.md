# Scene Editor Demo

> **Build-Along Milestone:** SceneEditor3DDemo save/load restores five edited entities with matching viewport appearance.

**Spark API note:** SceneEditor3DDemo viewport rendering goes through **`world.Rendering().SubmitLit`** like other lit demos. Mentions of **`FillStandardLitSceneFromWorld`** below mean the same **`LitSceneSubmitter`** gather.

**You will learn:**

- Run SceneEditor3DDemo as full editor vertical slice.
- Save and load scenes through editor menus with v4 format.
- Validate create → edit → persist → reload loop with async assets.
- Use demo as regression target after serializer changes.

## 1. The Problem We Are Solving

Parts XIII chapters 1–5 built serialization, async load, editor shell, and panels separately. **SceneEditor3DDemo** proves they work together — viewport, placement tools, hierarchy/inspector, `SceneSerializer` v4 output, `SceneManager` async finalize, and **`world.Rendering().SubmitLit`** HDR rendering in one executable.

The demo encodes **editor-specific policy**: which entities save (placed props and user lights), how project-relative paths resolve through capture callbacks, and how load clears stale placed arrays before async apply. Treat the five-case regression matrix as a merge gate — silent serializer regressions lose artist work without compile errors.

Milestone: place five props, save, reload async, verify transforms and mesh paths match within float tolerance.

## 1b. Concept Deep Dive — SceneEditor3DDemo Save Policy

### ContentModel filters what gets captured

Not every bootstrap entity belongs in a level file. `SceneEditorContentModel` supplies the capture predicate and path context:

```cpp
// src/spark/demo/SceneEditor3DDemo.cpp
void SceneEditor3DDemo::SaveSceneToFile(Spark::GameWorld& w) {
    SceneCaptureContext ctx = contentModel.BuildCaptureContext();
    const auto includeEntity = [this](const GameObject* go) -> bool {
        return contentModel.ShouldCapture(go);
    };

    SceneSerializer serializer;
    SceneDocument document = serializer.Capture(w, ctx, includeEntity);
    document.header.assetsRoot = Utf8String(ScenePathResolver::AssetsRoot());
    serializer.WriteToFile(document, path.CStr());
}
```

Placed props and user lights save; editor gizmo shells and bootstrap scaffolding do not — policy lives in one place, not scattered `if` checks in the serializer.

### Async load integrates through SceneManager

Load clears stale placed arrays, begins async apply, then finalizes when assets register:

```cpp
void SceneEditor3DDemo::FinalizeAsyncSceneLoad(Spark::GameWorld& w) {
    contentModel.IntegrateLoadedInstance(w, loadedSceneId, pendingLoadDocument, hooks);
    instanceTracker.Register(loadedSceneId);
    selectedObject = nullptr;  // rebind selection after graph swap
}
```

`Pump()` runs each Simulate tick; status line shows "Loading scene (async assets)…" until `IsSceneReady` returns true.

### Five-case regression matrix as merge gate

The demo encodes editor-specific invariants: create → edit transforms → save v4 → reload async → viewport matches. Silent serializer regressions lose artist work without compile errors — treat this demo like ParticleDemo's integration harness.

## 2. Core Concepts

### Vertical slice integration

`SceneEditor3DDemo` is Part XIII's **regression harness** — editor viewport, placement tools, hierarchy/inspector, save/load, async SceneManager, and standard HDR lit rendering in one executable. Same role as ParticleDemo (Part VIII) and Platformer2DDemo (Part XI).

```text
Ch1–2 SceneSerializer v4     → SaveSceneToFile / ReadFromFile
Ch3   SceneManager async     → BeginLoadSceneAsync + Pump + Finalize
Ch4   EditorApplication      → shared submit + bootstrap
Ch5   Panels                 → hierarchy selection + inspector readout
Ch6   This demo              → placed[] tracking + menu dispatch + checklist
```

### Menu action dispatch

ImGui or UI menu bar routes File → Save/Load through `SceneEditorMenuAction` enum — central dispatch keeps UI code readable. Save calls `SaveSceneToFile`; Load calls `LoadSceneFromFile` with v4 async path and legacy v1/v2 fallback for older classroom files.

Status line messages confirm entity count and format version — user-visible proof save succeeded.

### Placed entity tracking

Demo maintains `placed[]` and `placedRel[]` parallel arrays mapping GameObject* to project-relative asset paths. Capture callbacks consult these arrays for mesh and texture path resolution. Include predicate saves only placed props and `userLights` — not bootstrap sun/ground unless explicitly placed.

`ClearPlaced` before load prevents duplicate props when reloading into non-empty world. `userLights` array tracks designer-spawned point lights separate from bootstrap sun.

### Async load finalize

Load stores `pendingLoadDocument`, starts `BeginLoadSceneAsync`, sets `sceneLoadInProgress`. Update polls `IsSceneReady`; `FinalizeAsyncSceneLoad` sorts roots by id, repopulates placed arrays, restores selection hints. Status: "Loaded N entities (SceneManager, async assets ready)."

### Play vs edit simulation

Editor tick may gate `PhysicsSubsystem::Simulate` with a simulate flag — authoring focuses on transforms without balls falling through floor. Document which mode your team uses for level design vs physics tuning.

### End-to-end regression matrix

Run before every serializer merge:

| Case | Expected |
|------|----------|
| Empty placed | zero or minimal E lines |
| Unit cube | transform + mesh unit_cube |
| glTF + light | mesh path + point_light |
| Double reload | no duplicate entities |
| Corrupt C line | Apply false, status error |

Create → transform in inspector → save → reload → visual match proves serialization + submit + editor integration.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SceneEditor3DDemo target

**Concept:** Vertical slice integration

**Decision:** SceneEditor3DDemo target

**Why:** Single binary exercises full Part XIII stack; CI smoke test candidate.

**In Spark:** `include/spark/demo/SceneEditor3DDemo.hpp`

### SceneEditorMenuAction enum + handler lambda

**Concept:** Menu action dispatch

**Decision:** SceneEditorMenuAction enum + handler lambda

**Why:** SaveScene / LoadScene menu items map to member functions.

**In Spark:** `SceneEditor3DDemo.cpp` menu switch

### placed / placedRel arrays + includeEntity lambda

**Concept:** Placed entity tracking

**Decision:** placed / placedRel arrays + includeEntity lambda

**Why:** Editor-specific path mapping without changing core serializer.

**In Spark:** `SaveSceneToFile` capture callbacks

### FinalizeAsyncSceneLoad after IsSceneReady

**Concept:** Async load finalize

**Decision:** FinalizeAsyncSceneLoad after IsSceneReady

**Why:** Rebinds editor tracking arrays to loaded entities.

**In Spark:** `SceneEditor3DDemo::LoadSceneFromFile`

### Optional simulate gate in Update

**Concept:** Play vs edit simulation

**Decision:** Optional simulate gate in Update

**Why:** Skip physics while authoring; enable for physics-placement tests.

**In Spark:** Editor Update simulate flag

### Same renderer as ThreeDDemo

**Concept:** End-to-end checklist

**Decision:** Same renderer as ThreeDDemo

**Why:** No editor-specific shaders — validates architecture separation.

**In Spark:** `world.Rendering().SubmitLit` (legacy `FillStandardLitSceneFromWorld` delegates to same gather)

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/demo/SceneEditor3DDemo.cpp`
- `include/spark/demo/SceneEditor3DDemo.hpp`
- `src/spark/scene/SceneManager.cpp`

### Step 1 — Launch SceneEditor3DDemo

Run Spark scene editor demo — baseline viewport with placement UI, hierarchy, and file menu.

### Step 2 — Place mesh from project browser

Spawn glTF or unit cube prop; move via inspector; verify shadow + HDR in viewport. `placedRel` stores relative path for Capture resolver.

### Step 3 — Save to disk

File → Save — inspect `scene_editor/scene.txt` for `spark_scene_v4`, entity count, transform/mesh lines.

*SaveSceneToFile Capture + WriteToFile*

```cpp
// src/spark/demo/SceneEditor3DDemo.cpp (lines 925–934)
SceneSerializer serializer;
const SceneDocument document = serializer.Capture(w, ctx, includeEntity);
if (!serializer.WriteToFile(document, path)) {
    SetStatusMessage(Spark::Utf8String("Save failed (could not open file)."));
    return;
}
SetStatusMessage(Spark::Utf8String(
        std::format("Saved {} entities (spark_scene_v4) → scene_editor/scene.txt",
                document.entities.GetSize()).c_str()));
```

### Step 4 — Load async path

File → Load — trace BeginLoadSceneAsync, sceneLoadInProgress, status message during asset pump.

*LoadSceneFromFile async branch*

```cpp
// src/spark/demo/SceneEditor3DDemo.cpp (lines 1064–1084)
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

### Step 5 — Finalize when ready

Poll IsSceneReady in Update; FinalizeAsyncSceneLoad repopulates placed tracking arrays.

*FinalizeAsyncSceneLoad — scan instance entities*

```cpp
// src/spark/demo/SceneEditor3DDemo.cpp (lines 956–980)
void SceneEditor3DDemo::FinalizeAsyncSceneLoad(Spark::GameWorld& w) {
    Spark::Array<Spark::GameObject*> instanceObjects;
    w.ForEachGameObject([&](Spark::GameObject* object) {
        if (object != nullptr && object->GetSceneInstanceId() == loadedSceneId) {
            instanceObjects.PushBack(object);
        }
    });
    SortObjectsById(instanceObjects);
    // Rebuild placed[] and userLights from pendingLoadDocument component records
}
```

### Step 6 — Visual diff regression

Screenshot before save and after reload — compare transforms, materials, light count. Editor camera is not saved; manual alignment required.

### Step 7 — Break Transform handler test

Intentionally corrupt sscanf format; verify load error handling — restore before commit.

### Step 8 — Milestone — five-entity save/load

Complete save/load checklist with 5 placed entities including one point light and one glTF mesh path.

## 5. How This Connects to the Rest of Your Engine

Touches Part **IX** shadows in viewport, Part **X** post volumes in optional inspector extensions, Part **XIV** physics when simulate flag enabled, Part **XVI** resize with dock panels open. Production ships editor binary sharing same libraries as runtime game.

Legacy v1/v2 loaders in same file teach migration — read only to understand history; new coursework uses v4 exclusively.

## 6. Pitfalls and Debugging

- **Save without registry entry for new component** — data lost silently on reload.
- **Load into non-empty world without clear** — duplicate sun/ground; call ClearPlaced first.
- **Editor camera serialized** — expected omission; only authored entities persist.
- **Forgetting to Pump SceneManager** — load hangs at "async assets" forever.
- **placed array desync after manual DestroyGameObject** — remove from placed[] in delete handler.

**Debugging workflow:** when save/load or physics visuals diverge, work top-down: (1) log ECS component fields on CPU, (2) confirm update tick order, (3) breakpoint the failing handler, (4) capture RenderDoc and compare draw count against expected transform.

## 7. Exercises

1. Complete save/load checklist with 5 placed entities including one point light and one glTF mesh path.
2. Break Transform serializer sscanf format temporarily; verify load error status.
3. Profile editor Update vs OnRender with hierarchy + inspector open.
4. Load scene during async asset pump; finalize restores placed[] count matching saved entity count.
5. Run testing matrix from problem section: empty save, single cube, glTF+light, double reload, corrupt C line.

## Summary

SceneEditor3DDemo integrates editor shell, panels, serialization, and SceneManager async load into one vertical slice. Use it as the regression target for all Part XIII work. New coursework should standardize on spark_scene_v4.

---

Continue to [Part XIV — Physics](../part-14-physics/index.md).
