# Sky Pipeline Branch

> **Build-Along Milestone:** Sky recorded first in opaque pass; ground cubes correctly occlude background.

**You will learn:**

- Explain pipelineSky depth/raster differences from pipelineLit.
- Describe why sky draws sort before opaque meshes.
- Trace pipeline bind switches inside VulkanSceneOpaquePass::Record.

## 1. The Problem We Are Solving

Reusing the lit mesh pipeline for skies causes depth fighting and wrong cull — the camera sits **inside** the shell.
You need a **pipeline variant** (or dynamic state) with depth test/write off and no culling, recorded **before** world geometry.

Build along: diff `pipelineSky` vs `pipelineLit` create info and verify sort order puts sky draws first.

## 2. Core Concepts

### Shared shaders, different pipeline

Spark uses the **same** `scene.vert` / `scene.frag` modules for lit meshes and sky, but creates `pipelineSky` with different depth/raster state. Record loop binds sky pipeline when `d.skyMode != None`.


### Depth test off for sky

`depthStencilSky` disables depth test **and** write. Combined with vertex shader pushing `z = w * (1 - ε)`, sky fills color buffer first; opaque meshes always win depth later.


### Draw sort key −100

`DrawSortKey` returns −100 for sky items so `StableSortDrawItems` places them **before** ground and props even within one opaque pass.


### Dynamic cull override

`VulkanSceneApplyRasterState` sets `VK_CULL_MODE_NONE` for sky — view from inside the sphere requires no back-face cull.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Duplicate create info with depthStencilSky

**Concept:** pipelineSky sibling

**Decision:** Duplicate create info with depthStencilSky

**Why:** Minimal shader duplication; only depth and cull defaults differ from pipelineLit.

**In Spark:** `src/spark/render/scene/VulkanScenePipeline.cpp`

### gl_Position.z = w * (1 - 1e-4) when skyMode != 0

**Concept:** Clip-space z bias in vertex shader

**Decision:** gl_Position.z = w * (1 - 1e-4) when skyMode != 0

**Why:** Keeps sky at far plane when depth test is enabled in other engines; Spark disables test entirely.

**In Spark:** `shaders/scene.vert` lines 124–128

### Toggle pipelineSky / pipelineLit in draw loop

**Concept:** Runtime pipeline bind switch

**Decision:** Toggle pipelineSky / pipelineLit in draw loop

**Why:** Avoids separate render pass; sky and opaque share HDR target and descriptor set.

**In Spark:** `VulkanSceneOpaquePass::Record`

### Sky first, then mesh slot layers

**Concept:** Stable insertion sort by sort key

**Decision:** Sky first, then mesh slot layers

**Why:** Deterministic order across platforms; transparent pass excludes sky items.

**In Spark:** `SceneSubmitDrawPartition.cpp` DrawSortKey

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/scene/VulkanScenePipeline.cpp`
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp`
- `include/spark/render/scene/VulkanSceneRaster.hpp`
- `shaders/scene.vert`

### Step 1 — Inspect sky pipeline create info

*rasterizerSky cull none; depthStencilSky test/write disabled.*

```cpp
// src/spark/render/scene/VulkanScenePipeline.cpp (lines 127–152)
    VkPipelineRasterizationStateCreateInfo rasterizerSky{};
    rasterizerSky.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
    rasterizerSky.polygonMode = VK_POLYGON_MODE_FILL;
    rasterizerSky.lineWidth = 1.0F;
    rasterizerSky.cullMode = VK_CULL_MODE_NONE;
    rasterizerSky.frontFace = VK_FRONT_FACE_CLOCKWISE;

    VkPipelineMultisampleStateCreateInfo multisampling{};
    multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
    multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;

    VkPipelineDepthStencilStateCreateInfo depthStencil{};
    depthStencil.sType = VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO;
    depthStencil.depthTestEnable = VK_TRUE;
    depthStencil.depthWriteEnable = VK_TRUE;
    depthStencil.depthCompareOp = VK_COMPARE_OP_LESS;
    depthStencil.depthBoundsTestEnable = VK_FALSE;
    depthStencil.stencilTestEnable = VK_FALSE;

    VkPipelineDepthStencilStateCreateInfo depthStencilSky{};
    depthStencilSky.sType = VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO;
    depthStencilSky.depthTestEnable = VK_FALSE;
    depthStencilSky.depthWriteEnable = VK_FALSE;
    depthStencilSky.depthCompareOp = VK_COMPARE_OP_LESS;
    depthStencilSky.depthBoundsTestEnable = VK_FALSE;
    depthStencilSky.stencilTestEnable = VK_FALSE;
```

### Step 2 — Read vertex shader sky clip tweak

*Sky branch pushes clip z toward w for far-plane depth.*

