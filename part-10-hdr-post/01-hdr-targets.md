# HDR Render Targets

> **Build-Along Milestone:** RenderDoc shows scene draws targeting RGBA16F, not the swapchain image.

**You will learn:**

- Allocate float color and depth attachments for the main scene pass.
- Wire scene, sprite, and particle passes into the same HDR subpass.
- Trace Spark's HDR render pass owned by VulkanHdrTonemapPass.
- Recreate HDR targets when swapchain extent changes.

## 1. The Problem We Are Solving

If your scene pass writes **UNORM8** swapchain colors, lighting saturates at 1.0 before exposure or artistic grading. A sun disk, emissive panel, or specular highlight needs **headroom above 1.0** in linear space so post can compress dynamic range without clipping.

Production engines render into **floating-point offscreen color** with depth, accumulate lighting in **linear space**, then **tonemap** to the display-referred swapchain. Part X builds that pipeline on top of everything from Parts IV–IX.

Spark already routes world drawing through `VulkanHdrTonemapPass`'s HDR target. You will trace render pass creation, follow scene recording into the HDR framebuffer, and verify the migration checklist in RenderDoc before mirroring the pattern in your engine.

## 2. Core Concepts

### Linear vs display-referred color

Real-time rendering separates **scene-referred** and **display-referred** signals:

| Space | Where computed | Typical range | Example |
|-------|----------------|---------------|---------|
| Linear HDR | Scene pass attachment | 0 … unbounded | Sun-facing diffuse ≈ 3–8 |
| Display LDR | Swapchain after tonemap | 0 … 1 per channel | Monitor output |

**Albedo** textures store sRGB-encoded values; decode to linear before BRDF (`sparkSrgbToLinear`). **Lighting** accumulates in linear space — emissive and specular highlights may exceed 1.0. **Tonemap** (Chapter 2) compresses dynamic range; **gamma encode** prepares UNORM swapchain images for human perception.

```text
sRGB albedo → linear BRDF → HDR buffer (RGBA16F) → tonemap → gamma → swapchain
```

Clamping in `scene.frag` destroys headroom for exposure volumes and bloom-style effects. Spark outputs unclamped linear radiance and defers range compression to post.


### HDR attachment pairing

The main world pass targets an **offscreen framebuffer** with float color and depth:

```text
HDR framebuffer (per frame-in-flight):
  color0: R16G16B16A16_SFLOAT   COLOR_ATTACHMENT | SAMPLED
  depth:  D32_SFLOAT             DEPTH_STENCIL_ATTACHMENT
```

| Choice | Rationale |
|--------|-----------|
| RGBA16F | Enough mantissa for sun + emissive; half bandwidth vs RGBA32F |
| Shared depth | Opaque, sprite, and particle passes occlude consistently |
| Color finalLayout = SHADER_READ | Tonemap and SSAO sample without extra copy |

The **swapchain image is untouched** until the tonemap pass. Pipelines for scene, sky, transparent, tilemap, sprite, and particle passes must all declare compatibility with `hdrRenderPass` — not the present pass handle from Part I.


### Shared HDR subpass

Spark records **one** `vkCmdBeginRenderPass` for the entire world draw sequence:

```text
vkCmdBeginRenderPass (HDR)
  → sceneOpaquePass (sky, opaque, transparent)
  → composite2DPass (tilemap + sprite)
  → particlePass (additive billboards)
vkCmdEndRenderPass
```

Depth buffer persistence matters: Part VIII particles rely on depth testing against opaque geometry drawn earlier in the **same subpass**. Splitting sprites into a separate render pass would require depth resolve or re-load — expensive and error-prone.

Load ops: color and depth **clear** at pass start. Store ops: color **store** (tonemap reads it); depth **store** (SSAO may copy it in Chapter 3).


### Render pass ownership

Centralizing HDR render pass creation in `VulkanHdrTonemapPass` keeps attachment formats, subpass count, and final layouts consistent across every pipeline that draws the world. Scene modules receive the `VkRenderPass` handle at pipeline create time — when it changes (swapchain recreate), **all** dependent pipelines recreate.

```text
VulkanHdrTonemapPass
  ├── CreateRenderPass()      → hdrRenderPass handle
  ├── RecreateFlightTargets() → per-flight FB + RGBA16F image
  └── RecordTonemap()         → samples HDR, writes swapchain
```

This inversion — post module owns scene target — mirrors production engines where the **display pipeline** defines the HDR format the lighting team must target.


### Recreate on resize

Swapchain extent drives HDR image dimensions. When `RecreateSwapchain` fires:

