# Main Camera Resolution

> **Build-Along Milestone:** `TryBuildSceneCameraMatrices` selects the highest-priority enabled `CameraComponent` or `Camera2DComponent` and builds `viewProj` for scene submit.

**Spark API note:** ECS camera resolution pairs with **`world.Rendering().SubmitLitWithCamera`** or **`SubmitLit`** after manual matrix build. Legacy helpers such as **`SubmitStandardLitSceneFromWorldWithCamera`** and **`FillStandardLitSceneFromWorld`** delegate to the same **`LitSceneSubmitter`** pipeline.

**You will learn:**

- Attach perspective cameras to ECS `GameObject` transforms.
- Resolve main camera by priority when multiple cameras exist.
- Arbitrate between 2D and 3D cameras in hybrid scenes.
- Use auto-submit helpers that fill `SceneRenderParams` from the world.

## 1. The Problem We Are Solving

Demos can manually assign `params.viewProjection` from `FlyCamera` or `CharacterCameraRig`. Production scenes place cameras on entities with transforms, FOV, clip planes, and **priority**. At submit time Spark must answer: **which camera renders this frame?**

`TryResolveMainCamera`, `TryResolveMainCamera2D`, and `TryBuildSceneCameraMatrices` implement that policy. `SubmitStandardLitSceneFromWorldWithCamera` combines resolution with mesh/light gather — the path `Platformer2DDemo` and ECS 3D scenes use.

### Which camera renders? (beginner)

Scenes can have **many** cameras — minimap cam, cinematic cam, player cam. Only **one** supplies `viewProj` each frame.

```text
Entity "PlayerCamera"  → CameraComponent, priority = 10
Entity "CutsceneCam"   → CameraComponent, priority = 100  ← wins when enabled
```

**Resolution rules (simplified):**

1. Collect all enabled `CameraComponent` / `Camera2DComponent`.
2. Pick **highest priority** number.
3. Build `viewProj` from that entity's transform + FOV/ortho settings.

**Hybrid 2D/3D:** if both exist, Spark's resolver picks based on scene mode — platformer uses `Camera2DComponent`; 3D world uses `CameraComponent`.

**Manual path (early demos):** `FlyCamera` fills `params.viewProjection` directly — no ECS. Production path: attach camera to `GameObject`, let `TryBuildSceneCameraMatrices` find it at submit.

**Debug:** log resolved camera entity name — if wrong camera wins, check `priority` and `enabled` flag.

## 1b. Concept Deep Dive — TryBuildSceneCameraMatrices Resolution

### ECS CameraComponent fields

```cpp
// include/spark/ecs/components/camera/CameraComponent.hpp
class CameraComponent final : public GameComponent {
    CameraProjectionMode projectionMode = CameraProjectionMode::Perspective;
    float fovYDegrees = 60.0F;
    float nearPlane = 0.12F;
    float farPlane = 400.0F;
    std::int32_t priority = 0;
    bool enabled = true;
    [[nodiscard]] Matrix4 ViewMatrix(const GameObject& owner) const noexcept;
};
```

Higher `priority` wins when multiple enabled cameras exist. World pose comes from the owner's `TransformComponent` — the component stores projection parameters only.

### TryBuildSceneCameraMatrices

```cpp
// src/spark/scene/submit/SceneSubmit.cpp
bool TryFillSceneCameraFromWorld(
        const GameWorld& world, float fbW, float fbH, SceneRenderParams& outParams) {
    SceneCameraMatrices cam{};
    if (!TryBuildSceneCameraMatrices(world, fbW, fbH, cam)) return false;
    outParams.viewProjection = cam.viewProjection;
    outParams.cameraPositionWorld = cam.positionWorld;
    outParams.particleCameraRight = cam.rightWorld;
    outParams.particleCameraUp = cam.upWorld;
    return true;
}
```

Failure returns false — caller must fall back to manual matrices or skip submit. Particle billboards use `rightWorld`/`upWorld` extracted during resolution.

### Hybrid 2D vs 3D arbitration

