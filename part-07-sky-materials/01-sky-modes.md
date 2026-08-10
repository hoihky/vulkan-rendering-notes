# Sky Rendering Modes

> **Build-Along Milestone:** SkyDemo cycles three modes; HDR background stable under camera rotation.

**You will learn:**

- Contrast Box, Dome, Plane, and None sky modes in Spark.
- Explain why view-ray reconstruction beats interpolated mesh positions for HDR skies.
- Wire SkyComponent + MeshComponent + optional MaterialComponent on a sky entity.

## 1. The Problem We Are Solving

Outdoor scenes need a background at effectively infinite distance. A flat clear color breaks immersion; treating sky
as a lit mesh with sun **N·L** is wrong — sky radiance is **emitted**, not reflected.

Your build-along milestone: spawn a `SkyComponent` entity, cycle Box/Dome/Plane in `SkyDemo`, and confirm the HDR
background stays fixed when you orbit the camera.

## 2. Core Concepts

### SceneSkyMode taxonomy

Spark encodes sky **geometry + shading path** in `SceneSkyMode`: `None` (standard lit mesh), `Box` (closed environment sphere/cube), `Dome` (hemisphere-style gradient math), `Plane` (billboard quad UV gradient). The mode rides on each `SceneDrawItem::skyMode` and early-outs in `scene.frag` — not a global render flag.


### Per-pixel view rays vs mesh UV

HDR skies must stay fixed when the camera rotates. Spark reconstructs a **world view direction** from `invViewProj` and pixel NDC in the fragment shader instead of interpolating mesh positions — interpolated `vWorldPos` on a large sphere **swims** under rotation.


### Procedural vs textured fallback

When `textureLayer >= 0`, Box/Dome/Plane sample an **equirectangular** map via `(phi, mu)` spherical coords. Without a texture, each mode uses a distinct procedural gradient keyed off `push.albedoTint`.


### ECS sky entity pattern

A sky is a normal `GameObject` with `SkyComponent` + `MeshComponent` + optional `MaterialComponent` (equirect texture). Submit marks the draw item `skyMode != None`; the GPU skips BRDF entirely.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Per-draw sky flag in push constants

**Concept:** SceneSkyMode on draw item

**Decision:** Per-draw sky flag in push constants

**Why:** Same pipeline and descriptor set serve lit meshes and sky — fragment shader branches on `push.skyMode`.

**In Spark:** `include/spark/engine/SceneRenderParams.hpp` SceneSkyMode

### ComponentKind::Sky on GameObject

**Concept:** SkyComponent marks intent

**Decision:** ComponentKind::Sky on GameObject

**Why:** Artists toggle sky enable, tint, mode, and optional texture without special-case demo code.

**In Spark:** `include/spark/ecs/components/rendering/SkyComponent.hpp`

### invViewProj × NDC → far world point

**Concept:** View-ray reconstruction

**Decision:** invViewProj × NDC → far world point

**Why:** Fixes equirect swimming when yaw/pitch; mesh only provides raster coverage.

**In Spark:** `shaders/scene.frag` sky branch (lines 384–424)

### Early return before BRDF

**Concept:** No lighting on sky fragments

**Decision:** Early return before BRDF

**Why:** Sky outputs radiance directly — sun disc and IBL come from environment texture or gradient, not N·L.

**In Spark:** `scene.frag` `if (push.skyMode != 0) { … return; }`

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/ecs/components/rendering/SkyComponent.hpp`
- `include/spark/engine/SceneRenderParams.hpp`
- `shaders/scene.frag`
- `src/spark/demo/SkyDemo.cpp`

### Step 1 — Read SceneSkyMode and SkyComponent

*SceneSkyMode — None, Box, Dome, Plane.*

```cpp
// include/spark/engine/SceneRenderParams.hpp (lines 50–56)
/** When set on a draw item, fragment/vertex use infinite-sky path (see scene.vert / scene.frag). */
enum class SceneSkyMode : std::uint8_t {
    None = 0,
    Box = 1,
    Dome = 2,
    Plane = 3,
};
```


*SkyComponent — mode, enabled, tint, optional skyTexture.*

```cpp
// include/spark/ecs/components/rendering/SkyComponent.hpp (lines 12–42)
/**
 * Marks a GameObject as an environment sky (use with MeshComponent + TransformComponent).
 * SceneSkyMode must match the mesh/shader path: Box = closed env (cube or sphere), Dome = hemisphere,
 * Plane = billboard quad (camera-facing in demos).
 */
class SkyComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::Sky;

    [[nodiscard]] ComponentKind Kind() const noexcept override { return TypeKind; }

    explicit SkyComponent(SceneSkyMode mode) noexcept : skyMode(mode) {}

    [[nodiscard]] SceneSkyMode GetSkyMode() const noexcept { return skyMode; }
    void SetSkyMode(SceneSkyMode m) noexcept { skyMode = m; }

    [[nodiscard]] bool IsSkyEnabled() const noexcept { return enabled; }
    void SetSkyEnabled(bool e) noexcept { enabled = e; }

    void SetTint(const Vector3& c) noexcept { tint = c; }
    [[nodiscard]] const Vector3& GetTint() const noexcept { return tint; }

    void SetSkyTexture(SharedPtr<Texture2D> t) { skyTexture = MoveTemp(t); }
    [[nodiscard]] const SharedPtr<Texture2D>& GetSkyTexture() const noexcept { return skyTexture; }

private:
    SceneSkyMode skyMode = SceneSkyMode::Box;
    bool enabled = true;
    Vector3 tint{0.25F, 0.35F, 0.55F};
    SharedPtr<Texture2D> skyTexture{};
};
```

### Step 2 — Spawn a sky entity like SkyDemo

```cpp
GameObject* skyObj = world.CreateGameObject();
skyObj->AddComponent<TransformComponent>();
skyObj->AddComponent<MeshComponent>(skySphereMesh, Vector3::One);
skyObj->AddComponent<SkyComponent>(SceneSkyMode::Box);
skyObj->AddComponent<MaterialComponent>()->SetBaseColorTexture(equirectTex);
```

Run Spark's `SkyDemo` and press **TAB** to cycle Box → Dome → Plane.

### Step 3 — Trace submit → draw item skyMode

*RigidDrawableSubmitSink — sky branch sets skyMode, tint, texture layer.*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 72–90)
        const SkyComponent* sky = o->GetComponent<SkyComponent>();
        if (sky != nullptr && sky->IsSkyEnabled()) {
            SceneDrawItem item{};
            item.mesh = SceneMeshSlot::Custom;
            item.skyMode = sky->GetSkyMode();
            item.model = worldM;
            item.customMesh = mc.GetMesh();
            item.albedo = sky->GetTint();
            item.textureLayer = -1;
            item.metallic = 0.0F;
            item.roughness = 1.0F;
            if (mat != nullptr && mat->GetBaseColorTexture()) {
                const Vector3& t = mat->GetTint();
                item.albedo = {item.albedo.x * t.x, item.albedo.y * t.y, item.albedo.z * t.z};
                item.textureLayer = findOrAddTexture(mat->GetBaseColorTexture());
            }
            item.shadowFlags = 0;
            drawList.PushBack(item);
            return;
```

### Step 4 — Study fragment sky modes

*Sky path — view ray, equirect sample, procedural Box/Dome/Plane gradients.*

