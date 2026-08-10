# World Sprites

> **Build-Along Milestone:** RenderDoc shows a sprite draw in the HDR subpass, correctly occluded by a closer mesh.

**You will learn:**

- Submit textured quads as world-space sprite draws in SceneRenderParams.
- Explain how instance SSBOs batch many sprites over one shared quad.
- Place VulkanSpritePass inside the HDR scene subpass with correct depth and blend state.

## 1. The Problem We Are Solving

Hybrid games render **3D meshes** and **2D billboards** in the same camera view. A world sprite is a textured quad transformed in world space — not a screen-space HUD element. It must respect HDR lighting headroom (Part X), depth occlusion against terrain and props, and the same texture array as PBR materials (Part IV).

Spark implements world sprites as **instanced draws** over a baked unit quad, with per-sprite data in mapped SSBOs recorded by `VulkanSpritePass` inside the HDR subpass. Your build-along milestone: submit one textured sprite, capture a frame in RenderDoc, and confirm it draws into the float16 scene target with depth testing against a mesh occluder.
## 2. Core Concepts

### SceneSpriteDraw snapshot record

Each frame, `SceneSubmit` walks active entities with `SpriteComponent` and appends **plain structs** to `SceneRenderParams::sprites`. A `SceneSpriteDraw` carries everything the GPU needs without touching ECS:

| Field | Role at draw time |
|-------|-------------------|
| `model` | World transform for the unit quad |
| `tint` | RGBA multiplier (also used when `textureLayer < 0`) |
| `uvRect` | Atlas sub-rectangle `(minU, minV, maxU, maxV)` |
| `textureLayer` | Index into `sceneTextures` array (-1 = no sample) |
| `lightingMode` | Branch selector in `sprite.frag` (Chapter 2) |
| `blendMode` | Selects premultiplied vs additive pipeline variant |

Gameplay never calls Vulkan. Demos attach `SpriteComponent`, set UV rect and texture, and let submit flatten the world. The renderer copies `pendingScene` once per frame slot — same immutability contract as mesh draw items from Part VII.

When debugging missing sprites, print each draw's `textureLayer` and `uvRect` after submit. A layer of −1 with zero tint alpha discards early in `sprite.frag`. Compare Spark `Platformer2DDemo` coin entities for a minimal working setup.

### Instance SSBO packing

`VulkanSpritePass` owns **per-flight mapped SSBOs** (`instanceMapped[frameIndex]`). At record time, each `SceneSpriteDraw` is converted by `FillSpriteInstanceGpu` into a `VulkanSpriteInstanceGpu` struct whose layout matches `sprite_instance.glsl`. The vertex shader reads:

```glsl
SpriteInstanceGpu inst = instances[spriteBatch.instanceBase + uint(gl_InstanceIndex)];
```

**Instancing** means one indexed draw call expands to N sprites: shared topology (six indices), unique per-instance model matrix, tint, UV bounds, layer, and lighting params. CPU cost scales with sprite count as a memcpy loop, not as N separate pipeline creations. Spark resets `instanceWriteCursor` at the start of each frame slot so stale instances never leak across flights.

Study `sprite_instance.glsl` next to `VulkanSpriteInstanceGpu` — field order and std430 padding must match.

### Shared unit quad topology

Sprites do not upload unique vertex buffers per entity. During `VulkanRenderer::CreateSceneGeometry`, Spark appends four corners of a **unit quad** in the XY plane at Z=0 to the same interleaved VB that holds the unit cube and ground plane:

```text
local corners: (-0.5,-0.5,0) … (0.5,0.5,0)
UV: (0,0) … (1,1)
```

`spriteQuadFirstIndex` and `spriteQuadIndexCount` (six indices, two triangles) tell `VulkanSpritePass::Record` which index range to bind. The instance `model` matrix scales, rotates, and translates that template into world space — a tree billboard might scale to `(2, 4, 1)` and face the camera via transform. This is the same “one mesh, many draws” pattern as particles (Part VIII), but sprites retain full 4×4 transforms instead of camera-axis billboards.

Tilemap draws share this index range — corrupt index buffer upload breaks both systems simultaneously.

### HDR scene subpass placement

World sprites target the **RGBA16F HDR color attachment** from Part X, not the swapchain. In `RecordSceneCommandBuffer`, ordering inside the HDR subpass is:

```text
opaque meshes → transparent meshes → composite2D (tilemap + sprite) → particles
```

`composite2DPass.

Record` receives both `VulkanTilemapPass` and `VulkanSpritePass` contexts sharing the same static VB/IB and scene descriptor set. Sprites therefore **share depth** with 3D geometry written earlier in the subpass. A sprite behind a mesh fails the depth test and is occluded — essential for 2.5D games where props and billboards interleave. Drawing sprites after tonemap would ignore that depth buffer entirely.

