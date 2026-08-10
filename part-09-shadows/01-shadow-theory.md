# Shadow Mapping Theory

> **Build-Along Milestone:** RenderDoc capture shows directional shadow depth draws completing before the HDR scene pass begins.

**You will learn:**

- Explain shadow mapping as a two-pass depth compare from light space.
- Place shadow passes before the HDR scene pass in the frame graph.
- Identify Spark's directional and punctual shadow pass objects.
- Relate texel footprint to shadow map resolution and cascade count.

## 1. The Problem We Are Solving

Every direct-light shading model asks the same question: **Is this point visible from the light?** Ray tracing answers by intersecting scene geometry — accurate but expensive for a forward renderer drawing thousands of meshes each frame.

**Shadow mapping** trades exactness for speed: render the scene from the light's viewpoint, store the nearest depth per texel, then during the main pass compare each shaded point's light-space depth against that texture. Wrong answers appear as blocky edges, acne, or detached shadows — but the core recipe is stable across decades of engines.

You already have opaque mesh drawing with depth testing from earlier parts. Part IX adds offscreen depth images, depth-only pipelines, matrix upload for `worldToShadowClip`, and a visibility multiplier in the sun term. Spark implements the full path — you will trace it top-down before writing your own.

## 2. Core Concepts

### Two-pass shadow recipe

Shadow mapping approximates **visibility along a light ray** without tracing rays per pixel. The algorithm splits work into two raster passes:

| Pass | View matrix | Attachment | Question answered |
|------|-------------|------------|-------------------|
| Shadow | Light view-projection | Depth texture | What is the closest surface the light sees at each texel? |
| Scene | Camera view-projection | HDR color + depth | Is this pixel farther from the light than the stored depth? |

```text
Eye camera                    Light camera
     \                            /
      \   occluder blocks ray    /
       \-----------------------/
        * P (shaded point)
```

For world point **P**, transform to light clip space, divide by **w**, map XY to UV, and compare **z** against `texture(shadowMap, uv).r`. If stored depth is **closer** to the light, **P** is in shadow. The compare is a **discrete texel lookup**, not continuous geometry — the source of acne, Peter panning, and soft-edge filtering in later chapters.


### Light-space coordinates

Every shadow sample begins with the same matrix chain used when **writing** the map:

```text
clipLight = worldToShadowClip * vec4(P_world, 1)
ndc       = clipLight.xyz / clipLight.w          // perspective or ortho
uv        = ndc.xy * 0.5 + 0.5                   // [0,1]²
refDepth  = ndc.z                                  // after bias (Ch. 8)
mapDepth  = texture(shadowMap, uv).r
visible   = (refDepth <= mapDepth)                 // Spark: lower Z = closer
```

**Handedness** must match between shadow pass projection and scene compare. Spark applies a linear Z-flip matrix when building `worldToShadowClip` so Vulkan's `[0,1]` depth range agrees with stored texels. **Y-flip** depends on viewport convention — Spark checks `ubo.viewportSize.w` and may flip V when sampling.

The eye camera never renders occluders behind the viewer that still cast shadows **in front** of the camera. Only a light-space depth pass sees those silhouettes.


### Depth-only attachments

Shadow passes need **no color buffer**. Rasterization writes clip-space **Z** into a depth image; the fragment shader can be empty or omitted entirely.

| Property | Typical Spark value | Why |
|----------|---------------------|-----|
| Format | `VK_FORMAT_D32_SFLOAT` | Stable compare; avoids 16-bit banding when tuning bias |
| Usage | `DEPTH_STENCIL_ATTACHMENT \| SAMPLED` | Write in pass, read in scene shader |
| Final layout | `DEPTH_STENCIL_READ_ONLY_OPTIMAL` | Legal layout for `sampler2D` fetch |
| Compare sampler | `compareEnable = false` | Manual compare in GLSL for MoltenVK portability |

Pipeline state highlights: `depthTestEnable = true`, `depthWriteEnable = true`, `depthCompareOp = LESS`, **no** color attachments, cull mode usually back-face. After `vkCmdEndRenderPass`, the depth image must be in a layout the lit pass can sample — Spark sets this in the attachment description's `finalLayout`.


### Frame graph placement

Shadow work belongs **after uploads** and **before** the HDR scene subpass:

```text
vkBeginCommandBuffer
  → deferred uploads / mesh streaming
  → RecordShadowMapPass          ← Part IX
  → vkCmdBeginRenderPass (HDR)   ← Part X color target
       → opaque / transparent / 2D / particles
  → vkCmdEndRenderPass
  → SSAO / tonemap
vkEndCommandBuffer
```

