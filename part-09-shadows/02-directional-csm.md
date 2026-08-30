# Directional CSM

> **Build-Along Milestone:** Cascade false-color debug shows index 0 near the camera and higher indices at distance.

**You will learn:**

- Split the view frustum into cascades with stable orthographic light projections.
- Pack cascade depth views into one atlas.
- Upload per-cascade matrices and atlas UV rects to the scene UBO.

## 1. The Problem We Are Solving

One orthographic shadow map covering an entire open level blurs distant shadows and wastes resolution near the camera. **Cascaded shadow maps** render multiple light-space depth maps tuned to frustum slices, packed into a single atlas.

You will follow Spark's CPU cascade math, then confirm the GPU receives matching `worldToShadowClip` and `cascadeAtlas` arrays each frame.

### Why one shadow map is not enough (beginner)

Imagine a sun shadow map covering an entire open world at 2048×2048 texels. Near your character's feet, each texel might cover **50 cm** of ground — shadows look like chunky blocks. Mountains kilometers away get the same texel budget — wasted resolution.

**Cascaded Shadow Maps (CSM)** split the camera view into **near, mid, far** slices. Each slice gets its own shadow map (or region in an atlas) with an orthographic light projection fitted tightly to that slice:

```text
Camera frustum side view:

  near slice  ──► high-res shadow map (crisp character shadow)
  mid slice   ──► medium-res map
  far slice   ──► low-res map (distant hills)
```

The fragment shader picks which cascade to sample based on how far the pixel is from the camera.

### Step-by-step cascade flow in Spark

1. **CPU:** Compute camera frustum corner distances → split distances λ-blended near/far.
2. **CPU:** For each cascade, build orthographic light matrix enclosing that frustum slice.
3. **CPU:** Snap projection to texel grid (reduces shadow edge shimmering when camera moves).
4. **GPU:** Render depth-only pass per cascade into atlas regions.
5. **CPU:** Upload `worldToShadowClip[cascade]` + atlas UV rects to scene UBO.
6. **GPU:** Main pass fragment shader selects cascade, samples depth, applies PCF.

Beginners should visualize cascade **false-color debug** (Spark demo / RenderDoc) before tuning bias — confirms index 0 is near camera.

## 1b. Concept Deep Dive — VulkanDirectionalShadowCascadeMath

### λ-blended split distances

```cpp
// src/spark/render/shadow/VulkanDirectionalShadowCascadeMath.cpp (lines 31–41)
float ComputeSplitDistance(float nearDist, float farDist, int splitIndex,
                           int splitCount, float lambda) {
    const float p = static_cast<float>(splitIndex) / static_cast<float>(splitCount);
    const float logSplit = nearDist * std::pow(farDist / std::max(nearDist, 1e-3F), p);
    const float uniSplit = nearDist + (farDist - nearDist) * p;
    return lambda * logSplit + (1.0F - lambda) * uniSplit;
}
```

λ = 0.5 mixes logarithmic (more resolution near camera) with uniform splits. Spark uses four cascades; split distances feed both shadow pass viewport scissor and `ubo.cascadeSplits` for fragment shader selection.

### Ortho light fit with texel snapping

```cpp
// ComputeWorldToShadowClip — frustum slice → light-space AABB (concept)
const Matrix4 lightView = Matrix4::LookAt(lightPos, center, up);
// accumulate frustum corners + ground grid in light view
// pad X/Y/Z, then snap minX/minY to texel grid:
minX = std::floor((minX - cx) / texelX) * texelX + cx;
```

Snapping the orthographic projection to shadow-map texel boundaries reduces **shadow edge shimmering** when the camera translates. Ground corners at ±`kSceneGroundHalfExtent` are included so terrain near the player stays inside the near cascade.

### Z-flip matrix for Vulkan depth convention

```cpp
// ShadowClipLinearZFlip() — maps OpenGL-style [0,1] depth to Vulkan [0,1] with reversed Z
static const Matrix4 k = [] {
    Matrix4 r = Matrix4::IdentityMatrix();
    r.m[10] = -1.0F;
    r.m[14] = 1.0F;
    return r;
}();
// worldToShadowClip = ZFlip * lightProj * lightView
```

