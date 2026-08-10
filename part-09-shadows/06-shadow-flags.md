# Cast and Receive Flags

> **Build-Along Milestone:** A mesh can cast a shadow on the floor without receiving shadows on itself.

**You will learn:**

- Add per-draw cast and receive shadow flags to the render snapshot.
- Filter shadow pass draw lists to casters only.
- Skip shadow multiplication for non-receivers in the shader.

## 1. The Problem We Are Solving

Not every mesh should **cast** shadows (particles, invisible proxies). Not every surface should **receive** them (skies, emissive billboards). Without explicit flags you waste shadow pass fill rate and darken glowing surfaces incorrectly.

Spark stores these flags on scene draw items and filters at submit and record time.

## 2. Core Concepts

### Cast vs receive

Two independent booleans control complementary behavior:

| Flag | Shadow pass | Scene pass |
|------|-------------|------------|
| `castsShadow` | Include in depth draws | — |
| `receivesShadow` | — | Multiply sun term by visibility |


### CPU filtering

Build `shadowDraws[]` during snapshot by skipping non-casters — cheaper than GPU discard for large scenes.


### Defaults by draw type

Opaque world geometry: both true. Particles and screen UI: neither. Skies: never cast or receive.


### Serialization

Cast/receive flags should persist on prefabs in scene files (Part XIII) so authored content round-trips through the editor.


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


### Shadow flag bitfield

```cpp
kSceneShadowCast    = 1
kSceneShadowReceive = 2
kSceneShadowCastAndReceive = 3
```

`SceneDrawItem::shadowFlags` defaults from `DefaultShadowFlagsFor(resolvedLighting)` using profile `defaultCast` / `defaultReceive`. Per-draw overrides come from mesh/material authoring at submit time.


### CPU filter vs GPU branch

Shadow pass: skip draws without cast bit **before** issuing draw calls — saves vertex processing. Scene pass: branch **before** PCF texture fetches when receive bit clear — saves bandwidth on emissive/UI-proxy geometry that still should not darken.

Skies use `SceneSkyMode` and skip shadow pass entirely in `RecordShadowCastMeshes`.


### Light-level castsShadow

Point/spot lights carry separate `castsShadow` on GPU structs (`ScenePointLight`, `SceneSpotLight`). Punctual shadow pass filters lights before allocating atlas tiles — independent from mesh cast flags.


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


### SceneLightingProfile defaults

`DefaultShadowFlagsFor` reads `resolved.defaultShadowCast` and `defaultReceive` from outdoor/night/interior profiles. Outdoor preset enables both; particles and UI proxies should override at submit. Document defaults in your game's mesh import pipeline so glTF assets do not accidentally cast from invisible collision shells.


### Submit-time flag assignment

`SceneSubmit` sets draw flags when building `SceneDrawItem` from mesh components — not during Vulkan record. Renderer reads immutable snapshot per frame. Changing cast/receive at runtime goes through component API → next submit overwrites flags.

For punctual lights, `castsShadow` on `PointLightComponent` / `SpotLightComponent` is separate from mesh flags — light casts shadows only when both light flag and mesh casters exist in frustum.


### Performance implications

Receiver-off draws that still run PCF waste texture bandwidth — branch before first shadow map sample. Cast-off draws skipped on CPU save entire vertex transform + rasterization in shadow pass — significant on foliage-heavy scenes where artists disable cast on grass cards.


### RecordShadowCastMeshes filter logic

```cpp
if ((d.shadowFlags & kSceneShadowCast) == 0) continue;
if (d.skyMode != SceneSkyMode::None) continue;
```

Sky draws never cast — sun disk should not occlude world. Transparent draws with cast enabled still rasterize to shadow map — verify alpha cutout meshes cast correct silhouette, not full quad bounds.

Custom dynamic meshes route through custom VB/IB bindings with same flag check — procedural geometry must set flags at submit like static meshes.


### Scene.frag receive gate

```glsl
const bool receiveShadow = (push.shadowFlags & 2) != 0;
if (... && receiveShadow) {
    sunShadow = blendSunShadowCascades(...);
}
```