composite2DPass may skip record when both tile and sprite arrays empty — guard demo code so at least one 2D draw exists when testing. HDR depth clear value must match opaque pass expectations; sprites with depth test compare against terrain written earlier in same subpass.

End-to-end frame graph for 2D-in-3D: shadow maps → begin HDR → opaque scene (terrain, props) → transparent meshes → composite2D tilemap+sprite → particles → end HDR → SSAO/bloom optional → tonemap → screenUi → ImGui. Sprites never appear in tonemap input as UI quads — different lifecycle. If sprites look too dark, verify they are in HDR pass receiving same exposure UBO as meshes, not display-referred UI path.


### Alpha blending and depth policy

Sprite pipeline state in `VulkanSpritePass::CreateGraphicsPipeline` sets:

| State | Value | Why |
|-------|-------|-----|
| Depth test | ON | Occlude against world geometry |
| Depth write | OFF | Transparent quads must not block sprites behind |
| Cull mode | NONE | Quads may be rotated arbitrarily |
| Blend | Per `SceneBlendMode` | Alpha vs additive variants |

Default alpha uses `SRC_ALPHA` / `ONE_MINUS_SRC_ALPHA`. Additive sprites (muzzle flashes) select a different pipeline slot from the same pass object. **Do not** reuse the opaque mesh pipeline — it lacks instance SSBO bindings and uses depth write ON, which breaks correct transparency sorting within the sprite batch.

SceneBlendMode selects among pipelines created in CreateGraphicsPipeline loop — default vs additive. Record path should batch consecutive sprites sharing blend mode and texture to minimize pipeline binds. Premultiplied alpha assets need matching blend func or fringes appear dark.

Compare VkPipelineDepthStencilStateCreateInfo between scene lit pipeline and sprite pipeline in RenderDoc pipeline state panes — depthWriteEnable differs. For ground decals coplanar with terrain, artists use render layer and slight polygon offset in model matrix; depth test still prevents drawing behind hills. Additive sprites for muzzle flash may use depthTest OFF in their blend pipeline variant — confirm in VulkanSpritePass CreateGraphicsPipeline loop per SceneBlendMode.


### Texture array and sRGB decode

Sprites sample the same `sampler2DArray sceneTextures` binding as PBR meshes (set 0, binding 1). Submit registers atlas PNGs into `SceneRenderParams::sceneTextures` and stores the layer index on each draw. In `sprite.frag`, `sampleBase()` calls `texture(sceneTextures, vec3(vTex, float(vLayer)))` then **`sparkSrgbToLinear`** before tint multiply — identical color workflow to Part IV albedo sampling.

The vertex shader remaps unit-quad UVs into atlas space with half-texel inset (`sprite.vert` lines 25–29) to reduce bleeding when neighboring atlas cells differ. If `textureLayer < 0`, the fragment shader skips the sample and outputs `vTint` alone — useful for debug overlays and flat-color placeholders while iterating on transform and sort order.

findOrAddTexture in submit deduplicates by pointer equality on SharedPtr<Texture2D>. Two loads of same path create distinct layers if pointers differ — cache textures in GameWorld registry like Platformer2DDemo. Mip generation at upload reduces shimmer on minified distant sprites in 2.5D scenes.

Part IV texture upload chooses sRGB vs UNORM format; sprite.frag always decode albedo with sparkSrgbToLinear before tint and lighting. Debugging washed sprites: verify PNG is sRGB authored, decode runs once, and HDR exposure is not applied twice. Array layer clamp in fragment shader prevents out-of-range when texture deregistered but draw remains one frame stale.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Flat array on SceneRenderParams

**Concept:** SceneSpriteDraw snapshot record

**Decision:** Flat array on SceneRenderParams

**Why:** Renderer reads one contiguous list per frame; no virtual calls or component lookups during command recording.

**In Spark:** `include/spark/engine/SceneRenderParams.hpp` — `SceneSpriteDraw` at lines 108–128.

### FillSpriteInstanceGpu + mapped per-flight buffer

**Concept:** Instance SSBO packing

**Decision:** FillSpriteInstanceGpu + mapped per-flight buffer

**Why:** Host-visible coherent memory avoids staging for dynamic sprite counts; cursor reset prevents cross-frame garbage.

**In Spark:** `src/spark/render/sprites2d/VulkanSpritePass.cpp` — `FillSpriteInstanceGpu` lines 18–40.

### Embed quad in CreateSceneGeometry

**Concept:** Shared unit quad topology

