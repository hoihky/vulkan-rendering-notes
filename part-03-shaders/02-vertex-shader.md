# Vertex Shader Fundamentals

> **Build-Along Milestone:** You can read `scene.vert` and explain every input, uniform source, and varying that reaches the fragment stage.

**You will learn:**

- Declare vertex inputs that match `VkVertexInputAttributeDescription` in C++.
- Transform positions through model space → world space → clip space.
- Transport normals and tangents for lighting and normal mapping.
- Implement optional GPU skinning from an SSBO.

## 1. The Problem We Are Solving

Part II created a graphics pipeline and drew geometry. The **vertex shader** is the first programmable stage — it runs once per vertex and decides where that vertex lands on screen, plus what data the **fragment shader** receives via interpolation.

If vertex locations in GLSL disagree with C++ vertex layout, validation may pass but meshes explode or collapse. If matrix multiplication order is wrong, objects spin or slide. If normals are transformed with the model matrix instead of its inverse-transpose, lighting looks flat or inverted.

Spark's `scene.vert` is a production-grade example: six vertex attributes, per-frame UBO, per-draw push constants, optional skinning, and nine varyings. This chapter walks through it line by line so your build-along engine can match Spark's contract.

### What the vertex shader actually does (one paragraph)

For **each vertex** of **each triangle** being drawn, the GPU runs `scene.vert` once. That program:

1. Reads vertex attributes from the bound vertex buffer (position, normal, UV, …).
2. Applies optional skinning (animated characters).
3. Multiplies position by **model** matrix (object → world) and **viewProj** (world → clip).
4. Writes `gl_Position` — the clip-space position the rasterizer needs.
5. Passes normals, UVs, and other data to the fragment shader via **varying** outputs.

The fragment shader does **not** know about individual vertices — only interpolated values across the triangle interior.

### Matrix multiply order (the #1 beginner bug)

Spark uses column-major matrices matching GLSL:

```glsl
vec4 worldPos = push.model * vec4(inPosition, 1.0);
gl_Position = ubo.viewProj * worldPos;
```

Read right-to-left as math: first apply model, then viewProj. **Never** transpose one side "because DirectX did it differently" without verifying in RenderDoc.

Wrong order symptoms:

| Mistake | What you see |
|---------|--------------|
| `viewProj * model` swapped with model on wrong side | Objects orbit wrong axis |
| Row-major C++ matrix without transpose | Sheared / collapsed mesh |
| Forgot perspective w | Everything clips or vanishes |

### Varyings: what they are

**Varyings** (outputs from vertex shader, inputs to fragment shader) are **interpolated** across the triangle:

```glsl
// vertex shader
out vec2 vTexCoord;
vTexCoord = inTexCoord;

// fragment shader
in vec2 vTexCoord;  // smooth blend between three corner UVs
```

If `vTexCoord` is wrong at vertices, textures swim across the surface. Fix vertex layout and locations before debugging `texture()` in the fragment shader.

## 1b. Concept Deep Dive — scene.vert Inputs, Skinning, and Clip Output

### Vertex attribute contract (locations 0–7)

Spark's production vertex shader binds a single interleaved stream matching `VulkanSceneVertexLayout`:

```glsl
// shaders/scene.vert (lines 4–11)
layout(location = 0) in vec3 inPosition;
layout(location = 1) in vec3 inNormal;
layout(location = 2) in vec2 inTexCoord;
layout(location = 3) in vec4 inTangent;
layout(location = 4) in vec4 inJointsPacked;
layout(location = 5) in vec4 inWeights;
layout(location = 6) in vec2 inTexCoord1;
layout(location = 7) in vec4 inColor;
```

Rigid draws ignore locations 4–5; skinned draws decode joint indices with `floatBitsToUint` because MoltenVK historically mishandled integer vertex fetches. Every location must match `VkVertexInputAttributeDescription` stride **80 bytes** in `VulkanScenePipeline.cpp`.

### Frame UBO vs per-draw push in the transform chain

Shared layouts come from includes; the vertex stage reads both:

```glsl
#include "scene_ubo.glsl"   // set 0 binding 0 — viewProj, cameraPos, invViewProj
#include "scene_push.glsl"  // push_constant — model, material, skyMode, useSkinning
```

The non-skinned path is deliberately minimal:

```glsl
// shaders/scene.vert (lines 79–87, rigid branch)
vec4 wp = push.model * vec4(inPosition, 1.0);
worldPos = wp.xyz;
mat3 R = mat3(push.model);
mat3 nMat = (abs(detR) > 1e-8) ? transpose(inverse(R)) : mat3(1.0);
vNormal = normalize(nMat * inNormal);
gl_Position = ubo.viewProj * vec4(worldPos, 1.0);
```

