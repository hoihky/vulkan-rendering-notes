# UV Mapping

> **Build-Along Milestone:** Mesh UVs flow from `Mesh::Vertex::texCoord` through interleaved GPU buffers to `vTexCoord` and into `texture(sceneTextures, vec3(uv, layer))`.

**You will learn:**

- Store and upload 2D texture coordinates with vertex data.
- Match `VkVertexInputAttributeDescription` to GLSL `layout(location = 2) in vec2`.
- Understand UV conventions, tiling, and derivative-based tangent reconstruction.
- Use UV debug views to validate the texture pipeline before lighting.

## 1. The Problem We Are Solving

A texture is a 2D image; a mesh surface is 3D. **UV mapping** assigns each vertex a 2D coordinate `(u, v)` that indexes texels. Wrong UVs produce swimming, seams, or mirrored materials — often mistaken for shader or descriptor bugs.

Spark carries UVs from glTF and procedural meshes through a fixed interleaved vertex layout into the fragment shader. This chapter connects asset authoring, vertex buffer packing, and shader sampling — the geometric half of texturing (color space is Chapter 5).

### UV mapping explained without jargon

Imagine wrapping a gift box with patterned paper:

- The **3D box** is your mesh surface.
- The **2D wrapping paper** is your texture image.
- **UV mapping** is the instruction that says "corner A of the box uses this corner of the paper."

Each vertex stores a 2D coordinate `(u, v)`:

- `u = 0` is the left edge of the texture; `u = 1` is the right edge.
- `v = 0` is typically the bottom; `v = 1` is the top (conventions vary by DCC tool).

The GPU **interpolates** UVs across each triangle. A pixel halfway between two vertices gets a UV halfway between those vertices' UVs. The fragment shader uses that UV to read a texel color.

```text
Vertex A: position (1,0,0), uv (0,0)  ─┐
Vertex B: position (2,0,0), uv (1,0)   ├─ triangle → rasterizer → fragment uv ≈ (0.5, 0.2)
Vertex C: position (1,1,0), uv (0,1)  ─┘
                                              ↓
                              texture(brick, uv) → orange pixel
```

Without UVs, `texture()` has no coordinates — every pixel would sample the same corner of the image (or undefined behavior if UVs are zero).

### Where UVs are created vs where they are consumed

| Stage | Who sets UV | Format |
|-------|-------------|--------|
| Blender / Maya | Artist unwraps mesh | Exported in glTF `TEXCOORD_0` |
| glTF import | `AppendPrimitive` reads accessor | `Mesh::Vertex::texCoord` |
| GPU upload | `VulkanCustomMeshPool` memcpy | Interleaved `VkBuffer` |
| Vertex shader | Pass through | `out vec2 vTexCoord` |
| Fragment shader | Sample | `texture(sceneTextures, vec3(vTexCoord, layer))` |

If the brick pattern looks stretched, fix UVs in Blender — not in the fragment shader. If the pattern is missing entirely, debug the import → vertex layout → shader location chain.

## 1b. Concept Deep Dive — Vertex UV Path to Shader

### Authoritative CPU field

Every rigid mesh stores UVs on the vertex record:

```cpp
// include/spark/scene/mesh/Mesh.hpp
struct Vertex {
    Vector3 position;
    Vector3 normal;
    Vector2 texCoord;   // primary UV set (glTF TEXCOORD_0)
    Vector2 texCoord1;  // secondary set (lightmaps, detail)
    Vector4 color;
    Vector4 tangent;    // xyz + handedness for normal mapping
};
```

glTF import writes `texCoord` from accessor index 0. Procedural meshes (`CreateQuad`, brick patterns) set UVs explicitly — `(0,0)`–`(1,1)` for a full texture wrap.

### Vulkan vertex attribute location 2

The scene pipeline declares `texCoord` at location 2, matching `scene.vert`:

```glsl
layout(location = 2) in vec2 inTexCoord;
layout(location = 0) out vec2 vTexCoord;
void main() {
    vTexCoord = inTexCoord;
    // ...
}
```

If location numbers drift between C++ `VkVertexInputAttributeDescription` and GLSL, UVs appear as garbage while positions still draw correctly — a common "solid color mesh" bug.

### Scene-layer UV scale after upload

Large textures are fit into square GPU layers with letterboxing. `Texture2D::GetSceneLayerUvScale()` returns the occupied fraction; submit copies it to `SceneDrawItem::textureUvScale`. Shader code multiplies interpolated UVs so art fills the visible sub-rectangle of the layer, not the padded border.

