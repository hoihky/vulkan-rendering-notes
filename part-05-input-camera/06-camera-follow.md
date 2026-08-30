# Camera Follow

> **Build-Along Milestone:** You understand three Spark follow patterns: editor orbit on `FlyCamera`, ECS `SpringArm3DComponent`, and smooth `CameraFollow3DComponent`.

**You will learn:**

- Orbit a fly camera around a world pivot for editor-style navigation.
- Place a camera on a spring arm behind a character pivot.
- Smooth-follow a target with optional look-at rotation.
- Choose between standalone rigs and ECS component ticks.

## 1. The Problem We Are Solving

A free fly camera (Chapter 5) does not follow the player. Games need cameras that **track targets** — smoothly, with offset, sometimes with collision-shortened arms. Spark implements follow at three levels:

| Pattern | Use case | Spark type |
|---------|----------|------------|
| Orbit pivot | Scene editor, inspect mesh | `OrbitFlyCameraAroundPivot` |
| Spring arm | Third-person behind character ECS | `SpringArm3DComponent` |
| Smooth follow | Cutscene camera, simple tracking | `CameraFollow3DComponent` |

This chapter explains each math model and update order — not just "lerp the camera" in one sentence.

### Follow cameras for beginners

**Fly camera** = you control position directly. **Follow camera** = camera chases something else (player, pivot).

**Orbit (editor style):**

```text
Pivot at mesh center
  Mouse drag → rotate camera on sphere around pivot
  Scroll → zoom distance
  Result: always looking at the object
```

**Spring arm (third-person games):**

```text
Character at origin
  Arm extends backward from character forward
  Camera at end of arm
  Optional: shorten arm if wall blocks line of sight
```

**Smooth follow (simple tracking):**

```text
camera.position = lerp(camera.position, target.position + offset, smoothness * dt)
camera looks at target
```

| Pattern | Who moves? | Spark |
|---------|------------|-------|
| Orbit | Camera on sphere | `OrbitFlyCameraAroundPivot` |
| Spring arm | Camera behind ECS character | `SpringArm3DComponent` |
| Smooth follow | Camera lerps to target | `CameraFollow3DComponent` |

Pick orbit for tools, spring arm for action games, smooth follow for cutscenes or simple 2.5D.

## 1b. Concept Deep Dive — Orbit, Spring Arm, and Smooth Follow

### Editor orbit on FlyCamera

`SceneEditor3DDemo` wraps `FlyCamera` with pivot orbit — Alt+drag rotates on a sphere around a world point while scroll adjusts radius. The fly camera struct stays the same; orbit math writes `position` from pivot + spherical coordinates each frame. Same `ViewMatrix()` path feeds `params.viewProjection`.

### ECS SpringArm3DComponent

Third-person characters attach `SpringArm3DComponent` to the player entity and a `CameraComponent` on a child camera object. Each tick the arm:

- Extends backward from character forward by `armLength`
- Optionally raycasts to shorten the arm when geometry blocks line of sight
- Positions the child camera at the arm tip

Transform hierarchy supplies world matrices — submit reads the camera child's `CameraComponent::ViewMatrix`.

### CameraFollow3DComponent smooth tracking

For simpler chase cameras, `CameraFollow3DComponent` lerps owner transform toward `target` + offset and optionally `LookAt`s the target each frame. Lower complexity than spring arm — good for cutscenes and top-down follow without collision response.

### When to pick which pattern

| Need | Spark type |
|------|------------|
| Inspect static mesh in editor | Orbit on `FlyCamera` |
| Action third-person with wall collision | `SpringArm3DComponent` |
| Smooth chase without arm logic | `CameraFollow3DComponent` |

All three produce a `Matrix4` view matrix consumed identically by `SceneRenderParams::viewProjection` — follow style is a CPU update choice, not a separate render path.

## 2. Core Concepts

### Editor orbit on FlyCamera

Alt+drag orbit in `SceneEditor3DDemo` preserves distance from pivot:

```cpp
void OrbitFlyCameraAroundPivot(FlyCamera& cam, const Vector3& pivot,
    float& orbitDistance, float deltaX, float deltaY) {
    cam.AddLook(deltaX, deltaY);
    orbitDistance = std::max(1.5f, orbitDistance);
    const Vector3 f = cam.Forward();
    cam.position = pivot - f * orbitDistance;
}
```

**Pivot fixed**, camera slides on sphere shell defined by yaw/pitch and distance. Scroll wheel adjusts `orbitDistance`. Combines fly look with **positional constraint**.

```text
position = pivot - forward(distance)
LookAt implicit in forward direction from pivot
```

### SpringArm3DComponent

Spring arm attaches camera to **pivot entity** (player torso):

```cpp
void SpringArm3DComponent::Tick(...) {
    Vector3 pivotPos = PivotWorld(pivot, socketOffset);
    Vector3 back = BackVectorFromYawPitch(arm.yaw, arm.pitch);
    float length = arm.length;
    // optional ground probe shortens length
    Vector3 cameraPos = pivotPos + back * length;
    cameraTransform->SetTranslation(cameraPos);
    cameraTransform->SetRotation(Quaternion::FromShortestArc(-Z, normalize(pivotPos - cameraPos)));
}
```

| Parameter | Role |
|-----------|------|
| `socketOffset` | Pivot on target (shoulder height) |
| `yaw`, `pitch` | Orbit angles around pivot |
| `length` | Arm distance |
| Ground probe | Ray down shortens arm to avoid clipping floor |

