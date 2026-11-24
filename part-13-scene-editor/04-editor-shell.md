# Editor Application

> **Build-Along Milestone:** Editor viewport renders bootstrap scene with working fly camera and panel shell.

**Spark API note:** Editor viewport rendering uses **`world.Rendering().SubmitLit`** / **`FillLit`** like runtime demos. This chapter still references **`FillStandardLitSceneFromWorld`** when diffing against older snippets — that name delegates to **`LitSceneSubmitter`**.

**You will learn:**

- Structure an editor shell around the existing engine Update/Render loop.
- Bootstrap a default scene for immediate viewport feedback.
- Share EditorContext across panels without global state.
- Use the same `world.Rendering().SubmitLit` submit path as runtime demos.
- Apply dock viewport scissor so 3D scene draws only in the central panel region.

## 1. The Problem We Are Solving

Designers need a **viewport + hierarchy + inspector** without maintaining a second tech stack. Spark's `EditorApplication` bootstraps a default lit scene, runs the normal render path, and hosts dock panels through the same UI canvas system as Part XI overlays.

You will trace bootstrap, confirm submit matches ThreeDDemo, and wire EditorContext so Chapter 5 panels can list and edit entities live.

Milestone: launch editor — HDR lit viewport with ground, sun, sample cube, and working WASD + RMB fly camera before save menu wiring.

## 1b. Concept Deep Dive — EditorApplication Update Loop

### Same engine loop, extra UI canvases

`EditorApplication` does not fork a second renderer. It implements the same `OnUpdate` / `OnRender` hooks as ThreeDDemo, layering dock panels and selection on top:

```cpp
// src/spark/editor/EditorApplication.cpp
void EditorApplication::OnUpdate(const FrameTiming& timing, Scene& scene, IEngineContext& engineContext) {
    context.world = &scene.GetWorld();
    context.selection = &selection;
    context.viewport = &viewport;
    // ... wire EditorContext for panels ...

    ProcessUiCanvasesInput(scene, engineContext.GetInput(), fbW, fbH, ...);
    PumpSceneLoads(scene.GetWorld());

    const bool editingEnabled = !playSession.IsActive();
    if (editingEnabled) {
        HandleEditModeInput(engineContext, scene.GetWorld());
    }

    viewport.Simulate(timing, scene, engineContext, selection, worldViewport, ...);
    TickPanels(timing, scene, engineContext);
}
```

**Key ordering:** UI input first (panels capture pointer), then async scene pump, then viewport camera, then panel ticks.

### EditorContext replaces globals

Panels receive a shared `EditorContext` each frame — world pointer, selection, command stack, viewport, content model. No static singletons holding stale entity copies; inspector reads live `TransformComponent` through this context.

### Submit path matches runtime demos

`OnRender` builds `SceneRenderParams` via the same `world.Rendering().SubmitLit` path as ThreeDDemo (legacy `FillStandardLitSceneFromWorld`). The dock viewport supplies scissor rect so 3D draws only in the central panel region — HDR, shadows, and tonemap unchanged from Part X.

## 2. Core Concepts

### Editor vs runtime game loop

The editor runs the **same** engine Update/Render pipeline as ThreeDDemo — not a separate WPF or Electron shell. `EditorApplication` implements scene lifecycle hooks: `OnAttach` bootstraps content, `OnUpdate` drives camera and panels, `OnRender` builds `SceneRenderParams` and submits. No second Vulkan device, no duplicate renderer — only additional UI canvases and selection logic layered on top.

### Bootstrap default scene

Designers expect immediate viewport feedback. `BootstrapDefaultScene` creates Ground (registered mesh + MeshComponent ground plane slot), Sun (point light with elevated transform), and SampleCube (material tuning target). Matches runtime demo conventions so lighting and shadows look familiar before the user loads a saved file.

Bootstrap entities are typically **excluded** from save predicates — only user-placed props persist.

### EditorContext shared state

Panels and tools read/write through `EditorContext`: pointer to `GameWorld`, `EditorSelection`, project paths, workspace mode, status line buffer. Avoid globals — multiple editor modes swap context fields without recompiling panels.

Selection drives inspector and highlight pass; project root drives asset browser listing under `SPARK_BUILD_ASSETS_DIR` or user-opened folder.

### Viewport camera vs authored camera

Editor fly camera (`UpdateViewportCamera`) is independent of serialized `CameraComponent` entities. WASD + RMB look navigates the **view**; game cameras in the scene render only when play mode runs full game logic. Editor camera fills viewProj in `OnRender` — not saved in scene files.

F1 toggles cursor capture; scroll zoom respects `UiConsumesGamePointer` so panels eat wheel events first.

### Submit path unchanged

`OnRender` calls `world.Rendering().SubmitLit` (or legacy `FillStandardLitSceneFromWorld`) → populates `SceneRenderParams` → engine presents. Same function family as ThreeDDemo and SceneEditor3DDemo runtime viewport. World viewport scissor from dock layout clips 3D rendering to the central panel region — UI draws afterward via `PaintUiCanvases`.

Validates architecture: editor is a **client** of scene submit, not a fork of VulkanRenderer.