```glsl
// shaders/scene.frag (lines 384–424)
    if (push.skyMode != 0) {
        vec3 cam = ubo.cameraPos.xyz;
        // Per-pixel world view ray — linear interpolation of vWorldPos is inside the sky mesh, not on the
        // sphere/plane, so normalize(interp - cam) warps when the camera rotates (HDR equirect swims).
        vec2 vp = max(ubo.viewportSize.xy, vec2(1.0));
        float sx = (gl_FragCoord.x + 0.5) / vp.x;
        float sy = (gl_FragCoord.y + 0.5) / vp.y;
        // Y: no extra sign flip — negating made equirect appear vertically inverted vs PerspectiveVulkan.
        vec2 ndc = vec2(sx * 2.0 - 1.0, sy * 2.0 - 1.0);
        vec4 farH = ubo.invViewProj * vec4(ndc, 1.0, 1.0);
        vec3 farW = farH.xyz / max(farH.w, 1e-5);
        vec3 dir = normalize(farW - cam);
        vec3 skyCol;
        // Textured sky (box / dome / plane): equirect from view ray. TryLoadFromFile uses
        // stbi_set_flip_vertically_on_load(1), so GPU v=0 is the image file bottom; equirect zenith is
        // usually at the file top → map zenith (mu=0) to v=1 via vEq = 1 - mu.
        if (vTextureLayer >= 0) {
            int layer = clamp(vTextureLayer, 0, 15);
            float phi = atan(dir.z, dir.x) / (2.0 * PI) + 0.5;
            phi = fract(phi);
            float mu = acos(clamp(dir.y, -1.0, 1.0)) / PI;
            float vEq = 1.0 - mu;
            vec4 tex = textureLod(sceneTextures, vec3(phi, vEq, float(layer)), 0.0);
            skyCol = sparkSrgbToLinear(tex.rgb) * push.albedoTint.rgb;
        } else if (push.skyMode == 1) {
            vec3 horizon = push.albedoTint.rgb;
            vec3 zenith = mix(horizon, vec3(0.45, 0.62, 0.95), 0.65);
            skyCol = mix(horizon, zenith, clamp(dir.y * 0.5 + 0.5, 0.0, 1.0));
        } else if (push.skyMode == 2) {
            float mu = acos(clamp(dir.y, -1.0, 1.0)) / PI;
            vec3 low = push.albedoTint.rgb;
            vec3 high = vec3(0.55, 0.78, 1.0);
            skyCol = mix(low, high, clamp(1.0 - mu, 0.0, 1.0));
        } else {
            float h = clamp(vTexCoord.y, 0.0, 1.0);
            vec3 bottom = push.albedoTint.rgb;
            vec3 top = vec3(0.75, 0.88, 1.0);
            skyCol = mix(bottom, top, h);
        }
        outColor = vec4(skyCol, 1.0);
        return;
```

### Step 5 — Compare textured vs procedural

In `SkyDemo::ApplySkyModeVisuals`, note HDR path binds equirect on `MaterialComponent`; missing file falls back to tint-only procedural gradients. Toggle by renaming the asset path temporarily.

### Step 6 — Milestone — three modes under fly camera

You should see: rotating camera with **stable** horizon (no swimming), TAB cycles modes, reference cube still lit by standard BRDF. Capture one frame per mode for your notes.

## 5. How This Connects to the Rest of Your Engine

**Sky Geometry** builds the sphere/plane meshes this chapter references. **Sky Pipeline Branch** binds `pipelineSky`
and sort order so opaque props draw over the background. **IBL Basics** can reuse the same equirect layer registered
on the sky draw item.

## 6. Pitfalls and Debugging

- **Interpolating vWorldPos for direction** — equirect swims on camera rotation; use per-pixel rays like Spark.
- **Sky with BRDF enabled** — dark or incorrectly shaded background; early-out when `skyMode != 0`.
- **Wrong equirect V flip** — zenith at bottom; Spark maps `vEq = 1.0 - mu` to match stbi vertical flip.
- **Multiple enabled skies** — each submits a draw; keep one active sky per scene unless layering intentionally.

## 7. Exercises

1. Log `item.skyMode` for every draw in a frame — exactly one should be non-None.
2. Rotate camera 360° on Box mode with HDR texture; confirm horizon line stays level.
3. Map each `SceneSkyMode` value to the procedural gradient branch in `scene.frag`.
4. Disable `SkyComponent` and confirm lit meshes still render with clear color gaps.
5. Compare textured Box vs Dome — note identical equirect math, different procedural fallback.

## Summary

SceneSkyMode selects sky shading path on each draw item; SkyComponent + mesh + optional texture drive Box/Dome/Plane with view-ray equirect sampling.

---

Next: [Sky Geometry](02-sky-meshes.md).
