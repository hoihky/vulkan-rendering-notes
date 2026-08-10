# Blending and Transparency

> **Build-Along Milestone:** You understand `pipelineLitTransparent` blend factors and why Spark sorts transparent draws separately from opaque.

**You will learn:**

- Configure `VkPipelineColorBlendAttachmentState` for standard alpha blending.
- Combine blend state with depth test/write policy for transparency.
- Record transparent draws in a separate pass slice with correct pipeline binding.

## 1. The Problem We Are Solving

Opaque rendering is straightforward: write color and depth, nearest fragment wins. **Transparency** requires combining the new fragment color with what is already in the framebuffer — typically **alpha compositing**:

```text
C_out = C_src * α + C_dst * (1 - α)
```

Vulkan does not sort fragments for you. The **blend unit** runs after the fragment shader (and depth test) with factors you configure in the pipeline. Wrong factors produce dark halos, invisible glass, or premultiplied alpha bugs.

Spark uses a dedicated **lit transparent pipeline** sharing the same shaders as opaque but with blend enabled and depth write disabled. Transparent `SceneDrawItem`s are collected and recorded in `RecordTransparent` after opaque work.

## 2. Core Concepts

### Fixed-function blend equation

For each color attachment, Vulkan applies:

```text
color = (srcColor * srcColorFactor) OP (dstColor * dstColorFactor)
alpha = (srcAlpha * srcAlphaFactor) OP (dstAlpha * dstAlphaFactor)
```

Spark's transparent attachment:

```cpp
transparentBlendAttachment.blendEnable = VK_TRUE;
transparentBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
transparentBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
transparentBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
transparentBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
transparentBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
transparentBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

| Factor | Role in RGB equation |
|--------|----------------------|
| `SRC_ALPHA` | Multiply source color by fragment alpha |
| `ONE_MINUS_SRC_ALPHA` | Multiply destination by `(1 - α)` |

This is **straight alpha** blending — albedo.rgb is not premultiplied by α in the shader output.

### Color write mask

```cpp
colorWriteMask = R | G | B | A;
```

Blend only affects enabled channels. Disabling alpha write while blending RGB is valid when destination alpha is unused (Spark HDR color format may still store alpha for compositing).

### Blend state lives in the pipeline

Unlike dynamic depth bias on some paths, **blend enable and factors are fixed at pipeline creation** in Vulkan 1.0 core. Spark needs `pipelineLit` vs `pipelineLitTransparent` — cannot toggle `blendEnable` with a single dynamic call (Vulkan 1.3+ has limited extensions; Spark stays portable).

### Interaction with depth

Transparent pipeline in Spark:

| Stage | Setting |
|-------|---------|
| Depth test | ON — respect opaque occlusion |
| Depth write | OFF — do not block later transparent layers |
| Blend | ON — standard alpha over |

```text
Opaque pass:     depth write ON,  blend OFF
Transparent pass: depth write OFF, blend ON
```

Draw order for transparent: **back to front** approximates correct compositing when depth write is off. Spark packs transparent slices and sorts by depth heuristic in `RecordTransparent` — imperfect for intersecting alpha meshes, acceptable for foliage and glass panels.

### Alpha source in Spark

Per-draw opacity flows through push constants:

```cpp
push.albedo[3] = d.opacity > 0.0F ? d.opacity : 1.0F;
```

Fragment shader uses albedo alpha from texture × tint × `push.albedoTint.a` in the lit path. `outColor.a` carries opacity for blend.

### Premultiplied vs straight alpha

| Convention | Shader output | Blend factors |
|------------|---------------|---------------|
| Straight | `rgb = albedo`, `a = opacity` | `SRC_ALPHA`, `ONE_MINUS_SRC_ALPHA` |
| Premultiplied | `rgb = albedo * opacity`, `a = opacity` | `ONE`, `ONE_MINUS_SRC_ALPHA` |

Mixing conventions produces dark fringes on cutout textures. Spark textures are straight alpha in the standard material path; cutout leaves may use alpha test style `discard` in shader (hard edge) vs true transparency.

### HDR and blending

Scene pass targets **linear HDR** float buffer. Blending happens in linear space — correct for physically based transparency. Tonemapping runs later; do not blend in display gamma space.

### Separate blend attachment for opaque

Opaque pipeline:

```cpp
colorBlendAttachment.blendEnable = VK_FALSE;
```

Fragment output replaces destination color entirely (after depth pass). No need for `ONE, ZERO` factors when blend is disabled.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Second pipeline, same shaders

**Concept:** Blend state in pipeline

**Decision:** `pipelineLitTransparent` duplicates create info with different blend attachment

**Why:** Vulkan 1.0 requires separate pipelines; avoids maintaining two SPIR-V modules.

**In Spark:** Third `vkCreateGraphicsPipelines` call after sky variant.

### SRC_ALPHA over blend

**Concept:** Straight alpha

**Decision:** Standard Porter-Duff "over" for RGB

**Why:** Matches art authored in straight alpha PNG; artist-friendly.

**In Spark:** `transparentBlendAttachment` factors in `VulkanScenePipeline.cpp`.

### RecordTransparent split

**Concept:** Draw order

**Decision:** Opaque `Record` then transparent `RecordTransparent` with packed draw list

**Why:** Ensures depth buffer filled by opaque before transparent reads it; enables sort pass on transparent subset only.

**In Spark:** `VulkanSceneOpaquePass::RecordTransparent`.

### Depth write off for transparent

**Concept:** Interaction with depth

**Decision:** Keep depth test, disable write

**Why:** Standard forward transparency; multiple layers can contribute color.

**In Spark:** `depthStencilTransparent`.

### Alpha in albedo push vec4

**Concept:** Alpha source

**Decision:** Store opacity in `push.albedo[3]` / `albedoTint.a`

**Why:** Reuses vec4 already pushed for tint RGB; fragment shader has one tint source.

**In Spark:** `VulkanSceneOpaquePass.cpp` lines 97–100.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `src/spark/render/scene/VulkanScenePipeline.cpp` (blend attachments)
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp` (`RecordTransparent`)
- `shaders/scene.frag` (alpha in `outColor`)