`TryResolveMainCamera2D` competes with `TryResolveMainCamera` — platformer scenes with `Camera2DComponent` use orthographic `ViewProjection(fbW, fbH)`; pure 3D scenes use perspective FOV from `CameraComponent`. Same output fields on `SceneRenderParams`.

### One-call production submit

`SubmitStandardLitSceneFromWorldWithCamera` combines camera resolution, mesh gather, light gather, and texture dedup — the standard path once you outgrow manual `FlyCamera` assignment in demo code.

## 2. Core Concepts

### CameraComponent (3D ECS)

```cpp
class CameraComponent {
    CameraProjectionMode projectionMode;  // Perspective | Orthographic
    float verticalFovDegrees;
    float nearClip, farClip;
    float orthographicHalfExtentY;
    std::int32_t priority;
    bool enabled;
};
```

View from owner world matrix:

```cpp
Matrix4 CameraComponent::ViewMatrix(const GameObject& owner) const {
    Matrix4 world = owner.GetWorldMatrix();
    Matrix4 view;
    world.TryInvert(view);
    return view;
}
```

Projection from aspect:

```cpp
Matrix4 CameraComponent::ProjectionMatrix(float aspect) const {
    if (projectionMode == Perspective)
        return Matrix4::PerspectiveVulkan(DegreesToRadians(verticalFovDegrees), aspect, nearClip, farClip);
    return Matrix4::OrthographicVulkan(...);
}

Matrix4 ViewProjection(const GameObject& owner, float aspect) const {
    return ProjectionMatrix(aspect) * ViewMatrix(owner);
}
```

Camera **position** is transform translation; **orientation** is transform rotation — rigs (Chapter 6) update transform before resolve.

### Priority resolution (3D)

```cpp
bool TryResolveMainCamera(const GameWorld& world, ResolvedCamera& out) {
    world.ForEachActiveGameObject([&](GameObject* o) {
        const CameraComponent* cam = o->GetComponent<CameraComponent>();
        if (!cam || !cam->IsEnabled()) return;
        if (!found || cam->GetPriority() > bestPriority) {
            best = o; bestCam = cam; bestPriority = cam->GetPriority();
        }
    });
    return found;
}
```

Higher `priority` wins. Ties: first encountered in iteration — set unique priorities in content.

### Camera2DComponent (2D ECS)

Parallel resolver for orthographic XY cameras with `halfExtentY`, priority, enabled flag. `ViewProjection(owner, fbW, fbH)` delegates to `Camera2D` math (Chapter 9).

### 2D vs 3D arbitration

When **both** exist in scene:

```cpp
if (has2d && has3d) {
    if (cam2d.priority >= cam3d.priority)
        use 2D matrices;
    else
        use 3D matrices;
}
```

Only one path active per frame — hybrid games use separate scenes or disable unused camera.

### SceneCameraMatrices output

```cpp
struct SceneCameraMatrices {
    Matrix4 viewProjection;
    Vector3 positionWorld;
    Vector3 rightWorld;  // billboard basis
    Vector3 upWorld;
};
```

Billboard basis from camera for particles/sprites facing camera in 3D.

### TryFillSceneCameraFromWorld

```cpp
bool TryFillSceneCameraFromWorld(const GameWorld& world, int fbW, int fbH, SceneRenderParams& params) {
    SceneCameraMatrices mats;
    if (!TryBuildSceneCameraMatrices(world, fbW, fbH, mats)) return false;
    params.viewProjection = mats.viewProjection;
    params.cameraPositionWorld = mats.positionWorld;
  // particle basis fields...
    return true;
}
```

Called before draw gather — camera must exist or submit aborts.

### SubmitStandardLitSceneFromWorldWithCamera

```cpp
// Preferred: camera entity + options bundle
LitSceneSubmitOptions opts{};
opts.WithDirectionalLight(sunDir, sunColor, sunIntensity).WithAmbient(ambientColor);
if (!world.Rendering().SubmitLitWithCamera(context, opts)) return false;

// Legacy one-shot (still LitSceneSubmitter inside):
// SubmitStandardLitSceneFromWorldWithCamera(world, context, ...);
```