Push constants carry per-draw shadow flags alongside material fields — sizeof push block must accommodate int shadowFlags without shifting material offsets. RenderDoc: verify push bytes at draw match expected flag value when toggling receive in editor.


### Authoring workflow

MeshRenderer or equivalent exposes cast/receive toggles → serialized on prefab → submit writes flags → GPU consumes. QA checklist: collision proxy cast off, hero mesh cast on, floor receive on, emissive billboard receive off, particle systems both off unless stylized ground stain effect.


### Flag bitmask in tooling

Expose shadow flags as checkboxes in editor: Cast, Receive. Store as bitmask on prefab matching `kSceneShadowCast | kSceneShadowReceive`. Import pipeline from glTF should map extensions if present; default static geometry to both true, particles to both false, sky to neither.

Runtime toggle API: changing component → marks scene dirty → next submit updates GPU path without pipeline rebuild.


### Default flag matrix

| Draw type | Cast | Receive |
|-----------|------|---------|
| Opaque world mesh | yes | yes |
| Skinned character | yes | yes |
| Particle | no | no |
| Sky | no | no |
| Emissive billboard | optional | no |
| Collision proxy | no | no |

Publish this table in your engine docs — prevents artists from enabling cast on invisible physics hulls that darken entire regions incorrectly.


### kSceneShadow constants

```cpp
// SceneLightingProfile.hpp
inline constexpr std::int32_t kSceneShadowCast = 1;
inline constexpr std::int32_t kSceneShadowReceive = 2;
inline constexpr std::int32_t kSceneShadowCastAndReceive = 3;
```

Use bitmask tests — `(flags & kSceneShadowCast) != 0` — not equality on entire word unless you guarantee no future bits. Document reserved bits for engine extensions (e.g., cast-only for first-person weapon).


### Level design scenarios

**Floating platform** that casts onto ground but should not self-darken: cast on, receive off. **Light shaft decal** on floor: receive on, cast off. **Glass wall**: receive off if transparent pass skips shadow multiply; may still cast if opaque shadow caster desired. Playtest each scenario in RenderDoc with flags highlighted in ImGui overlay.


### Punctual light castsShadow

`SceneSubmit` copies `PointLightComponent::CastsShadow()` into GPU `ScenePointLight.castsShadow`. Independent from mesh flags — a scene can have shadow-casting spot with no mesh casters (empty pass) or casters with non-shadow lights (no punctual depth pass). Both checks must pass for visible punctual shadow.


### ImGui shadow flag overlay

During development, draw ImGui text above selected entity showing cast/receive bits pulled from live SceneDrawItem snapshot. When artist reports wrong shadow, compare overlay to RenderDoc draw push constants — catches submit bugs vs GPU bugs in under five minutes.


### Worked example

### Cast/receive worked example

Walk through Spark **ThreeDDemo** with RenderDoc:

1. Select ground draw — push constants show `shadowFlags = 3` (cast+receive).
2. Select character — same default unless overridden.
3. Temporarily set ground receive off in component — relaunch — sun term ignores PCF on floor pixels (verify in texture inspect: uniform brightness in sun direction).
4. Add invisible proxy with cast on — shadow pass draw count increases; scene shows dark blob — disable cast on proxy.

This four-step loop is the canonical QA script for flag wiring from ECS → submit → push → shader branch.


### Shadow flags in serialization

Part XIII scene files should persist cast/receive on mesh components — reloading level must not reset flags to engine defaults silently. Add round-trip test: save scene with receive disabled on prop, reload, confirm RenderDoc push constants still show bit cleared.


### Summary checkpoint

Milestone for this chapter: one mesh casts shadow on floor without receiving on itself — verify in RenderDoc that shadow pass includes caster draw and scene pass skips PCF multiply on caster pixels while ground still darkens.


### Engine porting note

When porting from Unity/Unreal terminology: Cast Shadows maps to kSceneShadowCast; Receive Shadows maps to kSceneShadowReceive. Unity Light layers do not exist in Spark — use light castsShadow plus mesh flags together for equivalent control.