If shadows run **after** the lit pass, the scene shader samples stale or cleared depth — shadows vanish or lag one frame. Matrix uploads for shadow sampling happen in `WriteUniformBuffer` on the CPU **before** command recording so shadow pass push constants and scene UBO share the same light transforms for that frame index.


### Resolution and texel footprint

Shadow quality scales with **world-space texel size** — meters per texel at the shaded point:

```text
texelWorldSize ≈ (orthoWidth_world) / shadowMapResolution
```

A single 2048² map covering 500 m horizontally yields ~0.24 m/texel — soft, blocky contact shadows. **Cascaded shadow maps** (Chapter 2) allocate multiple frustum slices so near geometry gets higher texel density. Memory budget matters: four 1024² D32 slices ≈ 16 MB per flight target before punctual atlases.

When profiling, measure **fill rate** (caster triangles × map resolution) separately from **sample cost** (scene pixels × PCF taps in Chapter 4).


### Shadow maps vs other visibility techniques

| Technique | Cost driver | Strength | Weakness |
|-----------|-------------|----------|----------|
| Shadow map | Extra passes + texture samples | Works in forward; artist-tunable | Acne, bias, resolution limits |
| Shadow volumes | Stencil / extra geometry | Sharp when exact | Silhouette extraction; fill cost |
| Ray tracing | BVH traversal | Physically correct penumbra | Hardware / denoise dependency |

Spark uses **forward PBR** with shadow maps because the same mesh buffers, skinning path, and draw lists serve both lit and shadow passes. Ray-traced shadows can replace the compare later, but the **visibility term** in the BRDF remains: multiply direct light by `shadow ∈ [0,1]`.


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


### Historical context and limits

Shadow maps date to classic real-time graphics — the algorithm is **biased** (single depth sample per texel) and **resolution-limited**, but predictable on programmable GPUs. Modern engines combine CSM, PCF, and slope bias as baseline; ray-traced shadows augment quality on high-end paths.

No bias removes acne **and** Peter panning on all surfaces simultaneously; no single map resolution serves contact detail and 500 m draw distance. Plan cascades and art-side offsets (Chapter 8) as part of the feature, not polish pass.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Dedicated pass objects per light class

**Concept:** Two-pass shadow recipe

**Decision:** Dedicated pass objects per light class

**Why:** Directional CSM and punctual spot/point maps use different projections, atlas layouts, and slot tables. Separate `VulkanDirectionalShadowPass` and `VulkanPunctualShadowPass` classes own render passes, images, pipelines, and record functions — the HDR scene pass stays a consumer, not a producer, of depth targets.

**In Spark:** `include/spark/render/shadow/VulkanDirectionalShadowPass.hpp` and `VulkanPunctualShadowPass.hpp`.

### Single depth attachment render pass

**Concept:** Depth-only attachments

**Decision:** Single depth attachment render pass

**Why:** Omitting color attachments simplifies load/store and guarantees no accidental color writes. `finalLayout = DEPTH_STENCIL_READ_ONLY_OPTIMAL` on the attachment description transitions the image for shader read when the pass ends.

**In Spark:** `VulkanDirectionalShadowPass::CreateResources` lines 104–112 — depth attachment with read-only final layout.

### RecordShadowMapPass before HDR begin

**Concept:** Frame graph placement

**Decision:** RecordShadowMapPass before HDR begin

**Why:** Uploads finish first; shadow maps populate and transition; then the HDR subpass samples them. Centralizing the call at line 213 makes pass order auditable in one glance.

**In Spark:** `VulkanRenderer::RecordSceneCommandBuffer` — `RecordShadowMapPass` immediately after upload record.

### Shared mesh bindings via VulkanShadowRecordContext

**Concept:** Light-space coordinates

**Decision:** Shared mesh bindings via VulkanShadowRecordContext

**Why:** Casters reuse static/custom VBs, index buffers, and skin SSBO from the opaque scene pass. Only pipeline, push constants (`lightViewProj`), and cull state differ — no duplicate geometry on CPU.

**In Spark:** `RecordShadowMapPass` builds `VulkanShadowRecordContext` with the same buffers as `VulkanSceneOpaqueRecordContext`.

### Fixed atlas sizes with documented caps

**Concept:** Resolution and texel footprint