The same matrix is uploaded to the scene UBO and pushed to `shadow_depth.vert` via `ShadowPushConstants.lightViewProj` — matrix mismatch between passes is a common bug source.

## 2. Core Concepts

### Cascade splits

One orthographic shadow map for an entire open level wastes texels near the camera. **Cascaded shadow maps (CSM)** render multiple light-space frusta aligned to slices of the camera frustum.

A common split scheme mixes logarithmic and uniform distances:

```text
λ = 0.5
split[i] = λ * (n * (f/n)^(i/C)) + (1-λ) * (n + (f-n)*i/C)
```

Near cascades get high texel density; far cascades cover the horizon.


### Ortho light fit

For each cascade slice, transform frustum corners to light space, take an axis-aligned bounding box, and optionally **snap texel grid** to reduce shadow shimmering when the camera moves.

```text
texelSize = (maxX - minX) / shadowMapRes
minX = floor(minX / texelSize) * texelSize
```


### Atlas packing

Multiple cascades can share one depth image with sub-rect viewports:

```text
┌─────────┬─────────┐
│ C0      │ C1      │
├─────────┼─────────┤
│ C2      │ C3      │
└─────────┴─────────┘
```

The fragment shader maps light UV to atlas coordinates with per-cascade scale/bias in the scene UBO.


### Z-flip convention

Vulkan depth range and your engine's projection conventions must match between shadow pass output and scene compare. Spark applies a linear Z-flip matrix when building `worldToShadowClip` — keep CPU and GPU on the same convention.


### Build-along verification checklist

After each milestone, verify four contracts before adding features:

| Check | Tool | Pass criteria |
|-------|------|---------------|
| Data layout | RenderDoc | Push/UBO bytes match shader struct |
| Pass order | RenderDoc event list | Dependencies before consumers |
| Format | Texture viewer | D32 / RGBA16F as documented |
| Visual | Demo run | No swimming, acne, or black pass |

When output disagrees with Spark, diff **data paths** (struct fields, matrix multiply order, binding indices) — not just Vulkan object names. Validation VUID strings link to spec language for the exact contract violated.


### Matrix and space conventions

Spark uses **column-major** `mat4` in GLSL matching C++ `Matrix4` storage. World-to-shadow transforms apply as:

```text
clip = worldToShadowClip * vec4(P_world, 1)
```

Normal vectors for bias use world-space **N** dotted with light direction **L** — not tangent space. Cascade selection in Spark uses **camera distance** from `length(vWorldPos - cameraPos)`, not clip-space Z, so split thresholds stay intuitive when the camera tilts.

Document your engine's handedness (Y-up, Z-forward) beside Spark's when porting cascade math — a transposed multiply shows up as shadows sliding opposite camera motion.


### Practical split tuning

Outdoor scenes typically use **λ = 0.5** between logarithmic and uniform splits. Logarithmic allocation concentrates cascades near the camera where parallax is highest; uniform splits prevent the farthest cascade from covering an enormous depth range with too few texels.

```text
i = cascade index (0 = nearest)
C = cascade count (Spark: 4)
n, f = practical near/far for split math (not always clip planes)
split[i] = λ * n * (f/n)^(i/C) + (1-λ) * (n + (f-n)*i/C)
```

When the camera **far plane** changes in editor, re-log split distances — artists notice seams first at split boundaries on flat ground. Start tuning with **two cascades** before enabling four; false-color debug (Chapter 4) tints by cascade index.


### Texel snapping stability

Moving the camera translates the light-space ortho box every frame. Without **texel snapping**, shadow edges shimmer as the quantized depth map slides against world geometry:

```text
texelSize = (lightSpaceMax.x - lightSpaceMin.x) / atlasTileResolution
minX = floor(minX / texelSize) * texelSize
maxX = minX + texelSize * atlasTileResolution
```

Snap in **light space** after fitting the frustum slice AABB. The stable texel grid trades slight shadow edge jitter at cascade boundaries for dramatically reduced swimming on static meshes.