`ubo.viewProj` is written once per frame by `VulkanSceneUniformWriter`; `push.model` changes every draw via `vkCmdPushConstants`.

### Skin SSBO and weighted palette lookup

When `push.useSkinning != 0`, the shader blends up to four bone matrices from binding 2:

```glsl
// shaders/scene.vert (lines 17–19, 44–58)
layout(std430, set = 0, binding = 2) readonly buffer SkinSSBO {
    mat4 bones[];
} skin;

if (push.useSkinning != 0) {
    for (int k = 0; k < 4; k++) {
        float w = inWeights[k];
        uint ji = inJoints[k];
        if (int(ji) >= push.jointCount) continue;
        mat4 BM = skin.bones[ji];
        sp += w * BM * vec4(inPosition, 1.0);
    }
    vec4 wp = push.model * vec4(sp.xyz, 1.0);
}
```

The CPU memcpy's the palette into mapped SSBO memory immediately before the skinned draw in `VulkanSceneOpaquePass.cpp`.

### Eleven varyings and the sky clip hack

Vertex outputs feed `scene.frag` — world position, perturbed normal/tangent, material scalars, and `flat int vTextureLayer`:

```glsl
// shaders/scene.vert (lines 95–111)
vWorldPos = worldPos;
vAlbedo = push.albedoTint.rgb;
vTextureLayer = push.textureLayer;
vec4 clip = ubo.viewProj * vec4(worldPos, 1.0);
if (push.skyMode != 0) {
    float zw = max(clip.w, 1e-5);
    gl_Position = vec4(clip.x, clip.y, zw * (1.0 - 1e-4), zw);
} else {
    gl_Position = clip;
}
```

Sky geometry nudges clip Z toward the far plane so dome pixels survive depth testing when other passes enable it. The fragment shader then reconstructs view rays from `ubo.invViewProj` — interpolated `vWorldPos` alone would swim on camera rotation (Chapter 3).

## 2. Core Concepts

### Coordinate spaces and the transform chain

A lit mesh vertex passes through these spaces:

```text
Local (mesh)  →  World  →  View  →  Clip  →  NDC  →  Screen
     model         (combined in viewProj)      ÷w    viewport
```

| Space | Meaning | Spark source |
|-------|---------|--------------|
| Local | Position in mesh file | `inPosition` |
| World | Scene placement | `push.model * vec4(inPosition, 1)` |
| Clip | Homogeneous clip coords | `ubo.viewProj * vec4(worldPos, 1)` |
| NDC | After perspective divide | GPU hardware |
| Screen | Pixel coordinates | Viewport + scissor |

Spark pre-multiplies view × projection on the CPU into `ubo.viewProj`. The vertex shader does **not** multiply view and projection separately — fewer uniforms, one matrix multiply per vertex.

### Vertex inputs and `layout(location = N)`

Vulkan binds vertex buffers to **locations** declared in GLSL:

```glsl
layout(location = 0) in vec3 inPosition;
layout(location = 1) in vec3 inNormal;
layout(location = 2) in vec2 inTexCoord;
layout(location = 3) in vec4 inTangent;
layout(location = 4) in vec4 inJointsPacked;
layout(location = 5) in vec4 inWeights;
```

Each `location` must match `VkVertexInputAttributeDescription::location` in `VulkanScenePipeline.cpp`. Spark uses one binding (binding 0) with stride **80 bytes** (20 floats):

| Location | Semantic | C++ offset (floats) | Format |
|----------|----------|---------------------|--------|
| 0 | Position | 0 | `R32G32B32_SFLOAT` |
| 1 | Normal | 3 | `R32G32B32_SFLOAT` |
| 2 | UV | 6 | `R32G32_SFLOAT` |
| 3 | Tangent | 8 | `R32G32B32A32_SFLOAT` (w = handedness) |
| 4 | Joint indices | 12 | `R32G32B32A32_SFLOAT` (packed bits) |
| 5 | Weights | 16 | `R32G32B32A32_SFLOAT` |

`VulkanSceneVertexLayout` in C++ defines the same offsets — the shader and pipeline are two views of one struct.

### Outputs (varyings) and interpolation

Vertex outputs become fragment inputs:

```glsl
layout(location = 0) out vec3 vWorldPos;
layout(location = 1) out vec3 vNormal;
// ... through location 8: vTangent
```