**Decision:** Fixed atlas sizes with documented caps

**Why:** Directional CSM packs into a known atlas; punctual caps (`kMaxSpotShadowMaps = 4`, `kMaxPointShadowMaps = 2`) bound worst-case GPU memory. Documented constants let gameplay code reason about shadow slot assignment.

**In Spark:** `punctual_shadows.glsl` constants and directional `kMapSize` in `VulkanDirectionalShadowPass.cpp`.

### Manual depth compare in GLSL

**Concept:** Shadow maps vs other visibility techniques

**Decision:** Manual depth compare in GLSL

**Why:** Spark avoids `sampler2DShadow` hardware compare for MoltenVK / portability subset compatibility. Scene and punctual includes implement PCF with explicit `refZ <= mapZ` tests.

**In Spark:** `scene.frag` comment at binding 3: depth compare in shader, not shadow sampler.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/core/VulkanRenderer.cpp`
- `src/spark/render/shadow/VulkanDirectionalShadowPass.cpp`
- `src/spark/render/shadow/VulkanPunctualShadowPass.cpp`
- `src/spark/render/shadow/VulkanDirectionalShadowCascadeMath.cpp`
- `shaders/shadow_depth.vert`
- `shaders/scene.frag`
- `include/spark/render/lighting/SceneLightingProfile.hpp`

### Step 1 — Trace pass order in Spark

Open `VulkanRenderer::RecordSceneCommandBuffer`. After deferred uploads and custom mesh uploads, **`RecordShadowMapPass`** runs — still outside the HDR render pass. This single call site is the frame-graph anchor for all of Part IX.

*Shadow pass slot before HDR work*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 211–214)
    deferredUploadBatch.Record(commandBuffer, device(), sceneTextureUploader, screenUi);
    customMeshPool.RecordUploads(commandBuffer);
    RecordShadowMapPass(commandBuffer, frameIndex);

```

### Step 2 — Inspect directional shadow resource creation

`CreateResources` allocates a compare sampler (for potential future use), builds a **one-attachment** render pass, and creates per-flight depth images with `SAMPLED` usage. Read the attachment description carefully: `storeOp = STORE`, `finalLayout = DEPTH_STENCIL_READ_ONLY_OPTIMAL`.

*Depth-only attachment and subpass*

```cpp
// src/spark/render/shadow/VulkanDirectionalShadowPass.cpp (lines 104–120)
    VkAttachmentDescription depthAttachment{};
    depthAttachment.format = depthFormat;
    depthAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
    depthAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
    depthAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    depthAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    depthAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    depthAttachment.initialLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
    depthAttachment.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_READ_ONLY_OPTIMAL;

    VkAttachmentReference depthRef{};
    depthRef.attachment = 0;
    depthRef.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    VkSubpassDescription subpass{};
    subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpass.pDepthStencilAttachment = &depthRef;
```

### Step 3 — Follow shadow record dispatch

`RecordShadowMapPass` early-outs if descriptor sets are unavailable, then fills `VulkanShadowRecordContext` with mesh and skin bindings identical to the opaque scene path. **Punctual** shadows record first, then **directional** CSM — both iterate the same `SceneRenderParams::draws` list filtered by cast flags.

