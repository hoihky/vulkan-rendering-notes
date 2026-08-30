# Sky Geometry

> **Build-Along Milestone:** Sky sphere follows camera with no visible parallax; full-screen coverage on all aspect ratios.

**You will learn:**

- Generate sky sphere and billboard plane meshes with Mesh::CreateSkySphere / CreateSkyBillboardPlane.
- Center and scale sky transforms on the camera each frame.
- Route sky geometry through SceneMeshSlot::Custom and the custom mesh pool.

## 1. The Problem We Are Solving

Fragment shaders need pixels to shade, so "infinite" skies still need **mesh shells** or fullscreen coverage.
The art is choosing topology and transform rules so coverage is complete while parallax stays imperceptible.

Build along: create `CreateSkySphere`, parent it under a sky entity, and follow the camera like `SkyDemo::UpdateSkyTransform`.

## 1b. Concept Deep Dive — Sky Mesh Geometry in Spark

### CreateSkySphere — UV sphere tessellation

Spark generates a latitude/longitude sphere with outward normals and equirect-friendly UVs:

```cpp
// src/spark/scene/mesh/Mesh.cpp (lines 329–371, concept)
Mesh Mesh::CreateSkySphere(float radius, int latitudeSegments, int longitudeSegments) {
    Mesh m(Utf8String("SkySphere"));
    for (int iy = 0; iy <= latitudeSegments; ++iy) {
        const float theta = (float)iy / latitudeSegments * Pi;
        for (int ix = 0; ix <= longitudeSegments; ++ix) {
            const float phi = (float)ix / longitudeSegments * TwoPi;
            Vector3 pos{ sin(theta)*cos(phi), cos(theta), sin(theta)*sin(phi) };
            pos *= radius;
            Vector3 n = pos.Normalized();
            Vector2 uv{ (float)ix / longitudeSegments, 1.0f - (float)iy / latitudeSegments };
            m.vertices.PushBack({pos, n, uv});
        }
    }
    // index pairs into quads → two triangles each
    return m;
}
```

**Why sphere not cube?** Cube skyboxes show face-edge seams when sampling equirect HDR — a tessellated sphere gives smoother raster coverage for per-pixel ray reconstruction.

### CreateSkyBillboardPlane — procedural Plane mode

```cpp
// src/spark/scene/mesh/Mesh.cpp (lines 374–387)
Mesh Mesh::CreateSkyBillboardPlane(float halfWidth, float halfHeight) {
    Mesh m(Utf8String("SkyPlane"));
    m.vertices.PushBack({ {-hw, -hh, 0}, {0,0,1}, {0,0} });
    m.vertices.PushBack({ { hw, -hh, 0}, {0,0,1}, {1,0} });
    m.vertices.PushBack({ { hw,  hh, 0}, {0,0,1}, {1,1} });
    m.vertices.PushBack({ {-hw,  hh, 0}, {0,0,1}, {0,1} });
    // CCW indices from +Z
    return m;
}
```

Plane mode uses **mesh UV** (`vTexCoord0.y`) for vertical gradient when no equirect texture is bound — the fragment shader does not reconstruct view rays for procedural Plane fallback.

### Sky follows camera — infinite distance illusion

```cpp
// SkyDemo::UpdateSkyTransform (concept)
skyTransform->SetTranslation(cameraPosition);  // no rotation — shader handles direction
skyTransform->SetScale({500.0f, 500.0f, 500.0f});  // large enough to fill frustum
```

Vertex shader pushes sky to **far plane** depth so it draws behind all geometry:

```glsl
// shaders/scene.vert (lines 105–111)
if (push.skyMode != 0) {
    float zw = max(clip.w, 1e-5);
    gl_Position = vec4(clip.x, clip.y, zw * (1.0 - 1e-4), zw);
}
```

## 2. Core Concepts

### Rasterization still needs geometry

Even when the fragment shader reconstructs view rays, the GPU must **rasterize triangles**. Spark uses a low-poly **sky sphere** (`Mesh::CreateSkySphere`) uploaded through the **custom mesh pool**, or a **billboard plane** for procedural Plane mode.


### Camera-centered transform

Infinite skies translate with the camera each frame — `skyTransform->SetTranslation(camera.position)` and large uniform scale (~92–98) keep the mesh shell inside clip space without parallax against mountains.


### Custom mesh slot

Sky geometry uses `SceneMeshSlot::Custom` with `SharedPtr<Mesh>` on the draw item. The Vulkan custom pool uploads CPU vertices each frame (or caches by mesh identity).


### Sphere vs plane tradeoff

Textured HDR uses a **full sphere** for all three modes (Dome math is per-pixel, not mesh UV). Procedural Plane mode may use a billboard quad with `vTexCoord` vertical gradient — finite quads leave gaps if too small.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Lat/long UV sphere generator

**Concept:** CreateSkySphere procedural

