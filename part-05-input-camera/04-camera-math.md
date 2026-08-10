# Camera Math

> **Build-Along Milestone:** You can build `viewProj = PerspectiveVulkan(aspect) * LookAt(eye, eye+f, up)` and verify column-major layout matches GLSL `mat4`.

**You will learn:**

- Use Spark's column-major `Matrix4` aligned with GLSL.
- Build perspective and orthographic projections for Vulkan depth [0,1].
- Construct view matrices with `LookAt`.
- Compose view and projection without transposing errors.

## 1. The Problem We Are Solving

A camera is two matrices:

- **View** — world space → camera space (inverse of camera world transform)
- **Projection** — camera space → clip space (perspective or orthographic)

Shaders multiply `viewProj * vec4(worldPos, 1)`. One transposed multiply spins the scene. One wrong clip Y flip inverts the image vertically. One OpenGL-style depth range breaks depth testing.

Spark centralizes math in `Matrix4` with **Vulkan-specific** `*Vulkan` helpers. This chapter is the numeric foundation for every camera rig in Part V.

## 2. Core Concepts

### Column-major storage

```cpp
struct Matrix4 {
    float m[16];  // column c, row r → m[c * 4 + r]
};
```

Matches GLSL `mat4` memory layout — `memcpy` to UBO safe. **Not** row-major DirectX math library layout unless transposed.

```text
| m[0]  m[4]  m[8]  m[12] |
| m[1]  m[5]  m[9]  m[13] |
| m[2]  m[6]  m[10] m[14] |
| m[3]  m[7]  m[11] m[15] |
```

### Right-handed coordinate system

Spark uses **Y-up, right-handed** world:

- **Forward** for cameras: typically **−Z** (OpenGL/Vulkan convention)
- **Right**: +X
- **Up**: +Y

`LookAt(eye, target, up)` builds camera basis: right `s`, up `u`, forward `f = normalize(target - eye)`, then view rows dot with basis and translate.

### LookAt view matrix

```cpp
static Matrix4 LookAt(const Vector3& eye, const Vector3& target, const Vector3& up) {
    const Vector3 f = (target - eye).Normalized();
    Vector3 s = Cross(f, up).Normalized();  // right
    Vector3 u = Cross(s, f);               // true up
    // columns: s, u, -f, translation
    r.m[12] = -Dot(s, eye);
    r.m[13] = -Dot(u, eye);
    r.m[14] = Dot(f, eye);
    return r;
}
```

Degenerate case: `f` parallel to `up` — Spark falls back to alternate up vector in `FlyCamera::ViewMatrix`.

### Perspective projection (Vulkan Z)

```cpp
static Matrix4 Perspective(float fovYRad, float aspect, float nearZ, float farZ) {
    const float tanHalf = tan(fovYRad * 0.5f);
    const float f = 1.0f / tanHalf;
    r.m[0] = f / aspect;
    r.m[5] = f;
    r.m[10] = farZ / (nearZ - farZ);
    r.m[11] = -1.0f;   // w = -z for perspective divide
    r.m[14] = (nearZ * farZ) / (nearZ - farZ);
    return r;
}
```

Depth maps to **[0, 1]** in clip space (Vulkan/D3D), not OpenGL's [−1, 1].

### PerspectiveVulkan — clip Y flip

```cpp
static Matrix4 PerspectiveVulkan(float fovYRad, float aspect, float nearZ, float farZ) {
    Matrix4 r = Perspective(fovYRad, aspect, nearZ, farZ);
    r.m[5] *= -1.0f;  // negate Y scale
    return r;
}
```

Vulkan framebuffer Y grows **down**; without negating Y scale, world +Y appears inverted when viewport height is positive. **Always** use `PerspectiveVulkan` / `OrthographicVulkan` with Spark's renderer — do not negate viewport height instead unless you know the full pipeline.

### Orthographic projection

```cpp
static Matrix4 OrthographicVulkan(float left, float right, float bottom, float top,
                                  float nearZ, float farZ);
```

Maps `[left,right] × [bottom,top]` to clip. `Camera2D` uses symmetric ±halfW, ±halfH from `halfExtentY` and aspect.

### Composing viewProj

```cpp
const float aspect = static_cast<float>(fbW) / static_cast<float>(fbH);
const Matrix4 proj = Matrix4::PerspectiveVulkan(DegreesToRadians(60.f), aspect, 0.12f, 400.f);
const Matrix4 view = camera.ViewMatrix();
const Matrix4 viewProj = proj * view;  // projection * view
```

Order: **projection on the left**, view on the right — applies view first to column vectors.

ECS `CameraComponent::ViewProjection` uses same multiply.

### Inverse view projection

```cpp
Matrix4 invVp;
if (viewProj.TryInvert(invVp)) { /* sky ray reconstruction */ }
```

