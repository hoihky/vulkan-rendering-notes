# Blend and Depth State

> **Build-Along Milestone:** Particles glow additively, respect depth occlusion, and do not write depth.

**You will learn:**

- Configure additive blending for particle glow in Vulkan.
- Explain depth test ON with depth write OFF for transparent billboards.
- Contrast particle pipeline state with opaque scene lit pipeline.

## 1. The Problem We Are Solving

Wrong blend mode makes fire look like muddy alpha-cutout cards. Wrong depth settings make particles draw through walls or block the world. Particle VFX needs **additive HDR blend** with **depth test but no depth write**.

Build along: read `CreateGraphicsPipeline` blend and depth structs in `VulkanParticlePass.cpp` and verify behavior in a scene with occluders (ParticleDemo reference cube).

## 2. Core Concepts

### Additive color blend

| Factor | Value | Effect |
|--------|-------|--------|
| `srcColorBlendFactor` | `VK_BLEND_FACTOR_SRC_ALPHA` | Scale particle color by fragment alpha |
| `dstColorBlendFactor` | `VK_BLEND_FACTOR_ONE` | Add into existing HDR buffer |
| `colorBlendOp` | `VK_BLEND_OP_ADD` | Accumulate glow |

Result: `out = src.rgb * src.a + dst.rgb` — premultiplied-friendly glow that brightens where particles overlap without darkening the background.

### Depth test on, write off

| Setting | Particle pass | Opaque lit pass |
|---------|---------------|-----------------|
| `depthTestEnable` | TRUE | TRUE |
| `depthWriteEnable` | **FALSE** | TRUE |
| `depthCompareOp` | LESS | LESS |

Particles respect scene depth (hidden behind walls) but do not stomp depth for later passes — standard for transparent/additive effects.

### No face culling

`VK_CULL_MODE_NONE` — both sides of the billboard quad are visible if the camera enters the plane. Billboards are view-dependent; culling CW/CCW causes pop when crossing the quad plane.

### HDR attachment

Pipeline targets the same HDR render pass as scene lit draws — additive energy accumulates in linear space before tonemap compresses highlights.

### Alpha channel write mask

Spark writes RGB only (`colorWriteMask` without alpha bit) — swapchain alpha is irrelevant for HDR offscreen target.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SRC_ALPHA, ONE

**Concept:** Additive color blend

**Decision:** Not standard alpha-over (`SRC_ALPHA`, `ONE_MINUS_SRC_ALPHA`)

**Why:** Fire and magic read as emissive glow; alpha-over darkens and looks like smoke cards.

**In Spark:** `CreateGraphicsPipeline` `blendAtt`

### LESS compare, depthWriteEnable FALSE

**Concept:** Depth test on, write off

**Decision:** Read depth, never write

**Why:** Particles hide behind geometry but do not block each other's depth order perfectly — acceptable for additive VFX.

**In Spark:** `VkPipelineDepthStencilStateCreateInfo` `ds`

### CULL_MODE_NONE

**Concept:** No face culling

**Decision:** Disable cull for particle pipeline only

**Why:** Billboard quads are camera-aligned; wrong cull mode makes half the particles vanish.

**In Spark:** `rast.cullMode` in `CreateGraphicsPipeline`

### hdrRenderPass at pipeline create

**Concept:** HDR attachment

**Decision:** `particlePass.CreateGraphicsPipeline(device, hdrTonemapPass.HdrRenderPass())`

**Why:** Tonemap pass later compresses accumulated glow; particles must write linear HDR color.

**In Spark:** `VulkanRenderer.cpp` init

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `src/spark/render/sprites2d/VulkanParticlePass.cpp`

### Step 1 — Read depth stencil state

```cpp
// src/spark/render/sprites2d/VulkanParticlePass.cpp
    VkPipelineDepthStencilStateCreateInfo ds{};
    ds.sType = VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO;
    ds.depthTestEnable = VK_TRUE;
    ds.depthWriteEnable = VK_FALSE;
    ds.depthCompareOp = VK_COMPARE_OP_LESS;
```

### Step 2 — Read blend attachment state

```cpp
    VkPipelineColorBlendAttachmentState blendAtt{};
    blendAtt.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT;
    blendAtt.blendEnable = VK_TRUE;
    blendAtt.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
    blendAtt.dstColorBlendFactor = VK_BLEND_FACTOR_ONE;
    blendAtt.colorBlendOp = VK_BLEND_OP_ADD;
    blendAtt.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
    blendAtt.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
    blendAtt.alphaBlendOp = VK_BLEND_OP_ADD;
```

### Step 3 — Read rasterization cull mode

```cpp
    VkPipelineRasterizationStateCreateInfo rast{};
    rast.polygonMode = VK_POLYGON_MODE_FILL;
    rast.cullMode = VK_CULL_MODE_NONE;
    rast.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE;
```

### Step 4 — Compare to scene opaque pipeline

| State | `pipelineLit` | `pipeline` (particle) |
|-------|---------------|-------------------------|
| Depth write | ON | OFF |
| Blend | OFF (opaque) | Additive |
| Cull | Back-face | None |

### Step 5 — Visual test — occlusion

Place emitter behind `UnitCube` in ParticleDemo — particles hidden when cube blocks line of sight.

### Step 6 — Visual test — glow overlap

Stack two emitters at same position — HDR values add brighter where overlap (before tonemap clip).

## 5. How This Connects to the Rest of Your Engine

**Particle Shaders** output alpha-friendly RGB for blend. **Frame Graph** draws particles after opaque meshes so depth buffer reflects world. **HDR tonemap** (Part X) compresses accumulated particle energy. **Part VII** opaque pass writes the depth particles test against.

## 6. Pitfalls and Debugging

- **Depth write ON** — particles pollute depth; later transparent meshes sort wrong.
- **ONE, ONE blend** — double-bright blowout everywhere; `SRC_ALPHA` scales by particle alpha.
- **No depth test** — fire draws through walls.
- **Tonemap before particles** — glow missing from final image; particles must be pre-tonemap in HDR subpass.

## 7. Exercises

1. Toggle `depthTestEnable` off — screenshot particles through floor.
2. Switch to standard alpha blend (`SRC_ALPHA`, `ONE_MINUS_SRC_ALPHA`) — compare fire appearance vs additive.
3. RenderDoc: inspect color blend attachment state on particle pipeline.
4. Enable depth write on particle pipeline — describe sorting artifacts on ground plane.
5. List three differences between particle and sprite blend state in Spark.

## Summary

Particle pipeline uses additive `SRC_ALPHA`/`ONE` blending, depth test `LESS` with write disabled, and no culling — tuned for HDR glow occluded by scene geometry.

---

Next: [Frame Graph Ordering](07-frame-graph.md).
