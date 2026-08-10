# Punctual Shadows

> **Build-Along Milestone:** One spot light casts a visible shadow on the ground plane in RenderDoc.

**You will learn:**

- Render spot-light shadow maps with perspective projection.
- Outline point-light cube shadow maps.
- Bind punctual shadow maps separately from the directional CSM atlas.

## 1. The Problem We Are Solving

Directional CSM covers the sun; **spot** and **point** lights need **perspective** depth maps from the light origin. Your forward renderer already loops punctual lights for shading (Part IV); Part IX adds optional shadowed variants gated by light flags and shadow map slots.

Spark implements punctual shadows as a separate pass object with its own SSBO and sampler bindings.

## 2. Core Concepts

### Spot light shadow

A spot light uses one **perspective** depth map — same shape as a camera frustum aligned to the spot direction and outer cone angle.


### Point light shadow

A point light traditionally renders **six cube faces** (or fewer with dual-paraboloid approximations). Each face uses a 90° perspective projection from the light origin.


### Shadow slot budget

Each shadow-casting punctual light costs a pass plus memory. Cap active shadowed lights per frame and assign by distance/priority — Spark uses fixed max spot and point slot counts in GLSL.


### Separate from directional atlas

Directional CSM and punctual maps use different projections, biases, and atlas layouts — bind them on separate descriptor bindings rather than forcing one atlas scheme.


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


### Spot perspective frustum

Spot lights use **perspective** projection aligned to local **−Z** axis in world space. Outer cone angle must match between shading falloff and shadow projection — mismatch leaks light outside the shadow frustum at cone edges.

Each spot shadow slot stores `spotWorldToClip` and `spotAtlas` rect in `PunctualShadowSSBO`. Scene shader maps light index → slot via `spotShadowSlotByLight[]`.


### Point cube face selection

Point lights render **six 90° perspective faces** stored in a 2D array texture. `selectPointShadowFace` picks the face from the dominant axis of the vector from light to fragment — standard cube-map shadow mapping.

```text
face 0/1: ±X   face 2/3: ±Y   face 4/5: ±Z
layer = pointBaseLayer[slot] + face
```

Perspective foreshortening near the light origin increases acne — bias tuning differs from directional (Chapter 8).


### Shadow slot budgeting

Spark caps **4 spot** and **2 point** shadow maps per frame. Gameplay must assign slots to nearest `castsShadow` lights. Drawing every punctual light with shadows explodes frame time — profile CSM-only vs +1 spot before enabling full night scenes.


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


### PunctualShadowPass record order

Punctual pass runs **before** directional in `RecordShadowMapPass` — spot/point atlases populate first while descriptor bindings 6–8 remain dedicated. `PrepareAndUpload` fills SSBO slot tables mapping clustered light indices to shadow slots; shader uses `spotShadowSlotByLight` lookup during punctual BRDF accumulation.


### Spot atlas layout

Four spot shadows pack into 1024² atlas as 512² tiles — `spotAtlas` vec4 stores UV scale/bias per slot. Scene shader converts light clip UV to atlas UV before PCF. Point shadows use `sampler2DArray` with six layers per point slot — `pointBaseLayer` indexes array base.

PrepareAndUpload selects which lights receive slots this frame — overflow logs when more than kMaxSpotShadowMaps lights request shadows.


### Cone angle alignment

Spot light shading uses outer cone cosine from `SceneSpotLight`; shadow projection must use the same outer angle for the perspective matrix field-of-view. Axis alignment: Spark uses object local **−Z** as spot direction in world space — rotating the GameObject rotates both shading and shadow frustum together when components share transform.


### VulkanPunctualShadowPass structure

Punctual pass owns separate render pass, atlas images, and record function from directional CSM. `Record` filters `SceneSpotLight` / `ScenePointLight` arrays for `castsShadow`, assigns slots, renders depth per tile/face using same `shadow_depth.vert` and `RecordShadowCastMeshes`. Descriptor bindings 6–8 on scene set must be populated before scene pass records.

When porting, implement **one spot shadow** end-to-end before point cube maps — spot uses single perspective matrix; point requires six face matrices per slot.


### Shader slot lookup tables

`spotShadowSlotByLight[kMaxClusteredSpotLights]` maps clustered light index → shadow slot or −1. CPU fills during PrepareAndUpload; shader checks slot before PCF to avoid sampling empty atlas tiles. Overflow: when fifth shadow-casting spot competes, log and skip — gameplay should prioritize nearest lights to camera.


### Memory footprint estimate

