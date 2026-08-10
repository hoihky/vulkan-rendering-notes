# Depth Testing and Face Culling

> **Build-Along Milestone:** You can explain why Spark uses three scene pipelines (lit, sky, transparent) and how depth/cull state differs for each.

**You will learn:**

- Configure depth test, depth write, and compare operations in the graphics pipeline.
- Choose front face and cull mode for your asset winding convention.
- Use dynamic raster state where draw items need per-object culling overrides.

## 1. The Problem We Are Solving

After the vertex shader emits `gl_Position`, the GPU rasterizes triangles and runs the fragment shader. **Fixed-function** stages then decide whether each fragment updates the color and depth attachments:

- **Depth test** — is this fragment closer than the stored depth?
- **Depth write** — should we store the new depth value?
- **Face culling** — discard back-facing (or front-facing) triangles entirely.

Wrong settings produce classic bugs: everything invisible (cull wrong winding), z-fighting (depth precision), sky occluding geometry (depth test on sky), transparent objects blocking each other (depth write on glass).

Spark encodes most depth/cull policy in `VulkanScenePipeline.cpp` with three pipeline variants and **dynamic** cull mode for per-draw overrides. Shaders influence depth only through `gl_Position` (sky Z nudge in `scene.vert`).

## 2. Core Concepts

### Depth buffer recap

The HDR scene pass attaches a **depth image** cleared to `1.0` (far plane in Vulkan). Each fragment computes a depth value in `[0, 1]` after perspective divide (unless depth range changed).

```text
Fragment depth  vs  Depth buffer at (x,y)
        │
        ▼
   Compare op (LESS, LEQUAL, ...)
        │
   pass → optional depth write → color blend
   fail → discard fragment
```

Spark default for opaque meshes:

| Field | Value | Meaning |
|-------|-------|---------|
| `depthTestEnable` | `TRUE` | Test against buffer |
| `depthWriteEnable` | `TRUE` | Update buffer on pass |
| `depthCompareOp` | `LESS` | Nearer wins (smaller Z) |

### Three Spark scene pipelines

| Pipeline | Depth test | Depth write | Cull | Blend |
|----------|------------|-------------|------|-------|
| `pipelineLit` | ON | ON | BACK | OFF |
| `pipelineSky` | OFF | OFF | NONE | OFF |
| `pipelineLitTransparent` | ON | OFF | BACK | ON (alpha) |

```cpp
// Opaque
depthStencil.depthTestEnable = VK_TRUE;
depthStencil.depthWriteEnable = VK_TRUE;
depthStencil.depthCompareOp = VK_COMPARE_OP_LESS;

// Sky
depthStencilSky.depthTestEnable = VK_FALSE;
depthStencilSky.depthWriteEnable = VK_FALSE;

// Transparent
depthStencilTransparent = depthStencil;
depthStencilTransparent.depthWriteEnable = VK_FALSE;
```

Transparent draws read depth (occluded by opaque) but do not write — later transparent pixels can still shade over earlier ones (sorting limitations remain; true order-independent transparency is advanced).

### Face culling and winding

```cpp
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;
```

Vulkan culls triangles whose vertices appear **clockwise** in screen space when `frontFace = CLOCKWISE`. glTF assets are often exported CCW with `frontFace = COUNTER_CLOCKWISE`.

| Symptom | Likely fix |
|---------|------------|
| Hollow mesh (see inside) | Cull mode too aggressive |
| Invisible mesh | Cull front instead of back, or wrong winding |
| Flipped normals only | Winding fix, not cull |

Sky uses `CULL_MODE_NONE` — you are inside the dome; both faces may be visible from interior.

### Dynamic cull and front face

Spark enables dynamic states:

```cpp
dynamicStatesList.PushBack(VK_DYNAMIC_STATE_CULL_MODE);
dynamicStatesList.PushBack(VK_DYNAMIC_STATE_FRONT_FACE);
```

`VulkanSceneApplyRasterState` calls `vkCmdSetCullMode` / `vkCmdSetFrontFace` per draw from `SceneDrawItem` flags — double-sided materials skip back-face cull without a separate pipeline.

### Shader-side depth: sky clip Z

Even with depth test disabled on sky pipeline, vertex shader sets:

```glsl
gl_Position = vec4(clip.x, clip.y, zw * (1.0 - 1e-4), zw);
```

If sky ever shares a pipeline with depth on, this keeps sky at the far plane. Fragment shader does not write `gl_FragDepth`.

### Depth format and precision

Depth attachment format (typically `D32_SFLOAT` or `D24_UNORM_S8_UINT`) affects z-fighting on large outdoor scenes. Spark's shadow and scene passes use formats chosen in render pass creation (Part II). **Logarithmic depth** and **reversed-Z** are engine-wide policy decisions — Spark uses classic LESS with cleared 1.0 far.

### Stencil (Spark default off)