| Output | `flat`? | Fragment use |
|--------|---------|--------------|
| `vWorldPos` | no | Lighting, shadows, IBL |
| `vNormal` | no | Diffuse, specular |
| `vAlbedo` | no | Base color tint |
| `vTexCoord` | no | Texture lookup |
| `vTextureLayer` | **yes** | Array layer index (no interpolation) |
| `vMetallic`, `vRoughness` | no | PBR parameters |
| `vEmissive` | no | Emissive color |
| `vTangent` | no | Normal mapping TBN |

`flat` prevents interpolating integers across triangles — texture layer must stay constant per draw.

### Per-frame UBO vs per-draw push constants

| Data | Update frequency | Binding | Examples |
|------|------------------|---------|----------|
| Scene UBO | Once per frame | set 0, binding 0 | `viewProj`, camera, lights |
| Push block | Every draw | push constants | `model`, material, `skyMode` |
| Skin SSBO | Per skinned draw | set 0, binding 2 | Bone palette |

The vertex shader reads `ubo.viewProj` for clip transform and `push.model` for object placement. Splitting frame vs draw data minimizes descriptor updates in the draw loop.

### Normal and tangent transport

Normals are directions, not positions — they transform with the **inverse-transpose** of the model's upper 3×3:

```glsl
mat3 R = mat3(push.model);
mat3 nMat = transpose(inverse(R));
vec3 nW = nMat * inNormal;
vNormal = normalize(nW);
```

Uniform scale commutes with normal transform; non-uniform scale (stretched meshes) requires inverse-transpose or lighting breaks. Spark guards near-singular matrices with determinant checks and falls back to `(0, 1, 0)`.

Tangents transform with the linear part of the model matrix (or blended skin matrix for skinned meshes) to build a TBN basis in `scene.frag`.

### GPU skinning

When `push.useSkinning != 0`, up to four bone influences blend:

```glsl
for (int k = 0; k < 4; k++) {
    float w = inWeights[k];
    uint ji = inJoints[k];
    mat4 BM = skin.bones[ji];
    sp += w * BM * vec4(inPosition, 1.0);
    acc += w * mat3(BM);
}
```

Joint indices are stored as **float bit patterns** (`floatBitsToUint`) because some MoltenVK paths had issues with integer vertex attributes — a portability decision documented in the shader header.

The CPU uploads the joint palette to an SSBO each draw (`VulkanSceneOpaquePass.cpp` memcpy into mapped SSBO memory) before `vkCmdPushConstants` and `vkCmdDrawIndexed`.

### Sky mode clip-space hack

Sky geometry must render behind everything. Spark's sky pipeline disables depth test, but the vertex shader also nudges clip Z:

```glsl
if (push.skyMode != 0) {
    float zw = max(clip.w, 1e-5);
    gl_Position = vec4(clip.x, clip.y, zw * (1.0 - 1e-4), zw);
}
```

This pushes NDC depth slightly below 1.0 (far plane) so sky pixels win when depth is enabled on other passes. The fragment shader then reconstructs view rays from `invViewProj` instead of trusting interpolated `vWorldPos` for equirect skies — avoiding swimming artifacts.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Interleaved 20-float vertex

**Concept:** Vertex inputs and locations

**Decision:** Single binding, 80-byte stride, all attributes interleaved

**Why:** Matches glTF-style packing; one `vkCmdBindVertexBuffers` per mesh batch; cache-friendly for position-first access.

**In Spark:** `VulkanSceneVertexLayout.hpp`, `VulkanScenePipeline.cpp` lines 71–102.

### Combined viewProj in UBO

**Concept:** Coordinate spaces

**Decision:** CPU multiplies view × projection once; shader uses single `mat4`

**Why:** Saves one `mat4` multiply and one uniform update per frame; standard for forward renderers.

**In Spark:** `VulkanSceneUniformWriter::Write` → `u.viewProj`; `scene_ubo.glsl`.

### Push constants for model matrix

**Concept:** Per-frame UBO vs per-draw push

**Decision:** `model` and material fields in 192-byte push block, not dynamic UBO offsets

**Why:** Push constants are ideal for small, per-draw data — no descriptor churn, visible in RenderDoc.

**In Spark:** `ModelPushConstants` in `VulkanSceneOpaquePass.hpp`; duplicated `Push` block in vert + frag.

### Packed joint indices as float

**Concept:** GPU skinning

**Decision:** `inJointsPacked` as `vec4` of floats, decoded with `floatBitsToUint`

**Why:** MoltenVK compatibility on Apple GPUs; avoids `R32G32B32A32_UINT` fetch quirks.

**In Spark:** `scene.vert` lines 8–10, 58–62.

### World-space varyings

**Concept:** Outputs and interpolation