### Closing note

Extend exercises: document three game-specific draw types and their intended cast/receive defaults in your design doc.


### Quick reference

Cast bit → shadow pass draw list. Receive bit → scene.frag PCF multiply. Both independent — four combinations possible; only cast+receive is typical for world geometry.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Fields on SceneDrawItem

**Concept:** Cast vs receive

**Decision:** Fields on SceneDrawItem

**Why:** Spark stores shadow flags on scene draw items gathered at submit — not hardcoded in Vulkan record code.

**In Spark:** `SceneRenderParams` draw items include cast/receive booleans.

### Filter in shadow cast loop

**Concept:** CPU filtering

**Decision:** Filter in shadow cast loop

**Why:** `VulkanShadowCastDraw` skips draws with cast disabled before issuing indexed draws.

**In Spark:** Shadow record context iterates packed draw list with flag check.

### Component-level authoring

**Concept:** Defaults by draw type

**Decision:** Component-level authoring

**Why:** `MeshRenderer` or material components expose toggles that map to draw flags at submit.

**In Spark:** ECS components → `SceneSubmit` → draw item flags.

### Branch before shadow sample

**Concept:** CPU filtering

**Decision:** Branch before shadow sample

**Why:** When receive is false, skip PCF texture fetches in the fragment shader — saves bandwidth on emissive and debug geometry.

**In Spark:** Push constant or per-draw flag gating in `scene.frag`.

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

- `include/spark/engine/SceneRenderParams.hpp`
- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/render/shadow/VulkanShadowCastDraw.cpp`

### Step 1 — Find flag fields in SceneRenderParams

Grep Spark for `castsShadow` and `receivesShadow` on draw item structs. Note default values when spawning demo entities.

### Step 2 — Trace submit path

Flags are set during `FillStandardLitSceneFromWorld` (or equivalent) when mesh components are gathered — not in the renderer.

### Step 3 — Filter shadow pass draws

In your shadow record loop:

```cpp
for (const Draw& d : scene.draws) {
    if (!d.castsShadow) continue;
    RecordShadowDraw(d);
}
```

### Step 4 — Gate scene shader

Pass `receivesShadow` via push constants. When false, set visibility to 1.0 before modulating the sun term.

### Step 5 — Test special cases

Disable receive on ground — uniform lighting. Add a mesh that casts but does not receive — shadow on floor without self-darkening.

### Step 6 — Particles and sprites

Confirm Part VIII particles and Part XI sprites default both flags false unless explicitly enabled.

### Step 9 — Read shadow cast filter in C++

CPU-side cast filter before draw issue — canonical Spark pattern.

*Cast flag and sky filter*

```cpp
// src/spark/render/shadow/VulkanShadowCastDraw.cpp (lines 43–50)
    for (std::size_t di = 0; di < ctx.scene->draws.GetSize(); ++di) {
        const SceneDrawItem& d = ctx.scene->draws[di];
        if (d.skyMode != SceneSkyMode::None) {
            continue;
        }
        if ((d.shadowFlags & kSceneShadowCast) == 0) {
            continue;
        }
```

## 5. How This Connects to the Rest of Your Engine

Flags bridge **gameplay authoring** and **GPU passes**. Part **VIII** particles: both false. Part **XII** terrain: both true. Part **XIII** serialization should persist flags on prefabs.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Forgetting to filter skinned draws** — same flags on skinned entries.
- **Receiver off but still sampling** — branch before texture fetch.
- **Proxy casters** — invisible collision mesh casting huge shadows; disable cast on proxies.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Disable receive on ground plane — confirm uniform lighting.
2. Add a floater that casts but does not receive.
3. Document default flag values in your snapshot builder.
4. Clear cast flag on reference cube — confirm it disappears from shadow pass.
5. Trace `shadowFlags` from `SceneDrawItem` to push constants.

## Summary

Cast and receive flags control shadow pass inclusion and shader visibility. Filter casters on CPU; gate PCF with a per-draw flag in the lit shader.

---

Next: [Skinned Mesh Shadows](07-skinned-shadows.md).