### Cascade blend in scene.frag

Spark soft-blends between adjacent cascades near split distances using `smoothstep` on camera distance — see `blendSunShadowCascades` in `scene.frag`. The UBO field `timeGlobal.w` stores blend fraction; `cascadeBlendFraction` on the lighting profile sets artist-controlled overlap.

Without blend, texel resolution jumps at splits look like a **sharp quality line** on terrain. Blend weights two PCF results — costs extra texture taps only in a narrow band around each split.


### RenderDoc shadow debugging workflow

Capture a frame with **ThreeDDemo** shadows enabled. In the event browser, expand the directional shadow pass and confirm indexed draws for casters. Select a shadow draw → verify push constant bytes for `lightViewProj` match the matrix uploaded to UBO for the same cascade index.

Switch to the HDR scene pass → select an opaque draw → fragment shader inputs should show `worldToShadowClip` arrays in the UBO. Texture viewer on binding 3 should show depth atlas tiles — not cleared white.

Common false leads: debugging PCF when casters never drew (empty shadow pass); tuning bias when projection is inverted (whole shadow map slides). Always confirm **pass order** and **non-empty shadow pass** before shader math.


### Memory and bandwidth budget

Directional CSM atlas at 2048² D32 ≈ 16 MB per flight target. Four spot 512² tiles plus two point cube faces add punctual memory on top. Budget:

```text
bytes ≈ width * height * 4 (D32) * cascades_or_faces * framesInFlight
```

PCF sample cost scales with scene resolution × kernel taps × cascade blend bands. Spark's 5×5 directional PCF with cascade blend is ~25–50 extra depth fetches per pixel in worst case — profile before enabling on mobile targets.

Reduce cost: lower atlas resolution on far cascades only (requires split atlas sizes), fewer punctual shadow slots, or smaller PCF kernel on distant LOD materials.


### ComputeWorldToShadowClip walkthrough

Spark unprojects NDC frustum corners through **inverse view-projection**, interpolates near/far slice bounds, transforms world points to light space, accumulates axis-aligned min/max, applies texel snap, builds orthographic projection, then multiplies by `ShadowClipLinearZFlip()` and light view matrix. The output `worldToShadowClip` is what **both** shadow pass push constants and scene UBO must share for cascade index *i*.

Ground extent hooks optionally expand bounds using scene ground AABB — prevents shadow casters near the frustum edge from clipping out of the ortho box on terrain scenes. When debugging missing terrain shadows, inspect whether ground extent expansion is active for your level.


### Atlas viewport record loop

Each cascade calls `vkCmdSetViewport` with origin offset into the atlas image — tile size typically 1024 within 2048 atlas. Clear depth once per cascade sub-rect or once per full atlas depending on implementation — Spark clears per cascade render pass begin. Scissor must match viewport to avoid bleeding draws into neighboring tiles.

In RenderDoc, verify four viewport regions fill distinct quadrants of the depth texture when four cascades are enabled.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### CPU cascade math module

**Concept:** Cascade splits

**Decision:** CPU cascade math module

**Why:** Pure functions for splits, light view, ortho projection, and atlas rects can be unit-tested without Vulkan.

**In Spark:** `VulkanDirectionalShadowCascadeMath` in `src/spark/render/shadow/VulkanDirectionalShadowCascadeMath.cpp`.

### Frustum corner unprojection

**Concept:** Ortho light fit

**Decision:** Frustum corner unprojection

**Why:** Invert the camera view-projection, unproject NDC frustum corners, transform to light space, and fit an ortho box around the slice.

**In Spark:** `ComputeWorldToShadowClip` accumulates light-space bounds from eight NDC corners.

### One depth image, multiple viewports

**Concept:** Atlas packing

**Decision:** One depth image, multiple viewports

**Why:** Cheaper than separate images on many drivers — each cascade renders into a sub-rectangle with `vkCmdSetViewport`.

**In Spark:** `VulkanDirectionalShadowPass::Record` loops cascades with atlas viewport offsets.

### Shared Z-flip matrix

**Concept:** Z-flip convention

**Decision:** Shared Z-flip matrix

