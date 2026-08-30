# Editor Panels

> **Build-Along Milestone:** Inspector Transform edits move entity in viewport immediately; hierarchy selection stays in sync.

**You will learn:**

- Build hierarchy, inspector, and project browser panels implementing IEditorPanel.
- Edit Transform and component fields through live ECS mutation.
- Rebuild hierarchy tree efficiently with scene revision hashing.
- Route input so UI captures pointer before viewport camera.

## 1. The Problem We Are Solving

An editor shell without panels is just a 3D viewport. **Hierarchy** navigates entity parent/child relationships; **Inspector** surfaces component data for the selection; **Project** browses assets on disk. Spark implements these as `IEditorPanel` plugins arranged by `EditorDockShell` — the same retained-mode UI stack from Part XI, not a separate Dear ImGui immediate-mode fork for panel content.

The architectural test: edit a Transform in inspector, save scene (Chapter 1–2), reload (Chapter 3), and confirm the value round-trips. If inspector held a **copy**, save would persist stale data while the viewport showed the copy's values — a classic student bug. Spark's InspectorPanel reads live components; editable widgets must call `SetTranslation` on the actual `TransformComponent`.

Milestone: select entity in hierarchy; inspector shows name and position; extend with editable FloatField and verify viewport moves same frame.

## 1b. Concept Deep Dive — Live ECS Editing

### IEditorPanel plugin contract

Each dock region is an `IEditorPanel` — hierarchy, inspector, project browser share one interface:

```cpp
// include/spark/editor/IEditorPanel.hpp
class IEditorPanel {
    [[nodiscard]] virtual Utf8String GetPanelId() const = 0;
    [[nodiscard]] virtual Utf8String GetDisplayName() const = 0;
    [[nodiscard]] virtual Ui::IUiElement* GetRootElement() noexcept = 0;
    virtual void OnAttach(EditorContext& ctx) {}
    virtual void OnTick(const FrameTiming& timing, EditorContext& ctx) {}
};
```

`EditorApplication` arranges panel roots in `EditorDockShell`; adding a profiler panel means a new class, not edits to hierarchy internals.

### Inspector mutates live components

`InspectorPanel::OnTick` rebuilds widget context from `EditorContext` each frame — it reads the **selected entity's actual components**, not cached copies:

```cpp
void InspectorPanel::OnTick(const FrameTiming& /*timing*/, EditorContext& ctx) {
    const InspectorWidgetContext widgetCtx = BuildWidgetContext(ctx);
    // FloatField commits call TransformComponent::SetTranslation on the live object
}
```

If inspector held a copy, save would persist stale data while the viewport showed edited values — the classic student bug this architecture prevents.

### Hierarchy rebuilds on scene revision hash

`HierarchyPanel::ComputeSceneRevision` XORs entity pointers, parent links, and names. When revision changes, the tree rebuilds without polling every node every frame:

```cpp
std::uint32_t HierarchyPanel::ComputeSceneRevision() const noexcept {
    world->ForEachGameObject([&](GameObject* obj) {
        rev ^= static_cast<std::uint32_t>(reinterpret_cast<std::uintptr_t>(obj) >> 4U);
        rev = rev * 16777619U + ... parent + name chars ...
    });
    return rev;
}
```

Selection sync stays O(1) when the graph is stable; structural edits (rep parent, rename, spawn) invalidate once.

## 2. Core Concepts

### IEditorPanel plugin contract

Each dock region implements `IEditorPanel`: `GetPanelId`, `GetDisplayName`, `GetRootElement`, lifecycle `OnAttach`/`OnDetach`/`OnTick`. Panels own their UI subtree; `EditorApplication` arranges roots in `EditorDockShell`. Adding a console or profiler panel means a new class — no edits to hierarchy internals.

`ReleaseRootElement` transfers ownership to dock layout during BuildEditorUi — factory pattern matches Part XI UI control creation.

### Hierarchy panel tree rebuild

`HierarchyPanel` walks `GameWorld` each tick (when revision changes), skipping internal objects `EditorGui` and `EditorStatusHud`. TreeView node ids map to GameObject pointers; selection callback updates `EditorSelection`. Parent/child structure mirrors ECS hierarchy — not a flat list.

`ComputeSceneRevision` XOR-mixes object addresses, parent pointers, and name characters into a 32-bit revision. When revision matches cached value and `needsTreeRebuild` is false, OnTick returns early. After async load finalize, call `needsTreeRebuild = true` so new entities appear without restart.