### Selection highlight feedback

`HighlightSelection` temporarily boosts emissive on selected entity's `MaterialComponent` — visible border glow without a separate outline pass. Clears previous highlight when selection changes. Editing transforms in inspector mutates ECS; next OnRender submit shows movement same frame.

### Editor tick vs game tick

```text
OnUpdate: UI input → TickPanels → fly camera
OnRender: world.Rendering().SubmitLit → scissor → PaintUiCanvases
```

| Mode | Simulate | Transform authority |
|------|----------|---------------------|
| Edit | off | inspector |
| Play | on | physics integrator |

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### EditorApplication scene hooks

**Concept:** Editor vs runtime game loop

**Decision:** EditorApplication scene hooks

**Why:** Reuses ISceneApplication pattern from demos — one engine binary hosts editor mode.

**In Spark:** `include/spark/editor/EditorApplication.hpp`

### BootstrapDefaultScene on OnAttach

**Concept:** Bootstrap default scene

**Decision:** BootstrapDefaultScene on OnAttach

**Why:** Immediate lit viewport; ground extent matches kSceneGroundHalfExtent.

**In Spark:** `EditorApplication::BootstrapDefaultScene`

### EditorContext struct passed to panels

**Concept:** EditorContext shared state

**Decision:** EditorContext struct passed to panels

**Why:** World, selection, project, statusLine — no singleton GameWorld access.

**In Spark:** `include/spark/editor/EditorContext.hpp`

### Editor-owned ViewportCamera

**Concept:** Viewport camera vs authored camera

**Decision:** Editor-owned ViewportCamera

**Why:** Navigation independent of saved CameraComponent; expected for DCC-style editors.

**In Spark:** `UpdateViewportCamera` in EditorApplication.cpp

### world.Rendering().SubmitLit in OnRender

**Concept:** Submit path unchanged

**Decision:** `world.Rendering().SubmitLit` in OnRender (legacy `FillStandardLitSceneFromWorld`)

**Why:** Identical lit gather as runtime; dock scissor for panel layout only.

**In Spark:** `EditorApplication::OnRender`

### Emissive bump on MaterialComponent

**Concept:** Selection highlight feedback

**Decision:** Emissive bump on MaterialComponent

**Why:** Cheap visual feedback without extra render pass.