**Decision:** Embed quad in CreateSceneGeometry

**Why:** Zero extra buffer binds at runtime; tilemap pass reuses the same quad index range.

**In Spark:** `src/spark/render/core/VulkanRenderer.cpp` — sprite verts lines 692–725.

### composite2DPass after opaque scene

**Concept:** HDR scene subpass placement

**Decision:** composite2DPass after opaque scene

**Why:** Single HDR subpass keeps depth attachment valid for 2D/3D compositing before particles.

**In Spark:** `VulkanRenderer.cpp` lines 303–327 — tilemapCtx + spriteCtx.

### Dedicated sprite pipelines per blend mode

**Concept:** Alpha blending and depth policy

**Decision:** Dedicated sprite pipelines per blend mode

**Why:** Separates depth-write policy and blend attachments from mesh pipelines without duplicating shader code.

**In Spark:** `VulkanSpritePass::CreateGraphicsPipeline` — depth lines 226–232.

### Reuse scene descriptor set layout

**Concept:** Texture array and sRGB decode

**Decision:** Reuse scene descriptor set layout

**Why:** One texture array for meshes, sprites, and tilemaps — submit assigns layers consistently.

**In Spark:** `shaders/sprite.frag` — `sampleBase()` lines 21–28.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/engine/SceneRenderParams.hpp`
- `src/spark/render/sprites2d/VulkanSpritePass.cpp`
- `shaders/sprite.vert`
- `shaders/sprite.frag`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Read SceneSpriteDraw fields

Note sort keys used in Chapter 5 stable sort.

*SceneSpriteDraw*

```cpp
// include/spark/engine/SceneRenderParams.hpp (lines 108–128)
struct SceneSpriteDraw {
    Matrix4 model = Matrix4::Identity;
    Vector4 tint{1.0F, 1.0F, 1.0F, 1.0F};
    /** Atlas UV bounds (minU, minV, maxU, maxV) in normalized 0–1 space. */
    Vector4 uvRect{0.0F, 0.0F, 1.0F, 1.0F};
    std::int32_t textureLayer = -1;
    /** Matches sprite push constant layout (SpriteLighting2DMode). */
    SpriteLighting2DMode lightingMode = SpriteLighting2DMode::None;
    float lightingPad0 = 0.0F;
    float lightingPad1 = 0.0F;
    /** Shader-specific (see SpriteLighting2DMode). */
    Vector4 lightingParam0{1.0F, 1.0F, 1.0F, 1.0F};
    Vector4 lightingParam1{1.0F, 0.0F, 0.0F, 0.0F};
    /** Lower values draw first (behind); higher values draw on top within the sorting layer. */
    std::int32_t sortOrder = 0;
    /** Global sorting layer order from <c>RenderLayerRegistry</c> (lower draws behind). */
    std::int16_t sortingLayerOrder = 0;
    /** World Y anchor for <c>SceneSpriteSortMode::SortOrderThenWorldY</c> (sorting-group aware). */
    float sortWorldY = 0.0F;
    SceneBlendMode blendMode = kSceneBlendModeDefault;
};
```

### Step 2 — Trace instance packing

Follow SceneSpriteDraw into SSBO layout.

*FillSpriteInstanceGpu*

```cpp
// src/spark/render/sprites2d/VulkanSpritePass.cpp (lines 18–40)
void FillSpriteInstanceGpu(const SceneSpriteDraw& sprite, VulkanSpriteInstanceGpu& out) noexcept {
    std::memcpy(out.model, sprite.model.m, sizeof(out.model));
    out.tint[0] = sprite.tint.x;
    out.tint[1] = sprite.tint.y;
    out.tint[2] = sprite.tint.z;
    out.tint[3] = sprite.tint.w;
    out.uvRect[0] = sprite.uvRect.x;
    out.uvRect[1] = sprite.uvRect.y;
    out.uvRect[2] = sprite.uvRect.z;
    out.uvRect[3] = sprite.uvRect.w;
    out.textureLayer = sprite.textureLayer;
    out.lightingMode = static_cast<std::int32_t>(sprite.lightingMode);
    out.lightingPad0 = sprite.lightingPad0;
    out.lightingPad1 = sprite.lightingPad1;
    out.lightingA[0] = sprite.lightingParam0.x;
    out.lightingA[1] = sprite.lightingParam0.y;
    out.lightingA[2] = sprite.lightingParam0.z;
    out.lightingA[3] = sprite.lightingParam0.w;
    out.lightingB[0] = sprite.lightingParam1.x;
    out.lightingB[1] = sprite.lightingParam1.y;
    out.lightingB[2] = sprite.lightingParam1.z;
    out.lightingB[3] = sprite.lightingParam1.w;
}
```

### Step 3 — Study vertex shader instancing

Instance SSBO fetch and UV remap.

*sprite.vert main*

```glsl
// shaders/sprite.vert (lines 20–37)
void main() {
    SpriteInstanceGpu inst = instances[spriteBatch.instanceBase + uint(gl_InstanceIndex)];
    vec4 wp = inst.model * vec4(inPosition, 1.0);
    gl_Position = ubo.viewProj * wp;
    vec2 uv01 = inTexCoord;
    const float kLayerSize = 512.0;
    vec2 halfTexel = vec2(0.5) / vec2(kLayerSize);
    vec2 uvMin = inst.uvRect.xy + halfTexel;
    vec2 uvMax = inst.uvRect.zw - halfTexel;
    vTex = mix(uvMin, uvMax, uv01);
    vLayer = inst.textureLayer;
    vLocalXY = inPosition.xy;
    vWorldPos = wp.xyz;
    vTint = inst.tint;
    vLightingMode = inst.lightingMode;
    vLightingA = inst.lightingA;
    vLightingB = inst.lightingB;
}
```

### Step 4 — Locate HDR composite record slot

composite2DPass after opaque scene.

*composite2DPass*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 303–327)
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
```

