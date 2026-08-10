# Frame Graph Ordering

> **Build-Along Milestone:** Frame graph diagram with particles slot annotated correctly.

**You will learn:**

- Place VulkanParticlePass::Record after sprites inside the HDR subpass.
- Explain why particles share the scene depth buffer.
- Trace post-HDR tonemap relative to particle contribution.

## 1. The Problem We Are Solving

Pass order is not cosmetic — drawing particles **before** opaque meshes ignores depth; drawing **after** tonemap
misses HDR glow. You must slot `particlePass.Record` correctly in the frame graph.

Build along: read RecordSceneCommandBuffer and annotate pass order in your engine diagram.

## 2. Core Concepts

### HDR subpass ordering

Within one HDR render pass: sky → opaque meshes → transparent meshes → tilemaps/sprites → **particles** → end.


### Shared depth buffer

Particles read the same depth attachment written by opaque scene draws — no separate particle depth target.


### Post-subpass tonemap

After vkCmdEndRenderPass, HDR color transitions to shader-read for SSAO/tonemap — particles already contributed to HDR color.


### Renderer ownership

`VulkanRenderer` owns `particlePass` alongside sceneOpaquePass, spritePass, hdrTonemapPass.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Particles after sprites

**Concept:** HDR subpass ordering

**Decision:** Particles after sprites

**Why:** Additive glow should overlay alpha-blended sprites; opaque world already established depth.

**In Spark:** `VulkanRenderer.cpp` Record order

### Same framebuffer as scene pass

**Concept:** Shared depth buffer

**Decision:** Same framebuffer as scene pass

**Why:** No extra render pass setup; particles piggyback HDR fb depth attachment.

**In Spark:** `hdrTonemapPass.Flight(frameIndex).framebuffer`

### EndRenderPass then tonemap

**Concept:** Post-subpass tonemap

**Decision:** EndRenderPass then tonemap

**Why:** Particles stay in linear HDR space with scene lighting before exposure.

**In Spark:** `hdrTonemapPass.RecordTonemap` after EndRenderPass

### particlePass member

**Concept:** Renderer ownership

**Decision:** particlePass member

**Why:** CreateGpuResources at init; CreateGraphicsPipeline after HDR render pass known; Destroy on teardown.

**In Spark:** `VulkanRenderer.hpp` particlePass field

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/core/VulkanRenderer.hpp`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Locate particlePass in renderer

*Comment listing particlePass among frame passes.*

```cpp
// include/spark/render/core/VulkanRenderer.hpp (lines 62–62)
 * `VulkanHdrTonemapPass`, `VulkanScreenSpaceEffectsPass` (SSAO), `VulkanParticlePass`,
```


*particlePass member.*

```cpp
// include/spark/render/core/VulkanRenderer.hpp (lines 135–135)
    VulkanParticlePass particlePass;
```

### Step 2 — Init and pipeline create

*CreateGpuResources at renderer init.*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 621–621)
    particlePass.CreateGpuResources(physicalDevice(), device(), VulkanFrameSync::kMaxFramesInFlight, shaderLoader);
```


*CreateGraphicsPipeline bound to HDR render pass.*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 125–125)
    particlePass.CreateGraphicsPipeline(device(), hdrTonemapPass.HdrRenderPass());
```

### Step 3 — Record order inside HDR subpass

*Opaque → tilemap/sprite composite → particlePass.Record.*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 299–328)
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
```

### Step 4 — End render pass and tonemap

*vkCmdEndRenderPass, HDR transition, tonemap to swapchain.*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 330–359)
    vkCmdEndRenderPass(commandBuffer);

    hdrTonemapPass.TransitionColorToShaderRead(commandBuffer, frameIndex);

    const bool ssaoActive = sceneParamsValid && pendingScene.ssaoEnabled;
    if (ssaoActive && frameIndex < sceneDescriptors.DescriptorSetCount() && hdrTonemapPass.HasFlight(frameIndex) &&
        screenSpaceEffectsPass.HasFlight(frameIndex)) {
        screenSpaceEffectsPass.UpdateDescriptor(
                device(),
                frameIndex,
                sceneDescriptors.UniformBuffer(frameIndex),
                hdrTonemapPass.Flight(frameIndex),
                screenSpaceEffectsPass.Flight(frameIndex));
        screenSpaceEffectsPass.Record(
                commandBuffer,
                frameIndex,
                presentSwapchain().extent,
                sceneDepthResources[frameIndex],
                pendingScene);
        hdrTonemapPass.UpdateTonemapDescriptor(
                device(), frameIndex, screenSpaceEffectsPass.OutputView(frameIndex));
    } else {
        hdrTonemapPass.UpdateTonemapDescriptor(device(), frameIndex);
    }

    hdrTonemapPass.RecordTonemap(
            commandBuffer,
            imageIndex,
            frameIndex,
            presentRenderPass.vkPass,
```

### Step 5 — Draw ordering diagram

```text
Shadow maps (pre-HDR) → Begin HDR RP
  sky → opaque → transparent → tilemaps → sprites → particles
End HDR RP → SSAO? → tonemap → UI → present
```

### Step 6 — Regression check

Move particle Record before opaque in a test branch — observe particles occluded incorrectly; revert.

## 5. How This Connects to the Rest of Your Engine

**Part V** established HDR render pass and tonemap. **Part VI** scene opaque/transparent passes write depth.
**2D sprites** draw alpha-blended before additive particles. **Particle Demo** relies on this order for glow over ground.

## 6. Pitfalls and Debugging

- **Particles before depth prepass** — depth buffer empty; occlusion wrong.
- **Separate render pass for particles** — extra bandwidth unless you need different resolution.
- **Forgot CreateGraphicsPipeline on resize** — HDR render pass handle changes; pipeline stale.
- **UI before particles in same subpass** — rare; Spark draws UI after tonemap on swapchain pass.

## 7. Exercises

1. List five `Record` calls in order inside HDR subpass from `VulkanRenderer.cpp`.
2. Capture frame — confirm particles in same render pass as opaque draws (RenderDoc).
3. Document where you would insert shadow-decals relative to particles.
4. Move `particlePass.Record` before opaque — screenshot occlusion bug.
5. Trace when `vkCmdEndRenderPass` runs relative to particle draw.

## Summary

VulkanRenderer records particlePass after scene opaque/transparent and sprites, inside HDR subpass sharing depth; tonemap runs after EndRenderPass.

---

Next: [Particle Demo](08-particle-demo.md).
