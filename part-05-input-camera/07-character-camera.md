# Character Camera Rig

> **Build-Along Milestone:** `CharacterCameraRig` switches first/third person; `CharacterCameraDemo` drives a skinned avatar with camera-relative movement and glTF facing offsets.

**You will learn:**

- Implement first-person eye position and third-person orbit camera.
- Separate character facing yaw from camera pitch.
- Handle glTF bind orientation and local forward axis.
- Integrate walk input differently per camera mode.

## 1. The Problem We Are Solving

Action games need a camera tied to a **character** — not a free fly cam. First-person puts the eye at the head; third-person orbits behind a pivot with a look target on the upper body. glTF avatars add complexity: bind pose forward may be −Z with extra quaternion fixes.

`CharacterCameraRig` is Spark's **standalone** character camera (non-ECS) used in `CharacterCameraDemo` and `Maze3DDemo`. It complements ECS `SpringArm3D` (Chapter 6) with tighter integration to walk input and skinned mesh facing.

### Character camera vs fly camera (beginner)

| | Fly camera | Character camera |
|---|------------|------------------|
| Position | You fly anywhere | Tied to avatar |
| Look | Mouse aims view | Mouse aims **character facing** or orbit |
| Movement | Camera-relative WASD | **Character** walks on ground; camera follows |

**First person:** camera eye at head height — you see through the avatar's eyes. Walk input moves along **camera forward projected on XZ plane** (no flying up slopes unless coded).

**Third person:** camera orbits behind a shoulder pivot — you see the character mesh. Walk input moves **character**, not camera; camera catches up.

**glTF facing gotcha:** many rigs face **−Z** in bind pose. Spark applies `gltfFacingOffset` quaternion so "forward" in code matches artist intent — without it, W walks sideways.

**Separate yaw:** store `characterYaw` (body rotation) and `cameraPitch` (look up/down) independently — standard third-person control.

## 1b. Concept Deep Dive — CharacterCameraRig Modes

### First vs third person fields

```cpp
// include/spark/scene/camera/CharacterCameraRig.hpp
struct CharacterCameraRig {
    CharacterCameraMode mode = CharacterCameraMode::ThirdPerson;
    Vector3 characterPosition{};
    float characterVisualYaw = 0.0F;
    float characterFacingYawOffset = 0.0F;  // glTF bind fix
    Quaternion characterRootBindOrientation{Quaternion::Identity};
    Vector3 characterLocalForward{0.0F, 0.0F, -1.0F};
    float thirdPersonDistance = 4.25F;
    float firstPersonEyeHeight = 1.62F;
    void ProcessWalk(IInput& input, float deltaSeconds) noexcept;
    [[nodiscard]] Matrix4 ViewMatrix() const noexcept;
};
```

First person: eye at feet + `firstPersonEyeHeight` + forward nudge along yaw. Third person: camera sits behind at `thirdPersonDistance`, looks at pivot height + `thirdPersonFocusAhead`.

### Walk input differs by mode

`ProcessWalk` in first person moves relative to **camera yaw** (strafe + forward). Third person moves W/S along **character facing** only — A/D are intentionally ignored; mouse yaw rotates the character. `CharacterCameraDemo` wires this to a skinned glTF avatar.

### glTF facing offsets

`characterFacingYawOffset` and `characterRootBindOrientation` align code forward with artist bind pose — without them, W walks sideways on −Z-facing rigs. Effective heading = wrapped sum of visual yaw + offset, then multiplied by bind orientation for third-person orbit.

### Same UBO handoff as FlyCamera

`ViewMatrix()` returns `LookAt(eye, target, up)` — demos assign `params.viewProjection = proj * rig.ViewMatrix()` identically to `ThreeDDemo`'s fly path. Input abstraction unchanged: `ProcessWalk` and `AddLook` take `IInput&`.

## 2. Core Concepts

### Modes

```cpp
enum class CharacterCameraMode {
    FirstPerson,
    ThirdPerson,
};
```

Toggle at runtime (`ToggleCameraMode`) — typically a key press in demo.

### State overview

| Field group | Purpose |
|-------------|---------|
| `characterPosition`, `groundY` | Feet on floor |
| `characterVisualYaw`, `characterFacingYawOffset` | Facing + glTF fix |
| `characterRootBindOrientation`, `characterLocalForward` | Mesh vs logical forward |
| `cameraYaw`, `cameraPitch` | Look (third person: orbit; first person: view) |
| `thirdPersonDistance`, `pivotHeight`, `focusAhead`, `cameraLift` | TPS framing |
| `firstPersonEyeHeight`, `firstPersonForwardNudge` | FPS eye clear mesh |

### AddLook — mode-dependent

**First person:** mouse adjusts `cameraYaw` and `cameraPitch` (like fly cam).

**Third person:** mouse adjusts `characterVisualYaw` (turn body) and `cameraPitch` (orbit elevation) — A/D not used for strafe; player turns with mouse.

Pitch clamped similar to fly camera.

### ProcessWalk — mode-dependent

**First person:** WASD relative to **camera yaw** (strafe + forward).

**Third person:** W/S along **character facing**; A/D ignored — turn with mouse.

Shift may increase speed (`runSpeed` vs `moveSpeed`) in demo wiring.

### ThirdPersonFlatForward

Combines yaw quaternions and bind orientation to get horizontal forward for movement and camera placement:

```text
effectiveYaw = characterVisualYaw + characterFacingYawOffset
forward = rotate(yaw) * bindOrientation * characterLocalForward
flatten Y for orbit math
```

Critical for glTF characters where mesh forward ≠ world −Z without offsets.

### CameraWorldPosition