Mouse typically adjusts yaw/pitch on the arm, not the player facing — classic third-person ECS split.

### CameraFollow3DComponent

Exponential smooth to target position + offset:

```cpp
Vector3 desired = TargetWorldPosition(*target, targetOffset);
float t = clamp(followSmoothRate * deltaSeconds, 0, 1);
current += (desired - current) * t;
transform->SetTranslation(current);

if (lookAtTarget) {
    Vector3 toTarget = targetPos - current;
    Quaternion look = Quaternion::FromShortestArc(-Z, normalize(toTarget));
    transform->SetRotation(look);
}
```

| Field | Effect |
|-------|--------|
| `followSmoothRate` | Higher = snappier follow |
| `targetOffset` | Over-shoulder offset |
| `lookAtTarget` | Camera always aims at target |

Update priority **300** — runs after target movement (spring arm 295 places camera first in some setups — check demo wiring).

### 2D follow (related)

`Camera2DRigComponent` modes:

- **FollowTarget** — smooth XY track
- **BoundedFollow** — clamp camera in level bounds
- Velocity look-ahead optional via `Rigidbody2DComponent`

Documented in Chapter 9 — same **follow** mental model in orthographic plane.

### Combining arm + follow

Production third-person often chains:

```text
Player moves → Spring arm computes camera position → CameraComponent builds view/proj
```

`CameraFollow3D` is simpler when you only need a loose tracking cam without orbit input.

### Pan vs orbit (editor)

`PanFlyCamera` moves position in camera-right and camera-up plane without changing yaw/pitch — middle-mouse pan in editor. Orbit changes orientation **and** recomputes position on sphere.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Orbit as FlyCamera helper

**Concept:** Editor orbit

**Decision:** Free function in `SceneEditor3DDemo_detail.hpp`

**Why:** Reuses `AddLook` / `Forward`; no duplicate yaw/pitch state.

**In Spark:** Inline helper header.

### Spring arm as ECS component

**Concept:** SpringArm3D

**Decision:** Tick updates child camera transform

**Why:** Designer places arm + camera objects in hierarchy; yaw from input system.

**In Spark:** `SpringArm3DComponent.cpp`.

### Exponential smooth follow

**Concept:** CameraFollow3D

**Decision:** `t = rate * dt` lerp, not fixed time constant spring

**Why:** Frame-rate independent first-order lag; simple tuning.

**In Spark:** `CameraFollow3DComponent::Tick`.

### LookAt via quaternion shortest arc

**Concept:** lookAtTarget

**Decision:** Align camera −Z to `toTarget`

**Why:** Matches `CameraComponent` forward convention.

**In Spark:** `FromShortestArc({0,0,-1}, toTarget)`.

### Ground probe on arm

**Concept:** Spring arm length

**Decision:** Shorten arm when floor hit

**Why:** Reduce camera clipping through terrain without full collision solver.

**In Spark:** `SpringArm3DComponent` probe logic.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/demo/SceneEditor3DDemo_detail.hpp`
- `include/spark/ecs/components/camera/SpringArm3DComponent.hpp`
- `src/spark/ecs/components/camera/SpringArm3DComponent.cpp`
- `include/spark/ecs/components/camera/CameraFollow3DComponent.hpp`
- `src/spark/ecs/components/camera/CameraFollow3DComponent.cpp`

### Step 1 — Editor orbit

Run `SceneEditor3DDemo`, Alt+LMB drag — mesh orbits in view, distance constant.

### Step 2 — Spring arm scene

Spawn player + arm + camera entities. Adjust `length` and `socketOffset` — camera sits behind shoulder.

### Step 3 — Follow only

Disable spring input, enable `CameraFollow3D` on empty camera — lag behind moving cube.

### Step 4 — lookAtTarget off

Camera follows position but keeps fixed rotation — observe difference.

### Step 5 — Tune smooth rate

`followSmoothRate` 2 vs 20 — cinematic vs responsive.

## 5. How This Connects to the Rest of Your Engine

Chapter **7** `CharacterCameraRig` is standalone alternative to spring arm + follow ECS stack. Chapter **8** resolves `CameraComponent` on followed object. Chapter **10** reads resulting world matrix for `viewProj`.

Pick **one follow stack** per game mode — do not stack orbit helper on ECS arm without clear ownership.

## 6. Pitfalls and Debugging

- **Orbit drifts distance** — forgot `orbitDistance` clamp or scroll update.
- **Camera inside head** — `socketOffset` or `length` too small.
- **Jitter on moving target** — follow tick before target physics — fix update order.
- **LookAt flip** — target too close; shortest arc unstable.
- **2D bounds clamp fights follow** — bounds too tight in `Camera2DRig`.

## 7. Exercises

1. Implement zoom on orbit distance with scroll wheel.
2. Add collision sphere cast along arm backward vector.
3. Plot follow error vs time for exponential smooth at different rates.
4. When should you use orbit helper vs spring arm? One paragraph each.
5. Wire `CameraFollow3D` to moving physics body — measure lag in meters.

## Summary

Spark follows targets through editor orbit on `FlyCamera`, ECS `SpringArm3DComponent` for third-person offset and ground-aware length, and `CameraFollow3DComponent` for smoothed position with optional look-at. Each pattern updates position (and rotation) before `CameraComponent` builds view matrices. Choose the pattern that matches your control scheme and entity hierarchy.

---

Next: [Character Camera Rig](07-character-camera.md).