### Tangent fallback when w = 0

When `tangent.xyz` is zero (no glTF tangents), `scene.frag` reconstructs TBN from screen-space derivatives of `vTexCoord` and world position — UV quality directly affects normal-map stability at grazing angles.

## 2. Core Concepts

### UV coordinates defined

| Symbol | Range (typical) | Meaning |
|--------|-----------------|--------|
| `u` | [0, 1] along texture width | horizontal axis |
| `v` | [0, 1] along texture height | vertical axis |

Values outside [0, 1] **repeat** when sampler `addressMode` is `REPEAT` — brick walls tile without duplicating vertices.

```text
Vertex (position, normal, texCoord)
    → rasterizer interpolates texCoord across triangle
        → fragment receives vTexCoord
            → texture(sampler, vec3(vTexCoord, layer))
```

### CPU mesh representation

Rigid meshes store UVs on `Mesh::Vertex`:

```cpp
struct Vertex {
    Vector3 position;
    Vector3 normal;
    Vector2 texCoord;  // UV
};
```

Skinned glTF meshes extend with tangent, joints, weights — UV unchanged. Exporters (Blender, glTF) define UV islands; Spark imports as-is.

### GPU interleaved layout

`VulkanSceneVertexLayout` places UV at float offset 6–7 in a 20-float stride:

| Location | Attribute | Offset (floats) | Format |
|----------|-----------|-----------------|--------|
| 0 | position | 0 | `R32G32B32_SFLOAT` |
| 1 | normal | 3 | `R32G32B32_SFLOAT` |
| **2** | **texCoord** | **6** | **`R32G32_SFLOAT`** |
| 3 | tangent | 8 | `R32G32B32A32_SFLOAT` |
| 4–5 | joints, weights | 12–16 | `R32G32B32A32_SFLOAT` |

`VulkanScenePipeline.cpp` and `scene.vert` must agree on location 2 — Part III covered this contract; here the focus is **what UV values mean** for texturing.

### Vertex shader pass-through

```glsl
layout(location = 2) in vec2 inTexCoord;
layout(location = 3) out vec2 vTexCoord;

void main() {
    vTexCoord = inTexCoord;
    // ...
}
```

No transform in the vertex shader for basic UV mapping — scaling/offset lives in material systems (Spark uses texture directly; UV transform can be added via push constants in extended engines).

### Fragment shader sampling

```glsl
vec2 uv = vTexCoord;
int layer = clamp(vTextureLayer, 0, 15);
vec4 tex = texture(sceneTextures, vec3(uv, float(layer)));
```

`texture()` applies sampler filtering (linear) and address wrapping. **Perspective-correct interpolation** keeps UVs consistent across depth — GPU hardware handles this when `vTexCoord` is not `flat`.

### UV convention and vertical flip

Texture coordinate **v = 0** maps to one horizontal edge of the image; **v = 1** to the other. Image files and APIs disagree on which edge is "top":

| Stage | Row 0 meaning |
|-------|---------------|
| glTF / Vulkan image upload (Spark memory load) | Top of image |
| stb default file load with flip on | Top after flip |
| OpenGL legacy | Often bottom = 0 |

Spark documents flip per load path in Chapter 1. If albedo appears upside-down, fix at **load or UV export**, not by negating `v` in shader without understanding the full pipeline.

### Tangent space and normal maps

Normal mapping needs a **TBN matrix** (tangent, bitangent, normal). Spark stores tangent `vec4` with `w` = handedness (+1 or -1).

When tangent is missing, `scene.frag` reconstructs TBN from screen-space derivatives:

```glsl
vec3 dp1 = dFdx(vWorldPos);
vec3 dp2 = dFdy(vWorldPos);
vec2 duv1 = dFdx(vTexCoord);
vec2 duv2 = dFdy(vTexCoord);
float det = duv1.x * duv2.y - duv1.y * duv2.x;
// build T, B from gradient chain rule
```

This requires **continuous UVs** across the surface — seams break derivative reconstruction. Author UV islands with care for normal-mapped assets.

### Sky UV special case

Procedural sky modes may use `vTexCoord.y` as gradient parameter for ground-to-horizon color — not a texture lookup. Equirect sky uses **view ray** reconstruction (Part III), not mesh UVs. Know which path your draw uses via `push.skyMode`.

### UV debug visualization

Before trusting albedo:

```glsl
outColor = vec4(fract(vTexCoord), 0.0, 1.0);           // tiling check
outColor = vec4(vTexCoord.x, vTexCoord.y, 0.0, 1.0);   // raw UV
```