Computed at UBO upload time (Chapter 10), not stored in every camera struct.

### Transform helpers

| Function | Use |
|----------|-----|
| `Translation` | Camera world position (object transform) |
| `Rotation(Quaternion)` | Camera orientation ECS |
| `TransformPoint` | World point through matrix |
| `TransformVector` | Direction, no translation |
| `TranslationVector` | Extract position from world matrix |

### Aspect ratio

Always derive from **framebuffer** width/height, not window size on HiDPI:

```cpp
context.GetFramebufferSize(fbW, fbH);
const float aspect = static_cast<float>(fbW) / std::max(fbH, 1);
```

Resize swapchain → aspect changes → projection must update every frame.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Column-major native

**Concept:** Storage layout

**Decision:** `m[c*4+r]` documented in header

**Why:** Zero transpose at GPU upload; matches GLSL spec.

**In Spark:** `Matrix4.hpp`.

### Vulkan Z [0,1]

**Concept:** Perspective projection

**Decision:** `m[10]`, `m[14]` formulas for Vulkan depth

**Why:** Matches depth buffer clear 1.0 far plane in render pass.

**In Spark:** `Perspective` comment and implementation.

### *Vulkan Y negation in matrix

**Concept:** PerspectiveVulkan

**Decision:** Flip `m[5]` not viewport height

**Why:** Positive viewport height in `vkCmdSetViewport`; matrix absorbs NDC Y convention.

**In Spark:** All 3D demos use `PerspectiveVulkan`.

### viewProj = proj * view

**Concept:** Composition

**Decision:** Same as `CameraComponent::ViewProjection`

**Why:** Consistent across manual rigs and ECS.

**In Spark:** `ThreeDDemo`, `CameraComponent.cpp`.

### Separate cameraPositionWorld

**Concept:** Beyond view matrix

**Decision:** Store eye position explicitly in `SceneRenderParams`

**Why:** Shaders need world position for specular, fog, sky — not trivially extracted from view matrix alone after projection multiply.

**In Spark:** Chapter 10.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/math/Matrix4.hpp`
- `include/spark/math/Constants.hpp` (`DegreesToRadians`)
- `include/spark/ecs/components/camera/CameraComponent.cpp`
- `src/spark/scene/Camera2D.cpp` (ortho)

### Step 1 — Identity chain

Set `view = Identity`, `proj = PerspectiveVulkan(60°, 16/9, 0.1, 100)`. Object at (0,0,-5) should appear centered.

### Step 2 — LookAt orbit

Fix eye at (0, 2, 5), target origin. Rotate eye around Y manually — view updates, cube stable at origin.

### Step 3 — Transpose experiment

Transpose `viewProj` once — scene shears/spins. Revert — appreciate layout discipline.

### Step 4 — Wrong Vulkan helper

Use `Perspective` without Vulkan Y flip — scene upside-down. Switch to `PerspectiveVulkan`.

### Step 5 — Near/far plane

Move camera through near plane — clipping. Increase near too much — Z-fighting outdoors.

### Step 6 — Verify shader multiply

`scene.vert`: `gl_Position = ubo.viewProj * vec4(worldPos, 1.0)` — matches CPU order.

## 5. How This Connects to the Rest of Your Engine

Chapters **5–9** produce `view` and `proj`. Chapter **10** uploads `viewProj`. Part **III** vertex shader consumes UBO. Part **IV** `invViewProj` rebuilds rays.

Camera math errors are **high visibility** — master this chapter before rig complexity.

## 6. Pitfalls and Debugging

- **Transposed matrix** — object orbits wrong axis when camera still.
- **aspect from window not framebuffer** — stretched image on Retina.
- **OpenGL depth** — using [−1,1] projection with Vulkan depth test.
- **view * proj order** — reversed multiply inverts transform chain.
- **Gimbal lock in Euler** — use quaternions for ECS camera orientation; fly cam clamps pitch.

## 7. Exercises

1. Derive clip coords for unit cube corner through your `viewProj` by hand (one point).
2. Implement `Matrix4::operator*` unit test with known translation × rotation.
3. Plot effect of FOV 45° vs 90° on same scene screenshot.
4. When is `OrthographicVulkan` ill-conditioned (near ≈ far)?
5. Extract camera forward from view matrix columns — which column sign for Spark −Z forward?

## Summary

Spark cameras use column-major `Matrix4` with `LookAt` view and `PerspectiveVulkan` / `OrthographicVulkan` projection for Vulkan depth and Y conventions. Compose `viewProj = projection * view` using framebuffer aspect ratio. Match this math in C++, UBO upload, and `scene.vert` — transpose or wrong Vulkan variant breaks the entire 3D pipeline.

---

Next: [Fly Camera](05-fly-camera.md).