```text
RecreateSwapchain()
  → RecreateHdrFlightTargets()   // RGBA16F + FB
  → scene pipelines recreate     // render pass compatibility
  → SSAO / tonemap pipelines
```

Mismatch between HDR extent and `vkCmdSetViewport` extent produces validation errors or stretched post. Spark hooks `RecreateHdrFlightTargets` from the renderer's swapchain path (line 119) so HDR images rebuild **before** pipelines that depend on them.

Frame-in-flight: each flight index gets its own HDR color image — the CPU writes UBO while GPU may still read last frame's HDR for post.


### HDR migration checklist

Moving from direct swapchain rendering requires touching every pass that drew the world:

| Subsystem | Must update |
|-----------|-------------|
| Scene opaque / sky / transparent | `renderPass` handle, framebuffer binding |
| Tilemap / sprite composite | Same HDR pass + depth |
| Particle pass | Additive blend into RGBA16F |
| Shadow passes | **No change** — still pre-HDR |
| Tonemap | New fullscreen pass to swapchain |

Verify in RenderDoc: scene draws show **RGBA16F** color attachment, not `B8G8R8A8_UNORM`. Sun-facing pixels should exceed 1.0 in HDR before tonemap crushes them.


### Linear workflow audit

Trace one texel from albedo file to monitor:

```text
sRGB PNG → sparkSrgbToLinear → BRDF → RGBA16F → × exposure → ACES → gamma → UNORM swapchain
```

Any extra `pow(..., 2.2)` or missing decode step washes out contrast. HDR headroom only helps if sun values **exceed 1.0** in the float buffer before tonemap — verify in RenderDoc texture inspect.


### Post pass isolation

Tonemap and SSAO share the **fullscreen triangle** pattern but target different render passes and input views. Keeping them in separate classes prevents scene code from accidentally binding swapchain during world draw. Each post pass owns its push constants — exposure vs ssaoRadius — avoiding UBO reallocation per frame.


### Operational depth and demo validation

### Operational depth

Spark's reference demos (**ThreeDDemo**, **TerrainDemo**, **TimeOfDayDemo**) exist to validate each rendering contract in isolation before you merge subsystems in your engine. Run the demo that matches this chapter's topic after every implementation step — if Spark looks correct and your port does not, the bug is almost always struct layout, pass order, or matrix convention rather than a missing Vulkan feature.

**Descriptor set 0** carries the scene UBO, texture arrays, shadow maps, and punctual shadow SSBO. When debugging binding errors, use RenderDoc's pipeline state view to compare Spark's bound descriptors against yours for the same draw index. A common regression when adding shadows or HDR is recreating the descriptor set layout without re-binding shadow map views after swapchain recreate.

**Frame-in-flight indexing** must stay consistent: CPU writes UBO slot *i* while GPU may read slot *i* from the previous frame's submit. Shadow matrices uploaded in `WriteUniformBuffer(frameIndex)` must match the `frameIndex` passed to `RecordShadowMapPass` and the scene pass descriptor bind — mixing indices produces one-frame-late shadows that look like temporal filtering bugs.

When extending Spark's pattern, prefer **pure CPU modules** (cascade math, render volume resolve) that you can unit-test without Vulkan instances. GPU record functions should consume precomputed arrays — not re-derive frustum corners inside tight draw loops.


### RGBA16F precision

Half float has **10-bit mantissa** — sufficient for HDR scene radiance before tonemap. Sun values ~10–20 remain distinguishable; after ACES they map to display range. Use RGBA32F only if scientific HDR or aggressive GPU bloom headroom is required — doubles bandwidth.

Verify extreme emissive does not Inf/NaN — fire particles with additive stacking can overflow if not clamped before store (Spark particles use moderated emissive factors).


### Depth sharing across passes

Tilemap, sprite, and particle passes read the same depth buffer written by opaque scene geometry. **LoadOp CLEAR** at HDR pass start resets depth; opaque draws first establish occlusion. Particles with additive blending still depth-test against opaque depth — Part VIII relies on this ordering.


### Recreate dependency graph

```text
Swapchain recreate
  → HDR flight targets (extent)
  → Scene pipelines (renderPass compatibility)
  → Particle / tilemap pipelines
  → SSAO / tonemap (input views)
```
Missing one step → `VK_ERROR_INCOMPATIBLE_RENDER_PASS` or validation about framebuffer extent mismatch.


### HDR debugging in RenderDoc

