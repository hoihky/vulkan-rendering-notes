# Descriptor Binding Map

> **Build-Along Milestone:** RenderDoc descriptor set 0 matches written binding table for scene pass draw.

**You will learn:**

- Match GLSL set/binding numbers to C++ layout entries.
- Avoid collisions when adding shaders.
- Debug descriptor-not-bound validation errors with RenderDoc.

## 1. The Problem We Are Solving

Shaders declare `layout(set=N, binding=M)`; C++ must create matching `VkDescriptorSetLayout`. One off-by-one causes black screens with no compile error. This appendix consolidates **forward renderer set 0** bindings from Parts III–IX.

Audit with grep across GLSL, `VulkanSceneDescriptors`, and RenderDoc together — not from memory.


Appendix chapter 03 supports active development — bookmark it. When stuck mid-chapter on Parts XV–XVI, grep appendix first before re-reading entire prior part. Keep local notes column for your renamed types alongside Spark names.

Schedule 30-minute appendix refresh after any PR touching `SceneSubmit`, descriptor layouts, or `RecreateSwapchain`.


Export descriptor-bindings.csv before and after any shader binding edit — diff the CSV in PR review. Binding drift is the silent killer when merging upstream Spark shader fixes into your fork.

## 2. Core Concepts

### Scene pass set 0 overview

Lit forward scene uses **one descriptor set layout** (set 0) shared by scene/sprite/shadow depth stages where compatible:

| Binding | GLSL resource | Stages |
|---------|---------------|--------|
| 0 | SceneUBO | vert + frag |
| 1 | sceneTextures (array) | frag |
| 2 | SkinSSBO | vert |
| 3 | shadowMap (CSM) | frag |
| 4 | ClusterLightsSSBO | frag |
| 5 | ClusterGridSSBO | frag |
| 6 | punctualShadow SSBO | frag |
| 7 | spotShadowMap | frag |
| 8 | pointShadowMap (array) | frag |
| 9 | SpriteInstanceBuffer | vert (sprite pass) |

Post/tonemap passes use **different** set 0 layouts — same index number, different layout object.

### UBO vs SSBO vs push

| Mechanism | Typical data |
|-----------|--------------|
| SceneUBO binding 0 | viewProj, lights, cascades, cluster grid |
| Skin SSBO binding 2 | Joint palettes per draw batch |
| Push constants | Per-draw model matrix, material layer indices |
| Cluster SSBO 4–5 | Point/spot lights culled per tile |

Changing UBO layout requires C++, GLSL, and **all** pipelines using layout — grep before edit.

### Punctual shadow block

Bindings 6–8 package spot/point shadow atlases (Part IX). GLSL struct in `punctual_shadows.glsl` must match C++ descriptor sizes and order — off-by-one breaks shadow sampling silently (full black or lit wrong).

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

### Per-pass layout isolation

`tonemap.frag`: binding 0 = hdrColor sampler — **not** SceneUBO. `post_common.glsl`: bindings 1–2 color/depth.

Never assume binding 0 means the same resource across passes — always scope by pipeline layout name.

### UpdateDescriptorSets timing

Scene descriptors update each frame for:

- Uniform buffer memcpy to mapped pointer
- Texture array layer bind when fingerprint ready
- Shadow map views after shadow pass (or bind placeholder)

RecreateSwapchain must re-write image view bindings when depth/HDR views change extent.

### Debugging descriptor-not-bound

Validation: `VUID-vkCmdDraw-*-DescriptorSet*` — compare RenderDoc **Descriptor Set 0** tab to table. Common causes: forgot Update after upload, wrong frameIndex set, layout mismatch with shader.

### Sprite binding 9

Sprite instance SSBO at binding 9 — scene mesh draw does not bind it; sprite pass pipeline layout must include binding 9 or instancing fails validation at draw. Adding skin SSBO binding 2 to shadow pass keeps skinned shadow casters working — duplicate layout entries across pipelines must match byte-for-byte where shared.

### Cluster SSBO sizing

Cluster buffers sized to `MaxPointLights` — CPU must not push more lights than SSBO capacity; submit clamps or culls excess. Mismatch shows as garbage lights in last tiles of cluster grid.

### Dynamic uniform buffer offset

Not used in Spark baseline — all per-frame data in dedicated uniform buffer per flight index. Alternative engines use dynamic offsets into one big UBO — different binding model; do not mix paradigms in one layout.

Push constants limited 128 bytes — model matrix + material indices consume most budget on scene pass.

### Combined image sampler count

Each binding consumes descriptor pool sizes — count bindings × framesInFlight when sizing pool. Out of pool memory fails at set allocation with opaque error — log pool sizes at init.

### Complete descriptor binding CSV

