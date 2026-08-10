# Terrain in Scene Pass

> **Build-Along Milestone:** TerrainDemo outdoor shot shows shadows, fog horizon, and tonemapped HDR sun.

**You will learn:**

- Draw terrain through the standard lit opaque pipeline with Custom mesh slot.
- Apply tiled UVs and MaterialComponent PBR maps to heightfields.
- Integrate terrain with CSM shadows, fog, SSAO, and HDR tonemap.

## 1. The Problem We Are Solving

Terrain is not a separate rendering subsystem — it is **big opaque geometry** submitted like any mesh with materials and shadow flags. Outdoor demos combine terrain with Part IX CSM, Part X fog/SSAO/tonemap, and Part VII materials.

Milestone: TerrainDemo screenshot with shadows on hills, fog horizon, and tonemapped HDR sun on slopes.

## 2. Core Concepts

### No special-case terrain renderer

Terrain is a **large custom opaque mesh** — it uses `VulkanSceneOpaquePass` with the lit pipeline, scene UBO, material push constants, and texture array like any `MeshComponent` draw. There is no `TerrainRenderer` singleton. Submit packs terrain into `SceneDrawItem` with `mesh = Custom` and shadow flags enabled.

This design keeps outdoor scenes on the **same code path** you debugged in Parts V–IX. Terrain receives CSM shadows, clustered lights (if enabled), fog, and SSAO from Part X post chain without terrain-specific shader variants for baseline grass.

### HDR scene pass integration

Recording order in `RecordSceneCommandBuffer`:

```text
shadow passes → HDR begin → opaque (terrain here) → transparent → 2D composite → particles → HDR end
```

Terrain writes **linear HDR radiance** and **depth** like other opaque meshes. Capture TerrainDemo in RenderDoc — terrain draws appear in the opaque lit pass on RGBA16F target with depth write ON. Tonemap happens later; judge outdoor look only after the full post chain.

Sun-facing slopes exceed 1.0 linear — tonemap compresses for display. Terrain depth written here is authoritative for SSAO contact and sprite depth test in hybrid scenes.

### Tiled materials on slopes

Planar UVs from the generator scale with world XZ:

```cpp
v.texCoord = { x / uvRepeat, z / uvRepeat };
```

**MaterialComponent** enables normal maps — slopes show tangent-space detail from Part VII ORM workflow. Steep cliffs may stretch UVs along the vertical world axis; consider cliff blend materials or triplanar extensions (outside Spark baseline) for production cliffs.

TerrainDemo loads a soil texture and assigns it through `MaterialComponent` — the generator does not set metallic or roughness.

### CSM shadow interaction

Large outdoor worlds need tuned **cascade splits** (Part IX) — near cascade covers ground detail under camera; far cascades cover horizon. Terrain both **casts** and **receives** shadows by default draw flags. Props on terrain show contact shadows; terrain hills cast onto valleys.

If ground looks floating, verify **receive shadows** flag on terrain draw item and that directional shadow pass recorded before HDR opaque. Self-shadow acne on noisy procedural hills may need normal bias tweaks on the directional light, not terrain code changes.

### Fog and SSAO contact

Part X **fog** fades distant terrain toward horizon color — match fog tint to sky for seamless outdoor shots. **SSAO** darkens creases at object bases resting on terrain — toggle `ssaoEnabled` in TerrainDemo params and compare ground contact under props.

SSAO samples scene depth from the HDR flight — terrain depth must be written correctly in the opaque pass for contact darkening to appear. Toggle fog and SSAO off to isolate albedo tuning when material looks wrong.

### Hybrid 2.5D with Part XI

Sprites and tilemaps (Part XI) decorate outdoor scenes at higher render layers — draw order is pass-level (HDR 2D after opaque terrain) plus sprite sort keys. Orthographic 2D overlays on perspective terrain require deliberate depth and sort policy.

Sprites in `composite2D` draw **after** opaque terrain with depth test — characters walk up hills visually if sprite anchor Y matches physics ground height sampled from the heightfield.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SceneMeshSlot::Custom opaque draw

**Concept:** No special-case terrain renderer