**In Spark:** `HighlightSelection`

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/editor/EditorApplication.cpp`
- `include/spark/editor/EditorApplication.hpp`
- `include/spark/editor/EditorContext.hpp`

### Step 1 — Read BootstrapDefaultScene

Ground registration, sun point light, sample cube with material.

*BootstrapDefaultScene entity creation*

```cpp
// src/spark/editor/EditorApplication.cpp (lines 39–70)
void EditorApplication::BootstrapDefaultScene(GameWorld& world) {
    groundMesh = MakeShared<Mesh>(Utf8String("EditorGround"));
    *groundMesh = Mesh::CreateGroundPlane(kSceneGroundHalfExtent);
    world.RegisterMesh(groundMesh, "spark/editor/ground");

    GameObject* ground = world.CreateGameObject();
    ground->GetName() = Utf8String("Ground");
    ground->AddComponent<TransformComponent>();
    ground->AddComponent<MeshComponent>(
            groundMesh, SceneMeshSlot::GroundPlane, Vector3{0.45F, 0.48F, 0.5F});

    GameObject* sun = world.CreateGameObject();
    sun->GetName() = Utf8String("Sun");
    TransformComponent* sunTr = sun->AddComponent<TransformComponent>();
    sunTr->SetTranslation({18.0F, 28.0F, 12.0F});
    sun->AddComponent<PointLightComponent>(Vector3{1.0F, 0.96F, 0.9F}, 2.4F, 120.0F);

    GameObject* sample = world.CreateGameObject();
    sample->GetName() = Utf8String("SampleCube");
    sample->AddComponent<TransformComponent>()->SetTranslation({0.0F, 0.5F, 0.0F});
    sample->AddComponent<MeshComponent>(SharedPtr<Mesh>{}, SceneMeshSlot::UnitCube, Vector3{0.72F, 0.35F, 0.28F});
}
```

### Step 2 — Trace OnAttach sequence

Spatial partition, fonts, bootstrap, BuildEditorUi — order matters for first-frame render.

*OnAttach hook*

```cpp
// src/spark/editor/EditorApplication.cpp (lines 191–198)
void EditorApplication::OnAttach(Scene& scene, IEngineContext& /*context*/) {
    scene.SetSpatialPartitionKind(ScenePartitionKind::BoundingVolumeHierarchy);
    MountEditorUiFonts(scene.GetWorld());
    BootstrapDefaultScene(scene.GetWorld());
    BuildEditorUi(scene.GetWorld());
    viewportCamera.position = {8.0F, 6.0F, 14.0F};
    viewportCamera.SnapLookAt({0.0F, 0.0F, 0.0F});
}
```

### Step 3 — BuildEditorUi and panel wiring

Creates Hierarchy, Inspector, Project panels; binds EditorContext; loads dock layout.

*BuildEditorUi panel setup*

```cpp
// src/spark/editor/EditorApplication.cpp (lines 68–117)
void EditorApplication::BuildEditorUi(GameWorld& world) {
    hierarchyPanel = MakeUnique<HierarchyPanel>();
    inspectorPanel = MakeUnique<InspectorPanel>();
    projectPanel = MakeUnique<ProjectBrowserPanel>();

    context.world = &world;
    context.selection = &selection;
    context.project = &project;
    context.statusLine = statusLine;

    for (IEditorPanel* panel : {hierarchyPanel.Get(), inspectorPanel.Get(), projectPanel.Get()}) {
        panel->OnAttach(context);
    }

    Ui::TryLoadSceneEditorLayout(layout);
    dock.SetPanels(
            hierarchyPanel->ReleaseRootElement(),
            projectPanel->ReleaseRootElement(),
            inspectorPanel->ReleaseRootElement());
    guiCanvas = guiCanvasObject->AddComponent<UiCanvasComponent>();
    guiCanvas->SetTheme(Ui::UiTheme::SceneEditorDark());
    guiCanvas->SetRoot(dock.ReleaseRootElement());
}
```

### Step 4 — Launch editor target

Build and run Spark editor application — confirm viewport shows ground, sun lighting, sample cube.

### Step 5 — Study OnUpdate input routing

ProcessUiCanvasesInput before fly camera — RMB must not scroll hierarchy.

*OnUpdate tick order*

```cpp
// src/spark/editor/EditorApplication.cpp (lines 227–241)
void EditorApplication::OnUpdate(const FrameTiming& timing, Scene& scene, IEngineContext& context) {
    ProcessUiCanvasesInput(scene, context.GetInput(), fbW, fbH, contentScaleX, contentScaleY);
    UpdateViewportCamera(timing, context);
    TickPanels(timing, scene, context);
    if (fpsText != nullptr) {
        fpsText->SetText(statusLine);
    }
}
```

### Step 6 — Confirm OnRender submit

world.Rendering().SubmitLit, viewport scissor from dock, PaintUiCanvases for editor chrome.

*OnRender scene submit + UI paint*

```cpp
// src/spark/editor/EditorApplication.cpp (lines 243–290)
void EditorApplication::OnRender(Scene& scene, IEngineContext& context) {
    HighlightSelection(scene);
    LitSceneSubmitOptions opts{}; /* lights, culling, … */
    world.Rendering().SubmitLit(context, viewProj, viewportCamera.position, opts);
    // Legacy excerpt: FillStandardLitSceneFromWorld(scene.GetWorld(), context, viewProj, ...);
    const Ui::Rect worldViewport = dock.GetWorldViewportRect();
    if (worldViewport.width > 1.0F && worldViewport.height > 1.0F) {
        params.worldViewportScissorEnabled = true;
        params.worldViewportScissorX = worldViewport.x;
        // ...
    }
    PaintUiCanvases(scene, params, fbW, fbH);
}
```

### Step 7 — Layout persistence

`TryLoadSceneEditorLayout` restores dock widths — verify panel arrangement survives restart.

### Step 8 — Milestone — lit bootstrap viewport

HDR lit viewport with shadows before save menu wiring.

## 5. How This Connects to the Rest of Your Engine

Chapter 5 panels consume EditorContext. Part **XI** ImGui/UI backend shares input routing with editor. Part **XVI** swapchain recreate must handle docked viewport resize. SceneEditor3DDemo (Chapter 6) extends this shell with placement tools and save/load menus.

Font mounting precedes BuildEditorUi; missing fonts show tofu in panel chrome. OnDetach destroys EditorGui canvas entity and resets panels — prevents duplicate canvases on hot reload.

## 6. Pitfalls and Debugging

- **Second Vulkan device for editor** — use same renderer instance as games.
- **Editing SceneRenderParams directly** — edit ECS; submit rebuilds snapshot each frame.
- **Forgetting layout save** — panel arrangement resets every launch.
- **Camera serialized by mistake** — editor fly camera is not gameplay state.
- **UI not consuming pointer** — camera fights panel clicks; call ProcessUiCanvasesInput first.

**Debugging workflow:** RenderDoc draws full framebuffer under panels — scissor not applied; check dock `GetWorldViewportRect`. Zero scissor size shows black 3D region with visible UI — layout load issue.

## 7. Exercises

1. Launch editor; verify ground + sun + cube with shadows; capture RenderDoc showing HDR scene pass draws.
2. Toggle F1 cursor capture; confirm WASD stops moving viewport when typing focus in panel.
3. Resize dock panels; restart; verify EditorLayoutStore restored sidebar width within 10px.
4. Select SampleCube; confirm emissive highlight.
5. Diff OnRender against ThreeDDemo — list only scissor and UI paint differences.

## Summary

EditorApplication wraps the engine loop with bootstrap scene, EditorContext, and shared lit submit. Rendering uses `world.Rendering().SubmitLit` with dock viewport scissor — same pipeline as games (legacy `FillStandardLitSceneFromWorld` delegates to `LitSceneSubmitter`).

---

Next: [Editor Panels](05-editor-panels.md).