After capturing **ThreeDDemo**, select the HDR color attachment before tonemap. Texture inspect on sun-facing pixels — linear float values should exceed 1.0. If all values ≤ 1.0, either exposure is too low, sun intensity is clamped in scene shader, or you are viewing the wrong attachment (swapchain instead of RGBA16F).

Compare the same pixel after tonemap pass — display values should sit in 0–1 without channel clipping to exactly 1.0 on all RGB unless intentionally blown out.

SSAO scratch (when enabled) should show modulated HDR — not identical to raw scene unless strength is zero.


### Post stack ordering invariants

These invariants must hold every frame:

```text
1. Shadow passes complete before HDR scene begin
2. HDR scene end → color in SHADER_READ layout
3. SSAO (if on) reads HDR + copied depth → scratch RGBA16F
4. Tonemap reads scratch OR raw HDR → swapchain
5. Screen UI + ImGui after tonemap (display space)
```

Violating order 4 with UI before tonemap makes widgets exposure-scaled. Violating order 2 causes validation errors when SSAO or tonemap sample HDR.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### No clamp in scene.frag

**Concept:** Linear vs display-referred color

**Decision:** No clamp in scene.frag

**Why:** Scene shaders output unclamped linear radiance; tonemap handles display compression. Clamping early makes exposure tuning and emissive highlights impossible.

**In Spark:** `scene.frag` HDR output near end of main — values may exceed 1.0.

### VulkanHdrTonemapPass owns HDR render pass

**Concept:** HDR attachment pairing

**Decision:** VulkanHdrTonemapPass owns HDR render pass

**Why:** One module defines RGBA16F + D32 attachment descriptions, subpass dependencies, and per-flight framebuffers — scene code consumes handles rather than duplicating attachment specs.

**In Spark:** `VulkanHdrTonemapPass::CreateRenderPass` lines 10–68.

### Single render pass for world draws

**Concept:** Shared HDR subpass

**Decision:** Single render pass for world draws

**Why:** Opaque meshes, 2D composite, and particles share depth for occlusion. One begin/end pair minimizes layout transitions and matches forward-renderer compositing models.

**In Spark:** `RecordSceneCommandBuffer` lines 269–330 — one HDR subpass.

### Post module creates, scene module consumes

**Concept:** Render pass ownership

**Decision:** Post module creates, scene module consumes

**Why:** When display format or HDR precision changes, one edit point propagates. Scene pipeline create receives `hdrTonemapPass.HdrRenderPass()`.

**In Spark:** `include/spark/render/post/VulkanHdrTonemapPass.hpp`.

### RecreateHdrFlightTargets hook

**Concept:** Recreate on resize

**Decision:** RecreateHdrFlightTargets hook

**Why:** HDR images match swapchain extent after every resize. Called from `RecreateSwapchain` before dependent pipeline rebuild.

**In Spark:** `VulkanRenderer::RecreateSwapchain` → `RecreateHdrFlightTargets()` at line 119.

### TransitionColorToShaderRead after scene pass

**Concept:** HDR migration checklist

**Decision:** TransitionColorToShaderRead after scene pass

**Why:** Explicit barrier after `vkCmdEndRenderPass` puts HDR color in `SHADER_READ_ONLY_OPTIMAL` for SSAO/tonemap sampling without validation hazards.

**In Spark:** `hdrTonemapPass.TransitionColorToShaderRead` at line 332.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/post/VulkanHdrTonemapPass.cpp`
- `src/spark/render/core/VulkanRenderer.cpp`
- `include/spark/render/post/VulkanHdrTonemapPass.hpp`
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp`
- `src/spark/render/sprites2d/VulkanParticlePass.cpp`

### Step 1 — Read HDR render pass creation

`CreateRenderPass` pairs float16 color with D32 depth. Note color `finalLayout = SHADER_READ_ONLY_OPTIMAL` — post passes sample this attachment without an extra copy.

*HDR render pass with color and depth attachments*

