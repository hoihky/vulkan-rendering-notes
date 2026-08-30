# Fly Camera

> **Build-Along Milestone:** `FlyCamera` moves with WASD and mouse look; `ThreeDDemo` toggles cursor capture with F1 and fills `viewProj` each frame.

**You will learn:**

- Implement yaw/pitch FPS orientation with clamped pitch.
- Map WASD and Space/Shift to world movement along camera basis.
- Build a view matrix with `LookAt` and gimbal-safe up vector.
- Integrate fly camera with `IInput` and the engine loop.

## 1. The Problem We Are Solving

Editors, debug views, and early 3D demos need a **free-flying camera** — no physics, no target entity, immediate response. `FlyCamera` is Spark's minimal FPS rig: a struct with position, yaw, pitch, and speeds — not an ECS component.

It expects:

- `PollEvents` then `BeginFrame` before use (Chapter 3)
- Cursor captured for mouse look (optional but typical)
- `PerspectiveVulkan` applied outside the struct (Chapter 4)

This chapter implements movement, look, and view matrix — the pattern most students recognize from game dev tutorials, wired through Spark's `IInput` abstraction.

### Fly camera controls explained (beginner)

A **fly camera** is like spectator mode in an editor — you are not walking on the ground mesh, you float through the world.

**Yaw and pitch:**

- **Yaw** — turn left/right (rotate around world up, usually +Y).
- **Pitch** — look up/down (clamp to ±89° so you never flip upside down).

```text
Mouse moved right → yaw increases → look toward +X
Mouse moved up    → pitch increases → look toward +Y
```

**WASD movement** uses the camera's **forward/right** vectors, not world axes blindly:

```text
W → move along forward (where you look)
S → backward
A/D → strafe left/right
Space/Shift → up/down in world space (optional)
```

**View matrix:** `LookAt(eye, eye + forward, up)` — tells the GPU "place the camera here, look there." Combined with perspective projection (Chapter 4) → `viewProj` for shaders.

**F1 cursor capture:** without capture, mouse hits screen edge and stops turning — capture hides cursor and re-centers each frame for infinite look.

## 1b. Concept Deep Dive — FlyCamera Movement and LookAt

### Struct, not ECS component

```cpp
// include/spark/scene/camera/FlyCamera.hpp
struct FlyCamera {
    Vector3 position{0.0F, 4.2F, 16.0F};
    float yaw = 0.0F;
    float pitch = 0.0F;
    float moveSpeed = 5.0F;
    float mouseSensitivity = 0.12F;
    void ProcessMovement(IInput& input, float deltaSeconds) noexcept;
    [[nodiscard]] Matrix4 ViewMatrix() const noexcept;
};
```

Demos keep a `FlyCamera` on the stack or in demo state — no `GameObject` required. Projection (FOV, aspect, clip planes) stays outside the struct.

### Yaw/pitch forward vector

```cpp
// src/spark/scene/camera/FlyCamera.cpp
Vector3 FlyCamera::Forward() const noexcept {
    const float cy = std::cos(yaw), sy = std::sin(yaw);
    const float cp = std::cos(pitch), sp = std::sin(pitch);
    return Vector3{sy * cp, sp, -cy * cp}.Normalized();
}
```

WASD moves along `forward` and `Cross(forward, UnitY)` — strafe stays horizontal even when looking down.

### IInput-only movement

`ProcessMovement` queries `input.IsKeyDown(GLFW_KEY_W)` etc. — no GLFW window handle on the struct. Mouse look calls `AddLook(input.GetMouseDeltaX(), input.GetMouseDeltaY())` from demo update code after capture is enabled.

### SnapLookAt for framing

`SnapLookAt(target)` derives yaw/pitch from a direction vector — demos call it once after placing the camera to frame a mesh without manual angle tuning.

## 2. Core Concepts

### FlyCamera state

```cpp
struct FlyCamera {
    Vector3 position{0.0F, 4.2F, 16.0F};
    float yaw = 0.0F;
    float pitch = 0.0F;
    float moveSpeed = 5.0F;
    float mouseSensitivity = 0.12F;
};
```