**First person:**

```cpp
return FirstPersonEyeWorld();  // feet + eyeHeight + forward nudge along yaw
```

**Third person:**

```cpp
Vector3 pivot = characterPosition + UnitY * thirdPersonPivotHeight;
Vector3 flatF = ThirdPersonFlatForward();
float cp = cos(cameraPitch), sp = sin(cameraPitch);
float horiz = cp * thirdPersonDistance;
float vert = sp * thirdPersonDistance;
return pivot - flatF * horiz + UnitY * (vert + thirdPersonCameraLift);
```

Camera sits on sphere shell around pivot — behind character, elevated by pitch.

### ViewMatrix

**First person:** `LookAt(eye, eye + ForwardWorld(), up)`.

**Third person:**

```cpp
Vector3 pivot = characterPosition + UnitY * thirdPersonPivotHeight;
Vector3 focus = pivot + flatF * thirdPersonFocusAhead;
Vector3 cam = CameraWorldPosition();
return LookAt(cam, focus, worldUp);
```

Look target slightly ahead of head — over-the-shoulder feel, not staring at feet pivot.

### Demo integration

```cpp
if (in.IsCursorCaptured() && timing.frameIndex > 0)
    rig.AddLook(in.GetMouseDeltaX(), in.GetMouseDeltaY());
rig.ProcessWalk(in, timing.deltaTimeSeconds);

params.viewProjection = proj * rig.ViewMatrix();
params.cameraPositionWorld = rig.CameraWorldPosition();
// character mesh transform from rig position + yaw
```

Physics demo may sync `characterPosition` from controller — camera follows body.

### vs ECS spring arm

| CharacterCameraRig | SpringArm3D + CameraComponent |
|------------------|-------------------------------|
| Single struct, demo-friendly | Hierarchy in editor |
| Built-in walk/look modes | Input wired separately |
| glTF facing baked in | General purpose |

Shipped content may migrate rigs to ECS; learning path starts here.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Standalone rig struct

**Concept:** Modes

**Decision:** Not a Component — used by specific demos

**Why:** Rapid iteration on TPS/FPS without ECS ceremony.

**In Spark:** `CharacterCameraRig.hpp`.

### Split visual vs facing offset

**Concept:** glTF bind

**Decision:** `characterFacingYawOffset` + bind quaternion

**Why:** Art pipeline fixes without re-exporting mesh.

**In Spark:** Comments in header; `ThirdPersonFlatForward`.

### Third person no A/D strafe

**Concept:** ProcessWalk

**Decision:** W/S only along facing

**Why:** Tank controls + mouse turn — common console-style on PC demos.

**In Spark:** `ProcessWalk` implementation.

### Focus ahead of pivot

**Concept:** ViewMatrix TPS

**Decision:** `thirdPersonFocusAhead` offset

**Why:** Aim horizon stable when pivot at chest height.

**In Spark:** `CharacterCameraRig.cpp`.

### Toggle mode at runtime

**Concept:** First/third switch

**Decision:** `ToggleCameraMode` resets conflicting state minimally

**Why:** Debug compare perspectives in one demo binary.

**In Spark:** `CharacterCameraDemo` key bind.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/scene/CharacterCameraRig.hpp`
- `src/spark/scene/CharacterCameraRig.cpp`
- `src/spark/demo/CharacterCameraDemo.cpp`
- `src/spark/demo/Maze3DDemo.cpp` (FPS maze)

### Step 1 — Third person default

Run `CharacterCameraDemo` — camera behind avatar, mouse turns, W walks forward.

### Step 2 — Toggle first person

Switch mode — eye inside head mesh if offsets wrong; tune `firstPersonForwardNudge`.

### Step 3 — Pitch orbit TPS

Look up/down — camera rises/falls on shell, focus stays on upper body.

### Step 4 — glTF facing

Rotate character mesh component offsets — verify walk direction matches visual.

### Step 5 — Maze FPS

`Maze3DDemo` first-person only — compare simpler rig usage without TPS fields.

## 5. How This Connects to the Rest of Your Engine

Chapter **6** ECS follow is alternative architecture. Chapter **8** auto-resolve uses `CameraComponent` when not using manual rig fill. Chapter **10** uploads same `viewProj` / `cameraPositionWorld` fields.

Character camera links **input → locomotion → view matrices → skinned mesh**.

## 6. Pitfalls and Debugging

- **Mesh slides wrong way** — `characterFacingYawOffset` or local forward wrong for asset.
- **TPS camera inside head** — increase `thirdPersonDistance` or `cameraLift`.
- **FPS clipping face** — increase `firstPersonForwardNudge` / `eyeHeight`.
- **Mouse turns wrong axis** — third person adjusts `characterVisualYaw` not `cameraYaw` for horizontal.
- **Ground float** — `groundY` not synced with physics height.

## 7. Exercises

1. Add over-the-shoulder offset (shift pivot right in world space).
2. Implement first-person weapon sway from mouse delta.
3. Compare `CharacterCameraRig` TPS to `SpringArm3D` — list pros/cons.
4. Log `CameraWorldPosition` and pivot each frame in circle strafe test.
5. Support inverted Y mouse option in `AddLook`.

## Summary

`CharacterCameraRig` provides first- and third-person cameras with glTF-aware facing, mode-specific walk and look input, third-person orbit around a upper-body pivot, and first-person eye positioning with forward nudge. Demos fill `SceneRenderParams` from `ViewMatrix` and `CameraWorldPosition`. Tune pivot, distance, and bind offsets per avatar; migrate to ECS spring arm when level design needs editor hierarchies.

---

Next: [Main Camera Resolution](08-main-camera.md).