**Decision:** Lat/long UV sphere generator

**Why:** One function produces sky env mesh; demos share topology without asset files.

**In Spark:** `src/spark/scene/Mesh.cpp` CreateSkySphere

### Unit quad in XY at z=0

**Concept:** CreateSkyBillboardPlane

**Decision:** Unit quad in XY at z=0

**Why:** CCW winding from +Z for camera-facing billboards in procedural Plane fallback.

**In Spark:** `Mesh.cpp` CreateSkyBillboardPlane

### Reposition + scale sky shell

**Concept:** UpdateSkyTransform each frame

**Decision:** Reposition + scale sky shell

**Why:** Box/Dome/HDR Plane follow camera; procedural Plane billboards overshoot frustum slice.

**In Spark:** `SkyDemo::UpdateSkyTransform`

### Implicit Custom slot for imported/sphere meshes

**Concept:** MeshComponent Custom constructor

**Decision:** Implicit Custom slot for imported/sphere meshes

**Why:** Sky entities always route through dynamic GPU buffers.

**In Spark:** `MeshComponent.hpp` constructors

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scene/Mesh.hpp`
- `src/spark/scene/Mesh.cpp`
- `src/spark/demo/SkyDemo.cpp`

### Step 1 — Read mesh generators

*CreateSkySphere lat/long indices; CreateSkyBillboardPlane quad winding.*

```cpp
// src/spark/scene/Mesh.cpp (lines 250–307)
Mesh Mesh::CreateSkySphere(float radius, int latitudeSegments, int longitudeSegments) {
    Mesh m(Utf8String("SkySphere"));
    if (latitudeSegments < 2) {
        latitudeSegments = 2;
    }
    if (longitudeSegments < 3) {
        longitudeSegments = 3;
    }
    const int latCount = latitudeSegments + 1;
    const int lonCount = longitudeSegments + 1;
    for (int iy = 0; iy < latCount; ++iy) {
        const float t = static_cast<float>(iy) / static_cast<float>(latitudeSegments);
        const float theta = t * Pi;
        const float sinT = std::sin(theta);
        const float cosT = std::cos(theta);
        const float y = radius * cosT;
        const float ringR = radius * sinT;
        for (int ix = 0; ix < lonCount; ++ix) {
            const float p = static_cast<float>(ix) / static_cast<float>(longitudeSegments);
            const float phi = p * TwoPi;
            const float x = ringR * std::cos(phi);
            const float z = ringR * std::sin(phi);
            const Vector3 pos{x, y, z};
            const Vector3 n = pos.Normalized();
            const Vector2 uv{p, 1.0F - t};
            m.vertices.PushBack({pos, n, uv});
        }
    }
    for (int iy = 0; iy < latitudeSegments; ++iy) {
        for (int ix = 0; ix < longitudeSegments; ++ix) {
            const std::uint32_t a = static_cast<std::uint32_t>(iy * lonCount + ix);
            const std::uint32_t b = a + 1U;
            const std::uint32_t c = a + static_cast<std::uint32_t>(lonCount);
            const std::uint32_t d = c + 1U;
            m.indices.PushBack(a);
            m.indices.PushBack(c);
            m.indices.PushBack(b);
            m.indices.PushBack(b);
            m.indices.PushBack(c);
            m.indices.PushBack(d);
        }
    }
    return m;
}