**Decision:** SceneMeshSlot::Custom opaque draw

**Why:** One lit pipeline to maintain — terrain benefits from all mesh improvements automatically.

**In Spark:** SceneSubmit rigid mesh path with Custom slot.

### sceneOpaquePass.Record

**Concept:** HDR scene pass integration

**Decision:** sceneOpaquePass.Record

**Why:** Terrain listed alongside props in same opaque sort — depth occludes correctly.

**In Spark:** `VulkanRenderer.cpp` lines 269–300.

### MaterialComponent on terrain entity

**Concept:** Tiled materials on slopes

**Decision:** MaterialComponent on terrain entity

**Why:** Shading decoupled from height generation — swap biomes without regenerating topology.

**In Spark:** TerrainDemo material setup.

### Default cast/receive shadow flags

**Concept:** CSM shadow interaction

**Decision:** Default cast/receive shadow flags

**Why:** Outdoor grounding requires terrain receive — flags on SceneDrawItem from submit.

**In Spark:** Part IX shadow flags in SceneSubmitDetail.

### SceneRenderParams post toggles

**Concept:** Fog and SSAO contact

**Decision:** SceneRenderParams post toggles

**Why:** Same SSAO/fog path as ThreeDDemo — outdoor integration is param-driven.

**In Spark:** TerrainDemo enables SSAO/fog like other outdoor demos.

### Separate composite2D after opaque

**Concept:** Hybrid 2.5D with Part XI

**Decision:** Separate composite2D after opaque

**Why:** Clear pass boundary — terrain never mixes with sprite instancing path.

**In Spark:** `VulkanRenderer.cpp` lines 303–327.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/demo/TerrainDemo.hpp`
- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Capture TerrainDemo opaque pass

Terrain in HDR opaque pass.

*opaque record*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 269–300)
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
```

### Step 2 — Verify shadow receive on ground

Mesh shadow on terrain — props should cast onto hills.

### Step 3 — Tune fog and sky horizon

Match fog color to sky gradient from Part X.

### Step 4 — Toggle SSAO

Contact darkening at prop bases on terrain.

### Step 5 — Apply ORM grass material

Normal map on slopes via MaterialComponent.

### Step 6 — HDR headroom check

Sun-facing slopes peak above 1.0 linear before tonemap.

### Step 7 — Screenshot integration shot

Shadows, fog, SSAO, tonemap together.

### Step 8 — Milestone — outdoor HDR shot

Complete outdoor frame.

## 5. How This Connects to the Rest of Your Engine

Part **IX** CSM + Part **X** fog/SSAO/tonemap + Part **VII** materials converge on TerrainDemo. Part **XI** sprites and tilemaps can overlay hybrid scenes at higher layers. Chapter 4 **frustum cull** reduces off-screen terrain draw cost on large subdiv.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

Terrain draws in opaque HDR before Part XI composite2D. Outdoor acceptance tests should capture one frame with shadows, fog, SSAO, and tonemap all enabled — toggling features in isolation hides integration seams at horizon and object bases.

## 6. Pitfalls and Debugging

- **Terrain without receive shadows** — props float visually; enable receive flag.
- **Single-tile UV on large extent** — stretched grass; tune `worldUnitsPerTextureRepeat`.
- **Judging look before tonemap** — outdoor exposure lives in post chain.
- **Skipping normal recompute after brush** — slopes shade incorrectly until RegenerateMesh.
- **CSM splits too coarse** — blurry shadows on near ground; tune cascades not material.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

## 7. Exercises

1. Screenshot TerrainDemo with SSAO on vs off.
2. Apply normal-mapped material to steep slope.
3. Measure shadow cascade index at camera vs 200 m away on terrain.
4. RenderDoc pass list — count events from shadow through tonemap.
5. Confirm terrain uses `scene.vert`/`scene.frag`, not a forked terrain shader.

## Summary

Terrain renders as custom opaque mesh with standard lighting and post. Spark integrates outdoor effects on TerrainDemo — shadows, fog, SSAO, tonemap — without a dedicated terrain renderer.

---

Next: [Frustum Culling](04-frustum-cull.md).