Resolve camera → build draws → publish snapshot. `Platformer2DDemo::OnRender` uses the `WorldRendering` path in current Spark.

### Manual vs auto paths

| Path | When |
|------|------|
| Manual `viewProj` in demo | Learning, `FlyCamera`, `CharacterCameraRig` |
| `TryFillSceneCameraFromWorld` | ECS levels with camera entities |
| Full submit helper | 2D platformer, standard lit scenes |

Both end at same `SceneRenderParams` fields — GPU path identical (Chapter 10).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Priority not tags

**Concept:** 3D resolution

**Decision:** Integer priority, higher wins

**Why:** Simple split-screen future: raise gameplay cam priority over debug cam.

**In Spark:** `TryResolveMainCamera`.

### View = inverse world matrix

**Concept:** CameraComponent view

**Decision:** No separate view storage

**Why:** Transform is source of truth for rigs and animation.

**In Spark:** `CameraComponent::ViewMatrix`.

### 2D/3D explicit arbitration

**Concept:** Hybrid scenes

**Decision:** Compare priorities across types

**Why:** Avoid undefined behavior when level accidentally has both.

**In Spark:** `TryBuildSceneCameraMatrices`.

### Enabled flag per component

**Concept:** Multiple cameras

**Decision:** Disabled cams skipped in scan

**Why:** Cutscene camera entity can exist but stay off until triggered.

**In Spark:** `IsEnabled()` checks.

### Billboard basis in resolve

**Concept:** SceneCameraMatrices

**Decision:** `BillboardBasisWorld` on component

**Why:** Particles/sprites need camera right/up without recomputing matrix.

**In Spark:** `Camera.cpp` out fields.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/ecs/components/camera/CameraComponent.hpp`
- `src/spark/ecs/components/camera/CameraComponent.cpp`
- `include/spark/scene/Camera.hpp`
- `src/spark/scene/Camera.cpp`
- `src/spark/scene/SceneSubmit.cpp` (`SubmitStandardLitSceneFromWorldWithCamera`)

### Step 1 — Single camera entity

Empty scene + camera GO at (0,2,5) looking at origin — resolve succeeds.

### Step 2 — Priority duel

Two cameras, priorities 0 and 10 — higher renders.

### Step 3 — Disable loser

Set enabled false on high priority — lower takes over.

### Step 4 — Auto submit

Call submit helper from `OnRender` — no manual matrix math in game code.

### Step 5 — Missing camera

Remove all cameras — `TryFill` returns false; handle gracefully (skip draw or assert in dev).

## 5. How This Connects to the Rest of Your Engine

Chapter **6** rigs write camera transform before resolve. Chapter **9** 2D path uses parallel resolver. Chapter **10** uploads resolved matrices. Part **VI** animation moves targets rigs follow.

Main camera resolution is the **bridge** from ECS scene graph to render snapshot.

## 6. Pitfalls and Debugging

- **Wrong camera** — equal priorities; ambiguous iteration order.
- **Jump cut** — enabling second camera mid-frame without blend.
- **Aspect stale** — resolve must use current framebuffer size each frame.
- **View not updated** — rig tick order after camera resolve — move resolve to render phase.
- **2D game uses 3D cam** — 3D priority higher accidentally.

## 7. Exercises

1. Add debug overlay listing active camera name and priority.
2. Implement blend between two cameras over 1 second (stretch goal).
3. Trace `Platformer2DDemo` render path to `SetSceneRenderParams`.
4. What happens with zero cameras? Trace return value handling.
5. Design priority scheme: gameplay=100, UI debug=50, cinematic=200.

## Summary

Spark resolves the main camera by scanning enabled `CameraComponent` and `Camera2DComponent` entities and choosing highest priority, with explicit 2D vs 3D arbitration. `CameraComponent` builds view from inverse world matrix and projection from FOV or ortho settings. `TryFillSceneCameraFromWorld` and submit helpers populate `SceneRenderParams` for the standard render path.

---

Next: [2D Camera](09-camera-2d.md).