**Scene pass set 0 — master reference (update when forking shaders):**

| binding | name | type | stages | GLSL file | C++ update |
|---------|------|------|--------|-----------|------------|
| 0 | SceneUBO | uniform buffer | vert, frag | scene_ubo.glsl | VulkanSceneUniformWriter |
| 1 | sceneTextures | combined image sampler array[16] | frag | scene.frag | SceneDescriptors Update |
| 2 | SkinSSBO | storage buffer | vert | skinning.glsl | Skin palette upload |
| 3 | shadowMap | combined image sampler | frag | scene.frag | Directional shadow pass |
| 4 | ClusterLights | storage buffer | frag | clustered_lights.glsl | Cluster builder |
| 5 | ClusterGrid | storage buffer | frag | clustered_lights.glsl | Cluster builder |
| 6 | punctualShadowData | storage buffer | frag | punctual_shadows.glsl | Punctual shadow pass |
| 7 | spotShadowMap | combined image sampler | frag | punctual_shadows.glsl | Punctual atlas |
| 8 | pointShadowMap | combined image sampler array | frag | punctual_shadows.glsl | Point cube faces |
| 9 | SpriteInstanceBuffer | storage buffer | vert | sprite.vert | Sprite pass only |

**Tonemap pass set 0 (different layout!):**

| binding | name | type | stages |
|---------|------|------|--------|
| 0 | hdrColor | combined image sampler | frag |
| 1 | hdrDepth | combined image sampler | frag |

**Shadow pass set 0:**

| binding | name | type | stages |
|---------|------|------|--------|
| 0 | (push constants) | push | vert |
| 1–2 | mesh VBs/IB | vertex/index | vert |

Export CSV to `docs/descriptor-bindings.csv` — diff in PR when changing any row.

### Shadow pass descriptor subset

Shadow passes use **subset** of scene bindings — not full set 0:

| binding | Shadow pass | Scene pass |
|---------|-------------|------------|
| 0 | push: lightViewProj | SceneUBO |
| 1 | static/custom VB | sceneTextures |
| 2 | skin SSBO (skinned casters) | skin SSBO |
| 3 | — | shadowMap sample |

Adding skinned shadow casters requires binding 2 in shadow pipeline layout — duplicate layout entry must match scene skin SSBO range exactly.

Punctual shadow **atlas** bindings 6–8 exist only on scene/tilemap/sprite pipelines that include `punctual_shadows.glsl` — shadow depth pass does not bind them.

When forking forward+ renderer, duplicate binding indices break UpdateDescriptorSets — grep all `CreateDescriptorSetLayout` calls before editing one file.

### Descriptor pool sizing formula

Size pool at init to avoid runtime allocation failure:

```text
setsPerFrame = kMaxFramesInFlight
bindingsPerSet = count(layout bindings)
poolSizes[type] = sum over layouts (count × setsPerFrame × descriptorCount)
```

**Spark scene set 0 rough counts (per frame index):**

| Type | Count per set |
|------|---------------|
| UNIFORM_BUFFER | 1 |
| COMBINED_IMAGE_SAMPLER | 1 + 3 shadow samplers |
| STORAGE_BUFFER | 4 (skin + cluster + punctual) |

Multiply by `kMaxFramesInFlight` (2). Add tonemap/UI pass pools separately — do not one mega-pool unless sizes computed together.

Out-of-pool error at set allocation — log requested vs max per type from `VkDescriptorPoolCreateInfo`. Fix by increasing count in CreateDescriptorPool, not by leaking sets.

After RecreateSwapchain, **image view handles change** — UpdateDescriptorSets must refresh view pointers; pool itself usually survives.

### Push constant layout reference

Scene opaque draw push constants (typical Spark layout):

| Offset | Field | Shader |
|--------|-------|--------|
| 0 | modelMatrix (64 bytes) | vert |
| 64 | materialIndex / layers packed | frag |

Shadow pass push:

| Offset | Field |
|--------|-------|
| 0 | lightViewProj (64 bytes) |

Exceeding 128-byte push limit fails pipeline create — pack indices as uint32 bitfields if needed. std430 SSBO preferred for large per-draw data — sprites use SSBO binding 9 instead of push.

Verify push size in RenderDoc — Pipeline State → Push Constants shows bytes; compare to C++ `sizeof` struct with padding. `#pragma pack` mismatch between C++ and GLSL causes sliding shadows or wrong UV layer.

When adding push field, update **every** pipeline layout with same push range — shadow and scene layouts differ; do not copy-paste range sizes blindly.

### Sampler and texture array binding notes

Binding 1 `sceneTextures` uses **array of combined image samplers** — each layer may use different sampler state (albedo linear clamp vs normal repeat). Spark typically shares one sampler per array for simplicity; fork with anisotropic filtering may use separate sampler per material class.