**Why:** Apply the same flip when writing shadow pass matrices and when uploading `worldToShadowClip` to the scene UBO.

**In Spark:** `ShadowClipLinearZFlip()` returns a static matrix used in cascade math.

### RenderDoc-first debugging

**Concept:** Build-along verification checklist

**Decision:** RenderDoc-first debugging

**Why:** Capture one known-good frame from ThreeDDemo before changing splits, bias, or HDR order — regression compare pass list and bound descriptors each milestone.

**In Spark:** Demo README capture instructions; compare against Spark reference captures.

### Trace comment in cascade math

**Concept:** Matrix and space conventions

**Decision:** Trace comment in cascade math

**Why:** Spark documents CPU→GPU shadow path in source comments — update your port's equivalent when conventions change.

**In Spark:** `VulkanDirectionalShadowCascadeMath.cpp` lines 13–20 pipeline trace.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/shadow/VulkanDirectionalShadowCascadeMath.cpp`
- `src/spark/render/shadow/VulkanDirectionalShadowPass.cpp`
- `src/spark/render/scene/VulkanSceneUniformWriter.cpp`

### Step 1 — Read the cascade math trace comment

Spark documents the full pipeline from light direction → CPU matrix → shadow push constants → scene fragment compare. Read this comment before editing splits.

*Shadow pipeline trace and Z-flip*

```cpp
// src/spark/render/shadow/VulkanDirectionalShadowCascadeMath.cpp (lines 13–29)
/*
 * Shadow pipeline trace (single directional cascade):
 * 1) VulkanSceneUniformWriter: lightDir = normalize(scene.lightDirectionWorld) — toward sun.
 * 2) ComputeWorldToShadowClip(...) per cascade → worldToShadowClip[i] = ZFlip * lightProj * lightView.
 * 3) VulkanDirectionalShadowPass::Record: same matrix in ShadowPushConstants.lightViewProj.
 * 4) scene.frag: light clip from vWorldPos; depth compare uses slope-scaled bias.
 * 5) scene.vert: transpose(inverse(mat3(model))) * n for non-uniform scale.
 */