```glsl
// shaders/scene.vert (lines 123–130)
    vec4 clip = ubo.viewProj * vec4(worldPos, 1.0);
    if (push.skyMode != 0) {
        // Push clip z slightly below w so NDC depth < 1 (depth buffer clears to 1; pipeline skips depth test).
        float zw = max(clip.w, 1e-5);
        gl_Position = vec4(clip.x, clip.y, zw * (1.0 - 1e-4), zw);
    } else {
        gl_Position = clip;
    }
```

### Step 3 — Trace record-time pipeline bind

*Bind pipelineSky when draw is sky; apply raster override.*

```cpp
// src/spark/render/scene/VulkanSceneOpaquePass.cpp (lines 56–80)
    bool skyPipelineBound = false;
    ModelPushConstants push{};
    SceneMeshGeometryBinding bound = SceneMeshGeometryBinding::None;

    for (std::size_t di = 0; di < ctx.scene->draws.GetSize(); ++di) {
        const SceneDrawItem& d = ctx.scene->draws[di];
        const bool isSky = (d.skyMode != SceneSkyMode::None);
        if (isSky) {
            if (!skyPipelineBound && ctx.pipelineSky != VK_NULL_HANDLE) {
                vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, ctx.pipelineSky);
                skyPipelineBound = true;
            }
        } else {
            if (skyPipelineBound) {
                vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, ctx.pipelineLit);
                skyPipelineBound = false;
            }
        }

        const SceneMeshDrawRange range = ResolveSceneMeshDrawRange(d, di, ctx.meshBindings);
        if (!range.drawable) {
            continue;
        }

        VulkanSceneApplyRasterState(commandBuffer, d, d.skyMode);
```

### Step 4 — Confirm sort order at submit

*DrawSortKey — sky items sort before mesh layers.*

```cpp
// src/spark/scene/SceneSubmitDrawPartition.cpp (lines 29–34)
int DrawSortKey(const SceneDrawItem& it) {
    if (it.skyMode != SceneSkyMode::None) {
        return -100;
    }
    return DrawSortLayer(it.mesh);
}
```


Demos call `StableSortDrawItems(drawList)` before pushing into `params.draws`.

### Step 5 — Dynamic cull for interior view

*VulkanSceneApplyRasterState — CULL_MODE_NONE for sky.*

```cpp
// include/spark/render/scene/VulkanSceneRaster.hpp (lines 17–33)
/** Sets dynamic cull/front-face state required by <c>VulkanScenePipeline</c> before each indexed draw. */
inline void VulkanSceneApplyRasterState(
        VkCommandBuffer commandBuffer,
        const SceneDrawItem& item,
        SceneSkyMode skyMode) noexcept {
    if (skyMode != SceneSkyMode::None) {
        vkCmdSetCullMode(commandBuffer, VK_CULL_MODE_NONE);
        vkCmdSetFrontFace(commandBuffer, VK_FRONT_FACE_CLOCKWISE);
        return;
    }
    if (item.doubleSided || item.mesh == SceneMeshSlot::GroundPlane) {
        vkCmdSetCullMode(commandBuffer, VK_CULL_MODE_NONE);
    } else {
        vkCmdSetCullMode(commandBuffer, VK_CULL_MODE_BACK_BIT);
    }
    vkCmdSetFrontFace(commandBuffer, VulkanSceneFrontFaceForModel(item.model));
}
```

### Step 6 — Milestone — ground occludes sky

Capture RenderDoc: sky draw appears **first** in the opaque pass, depth test off, ground pixels pass depth test and overwrite sky where geometry exists.

## 5. How This Connects to the Rest of Your Engine

**Sky Rendering Modes** sets `push.skyMode` consumed by the same shaders. **Material Submit** must preserve sort order
when flattening ECS into `params.draws`. Part **X** tonemap runs after this HDR pass unchanged.

## 6. Pitfalls and Debugging

- **Sky after opaque draws** — wastes fill rate tinting the whole screen wrong; sort key must be lowest.
- **Depth write on sky** — z-fighting with distant terrain; keep depthStencilSky write disabled.
- **Back-face cull on sphere** — invisible from interior; force CULL_MODE_NONE.
- **Separate render pass for sky** — valid but Spark avoids extra pass; match one-pass pattern unless profiling demands split.

## 7. Exercises

1. Intentionally draw sky last — measure overdraw or capture wrong layering in RenderDoc.
2. Compare pipeline create info fields side-by-side for Lit vs Sky.
3. Log draw order indices for sky vs ground in `StableSortDrawItems` output.
4. Re-enable depth test on `pipelineSky` — describe artifacts when camera is inside sphere.
5. Find `DrawSortKey` return value for sky items in submit code.

## Summary

pipelineSky disables depth test/write and culling; sky draws sort first and share scene shaders with an early-out sky branch.

---

Next: [IBL Basics](04-ibl-basics.md).