`stencilTestEnable = VK_FALSE` on all three pipelines. Stencil masking appears in UI or special effects later — not scene opaque pass.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Pipeline variants not shader variants

**Concept:** Three Spark scene pipelines

**Decision:** Separate `VkPipeline` for opaque, sky, transparent — same SPIR-V modules

**Why:** Depth/blend/raster fixed state is immutable per pipeline; cheaper than dynamic depth enable (not universally dynamic).

**In Spark:** `VulkanScenePipeline::CreateGraphicsPipeline` three `vkCreateGraphicsPipelines` calls.

### LESS compare, clear 1.0

**Concept:** Depth buffer recap

**Decision:** Standard forward depth; no reversed-Z in Spark baseline

**Why:** Simpler mental model for book readers; works with default projection matrices.

**In Spark:** Render pass clear depth `1.0f`; `VK_COMPARE_OP_LESS`.

### Transparent: test on, write off

**Concept:** Transparent pipeline

**Decision:** Read depth for occlusion, never write

**Why:** Standard alpha blend pass; avoids punching holes in depth for overlapping glass.

**In Spark:** `depthStencilTransparent`.

### Dynamic cull per draw

**Concept:** Dynamic cull and front face

**Decision:** Pipeline default BACK cull; disable via command for double-sided

**Why:** Avoids doubling pipeline count for every material flag.

**In Spark:** `VulkanSceneApplyRasterState` in opaque pass record.

### Sky: no depth, no cull

**Concept:** Sky pipeline

**Decision:** Dedicated pipeline bound when `SceneDrawItem::skyMode != None`

**Why:** Interior sky dome; depth test would fight with scene geometry at infinity.

**In Spark:** `VulkanSceneOpaquePass` pipeline switch around line 62–72.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `src/spark/render/scene/VulkanScenePipeline.cpp` (depth, raster, blend)
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp` (pipeline bind order)
- `src/spark/render/scene/VulkanSceneRaster.cpp` (dynamic cull)
- `shaders/scene.vert` (sky `gl_Position` hack)

### Step 1 — Confirm depth clear

In render pass begin, clear depth to `1.0`. RenderDoc → Texture Viewer → depth attachment after clear should be white (far).

### Step 2 — Opaque only

Bind `pipelineLit`. Draw cube. Rotate — back faces culled, front visible. Set `cullMode = NONE` temporarily — see back faces.

### Step 3 — Flip frontFace

Toggle `CLOCKWISE` ↔ `COUNTER_CLOCKWISE`. One setting makes mesh disappear — you found your asset winding.

### Step 4 — Transparent pass

Draw semi-transparent quad over opaque. With depth write ON (wrong), second transparent layer fails. Fix: `pipelineLitTransparent`.

### Step 5 — Sky pass

Bind sky mesh with `pipelineSky`. Verify depth test disabled in RenderDoc pipeline state. Rotate camera — geometry should cover sky where closer.

### Step 6 — Z-fighting experiment

Place two coplanar quads. Observe flicker. Mitigations: slight offset, different materials, or polygon offset (`VkPipelineRasterizationStateCreateInfo::depthBias*` — Spark uses selectively in shadow passes).

## 5. How This Connects to the Rest of Your Engine

Chapter **8** covers alpha blending on `pipelineLitTransparent`. Part **V** shadow passes use depth-only rendering with different compare ops for bias. Part **II** defined the render pass attachment that owns the depth image.

Depth and cull are **pipeline state** — changing them requires new `VkPipeline` or dynamic commands, not shader recompile (except `gl_Position`).

## 6. Pitfalls and Debugging

- **Everything invisible** — 90% winding/cull; fix `frontFace` before rewriting shaders.
- **Sky covers world** — sky pipeline still writes depth, or drawn after opaque without depth test off.
- **Transparent sorting wrong** — depth write off helps but does not sort fragments; sort draws back-to-front approximately.
- **UI depth fighting scene** — UI pass usually disables depth or uses separate pass with guaranteed on-top draw order.
- **Dynamic state not in pipeline create** — `vkCmdSetCullMode` without `VK_DYNAMIC_STATE_CULL_MODE` → validation error.

## 7. Exercises

1. Draw same mesh with `CULL_MODE_FRONT` — photograph inside-out result.
2. Enable depth write on transparent pipeline — describe artifact on overlapping quads.
3. Log which pipeline (lit/sky/transparent) binds per draw index in one frame.
4. Plot NDC Z vs eye Z for perspective matrix — why precision clusters near far plane.
5. Add `VK_DYNAMIC_STATE_DEPTH_BIAS` for shadow pass preview (Part V teaser).

## Summary

Spark opaque meshes use depth test LESS with read/write, back-face cull, and clockwise front face. Sky disables depth and culling; transparent keeps depth test but disables depth write and enables alpha blend. Per-draw cull overrides use dynamic state. Coordinate pipeline setup with asset winding and render pass clear values.

---

Next: [Blending and Transparency](08-blending.md).