Four spot tiles at 512² D32 ≈ 4 MB; two point cubes × 6 faces × 512² ≈ 12 MB — punctual shadow memory rivals directional atlas. Disable punctual shadows in 2D-only games (Part XI) — CSM alone suffices for sun. Profile with timestamp queries: punctual record adds draw calls equal to active shadow lights × caster count.


### Flashlight spot test scene

Minimal punctual shadow test: spot light parented to camera, narrow outer cone, one ground plane receiver, one caster cube. Confirm perspective depth pass, SSBO slot 0 assignment, and scene shader PCF branch before enabling four spot slots or point cube maps. Measure frame time delta vs sun-only baseline on target GPU.


### Point shadow face matrices

Each point shadow slot stores six `pointFaceWorldToClip[slot][face]` matrices — CPU renders six passes or geometry shader/multiview equivalent per point light. Face selection on GPU uses dominant axis of `worldPos - lightPos`. Near the light, perspective aliasing is severe — tighten near plane and increase bias (Chapter 8) before raising map resolution.

Spark stores faces in 2D array layers — `pointBaseLayer[slot] + face` indexes `sampler2DArray pointShadowMap` at binding 8.


### Integration with clustered forward

Punctual shadow lookup happens inside clustered light loops — only lights with assigned shadow slots pay PCF cost. Directional sun shadow is separate code path in main sun BRDF block. When disabling punctual shadows globally via `punctualShadowsEnabled`, SSBO may still upload — skip record pass entirely to save fill rate.


### Spot shadow RenderDoc validation

Select spot shadow pass tile in RenderDoc — depth should show cone-frustum clipped scene. Scene pass spot BRDF draw should sample binding 7 at PCF UVs inside atlas rect from SSBO slot 0. Mismatch between atlas rect in SSBO and viewport used during spot shadow record produces shifted punctual shadows independent of directional CSM quality.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Dedicated punctual shadow pass

**Concept:** Spot light shadow

**Decision:** Dedicated punctual shadow pass

**Why:** `VulkanPunctualShadowPass` records spot and point depth draws after directional CSM, sharing the depth-only pipeline and caster filter.

**In Spark:** `punctualShadow.Record` in `RecordShadowMapPass`.

### Cube map array layers

**Concept:** Point light shadow

**Decision:** Cube map array layers

**Why:** Store six face renders in a 2D array or cube-compatible image; fragment shader selects face from vector to fragment.

**In Spark:** `pointShadowMap` sampler2DArray and `selectPointShadowFace` in `punctual_shadows.glsl`.

### Fixed SSBO slot tables

**Concept:** Shadow slot budget

**Decision:** Fixed SSBO slot tables

**Why:** Upload `spotLightIndex[]` and `pointLightIndex[]` mapping clustered lights to shadow slots — shader looks up slot by light index.

**In Spark:** `PunctualShadowSSBO` in `punctual_shadows.glsl` lines 9–25.

### Bindings 7 and 8 for punctual maps

**Concept:** Separate from directional atlas

**Decision:** Bindings 7 and 8 for punctual maps

**Why:** Keeps descriptor layout explicit — scene shader samples directional map and punctual maps independently.

**In Spark:** `spotShadowMap` binding 7, `pointShadowMap` binding 8.

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

- `src/spark/render/shadow/VulkanPunctualShadowPass.cpp`
- `shaders/punctual_shadows.glsl`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Read punctual shadow SSBO layout

The SSBO carries slot counts, light index maps, world-to-clip matrices, and atlas rects — mirror this layout if you port punctual shadows.

*Punctual shadow SSBO and samplers*

```glsl
// shaders/punctual_shadows.glsl (lines 9–28)
layout(std430, set = 0, binding = 6) readonly buffer PunctualShadowSSBO {
    uint numSpotShadows;
    uint numPointShadows;
    uint punctualShadowEnabled;
    uint _psPad0;
    int spotLightIndex[kMaxSpotShadowMaps];
    int pointLightIndex[kMaxPointShadowMaps];
    int _psPad1[2];
    mat4 spotWorldToClip[kMaxSpotShadowMaps];
    vec4 spotAtlas[kMaxSpotShadowMaps];
    vec4 pointPosRange[kMaxPointShadowMaps];
    uint pointBaseLayer[kMaxPointShadowMaps];
    uint _psPad2[2];
    mat4 pointFaceWorldToClip[kMaxPointShadowMaps][6];
    int pointShadowSlotByLight[kMaxClusteredPointLights];
    int spotShadowSlotByLight[kMaxClusteredSpotLights];
} punctualShadow;

layout(set = 0, binding = 7) uniform sampler2D spotShadowMap;
layout(set = 0, binding = 8) uniform sampler2DArray pointShadowMap;
```