### Step 5 — Inspect pipeline depth/blend state

Depth test ON, write OFF.

*sprite depth*

```cpp
// src/spark/render/sprites2d/VulkanSpritePass.cpp (lines 226–232)
    VkPipelineDepthStencilStateCreateInfo ds{};
    ds.sType = VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO;
    ds.depthTestEnable = VK_TRUE;
    ds.depthWriteEnable = VK_FALSE;
    ds.depthCompareOp = VK_COMPARE_OP_LESS_OR_EQUAL;
    ds.depthBoundsTestEnable = VK_FALSE;
    ds.stencilTestEnable = VK_FALSE;
```

### Step 6 — Read fragment base sample path

sRGB decode and unlit mode.

*sampleBase*

```glsl
// shaders/sprite.frag (lines 21–44)
vec4 sampleBase() {
    if (vLayer < 0) {
        return vTint;
    }
    vec4 tex = texture(sceneTextures, vec3(vTex, float(vLayer)));
    tex.rgb = sparkSrgbToLinear(tex.rgb);
    return tex * vTint;
}

float pulse01(float hz) {
    float t = ubo.timeGlobal.x * hz * 6.2831853;
    return 0.5 + 0.5 * sin(t);
}

void main() {
    vec4 base = sampleBase();
    if (base.a < 1e-4) {
        discard;
    }

    if (vLightingMode == 0) {
        outColor = base;
        return;
    }
```

### Step 7 — Submit a test sprite from gameplay

Add SpriteComponent; capture RenderDoc on HDR target.

### Step 8 — Milestone — depth occlusion against 3D

Sprite occludes against unit cube in ThreeDDemo.

## 5. How This Connects to the Rest of Your Engine

Sprites consume the **HDR render target** (Part X ch1) and **texture array** (Part IV). They record after opaque scene draws and before **particles** (Part VIII). **Shadows** (Part IX) typically skip sprite draws — lit sprites use simplified 2D modes in Chapter 2 instead of shadow maps. **Screen UI** (Chapter 6) draws after tonemap into an entirely different pass.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

## 6. Pitfalls and Debugging

- **Drawing sprites in the present pass** — ignores world depth; sprites float over everything.
- **Depth write ON for sprites** — later transparent sprites fail depth and disappear.
- **Forgetting sRGB decode** — atlas colors look washed out in HDR linear space.
- **One VkDraw per sprite without instancing** — CPU record cost explodes past a few hundred sprites.
- **Wrong UV rect orientation** — Spark uses bottom-left origin for uniform-grid atlases (Chapter 3); Tiled exports may need Y flip at import.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

## 7. Exercises

1. Log `params.sprites.GetSize()` each frame in a 2D demo — correlate with RenderDoc draw instance counts.
2. Draw a tint-only sprite (`textureLayer = -1`) and a textured sprite side by side.
3. Toggle depth test off temporarily and screenshot the incorrect overdraw.
4. List three differences between sprite pipeline state and lit mesh pipeline state.
5. Trace `FillSpriteInstanceGpu` field mapping for one `SceneSpriteDraw`.

## Summary

World sprites are instanced textured quads submitted via SceneSpriteDraw arrays, packed into per-flight SSBOs, and recorded by VulkanSpritePass inside the HDR scene subpass with depth test ON and depth write OFF.

---

Next: [2D Lighting](02-sprite-lighting.md).