```cpp
// src/spark/render/post/VulkanHdrTonemapPass.cpp (lines 10–68)
void VulkanHdrTonemapPass::CreateRenderPass(VkDevice device, VkFormat depthFormat) {
    DestroyRenderPass(device);

    VkAttachmentDescription colorAttachment{};
    colorAttachment.format = kColorFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
    colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
    colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    colorAttachment.initialLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    colorAttachment.finalLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;

    VkAttachmentDescription depthAttachment{};
    depthAttachment.format = depthFormat;
    depthAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
    depthAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
    depthAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    depthAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    depthAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    depthAttachment.initialLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
    depthAttachment.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    VkAttachmentReference colorRef{};
    colorRef.attachment = 0;
    colorRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

    VkAttachmentReference depthRef{};
    depthRef.attachment = 1;
    depthRef.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    VkSubpassDescription subpass{};
    subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpass.colorAttachmentCount = 1;
    subpass.pColorAttachments = &colorRef;
    subpass.pDepthStencilAttachment = &depthRef;

    VkSubpassDependency dep{};
    dep.srcSubpass = VK_SUBPASS_EXTERNAL;
    dep.dstSubpass = 0;
    dep.srcStageMask =
            VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
    dep.srcAccessMask = 0;
    dep.dstStageMask =
            VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
    dep.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;

    const VkAttachmentDescription attachments[] = {colorAttachment, depthAttachment};
    VkRenderPassCreateInfo rpInfo{};
    rpInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
    rpInfo.attachmentCount = 2;
    rpInfo.pAttachments = attachments;
    rpInfo.subpassCount = 1;
    rpInfo.pSubpasses = &subpass;
    rpInfo.dependencyCount = 1;
    rpInfo.pDependencies = &dep;
    if (vkCreateRenderPass(device, &rpInfo, nullptr, &hdrRenderPass) != VK_SUCCESS) {
        throw std::runtime_error("vkCreateRenderPass (HDR) failed");
    }
```

### Step 2 — Follow scene recording into HDR

After shadow passes, the renderer clears HDR color and depth, records opaque scene, 2D composite, and particles inside **one** subpass, then ends the pass.

*HDR subpass: scene, 2D composite, particles*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 255–330)
    VkRenderPassBeginInfo renderPassBeginInfo{};
    renderPassBeginInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
    if (!hdrTonemapPass.HasFlight(frameIndex) ||
        hdrTonemapPass.Flight(frameIndex).framebuffer == VK_NULL_HANDLE) {
        throw std::runtime_error("RecordSceneCommandBuffer: HDR framebuffer missing");
    }

    renderPassBeginInfo.renderPass = hdrTonemapPass.HdrRenderPass();
    renderPassBeginInfo.framebuffer = hdrTonemapPass.Flight(frameIndex).framebuffer;
    renderPassBeginInfo.renderArea.offset = {0, 0};
    renderPassBeginInfo.renderArea.extent = presentSwapchain().extent;
    renderPassBeginInfo.clearValueCount = 2;
    renderPassBeginInfo.pClearValues = clearColors;

    vkCmdBeginRenderPass(commandBuffer, &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);

    if (frameIndex < sceneDescriptors.DescriptorSetCount()) {
        const VulkanCustomMeshPool::Bindings customMesh = customMeshPool.GetBindings();
        const VulkanSceneOpaqueRecordContext opaqueCtx{
                .scene = &pendingScene,
                .sceneParamsValid = sceneParamsValid,
                .frameIndex = frameIndex,
                .extent = presentSwapchain().extent,
                .pipelineLit = scenePipeline.PipelineLit(),
                .pipelineSky = scenePipeline.PipelineSky(),
                .pipelineLitTransparent = scenePipeline.PipelineLitTransparent(),
                .pipelineLayout = scenePipeline.PipelineLayout(),
                .descriptorSet = sceneDescriptors.DescriptorSet(frameIndex),
                .meshBindings =
                        {
                                .staticVertexBuffer = vertexBuffer,
                                .staticIndexBuffer = indexBuffer,
                                .customVertexBuffer = customMesh.vertexBuffer,
                                .customIndexBuffer = customMesh.indexBuffer,
                                .cubeIndexCount = cubeIndexCount,
                                .planeIndexCount = planeIndexCount,
                                .planeFirstIndex = planeFirstIndex,
                                .cubeVertexOffset = cubeVertexOffset,
                                .planeVertexOffset = planeVertexOffset,
                                .customDrawPacked = &customDrawPacked,
                        },
                .skinSsboMapped = &sceneDescriptors.SkinSsboMapped(),
                .maxSkinJoints = VulkanSceneDescriptors::kMaxSkinJoints,
        };
        sceneOpaquePass.Record(commandBuffer, opaqueCtx);
        sceneOpaquePass.RecordTransparent(commandBuffer, opaqueCtx, customDrawPackedTransparent);
    }

    if (sceneParamsValid && frameIndex < sceneDescriptors.DescriptorSetCount()) {
        const VulkanTilemapRecordContext tilemapCtx{
                .scene = &pendingScene,
                .sceneParamsValid = sceneParamsValid,
                .frameIndex = frameIndex,
                .extent = presentSwapchain().extent,
                .vertexBuffer = vertexBuffer,
                .indexBuffer = indexBuffer,
                .quadFirstIndex = spriteQuadFirstIndex,
                .quadIndexCount = spriteQuadIndexCount,
                .descriptorSet = sceneDescriptors.DescriptorSet(frameIndex),
        };
        const VulkanSpriteRecordContext spriteCtx{
                .scene = &pendingScene,
                .sceneParamsValid = sceneParamsValid,
                .frameIndex = frameIndex,
                .extent = presentSwapchain().extent,
                .vertexBuffer = vertexBuffer,
                .indexBuffer = indexBuffer,
                .quadFirstIndex = spriteQuadFirstIndex,
                .quadIndexCount = spriteQuadIndexCount,
                .descriptorSet = sceneDescriptors.DescriptorSet(frameIndex),
        };
        composite2DPass.Record(commandBuffer, tilemapPass, spritePass, tilemapCtx, spriteCtx);
    }
    particlePass.Record(commandBuffer, frameIndex, presentSwapchain().extent, pendingScene, sceneParamsValid);

    vkCmdEndRenderPass(commandBuffer);
