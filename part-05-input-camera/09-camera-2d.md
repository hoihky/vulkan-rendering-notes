# 2D Camera

> **Build-Along Milestone:** `Camera2D::ViewProjection` maps world XY to clip with `OrthographicVulkan`; `Platformer2DDemo` uses `Camera2DRigComponent` for bounded follow.

**You will learn:**

- Build orthographic view and projection for side-view and top-down 2D.
- Rotate the 2D camera around Z for slight world tilt.
- Wire ECS `Camera2DComponent` and follow rigs with level bounds.
- Understand framebuffer Y convention vs world +Y up.

## 1. The Problem We Are Solving

2D games still need a **camera** — position, optional rotation, zoom (`halfExtentY`), and follow behavior. Spark does not use a separate renderer; the same Vulkan path draws sprites with an orthographic `viewProj` from a `Camera2D` or `Camera2DComponent`.

This chapter covers standalone `Camera2D` math, ECS wrapper, and `Camera2DRigComponent` modes — the 2D parallel to Chapters 5–8 in 3D.

### 2D camera in plain language (beginner)

2D games draw on the **XY plane** (side-scroller) or **XZ** (top-down). The camera still needs:

- **Position** — which part of the world is centered on screen
- **Zoom** — `halfExtentY` controls how many world units fit vertically (bigger = zoomed out)
- **Rotation** — usually 0; slight Z rotation for isometric tilt

**Orthographic projection** means parallel lines stay parallel — no perspective foreshortening. A coin at the back of the scene is the same size as one in front (correct for Mario-style platformers).

```text
World units: player at (100, 50)
Camera at (100, 50) → player centered
Camera at (90, 50)  → player appears right of center (camera lag behind)
```

**Follow rig with bounds:** `Camera2DRigComponent` clamps camera so you never show void outside the level — `minX/maxX` keep the view inside designed tilemap edges.

**Y-flip note:** Vulkan clip space Y may point down; Spark's `OrthographicVulkan` matches sprite batching — if sprites appear upside down, check projection sign vs texture V coordinate (Part XI).

## 1b. Concept Deep Dive — Camera2D Orthographic Path

### Standalone struct for 2D math

```cpp
// include/spark/scene/camera/Camera2D.hpp
struct Camera2D {
    Vector3 position{};
    float rotationRad = 0.0F;
    float halfExtentY = 5.0F;
    float clipNearZ = -500.0F;
    float clipFarZ = 500.0F;
    [[nodiscard]] Matrix4 ViewProjection(float fbW, float fbH) const noexcept;
};
```

`halfExtentY` is half the visible world height — halving it zooms in. Width follows aspect: `halfExtentX = halfExtentY * (fbW / fbH)`.

### ViewProjection composition

`ViewMatrix()` applies inverse pan + Z rotation; `ViewProjection` multiplies `OrthographicVulkan(halfExtentY, aspect, clipNear, clipFar) * view`. Result goes directly to `SceneRenderParams::viewProjection` — same field 3D cameras use.

### ECS Camera2DComponent + rig

`Camera2DComponent` mirrors projection fields on a `GameObject`. `Camera2DRigComponent` adds follow modes and level bounds — `Platformer2DDemo` clamps camera X/Y so the view never shows void outside the tilemap. Rig ticks in ECS update; submit reads resolved component matrices.

### Sprite sort interaction

Top-down 2D scenes set `SceneSpriteSortMode::SortOrderThenWorldY` on submit so characters with equal sort order occlude by world Y (+Y-up). The orthographic camera's `position` and `halfExtentY` define which slice of world Y is visible for that occlusion test.

## 2. Core Concepts

### Camera2D state

```cpp
struct Camera2D {
    Vector3 position;      // XY gameplay, Z often 0
    float rotationRad;     // rotation around Z
    float halfExtentY;     // half-height of ortho frustum in world units
    float clipNearZ, clipFarZ;
};
```

**halfExtentY** is the primary zoom control — smaller = zoomed in (fewer world units visible vertically).

### View matrix (2D)

```cpp
Matrix4 Camera2D::ViewMatrix() const {
    Quaternion q = Quaternion::FromAxisAngle(UnitZ, -rotationRad);
    Matrix4 rot = Matrix4::Rotation(q);
    Matrix4 trans = Matrix4::Translation({-position.x, -position.y, -position.z});
    return rot * trans;
}
```

Negated rotation and translation — standard inverse of camera transform. Gameplay moves `position`; view moves opposite.

### Orthographic projection

```cpp
Matrix4 Camera2D::ViewProjection(float fbW, float fbH) const {
    const float aspect = fbW / fbH;
    const float halfW = halfExtentY * aspect;
    const float halfH = halfExtentY;
    Matrix4 proj = Matrix4::OrthographicVulkan(-halfW, halfW, -halfH, halfH, clipNearZ, clipFarZ);
    return proj * ViewMatrix();
}
```

Symmetric frustum centered on camera position after view transform.

### World +Y up vs framebuffer Y down

Spark documents: world **+Y is up**, framebuffer **Y grows down**. `OrthographicVulkan` maps world +Y to increasing framebuffer Y — picking and `gl_FragCoord` align without extra NDC flip in 2D sprite shaders.

When porting art from tools with inverted Y, fix at import — not by negating projection ad hoc.

### Camera2DComponent (ECS)

Wraps parameters on `GameObject` transform:

```cpp
Camera2D Camera2DComponent::BuildCamera2D(const GameObject& owner) const {
    Camera2D cam;
    cam.position = owner.GetWorldMatrix().TranslationVector();
    cam.rotationRad = owner.GetEulerZ();  // simplified — see implementation
    cam.halfExtentY = halfExtentY;
    return cam;
}
```

`ViewProjection(owner, fbW, fbH)` builds matrices from component + transform.

### Camera2DRigComponent modes

| Mode | Behavior |
|------|----------|
| **Manual** | Designer sets transform only |
| **FollowTarget** | Smooth follow player with offset |
| **BoundedFollow** | Follow + clamp to axis-aligned bounds |

```cpp
// Platformer2DDemo setup
cameraRig->SetMode(Camera2DRigMode::BoundedFollow);
cameraRig->SetTarget(playerObject);
cameraRig->SetTargetOffset({0, 1.48f, 0});
cameraRig->SetUseBounds(true);
cameraRig->SetBoundsMin({...});
cameraRig->SetBoundsMax({...});
```

Bounds prevent showing void outside tilemap — common in platformers.

### Velocity look-ahead (optional)

Rig can offset camera slightly in movement direction using `Rigidbody2DComponent` velocity — reduces "leading edge" empty space when running. Tune strength for feel vs nausea.

### 2D main camera resolve

`TryResolveMainCamera2D` — same priority scan as 3D (Chapter 8). `TryBuildSceneCamera2DMatrices` fills `SceneCameraMatrices` with orthographic `viewProj` and billboard basis for sprite facing.

### Sprites and viewProj

2D sprite pass uses same `SceneRenderParams::viewProjection` — vertex shader transforms quad corners to clip. UI may use separate ortho (Part XI) — do not mix UI and world cameras without clear pass split.

### Zoom dynamics

Change `halfExtentY` over time for zoom punch — lerp in rig tick or script. Aspect from framebuffer keeps circles circular on resize.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### halfExtentY zoom metaphor

**Concept:** Camera2D state

**Decision:** Vertical half-extent not "zoom factor"

**Why:** Directly matches ortho top/bottom planes; width follows aspect.

**In Spark:** `Camera2D.hpp`.

### Z rotation only

**Concept:** View matrix

**Decision:** 2D camera rotates around Z axis

**Why:** Side-scrollers rarely need X/Y tilt; keeps math in XY plane.

**In Spark:** `ViewMatrix` axis angle.

### OrthographicVulkan for 2D

**Concept:** Projection

**Decision:** Same Vulkan Y convention as 3D

**Why:** One viewport/scissor pipeline; sprites respect flip.

**In Spark:** `Camera2D.cpp`.

### Bounded follow for tilemaps

**Concept:** Camera2DRig

**Decision:** Clamp camera center after smooth follow

**Why:** Tilemaps have finite extent; avoid black margins.

**In Spark:** `Platformer2DDemo`.

### Shared SceneRenderParams path

**Concept:** Sprites

**Decision:** 2D games use same submit + UBO as 3D

**Why:** One renderer; only matrices change.

**In Spark:** `SubmitStandardLitSceneFromWorldWithCamera`.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/scene/Camera2D.hpp`
- `src/spark/scene/Camera2D.cpp`
- `include/spark/ecs/components/camera/Camera2DComponent.hpp`
- `include/spark/ecs/components/camera/Camera2DRigComponent.hpp`
- `src/spark/demo/Platformer2DDemo.cpp`

### Step 1 — Fixed camera

Set position (0,0), halfExtentY 8.5 — see level framing.

### Step 2 — Follow player

Enable FollowTarget — camera lags behind movement.

### Step 3 — Bounds

Walk to level edge — camera stops, player approaches border.

### Step 4 — Zoom

Animate halfExtentY 8.5 → 4 — closer view.

### Step 5 — Rotate slightly

Small `rotationRad` — parallax layers may need adjustment (Part VII).

### Step 6 — Resolve path

Confirm `OnRender` uses submit helper — matrices from `Camera2DComponent`, not manual.

## 5. How This Connects to the Rest of Your Engine

Chapter **8** resolves 2D vs 3D camera. Chapter **10** uploads same UBO fields. Part **VII** sprites/particles use billboard basis. Part **XI** UI is separate ortho stack.

2D camera is **simpler math** but same snapshot contract as 3D.

## 6. Pitfalls and Debugging

- **Stretch on resize** — aspect not updated from framebuffer each frame.
- **Upside-down sprites** — art UV vs world Y — fix assets or sprite shader flip.
- **Bounds too tight** — player hits invisible wall before tile edge.
- **Z fighting** — clip near/far too tight for layered sprites.
- **Wrong camera in hybrid scene** — 3D cam higher priority steals 2D level.

## 7. Exercises

1. Implement smooth zoom lerp on scroll wheel changing `halfExtentY`.
2. Draw debug rect for `boundsMin/Max` in world space.
3. Calculate world units visible at halfExtentY=10, aspect 16:9.
4. Compare 2D view matrix to 3D LookAt looking down −Z — relationship?
5. Add screen shake by offsetting position per frame with noise.

## Summary

`Camera2D` provides orthographic view-projection with Z-rotation and `halfExtentY` zoom. ECS components and follow rigs track targets with optional level bounds. `OrthographicVulkan` matches Vulkan and framebuffer conventions. Platformer demos auto-submit through main camera resolution — same `SceneRenderParams` path as 3D.

---

Next: [Camera to UBO](10-camera-ubo.md).