Checkerboard texture on layer 0 makes seams and scale obvious.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### UV in same binding as position

**Concept:** Interleaved layout

**Decision:** Single vertex buffer binding 0, stride 80 bytes

**Why:** One `vkCmdBindVertexBuffers` per mesh; cache-friendly for position-first transforms.

**In Spark:** `VulkanGpuMeshInterleaved.cpp` packer.

### Location 2 for vec2 UV

**Concept:** Vertex shader pass-through

**Decision:** Matches glTF TEXCOORD_0 → location 2

**Why:** Stable across rigid and skinned paths; six attributes unified.

**In Spark:** `scene.vert` location 2.

### Repeat addressing on sampler

**Concept:** UV outside [0,1]

**Decision:** `VK_SAMPLER_ADDRESS_MODE_REPEAT`

**Why:** Terrain and walls tile without UV scissor at 1.0.

**In Spark:** `CreateTextureSampler`.

### Derivative fallback for TBN

**Concept:** Tangent reconstruction

**Decision:** `dFdx`/`dFdy` path when tangent length near zero

**Why:** Legacy meshes without tangents still get approximate normal maps.

**In Spark:** `scene.frag` normal map block.

### Separate layers for separate maps

**Concept:** Fragment sampling

**Decision:** Albedo, normal, MR, emissive use different `*MapLayer` push fields

**Why:** glTF materials ship separate images; same array, different indices.

**In Spark:** `ModelPushConstants` map layer ints.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/scene/mesh/Mesh.hpp`
- `include/spark/render/scene/VulkanSceneVertexLayout.hpp`
- `src/spark/render/gpu/VulkanGpuMeshInterleaved.cpp`
- `shaders/scene.vert`, `shaders/scene.frag`
- `src/spark/render/scene/VulkanScenePipeline.cpp`

### Step 1 — Author UVs on a quad

Create a unit quad with corners (0,0), (1,0), (1,1), (0,1). Draw with checkerboard — should tile once across quad.

### Step 2 — Verify buffer packing

Hex-dump first vertex: floats at index 6–7 should match authored UV. Compare to RenderDoc mesh viewer after draw.

### Step 3 — UV gradient debug

Output `vTexCoord` as color. Rotate camera — gradient should stick to mesh surface (perspective correct), not slide.

### Step 4 — Normal map with tangent

Load glTF with tangent. Enable normal map layer. Compare lighting to derivative-only path — quality at silhouettes and UV seams.

### Step 5 — REPEAT test

Use UVs from 0 to 3 on a plane. Confirm three tiles with REPEAT sampler.

### Step 6 — Flip diagnosis

If texture upside-down, trace: exporter UV → CPU load flip → sample. Fix one stage only.

## 5. How This Connects to the Rest of Your Engine

Chapter **1–3** delivered the image and binding; this chapter delivers **coordinates**. Chapter **5** interprets sampled RGB as sRGB albedo. Chapter **6** multiplies lit result by albedo from UV lookup.

UV errors look like lighting errors — always debug UV before BRDF.

## 6. Pitfalls and Debugging

- **All black UV debug** — attribute location 2 mismatch; UVs read as zero.
- **Seams on normal map** — UV island boundary; derivative TBN breaks.
- **Mirrored text** — negative UV scale in DCC tool or flipped U in export.
- **Sky uses mesh UV by mistake** — `skyMode` path should not use albedo UV for equirect.
- **flat qualifier on UV** — never use `flat` on `vTexCoord` (only on layer index).

## 7. Exercises

1. Map a unit cube with unique UV per face — verify six regions on one texture atlas.
2. Implement UV scale via push constants `vec2 uvScale` in vertex shader.
3. Measure max UV derivative on a cube edge in shader (`fwidth(vTexCoord)`) — when do mips matter?
4. Export same mesh from Blender with different UV unwraps — compare on checkerboard.
5. Draw UV wireframe in Blender, screenshot, compare to Spark `fract(vTexCoord)` debug.

## Summary

UVs travel from `Mesh::Vertex::texCoord` through interleaved GPU buffers to `vTexCoord` and into `texture()` as the first two components of a `vec3` with layer as the third. Match vertex attribute location 2, respect load-time vertical flip conventions, and debug with false-color UV before enabling PBR. Normal maps need tangents or derivative reconstruction — both depend on well-authored UVs.

---

Next: [Color Spaces](05-color-space.md).