### Step 1 — Opaque baseline

Draw fully opaque cube with `pipelineLit`. Confirm blend disabled in RenderDoc.

### Step 2 — Force wrong pipeline

Draw semi-transparent material with `pipelineLit` — object looks opaque or wrong. Switch to `pipelineLitTransparent` — see through.

### Step 3 — Single transparent quad

Place quad between camera and cube. Alpha 0.5. Verify cube visible through quad and quad reads depth from cube when behind it.

### Step 4 — Overlapping transparent quads

Two quads same depth, different colors — order matters. Swap draw order in `RecordTransparent` — observe compositing change.

### Step 5 — Premultiplied experiment

Temporarily use `VK_BLEND_FACTOR_ONE` for src color with premultiplied shader output — compare fringe on texture edges with straight path.

### Step 6 — HDR headroom

Bright emissive transparent (fire) — `outColor.rgb` can exceed 1.0; blend still linear; tonemap later clamps for display.

## 5. How This Connects to the Rest of Your Engine

Chapter **7** depth policy pairs with this chapter's blend policy. Part **VII** particles often use additive blend (`ONE, ONE`) — different factors, same pipeline mechanism. UI pass (Part XI) composites over tonemapped swapchain with its own blend tables.

Transparency is **order-dependent** — engine architecture must allocate a sort key or accept approximations.

## 6. Pitfalls and Debugging

- **Dark halos on cutouts** — straight alpha texture with premultiplied blend factors (or vice versa).
- **Invisible transparent** — alpha 0 in push or texture; or blend disabled on wrong pipeline.
- **Transparent disappears behind self** — depth write still on.
- **Z-sort popping** — sort heuristic wrong for intersecting triangles; only fix is OIT or mesh splitting.
- **Blend on opaque pass** — scene looks washed out; everything semi-transparent.

## 7. Exercises

1. Configure additive blend for particles preview — `src = ONE`, `dst = ONE`.
2. Draw transparent before opaque — document failure mode.
3. Visualize `outColor.a` as grayscale in shader debug mode.
4. Compare alpha blend in linear vs sRGB framebuffer (why HDR float matters).
5. Implement `discard` for alpha &lt; 0.5 cutout — does it need transparent pipeline?

## Summary

Spark enables standard straight-alpha blending on `pipelineLitTransparent` with `SRC_ALPHA` and `ONE_MINUS_SRC_ALPHA`, while opaque geometry uses blend disabled. Transparent draws disable depth write but keep depth test, and are recorded after opaque passes with approximate back-to-front sorting. Match alpha convention across art, shader output, and blend factors.

---

Next: [Debugging Shaders](09-shader-debug.md).