### Step 2 — Study point face selection

The shader picks the cube face from the dominant axis of the vector from light to fragment.

*Point shadow face select*

```glsl
// shaders/punctual_shadows.glsl (lines 75–90)
int selectPointShadowFace(int slot, vec3 worldPos) {
    vec3 toFrag = worldPos - punctualShadow.pointPosRange[slot].xyz;
    vec3 a = abs(toFrag);
    if (a.x >= a.y && a.x >= a.z) {
        return toFrag.x > 0.0 ? 0 : 1;
    }
    if (a.y >= a.z) {
        return toFrag.y > 0.0 ? 2 : 3;
    }
    return toFrag.z > 0.0 ? 4 : 5;
}

float samplePointShadowAtSlot(int slot, vec3 worldPos, vec3 N, vec3 L) {
    vec3 lightPos = punctualShadow.pointPosRange[slot].xyz;
    float range = max(punctualShadow.pointPosRange[slot].w, 1e-3);
    vec3 toFrag = worldPos - lightPos;
```

### Step 3 — Trace PrepareAndUpload

Punctual shadow matrices upload during uniform buffer write — before scene recording.

*Punctual shadow upload before scene UBO*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 649–655)
void VulkanRenderer::WriteUniformBuffer(std::uint32_t frameIndex) {
    void* const uniformMapped = sceneDescriptors.UniformMapped(frameIndex);
    if (!sceneParamsValid || uniformMapped == nullptr) {
        return;
    }
    punctualShadow.PrepareAndUpload(frameIndex, pendingScene, resolvedLighting, punctualShadowFrameState);
    clusteredForwardLights.BuildAndUpload(
```

### Step 4 — Implement one spot shadow first

Follow the camera with a flashlight-style spot. Confirm perspective frustum, depth pass, and scene shader branch before adding point cube faces.

### Step 5 — Cap shadowed lights

Assign shadow slots to the nearest shadow-casting spots/points each frame. Log when requests exceed `kMaxSpotShadowMaps`.

### Step 6 — Profile pass cost

Measure GPU time: directional CSM only vs +1 spot shadow pass. Budget memory: four 1024² spot maps ≈ 16 MB depth.

### Step 9 — Read punctual shadow SSBO

Slot tables and atlas rects upload to binding 6 — mirror layout when porting.

*PunctualShadowSSBO layout*

```glsl
// shaders/punctual_shadows.glsl (lines 9–25)
layout(std430, set = 0, binding = 6) readonly buffer PunctualShadowSSBO {
    uint numSpotShadows;
    uint numPointShadows;
    uint punctualShadowEnabled;
    uint _psPad0;
    int spotLightIndex[kMaxSpotShadowMaps];
    int pointLightIndex[kMaxPointShadowMaps];
    int _psPad1[2];
    mat4 spotWorldToClip[kMaxSpotShadowMaps];
    vec4 spotAtlas[kMaxSpotShadowMaps];
    vec4 pointPosRange[kMaxPointShadowMaps];
    uint pointBaseLayer[kMaxPointShadowMaps];
    uint _psPad2[2];
    mat4 pointFaceWorldToClip[kMaxPointShadowMaps][6];
    int pointShadowSlotByLight[kMaxClusteredPointLights];
    int spotShadowSlotByLight[kMaxClusteredSpotLights];
} punctualShadow;
```

## 5. How This Connects to the Rest of Your Engine

Punctual shadows share **caster lists** and **depth-only pipeline** with CSM (Chapter 3). Part **XI** 2D games may disable punctual shadows entirely. Part **X** HDR does not change shadow pass order.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Spot angle mismatch** between shading cone and shadow proj — leaks at cone edge.
- **Point light acne** — perspective aliasing; use larger bias and careful near plane.
- **Drawing all lights shadowed** — frame time explodes; prioritize closest to camera.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Implement one spot shadow for a flashlight following the camera.
2. Visualize spot shadow frustum as wireframe in debug draw.
3. Measure GPU time: CSM only vs +1 spot shadow pass.
4. List `kMaxSpotShadowMaps` and `kMaxPointShadowMaps` in Spark GLSL.
5. Compare punctual 3×3 PCF bias constants vs directional 5×5.

## Summary

Spot lights use a single perspective depth map; point lights need multi-face cube rendering. Spark schedules punctual passes in `RecordShadowMapPass` with fixed slot tables in GLSL.

---

Next: [Cast and Receive Flags](06-shadow-flags.md).