**Decision:** Pass `vWorldPos`, `vNormal` in world space, not view space

**Why:** Lighting, shadows, and IBL in `scene.frag` share one space; avoids per-light view transforms in the fragment shader.

**In Spark:** `vWorldPos = worldPos` after model/skin transform.

## 4. Implementation Walkthrough

Follow these steps in order. Open `shaders/scene.vert` in Spark alongside your engine.

### Spark Files We Touch This Chapter

- `shaders/scene.vert`
- `shaders/scene_ubo.glsl`
- `include/spark/render/scene/VulkanSceneVertexLayout.hpp`
- `src/spark/render/scene/VulkanScenePipeline.cpp` (vertex input state)
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp` (push + skin SSBO)

### Step 1 — Verify vertex layout parity

Compare your `VkVertexInputAttributeDescription` array against `VulkanSceneVertexLayout` constants. A single wrong offset shifts every attribute after it — normals point sideways, UVs scroll.

**Build-along:** Log the first vertex's position from CPU staging buffer and from a debug vertex shader that outputs `inPosition` as color (temporary).

### Step 2 — Minimal transform path

Start without skinning. Implement the non-skinned branch:

```glsl
vec4 wp = push.model * vec4(inPosition, 1.0);
worldPos = wp.xyz;
gl_Position = ubo.viewProj * vec4(worldPos, 1.0);
```

Confirm the mesh appears at the correct place when you move the camera (UBO updates) vs move the object (push `model` updates).

### Step 3 — Add varyings incrementally

Add outputs one at a time and verify in the fragment shader:

1. `vTexCoord` → sample albedo, confirm UV orientation
2. `vNormal` → `outColor = vec4(vNormal * 0.5 + 0.5, 1.0)` debug view
3. `vWorldPos` → distance-based grayscale

### Step 4 — Normal transform

Copy Spark's inverse-transpose path with determinant guard. Compare lit result against flat normal transform — stretched cubes show the difference immediately.

### Step 5 — Skinning (optional milestone)

Enable skinning only after static meshes work:

1. Create SSBO binding 2 in descriptor layout
2. Map SSBO, memcpy joint palette before draw
3. Set `push.useSkinning = 1` and `push.jointCount`
4. Match joint index unpacking in the shader

### Step 6 — Sky branch

Bind `pipelineSky` for sky draws (`VulkanSceneOpaquePass` switches pipelines when `d.skyMode != None`). Verify depth is disabled on sky pipeline and clip Z hack is active in the vertex shader.

## 5. How This Connects to the Rest of Your Engine

Chapter **3** consumes every varying in `scene.frag`. Chapter **4** documents the `SceneUBO` layout backing `ubo.viewProj`. Chapter **5** expands push constant fields beyond `model`. Part **VI** feeds the skin SSBO from animation.

The vertex shader is the **data contract** between mesh assets and shading — get locations and transforms right before investing in PBR math.

## 6. Pitfalls and Debugging

- **Winding / culling makes mesh invisible** — Spark uses `VK_FRONT_FACE_CLOCKWISE`; glTF is often CCW. Flip `frontFace` or flip index order, not both.
- **Transposed matrix** — C++ column-major `Matrix4` must match GLSL `mat4` memory layout; use `static_assert(sizeof(Matrix4) == 64)`.
- **Wrong location index** — Validation error `VUID-VkGraphicsPipelineCreateInfo-pVertexInput-08930` means attribute location mismatch.
- **Skinning explodes mesh** — joint indices out of range or SSBO not updated for current frame slot.
- **Sky swims on camera rotate** — using interpolated `vWorldPos` for equirect; fragment shader must reconstruct ray (Chapter 3).

## 7. Exercises

1. Draw a mesh with `outColor = vec4(inTexCoord, 0.0, 1.0)` in the fragment shader — confirm V-flip matches your texture loader.
2. Output `gl_Position.w` as grayscale — visualize perspective foreshortening.
3. Disable normal renormalization — when does lighting break on large triangles?
4. Compare `push.model` alone vs full chain for a child transform hierarchy (Part VI preview).
5. Capture one draw in RenderDoc — verify six vertex attributes and push constant bytes at offset 0.

## Summary

`scene.vert` maps six interleaved vertex attributes through optional skinning and model transform into world space, then multiplies by `ubo.viewProj` for clip position. It exports world-space normals, tangents, and material interpolants for `scene.frag`. Match `VulkanSceneVertexLayout` exactly, split frame UBO from draw push constants, and treat sky geometry as a special case for depth and ray reconstruction.

---

Next: [Fragment Shader Fundamentals](03-fragment-shader.md).