Mesh Mesh::CreateSkyBillboardPlane(float halfWidth, float halfHeight) {
    Mesh m(Utf8String("SkyPlane"));
    m.vertices.PushBack({ {-halfWidth, -halfHeight, 0.0F}, {0.0F, 0.0F, 1.0F}, {0.0F, 0.0F}});
    m.vertices.PushBack({ {halfWidth, -halfHeight, 0.0F}, {0.0F, 0.0F, 1.0F}, {1.0F, 0.0F}});
    m.vertices.PushBack({ {halfWidth, halfHeight, 0.0F}, {0.0F, 0.0F, 1.0F}, {1.0F, 1.0F}});
    m.vertices.PushBack({ {-halfWidth, halfHeight, 0.0F}, {0.0F, 0.0F, 1.0F}, {0.0F, 1.0F}});
    // CCW from +Z so front face points toward the camera when billboarding.
    m.indices.PushBack(0);
    m.indices.PushBack(1);
    m.indices.PushBack(2);
    m.indices.PushBack(0);
    m.indices.PushBack(2);
    m.indices.PushBack(3);
```

### Step 2 — Build sky sphere asset in Load()

*SkyDemo allocates skyBoxMesh and skyPlaneMesh from generators.*

```cpp
// src/spark/demo/SkyDemo.cpp (lines 10–13)
        skyBoxMesh = Spark::MakeShared<Spark::Mesh>(Spark::Utf8String("SkyEnvSphere"));
        *skyBoxMesh = Spark::Mesh::CreateSkySphere(1.0F, 20, 40);
        skyPlaneMesh = Spark::MakeShared<Spark::Mesh>(Spark::Utf8String("SkyPlaneMesh"));
        *skyPlaneMesh = Spark::Mesh::CreateSkyBillboardPlane(1.0F, 1.0F);
```

### Step 3 — Attach MeshComponent on sky entity

```cpp
skyMesh = skyObject->AddComponent<MeshComponent>(skyBoxMesh, Vector3::One);
```

Custom slot is implicit when passing a SharedPtr mesh.

### Step 4 — Center shell on camera

*UpdateSkyTransform — camera position, large scale, billboard math for Plane.*

```cpp
// src/spark/demo/SkyDemo.cpp (lines 315–351)
void SkyDemo::UpdateSkyTransform(int fbW, int fbH)
{
        if (skyTransform == nullptr) {
            return;
        }
        if (skyModeIndex < 2 || (skyModeIndex == 2 && skyHasEquirect)) {
            skyTransform->SetTranslation(camera.position);
            skyTransform->SetRotation(Spark::Quaternion::Identity);
            // Large enough that clip-space z stays inside (0,w) after sky vertex offset; sphere avoids cube seams.
            const float sc = (skyModeIndex == 1) ? 92.0F : 98.0F;
            skyTransform->SetUniformScale(sc);
        } else {
            const float aspect = (fbH > 0) ? static_cast<float>(fbW) / static_cast<float>(fbH) : 1.0F;
            // Billboard must overshoot the frustum slice: a tight quad leaves clear-color gaps / clip holes
            // when yaw/pitch (HDR uses per-pixel rays, but fragments only exist under the mesh).
            constexpr float kBackDist = 40.0F;
            constexpr float kFovMargin = 1.72F;
            const Spark::Vector3 f = camera.Forward().Normalized();
            const Spark::Vector3 center = camera.position - f * kBackDist;
            // Same handedness as FlyCamera / Matrix4::LookAt: right = f × worldUp, up = right × f.
            Spark::Vector3 r = Spark::Vector3::Cross(f, Spark::Vector3::UnitY);
            if (r.LengthSquared() < 1.0e-10F) {
                r = Spark::Vector3::Cross(f, Spark::Vector3::UnitX);
                if (r.LengthSquared() < 1.0e-10F) {
                    r = Spark::Vector3::UnitX;
                }
            }
            r = r.Normalized();
            const Spark::Vector3 u = Spark::Vector3::Cross(r, f).Normalized();
            const Spark::Quaternion q = QuaternionFromRotationColumns(r, u, f).Normalized();
            skyTransform->SetTranslation(center);
            skyTransform->SetRotation(q);
            const float tanHalf = std::tan(Spark::DegreesToRadians(kSkyDemoFovYDeg) * 0.5F);
            const float halfH = tanHalf * kBackDist * kFovMargin;
            const float halfW = halfH * aspect;
            skyTransform->SetScale({halfW, halfH, 1.0F});
        }
```

### Step 5 — Verify custom pool binding

In RenderDoc, confirm sky draw binds **custom** vertex/index buffers (not UnitCube static slot). Draw item fields: `mesh = Custom`, `customMesh = skyBoxMesh`.

### Step 6 — Milestone — no parallax or clip holes

Walk 100 units sideways — sky should not slide relative to distant hills (sphere centered on eye). Yaw 360° on procedural Plane — no clear-color wedges at screen corners.

## 5. How This Connects to the Rest of Your Engine

**Sky Pipeline Branch** sets cull mode none and skips depth test for these draws. **Sky Rendering Modes** decides
whether fragments use equirect rays or UV gradients. Part **VIII** custom mesh pool details upload mechanics.

## 6. Pitfalls and Debugging

- **Sphere too small** — parallax against terrain; scale to ~90+ units or push clip z in vertex shader.
- **Tight billboard quad** — clear-color gaps at screen edges when rotating; overshoot frustum (see `kFovMargin`).
- **Dome mesh as hemisphere** — raster holes toward zenith; Spark uses full sphere for Dome mode too.
- **Forgetting to update transform** — sky lags one frame behind camera; call UpdateSkyTransform before submit.

## 7. Exercises

1. Compare vertex counts: `CreateSkySphere(1, 20, 40)` vs `CreateSkyBillboardPlane`.
2. Log sky world matrix translation — should match camera position each frame.
3. Disable `UpdateSkyTransform` — describe the parallax artifact you see.
4. Shrink sky scale to 10 — note clip holes or swimming at screen edges.
5. Trace `SceneMeshSlot::Custom` path in opaque pass for sky draw.

## Summary

Sky meshes provide raster coverage via CreateSkySphere or billboard plane; camera-centered transform and Custom slot feed the dynamic mesh pool.

---

Next: [Sky Pipeline Branch](03-sky-pipeline.md).