*Shadow record entry point and context*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 624–647)
void VulkanRenderer::RecordShadowMapPass(VkCommandBuffer commandBuffer, std::uint32_t frameIndex) {
    if (frameIndex >= sceneDescriptors.DescriptorSetCount()) {
        return;
    }
    const VulkanCustomMeshPool::Bindings customMesh = customMeshPool.GetBindings();
    VulkanShadowRecordContext ctx{};
    ctx.scene = &pendingScene;
    ctx.sceneParamsValid = sceneParamsValid;
    ctx.vertexBuffer = vertexBuffer;
    ctx.indexBuffer = indexBuffer;
    ctx.customVertexBuffer = customMesh.vertexBuffer;
    ctx.customIndexBuffer = customMesh.indexBuffer;
    ctx.cubeIndexCount = cubeIndexCount;
    ctx.planeIndexCount = planeIndexCount;
    ctx.planeFirstIndex = planeFirstIndex;
    ctx.cubeVertexOffset = cubeVertexOffset;
    ctx.planeVertexOffset = planeVertexOffset;
    ctx.customDrawPacked = &customDrawPacked;
    ctx.skinSsboMapped = &sceneDescriptors.SkinSsboMapped();
    ctx.descriptorSet = sceneDescriptors.DescriptorSet(frameIndex);
    ctx.maxSkinJoints = VulkanSceneDescriptors::kMaxSkinJoints;
    punctualShadow.Record(commandBuffer, frameIndex, ctx, punctualShadowFrameState);
    directionalShadow.Record(commandBuffer, frameIndex, ctx, shadowFrameState);
}
```

### Step 4 — Read the cascade math trace comment

Before editing matrices, read Spark's documented pipeline from `lightDir` → CPU `worldToShadowClip` → shadow push constants → scene PCF. This comment is the authoritative checklist when shadows look offset or swimming.

*End-to-end shadow data path trace*

```cpp
// src/spark/render/shadow/VulkanDirectionalShadowCascadeMath.cpp (lines 13–20)
/*
 * Shadow pipeline trace (single directional cascade):
 * 1) VulkanSceneUniformWriter: lightDir = normalize(scene.lightDirectionWorld) — toward sun.
 * 2) ComputeWorldToShadowClip(...) per cascade → worldToShadowClip[i] = ZFlip * lightProj * lightView.
 * 3) VulkanDirectionalShadowPass::Record: same matrix in ShadowPushConstants.lightViewProj.
 * 4) scene.frag: light clip from vWorldPos; depth compare uses slope-scaled bias.
 * 5) scene.vert: transpose(inverse(mat3(model))) * n for non-uniform scale.
 */
```

### Step 5 — Confirm uniform upload ordering

Shadow matrices for sampling upload during `WriteUniformBuffer` — **before** command buffer recording for that frame. Punctual `PrepareAndUpload` runs first; then `sceneUniformWriter.Write` fills cascade arrays consumed by `scene.frag`.

*CPU shadow state before GPU record*

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

### Step 6 — Verify in RenderDoc

Run **ThreeDDemo** or **TerrainDemo** with shadows enabled. Capture one frame and confirm:

1. Directional shadow pass shows depth-only draws (no color attachment).
2. HDR scene pass binds `shadowMap` at set 0 binding 3.
3. Pass order: uploads → shadow → HDR opaque → post.

If the shadow pass is empty, check `shadowFlags` / `castsShadow` (Chapter 6) before debugging projection math.

### Step 7 — Mirror the slot in your engine

Add a `RecordShadowMaps(cmd, frameIndex)` call in the same position: after resource uploads, before your HDR (or swapchain) scene pass. Keep matrix preparation on the CPU path that fills your scene UBO so shadow pass push constants and lit-shader UBO stay in sync for each frame-in-flight index.

### Step 8 — Document your depth convention

Write down: ortho vs perspective per light type, NDC Z range, whether you flip Y at sample time, and whether compare uses `<=` or `>=`. Spark's Z-flip matrix and viewport flag exist because mismatches here produce **inverted or swimming** shadows with otherwise correct geometry.

## 5. How This Connects to the Rest of Your Engine

Part **IV** directional and punctual light code supplies radiance; Part IX multiplies it by **visibility**. Part **VI** skinned meshes reuse the bone SSBO in `shadow_depth.vert` (Chapter 7). Part **III** UBO and push constant layout holds cascade matrices and `shadowParams`. Part **X** HDR tonemap runs **after** shadows modulate linear radiance — tuning shadows in display space hides contrast problems.

## 6. Pitfalls and Debugging

- **Sampling before layout transition** — validation error or garbage compare values; confirm `finalLayout` on shadow attachments.
- **Shadow pass after scene pass** — shadows never appear or flicker one frame late.
- **Projection handedness mismatch** between shadow pass output and scene UV math — shadows swim when the camera rotates.
- **One map for an open world** — texel footprint meters wide; plan CSM (Chapter 2) early on terrain projects.

## 7. Exercises

1. Draw the data flow: mesh VB → shadow pass → depth image → scene FS compare.
2. Capture RenderDoc on ThreeDDemo; list every pass before the HDR opaque draws.
3. Compare pipeline create info: list three differences between scene lit and shadow depth pipelines.
4. Estimate world texel size at 10 m from camera given Spark's directional atlas dimensions.
5. Write a one-paragraph explanation of why the light must render occluders the eye cannot see.

## Summary

Shadow mapping is a light-space depth pre-pass plus a depth compare in the lit shader. Spark records directional and punctual shadow passes before the HDR subpass, shares mesh bindings with the opaque scene pass, and uploads cascade matrices on the CPU before command buffer recording.

---

Next: [Directional CSM](02-directional-csm.md).