```

### Step 3 — Transition HDR to shader read

Immediately after `vkCmdEndRenderPass`, Spark transitions the HDR color image for post sampling. SSAO and tonemap depend on this layout.

*Post-scene HDR layout transition*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 330–333)
    vkCmdEndRenderPass(commandBuffer);

    hdrTonemapPass.TransitionColorToShaderRead(commandBuffer, frameIndex);

```

### Step 4 — Trace swapchain recreate path

When the window resizes, HDR flight targets rebuild alongside presentation resources. Confirm your engine calls the equivalent **before** recreating scene pipelines.

*HDR targets recreated with swapchain*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 112–119)
    frameCapture.RecreateSwapchainResources(
            physicalDevice(), device(), presentSwapchain().imageFormat, presentSwapchain().extent);

    sceneDepthFormat = VulkanRendererGpu::FindDepthFormat(physicalDevice());
    CreateDepthResources();
    CreateRenderPass();
    screenSpaceEffectsPass.CreateRenderPass(device());
    RecreateHdrFlightTargets();
```

### Step 5 — Verify in RenderDoc

Capture **ThreeDDemo**. Scene draws should target **RGBA16F** — the swapchain image stays unused until tonemap. Inspect a sun-facing pixel's HDR value before post; it should exceed 1.0.

### Step 6 — Point scene pipelines at HDR

Recreate lit/sky/transparent/tilemap/sprite/particle pipelines when `hdrRenderPass` handle changes — same flow as swapchain recreate from Part I. Pass the HDR framebuffer handle at record time via `hdrTonemapPass.Flight(frameIndex).framebuffer`.

### Step 7 — Test window resize

Drag the window edge repeatedly. Confirm HDR extent updates, frames continue without validation errors, and aspect ratio stays correct. Guard `OUT_OF_DATE` during resize before capturing screenshots.

### Step 8 — Audit pass list

List every pass that must switch render pass handle when you add HDR. Missing one pass produces black frames or `VK_ERROR_INCOMPATIBLE_RENDER_PASS` at pipeline bind time.

## 5. How This Connects to the Rest of Your Engine

Parts **IV–VII** scene shaders output HDR without clamping. Part **IX** shadows modulate lighting into the same buffer. Part **XI** sprites draw here — not into swapchain. Chapter 2 tonemap consumes this image. Chapter 3 SSAO samples HDR color and scene depth derived from the same pass.

## 6. Pitfalls and Debugging

- **Pipeline still targets swapchain render pass** — black or validation error after HDR migration.
- **sRGB format on HDR target** — wrong for linear lighting math; use float UNORM/SFLOAT.
- **Forgetting recreate on resize** — HDR extent must match swapchain every frame.
- **Separate subpass per pass type** — loses shared depth; particles may draw through walls.

## 7. Exercises

1. Capture RenderDoc: confirm scene pass color format is RGBA16F.
2. List every pass that must switch render pass handle when you add HDR.
3. Visualize HDR buffer before tonemap — see values above 1.0 on sun-facing surfaces.
4. Measure memory: RGBA16F + D32 per flight target at 1080p vs 4K.
5. Document load/store ops on HDR color and depth attachments.

## Summary

HDR render targets are float color plus depth offscreen images. Spark routes all world drawing through one HDR subpass owned by VulkanHdrTonemapPass, transitions color to shader-read after the pass, then samples it for SSAO and tonemap.

---

Next: [ACES Tonemapping](02-aces-tonemap.md).