**Layer index -1** means unbound / placeholder — shader should branch to flat magenta or checker in debug; shipping may use default gray albedo.

When adding 17th texture, fail gather with LOG — do not silently wrap layer index. Increase `kMaxSceneTextures` constant in C++, GLSL, and descriptor layout together.

Normal/ORM/emissive use **same layer index space** as albedo — different draw items reference different layers for each map type from MaterialComponent gather.

### Tonemap and post descriptor quick ref

| Pass | Set | Binding | Resource |
|------|-----|---------|----------|
| SSAO | 0 | 0 | depth sampler |
| SSAO | 0 | 1 | noise texture |
| Tonemap | 0 | 0 | hdrColor |
| Tonemap | 0 | 1 | hdrDepth optional |
| Bloom (if fork) | 0 | 0 | bright extract |

Post passes never bind SceneUBO — exposure may come from small post UBO or push. Confusing tonemap binding 0 with scene binding 0 is the #1 appendix ch03 support ticket — always note pipeline layout name in CSV `layout` column.

SSAO enable flag on SceneRenderParams skips record entirely when false — descriptors may stale; safe because pass not executed.

### Clustered lights binding debug workflow

When point lights look wrong (garbage in last cluster cells, or all black), debug bindings 4–5 together — not scene UBO sun term.

**RenderDoc steps:**

1. Capture demo 17 at night preset with multiple point lights placed.
2. Select opaque scene draw → Descriptor Set 0 → binding 4 ClusterLights SSBO — verify non-zero byte size.
3. Binding 5 ClusterGrid — tile count matches `clusterDepth` in SceneUBO.
4. Fragment shader `clustered_lights.glsl` loop bound — compare `kMaxClusteredPointLights` in GLSL to C++ `MaxPointLights`.

**CPU-side grep:**

```bash
rg 'pointLights' src/spark/scene/SceneSubmit.cpp
rg 'ClusterGrid' src/spark/render/
```

Submit must clamp light count to SSBO capacity — excess lights cause shader to read uninitialized tail cells. Disabling clustered path (`clusterDepth.w < 0.5`) should degrade to directional-only without validation errors — use as bisect step when light pass regresses.

Update appendix CSV rows 4–5 whenever cluster struct layout changes — std430 alignment padding breaks silently.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanSceneDescriptors

**Concept:** Scene pass set 0 overview

**Decision:** VulkanSceneDescriptors

**Why:** C++ CreateSetLayout mirrors scene.vert/frag includes — single maintenance point.

**In Spark:** `include/spark/render/scene/VulkanSceneDescriptors.hpp`.
### punctual_shadows.glsl included by scene.frag

**Concept:** Punctual shadow block

**Decision:** punctual_shadows.glsl included by scene.frag

**Why:** Keeps shadow SSBO definition near sampling functions.

**In Spark:** Bindings 6–8 synchronized with C++ layout.
### Separate CreateDescriptorSetLayout per pass

**Concept:** Per-pass layout isolation

**Decision:** Separate CreateDescriptorSetLayout per pass

**Why:** Tonemap and UI do not inherit scene bindings — reduces accidental coupling.

**In Spark:** HdrTonemapPass vs VulkanSceneDescriptors.
### Model in push, frame data in UBO

**Concept:** UBO vs SSBO vs push

**Decision:** Model in push, frame data in UBO

**Why:** 128-byte push budget — per-draw model + material indices fit; per-frame data too large for push.

**In Spark:** Part III push constant chapter.
### Update after upload batch

**Concept:** UpdateDescriptorSets timing

**Decision:** Update after upload batch

**Why:** Descriptor image view must reference layout-ready image — DeferredUploadBatch ordering.

**In Spark:** RecordSceneCommandBuffer flow.
### docs/descriptor-bindings.csv export

**Concept:** Complete descriptor binding CSV

**Decision:** docs/descriptor-bindings.csv export

**Why:** Binding 0 means different resources per pass — CSV scopes by pipeline layout.