| Field | Meaning |
|-------|---------|
| `position` | Camera eye in world space |
| `yaw` | Rotation around world Y (radians) |
| `pitch` | Rotation up/down (radians, clamped) |
| `moveSpeed` | Units per second for WASD |
| `mouseSensitivity` | Scales mouse delta in `AddLook` |

No roll — standard FPS constraint.

### Forward from yaw/pitch

```cpp
Vector3 FlyCamera::Forward() const {
    const float cy = cos(yaw), sy = sin(yaw);
    const float cp = cos(pitch), sp = sin(pitch);
    Vector3 f{sy * cp, sp, -cy * cp};
    return f.Normalized();
}
```

Spark forward: **−Z** at yaw=0, pitch=0 — matches camera and glTF conventions.

### AddLook — mouse integration

```cpp
void AddLook(float deltaX, float deltaY) {
    yaw += deltaX * mouseSensitivity * 0.01f;
    pitch -= deltaY * mouseSensitivity * 0.01f;
    pitch = clamp(pitch, -1.54f, 1.54f);  // ~±88°
}
```

Pitch sign: moving mouse **up** decreases pitch in Spark's convention (look up). Tune `mouseSensitivity` per game feel.

### ProcessMovement — keyboard

```cpp
void FlyCamera::ProcessMovement(IInput& input, float dt) {
    Vector3 forward = Forward();
    Vector3 right = Cross(forward, UnitY).Normalized();
    float s = moveSpeed * dt;
    if (input.IsKeyDown(GLFW_KEY_W)) position += forward * s;
    if (input.IsKeyDown(GLFW_KEY_S)) position -= forward * s;
    if (input.IsKeyDown(GLFW_KEY_A)) position -= right * s;
    if (input.IsKeyDown(GLFW_KEY_D)) position += right * s;
    if (input.IsKeyDown(GLFW_KEY_SPACE)) position += UnitY * s;
    if (input.IsKeyDown(GLFW_KEY_LEFT_SHIFT)) position -= UnitY * s;
}
```

Movement is **camera-relative** horizontal with world-vertical fly. No sprint in base struct — multiply `moveSpeed` in demo if needed.

### ViewMatrix and gimbal guard

```cpp
Matrix4 FlyCamera::ViewMatrix() const {
    Vector3 f = Forward();
    Vector3 worldUp = UnitY;
    if (fabs(Dot(f, worldUp)) > 0.92f)
        worldUp = UnitZ;  // near vertical look — avoid degenerate Cross
    return Matrix4::LookAt(position, position + f, worldUp);
}
```

Without alternate up hint, looking straight up/down can break `LookAt` and distort ground plane projection.

### SnapLookAt — initialization

```cpp
void FlyCamera::SnapLookAt(const Vector3& target) {
    Vector3 d = normalize(target - position);
    pitch = asin(clamp(d.y, -1, 1));
    yaw = atan2(d.x, -d.z);
}
```

Call after placing `position` to aim at a mesh or origin without manual yaw/pitch tuning.

### Demo integration pattern

```cpp
void ThreeDDemo::Simulate(const FrameTiming& timing, IEngineContext& context) {
    IInput& in = context.GetInput();
    if (in.IsKeyPressedThisFrame(GLFW_KEY_F1))
        in.SetCursorCaptured(!in.IsCursorCaptured());

    if (in.IsCursorCaptured()) {
        if (timing.frameIndex > 0)
            camera.AddLook(in.GetMouseDeltaX(), in.GetMouseDeltaY());
        camera.ProcessMovement(in, timing.deltaTimeSeconds);
    }
}
```

**frameIndex > 0** skips one-frame mouse spike when capture enables.

### Projection outside struct