### Inspector live ECS binding

Inspector reads **live** components on selected entity — not copies. Transform position display comes from `GetLocalTransform().translation`; editable widgets write directly to `TransformComponent`. Same-frame submit shows inspector edits in viewport immediately.

When selection is null, show muted placeholder text. Stale selection after delete must clear `EditorSelection` to avoid dangling pointer reads.

### Project browser roots

`ProjectBrowserPanel` lists filesystem under editor project path — meshes, textures, scenes. Opens default `SPARK_BUILD_ASSETS_DIR` on attach via `project.OpenExisting`. Relative paths stored in snapshots must resolve against document `assets_root` on load.

SceneEditor3DDemo wires double-click placement: pick file → raycast ground → CreateGameObject with MeshComponent → append to `placed[]` and `placedRel[]`.

### Input capture between UI and viewport

`ProcessUiCanvasesInput` runs before camera movement. `UiConsumesGamePointer` and `UiScrollWheelConsumed` gate viewport navigation when mouse is over panels. Without this, RMB look fights tree view scrolling — classic editor UX bug.

Panel `OnTick` should not call Simulate or Submit — presentation stays in EditorApplication::OnRender.

### Dock shell and layout store

`EditorDockShell` positions hierarchy left, inspector right, project bottom — configurable widths from `SceneEditorLayoutSettings`. `TryLoadSceneEditorLayout` / save on exit persists arrangement. World viewport rect from dock drives scissor in OnRender (Chapter 4).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### IEditorPanel interface

**Concept:** IEditorPanel plugin contract

**Decision:** IEditorPanel interface

**Why:** Uniform lifecycle; EditorApplication registers three built-in panels.

**In Spark:** `include/spark/editor/IEditorPanel.hpp`

### HierarchyPanel + TreeView

**Concept:** Hierarchy panel tree rebuild

**Decision:** HierarchyPanel + TreeView

**Why:** Revision-based rebuild; filters editor-internal GameObjects.

**In Spark:** `src/spark/editor/panels/HierarchyPanel.cpp`

### InspectorPanel reads GetPrimary selection

**Concept:** Inspector live ECS binding

**Decision:** InspectorPanel reads GetPrimary selection

**Why:** Displays Transform + component list; extend with editable fields.

**In Spark:** `src/spark/editor/panels/InspectorPanel.cpp`

### ProjectBrowserPanel + EditorProject

**Concept:** Project browser roots

**Decision:** ProjectBrowserPanel + EditorProject

**Why:** Filesystem listing for asset assignment workflows.

**In Spark:** `include/spark/editor/panels/ProjectBrowserPanel.hpp`

### ProcessUiCanvasesInput first in OnUpdate

**Concept:** Input capture between UI and viewport

**Decision:** ProcessUiCanvasesInput first in OnUpdate

**Why:** Standard retained-mode UI hit testing before fly camera.

**In Spark:** `EditorApplication::OnUpdate`

### EditorDockShell + EditorLayoutStore

**Concept:** Dock shell and layout store

**Decision:** EditorDockShell + EditorLayoutStore

**Why:** Persist panel geometry across sessions.

**In Spark:** `src/spark/ui/runtime/EditorLayoutStore.hpp`

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/editor/IEditorPanel.hpp`
- `src/spark/editor/panels/HierarchyPanel.cpp`
- `src/spark/editor/panels/InspectorPanel.cpp`
- `src/spark/editor/panels/EditorDockShell.cpp`

### Step 1 — Read IEditorPanel interface

Panel contract — implement for custom tools (profiler, console).

*IEditorPanel virtual API*

```cpp
// include/spark/editor/IEditorPanel.hpp (lines 10–29)
class IEditorPanel {
public:
    virtual ~IEditorPanel() = default;

    [[nodiscard]] virtual Utf8String GetPanelId() const = 0;
    [[nodiscard]] virtual Utf8String GetDisplayName() const = 0;
    [[nodiscard]] virtual Ui::IUiElement* GetRootElement() noexcept = 0;

    virtual void OnAttach(EditorContext& /*ctx*/) {}
    virtual void OnDetach() {}
    virtual void OnTick(const FrameTiming& /*timing*/, EditorContext& /*ctx*/) {}