const Matrix4& VulkanDirectionalShadowCascadeMath::ShadowClipLinearZFlip() noexcept {
    static const Matrix4 k = [] {
        Matrix4 r = Matrix4::IdentityMatrix();
        r.m[10] = -1.0F;
        r.m[14] = 1.0F;
        return r;
    }();
    return k;
}
```

### Step 2 — Study split distance formula

The lambda mix between logarithmic and uniform splits is the default starting point for outdoor scenes.

*ComputeSplitDistance*

```cpp
// src/spark/render/shadow/VulkanDirectionalShadowCascadeMath.cpp (lines 31–41)
float VulkanDirectionalShadowCascadeMath::ComputeSplitDistance(
        const float nearDist,
        const float farDist,
        const int splitIndex,
        const int splitCount,
        const float lambda) noexcept {
    const float p = static_cast<float>(splitIndex) / static_cast<float>(splitCount);
    const float logSplit = nearDist * std::pow(farDist / std::max(nearDist, 1e-3F), p);
    const float uniSplit = nearDist + (farDist - nearDist) * p;
    return lambda * logSplit + (1.0F - lambda) * uniSplit;
}
```

### Step 3 — Follow ortho box fitting

`ComputeWorldToShadowClip` unprojects frustum corners, builds a light view matrix, and accumulates min/max in light space before orthographic projection.

*Light view from frustum center*

```cpp
// src/spark/render/shadow/VulkanDirectionalShadowCascadeMath.cpp (lines 43–86)
bool VulkanDirectionalShadowCascadeMath::ComputeWorldToShadowClip(
        const Matrix4& viewProj,
        const Vector3& lightDirectionWorldTowardLight,
        const float distNear,
        const float distFar,
        const std::uint32_t cascadeTileSize,
        Matrix4& outWorldToShadowClip) {
    Matrix4 invVp{};
    if (!viewProj.TryInvert(invVp)) {
        return false;
    }
    const Vector4 ndcCorners[8] = {
            {-1.0F, -1.0F, 0.0F, 1.0F},
            {1.0F, -1.0F, 0.0F, 1.0F},
            {1.0F, 1.0F, 0.0F, 1.0F},
            {-1.0F, 1.0F, 0.0F, 1.0F},
            {-1.0F, -1.0F, 1.0F, 1.0F},
            {1.0F, -1.0F, 1.0F, 1.0F},
            {1.0F, 1.0F, 1.0F, 1.0F},
            {-1.0F, 1.0F, 1.0F, 1.0F},
    };
    Vector3 center{};
    for (const Vector4& c : ndcCorners) {
        const Vector4 w = invVp * c;
        const float iw = (w.w != 0.0F) ? (1.0F / w.w) : 1.0F;
        center.x += w.x * iw;
        center.y += w.y * iw;
        center.z += w.z * iw;
    }
    constexpr float inv8 = 1.0F / 8.0F;
    center.x *= inv8;
    center.y *= inv8;
    center.z *= inv8;

    const Vector3 L = lightDirectionWorldTowardLight.Normalized();
    constexpr float kPullDist = 260.0F;
    const Vector3 lightPos = center + L * kPullDist;

    Vector3 up = Vector3::UnitY;
    if (std::fabs(Vector3::Dot(L, up)) > 0.95F) {
        up = Vector3::UnitX;
    }
    const Matrix4 lightView = Matrix4::LookAt(lightPos, center, up);

```

### Step 4 — Confirm UBO upload path

Cascade matrices flow through `sceneUniformWriter.Write` after `directionalShadow` prepares per-frame state. When you tune splits, set a breakpoint here — not in the fragment shader.

*Uniform write includes shadow state*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 649–668)
void VulkanRenderer::WriteUniformBuffer(std::uint32_t frameIndex) {
    void* const uniformMapped = sceneDescriptors.UniformMapped(frameIndex);
    if (!sceneParamsValid || uniformMapped == nullptr) {
        return;
    }
    punctualShadow.PrepareAndUpload(frameIndex, pendingScene, resolvedLighting, punctualShadowFrameState);
    clusteredForwardLights.BuildAndUpload(
            frameIndex,
            pendingScene,
            resolvedLighting,
            presentSwapchain().extent);
    sceneUniformWriter.Write(
            uniformMapped,
            pendingScene,
            resolvedLighting,
            presentSwapchain().extent,
            directionalShadow,
            frameIndex,
            shadowFrameState);
}
```

### Step 5 — Debug with cascade false color

Add a temporary debug mode in your scene fragment shader that tints by cascade index. Walk the camera forward — index should increase as view-space depth crosses split thresholds.

Start with **two cascades** in a 1024² atlas before enabling four.

### Step 6 — Validate on TerrainDemo

Outdoor terrain shows cascade seams and texel density problems quickly. Run Spark **TerrainDemo**, capture RenderDoc, and measure world-space texel size at 10 m vs 200 m from the camera.

## 5. How This Connects to the Rest of Your Engine

CSM matrices live in the frame UBO from Part III. Part **XII** terrain benefits strongly from near-cascade detail on ground near the player. Part **X** HDR tonemap does not change cascade selection — shadows modulate radiance before tonemap.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Seams between cascades** — different biases per cascade; slight split overlap helps.
- **Atlas bleed** — add gutter texels or clamp UVs when PCF kernels straddle boundaries.
- **Moving sun every frame** — temporal instability; snap or slow-update light direction outdoors.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Implement 2 cascades only; visualize cascade index as false color.
2. Log split distances when the camera far plane changes.
3. Measure texel size in world meters at 10 m vs 200 m on TerrainDemo.
4. Log `cascadeAtlas[i]` scale/bias for each cascade after camera move.
5. Disable texel snapping — observe shadow swimming on ground plane.

## Summary

CSM renders multiple light-space depth maps for frustum slices, packed in one atlas. Spark computes splits and ortho boxes on CPU in `VulkanDirectionalShadowCascadeMath` and uploads matrices before the scene pass.

---

Next: [Shadow Depth Shader](03-shadow-depth-shader.md).