```cpp
int fbW, fbH;
context.GetFramebufferSize(fbW, fbH);
const float aspect = static_cast<float>(fbW) / std::max(fbH, 1);
const Matrix4 proj = Matrix4::PerspectiveVulkan(DegreesToRadians(60.f), aspect, 0.12f, 400.f);
const Matrix4 view = camera.ViewMatrix();
params.viewProjection = proj * view;
params.cameraPositionWorld = camera.position;
```

Near/far and FOV are scene policy — not hardcoded in `FlyCamera`.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Struct not component

**Concept:** FlyCamera state

**Decision:** Plain struct in `spark/scene/`

**Why:** Demos/editors use without ECS overhead; copy into scene when needed.

**In Spark:** `FlyCamera.hpp`.

### Yaw/pitch not quaternion

**Concept:** Forward from yaw/pitch

**Decision:** Euler for fly cam only

**Why:** Intuitive mouse mapping; clamp pitch explicitly.

**In Spark:** `FlyCamera.cpp`.

### GLFW keys in ProcessMovement

**Concept:** ProcessMovement

**Decision:** `FlyCamera.cpp` includes GLFW for key constants

**Why:** Movement binding is implementation detail; could move to key map later.

**In Spark:** Only `.cpp` includes GLFW, not header consumers.

### Vertical movement on world Y

**Concept:** Space/Shift

**Decision:** Not along camera up when pitched

**Why:** Editor-style "elevator" fly; differs from some FPS games — document for players.

**In Spark:** `ProcessMovement`.

### Alternate up in LookAt

**Concept:** ViewMatrix guard

**Decision:** Switch to UnitZ when forward ≈ ±Y

**Why:** Prevents horizon flip when looking at ground from steep angle.

**In Spark:** `ViewMatrix` comment block.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/scene/FlyCamera.hpp`
- `src/spark/scene/FlyCamera.cpp`
- `src/spark/demo/ThreeDDemo.cpp`
- `src/spark/demo/SceneEditor3DDemo.cpp` (orbit extensions)

### Step 1 — Static camera

Set position, `SnapLookAt(origin)`, render — verify framing before enabling movement.

### Step 2 — Enable capture + look

F1, move mouse — yaw/pitch change, cube rotates on screen.

### Step 3 — WASD

Confirm W moves toward screen center target, S backward, strafe A/D.

### Step 4 — Pitch limit

Look up until clamp — no flip over the top.

### Step 5 — Steep pitch ground test

Look nearly straight down at plane — no matrix singularity with up hint.

### Step 6 — Speed scale

Double `moveSpeed` — travel across scene faster with same dt.

## 5. How This Connects to the Rest of Your Engine

Chapter **6** adds orbit around pivot (editor). Chapter **7** replaces fly with character rig. Chapter **8** ECS `CameraComponent` is production path for shipped scenes. Chapter **10** uploads matrices from demo fill or auto-submit.

`FlyCamera` is the **fastest path** to navigable 3D while building renderer features.

## 6. Pitfalls and Debugging

- **No mouse look** — capture off or frame 0 spike not guarded.
- **Inverted mouse Y** — flip sign in `AddLook` if preferred.
- **Sliding diagonally faster** — normalize combined WASD input if you add vector merge (Spark moves axis separately — slight diagonal speedup possible).
- **GLFW in header** — keep keys in `.cpp` only for module boundaries.
- **Wrong forward** — verify `Forward()` matches `LookAt` target direction.

## 7. Exercises

1. Add sprint on Left Ctrl (2× speed).
2. Implement `FlyCamera` with quaternion orientation — compare to yaw/pitch.
3. Log position every second flying in a straight line — verify `speed * dt` distance.
4. Wire scroll wheel to change `moveSpeed`.
5. Use `SnapLookAt` on a moving target each frame — what breaks?

## Summary

`FlyCamera` provides FPS-style yaw/pitch look, WASD+Space/Shift movement, and a gimbal-safe `ViewMatrix` via `LookAt`. Demos toggle cursor capture, guard first frame after capture, and multiply `PerspectiveVulkan` outside the struct into `SceneRenderParams`. Use it for debug navigation before investing in ECS camera components.

---

Next: [Camera Follow](06-camera-follow.md).