    [[nodiscard]] virtual UniquePtr<Ui::IUiElement> ReleaseRootElement() {
        return UniquePtr<Ui::IUiElement>{};
    }
};
```

### Step 2 — Hierarchy tree build and selection

EnsureBuilt creates TreeView; OnTick rebuilds when revision changes.

*OnAttach and revision-based OnTick*

```cpp
// src/spark/editor/panels/HierarchyPanel.cpp (lines 42–61)
void HierarchyPanel::OnAttach(EditorContext& ctx) {
    EnsureBuilt();
    world = ctx.world;
    selection = ctx.selection;
    needsTreeRebuild = true;
}

void HierarchyPanel::OnTick(const FrameTiming& /*timing*/, EditorContext& /*ctx*/) {
    const std::uint32_t revision = ComputeSceneRevision();
    if (!needsTreeRebuild && revision == sceneRevision) {
        return;
    }
    RebuildTree();
    needsTreeRebuild = false;
    sceneRevision = revision;
}
```

### Step 3 — Inspector selection display

Trace name, position, component list formatting for selected object.

*InspectorPanel OnTick body*

```cpp
// src/spark/editor/panels/InspectorPanel.cpp (lines 54–79)
void InspectorPanel::OnTick(const FrameTiming& /*timing*/, EditorContext& /*ctx*/) {
    GameObject* obj = selection->GetPrimary();
    if (obj == nullptr) {
        body->SetText(Utf8String("Select an entity in the Hierarchy."));
        return;
    }
    char buf[512]{};
    std::snprintf(buf, sizeof(buf), "Name: %s\n", obj->GetName().CStr());
    if (TransformComponent* tr = obj->GetComponent<TransformComponent>()) {
        const Vector3 p = tr->GetLocalTransform().translation;
        std::snprintf(buf + std::strlen(buf), sizeof(buf) - std::strlen(buf),
                "Position: %.2f, %.2f, %.2f\n", p.x, p.y, p.z);
    }
    if (obj->GetComponent<MeshComponent>() != nullptr) {
        std::strncat(buf, "MeshComponent\n", sizeof(buf) - std::strlen(buf) - 1);
    }
    body->SetText(Utf8String(buf));
}
```

### Step 4 — Extend inspector with editable Transform

Add FloatField bound to SetTranslation — verify viewport moves same frame.

### Step 5 — Add entity button

Create empty GameObject with Transform — appears in hierarchy tree on next revision bump.

### Step 6 — Project browser listing

Open project folder; confirm mesh/texture files visible; note paths for save resolver callbacks.

### Step 7 — Dock layout restore

Resize sidebar; restart editor; `TryLoadSceneEditorLayout` restores widths.

### Step 8 — Milestone — inspector + hierarchy sync

Select entity; edit transform; viewport and hierarchy stay in sync.

## 5. How This Connects to the Rest of Your Engine

Part **X** post volumes benefit from inspector fields and gizmo overlays. Part **XII** terrain brushes need hierarchy selection of terrain entity. Serialization (Ch 1–2) persists inspector edits on Save. SceneEditor3DDemo (Ch 6) adds placement tools atop this panel foundation.

Replace Label body with FormLayout of FloatField for vec3 — each change invokes TransformComponent setter and increments scene revision.

## 6. Pitfalls and Debugging

- **Inspector edits copy not live component** — bind widgets to actual ECS pointers.
- **Selection stale after delete** — clear EditorSelection when DestroyGameObject runs.
- **Panel draw without input routing** — camera fights UI; ProcessUiCanvasesInput first.
- **Rebuilding tree every frame** — use revision hash like HierarchyPanel.
- **Absolute paths in project browser** — store relative to assets root for snapshots.

**Debugging workflow:** edit transform, Save, grep scene.txt — must match inspector within float format precision.

## 7. Exercises

1. Move entity via inspector Transform display (extend to editable fields); verify viewport motion same frame.
2. Add PointLightComponent in code to selected entity; inspector lists it.
3. Dock panels; restart; layout restores; document layout file path on disk.
4. Delete selected entity from gameplay code; hierarchy clears selection without crash.
5. Stress test: spawn 200 entities; measure hierarchy rebuild ms when revision changes.

## Summary

Editor panels implement IEditorPanel for hierarchy, inspection, and project browse. Spark mutates ECS live and relies on standard scene submit for viewport feedback. Hierarchy revision hashing avoids rebuilding TreeView when the scene graph is static across frames.

---

Next: [Scene Editor Demo](06-scene-editor-demo.md).