**In Spark:** VulkanSceneDescriptors.hpp + shaders/.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/scene/VulkanSceneDescriptors.hpp`
- `shaders/scene.frag`
- `shaders/scene_ubo.glsl`
- `shaders/punctual_shadows.glsl`

### Step 1 — Grep scene shader bindings

```bash
rg 'layout\(set = 0, binding' shaders/scene*.glsl shaders/*ubo*.glsl shaders/clustered*.glsl
```

Build master table in spreadsheet.
### Step 2 — Read SceneUBO layout

Correlate std140 block with C++ struct writer.

*Scene UBO std140 block binding 0*

```glsl
// shaders/scene_ubo.glsl (lines 4–23)
layout(std140, set = 0, binding = 0) uniform SceneUBO {
    mat4 viewProj;
    vec4 lightDir;
    vec4 cameraPos;
    vec4 lightColor;
    vec4 ambientColor;
    vec4 ambientSky;
    vec4 ambientProbe;
    mat4 invViewProj;
    vec4 viewportSize;
    vec4 timeGlobal;
    mat4 worldToShadowClip[4];
    vec4 cascadeSplits;
    vec4 cascadeAtlas[4];
    vec4 shadowParams;
    vec4 clusterGrid;
    vec4 clusterDepth;
    /** x = env equirect layer (-1 = procedural sky); y = intensity; w = enabled (1). */
    vec4 iblParams;
} ubo;
```
### Step 3 — Open VulkanSceneDescriptors

Match binding index, count, stage flags to GLSL — fix any drift.

*Scene descriptor class — bindings 0–9 comment*

```cpp
// include/spark/render/scene/VulkanSceneDescriptors.hpp (lines 18–40)
/**
 * Scene descriptor set layout, per-flight uniform/skin SSBOs, pool, and bound descriptor sets.
 * Binding slots 0–9 match the lit-scene shader layout (UBO, textures, lights, shadows, sprites).
 */
class VulkanSceneDescriptors {
public:
    static constexpr std::uint32_t kMaxSkinJoints = 64;
    /** std430 mat4 skinPalette[kMaxSkinJoints] — 64 joints × 64 bytes. */
    static constexpr VkDeviceSize kSkinSsboBytes = 4096;

    struct BindingSources {
        const VulkanSceneTextureUploader& sceneTextureUploader;
        const VulkanClusteredForwardLights& clusteredForwardLights;
        const VulkanDirectionalShadowPass& directionalShadow;
        const VulkanPunctualShadowPass& punctualShadow;
        const VulkanSpritePass& spritePass;
    };

    void CreateSetLayout(VkDevice device);
    void CreateUniformBuffers(VkPhysicalDevice physicalDevice, VkDevice device, std::uint32_t framesInFlight);
    void CreateSkinSsboBuffers(VkPhysicalDevice physicalDevice, VkDevice device, std::uint32_t framesInFlight);
    void CreatePoolAndSets(VkDevice device, std::uint32_t framesInFlight, const BindingSources& sources);
    void Destroy(VkDevice device) noexcept;
```
### Step 4 — RenderDoc verify

Capture ThreeDDemo draw — Descriptor Set 0 vs your table for one opaque mesh draw.
### Step 5 — Break test binding 1

Skip one texture write — save validation message text for onboarding doc.
### Step 6 — Adding binding 10 checklist

Layout entry → GLSL declare → UpdateDescriptorSets → rebuild **all** pipelines using layout → update appendix table.
### Step 7 — Post pass contrast

Document tonemap binding 0 separately — students confuse with SceneUBO binding 0.
### Step 8 — Export CSV

Columns: binding, type, stages, shader file, C++ update function — attach to PR when changing layout.

## 5. How This Connects to the Rest of Your Engine

Part **III** UBO; Part **IV** texture array; Part **IX** shadows and clustered lights. **SceneRenderParams** feeds descriptor writes. **Lifecycle** appendix — destroy pool after pipelines.


Appendix **lifecycle** destroy order requires descriptor pool freed after pipelines — binding table useless if pool destroyed while sets in use. Part **III** introduced layouts; this appendix maintains index truth during forks.

## 6. Pitfalls and Debugging

- **Same binding index, different types across passes** — layout conflation bug.
- **Stale image view after resize** — re-write descriptors on RecreateSwapchain.
- **Assuming global set 0** — tonemap set 0 ≠ scene set 0.
- **Forgotten fragment stage flag** — UBO invisible in frag.
- **Skin SSBO size mismatch** — kMaxSkinJoints × mat4 must match GLSL.


- **Assuming binding 0 is SceneUBO in tonemap pass** — layout isolation; check pipeline name.
- **Adding binding 10 without rebuilding all scene pipelines** — validation at first draw.

## 7. Exercises

1. Export descriptor-bindings.csv from appendix table; diff against your fork.
2. RenderDoc verify bindings 0–3 on one opaque draw in demo 17.
3. Break binding 1 intentionally; save VUID for onboarding doc.

## Summary

Keep scene, shadow, and post descriptor layouts synchronized with GLSL. Spark centralizes scene bindings in VulkanSceneDescriptors and punctual_shadows.glsl. Master binding CSV, shadow subset, pool sizing, and push layout reference prevent silent descriptor drift across passes.

---

Next: [SparkDemo Mode Catalog](04-demo-catalog.md).
