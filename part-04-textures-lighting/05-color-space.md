# Color Spaces

> **Build-Along Milestone:** Albedo samples pass through `sparkSrgbToLinear` before BRDF math; normal and ORM maps stay linear.

**You will learn:**

- Distinguish sRGB-encoded albedo from linear data textures.
- Explain why Spark uses `R8G8B8A8_UNORM` storage with shader-side gamma decode.
- Apply `sparkSrgbToLinear` consistently for sun, sky, and material color.
- Avoid double gamma correction in HDR forward rendering.

## 1. The Problem We Are Solving

Human eyes perceive brightness **non-linearly** — doubling pixel value does not look twice as bright. Monitors and PNG files encode albedo colors in **sRGB** to use 8 bits efficiently. Physically based lighting, however, must add and multiply radiance in **linear** space.

Mix the two and you get:

- Dark muddy shadows (lighting applied twice in gamma space)
- Blown highlights on metals
- Wrong tint when blending lights

Spark stores textures as **UNORM** (linear 0–1 bytes) but treats albedo art as sRGB-encoded, decoding in the fragment shader via `color_space.glsl`. This chapter explains that contract and where it applies.

## 2. Core Concepts

### sRGB vs linear (intuition)

| Space | What it represents | Where it lives |
|-------|-------------------|----------------|
| sRGB (encoded) | Perceptual color in files | PNG, JPEG, Photoshop export |
| Linear | Physical radiance/reflectance | GPU lighting math |

The transfer function is piecewise:

```text
linear = srgb / 12.92                    if srgb ≤ 0.04045
linear = ((srgb + 0.055) / 1.055)^2.4   otherwise
```

(Shader uses normalized 0–1 channel values, not 0–255.)

### Spark's storage decision

```cpp
// GPU image format
VK_FORMAT_R8G8B8A8_UNORM   // NOT SRGB8_ALPHA8
```

| Approach | Pros | Cons |
|----------|------|------|
| `SRGB8_ALPHA8` format | Hardware decode on sample | Same format for normal/ORM maps is wrong |
| `UNORM` + shader decode | One array for all map types | Must remember which maps decode |

Spark chooses **UNORM + selective decode** — albedo and sky art yes; normal, metallic-roughness, emissive data no.

### sparkSrgbToLinear implementation

```glsl
vec3 sparkSrgbToLinear(vec3 srgb) {
    vec3 lo = srgb / 12.92;
    vec3 hi = pow((srgb + vec3(0.055)) / vec3(1.055), vec3(2.4));
    return mix(lo, hi, step(vec3(0.04045), srgb));
}
```

Included from `color_space.glsl` in `scene.frag` and sky paths.

### Where decode happens in scene.frag

**Albedo base color:**

```glsl
vec4 tex = texture(sceneTextures, vec3(vTexCoord, float(layer)));
base *= sparkSrgbToLinear(tex.rgb);
```

**Equirect sky texture:**

```glsl
skyCol = sparkSrgbToLinear(tex.rgb) * push.albedoTint.rgb;
```

**Not decoded:**

- Normal map XYZ (tangent space vectors)
- Metallic-roughness packed channels
- Emissive map when treated as linear HDR emissive mask (Spark multiplies without sRGB on emissive map sample in some paths — verify: emissive map uses raw `.rgb` multiply)

Check emissive path:

```glsl
emissiveT *= texture(sceneTextures, vec3(vTexCoord, float(el))).rgb;
```

Emissive maps are typically **linear** (not sRGB) — correct to skip decode.

### Linear workflow through the frame

```text
sRGB file → CPU RGBA8 → GPU UNORM sample → sparkSrgbToLinear → BRDF → HDR float out
                                                                              ↓
                                                                    tonemap (Part VIII) → display sRGB
```

Lighting never converts back to sRGB mid-shader. `outColor` is linear HDR until tonemap pass.

### CPU side does not gamma-correct

`Texture2D` stores raw bytes from stb_image — no pow(2.2) on load. The **file is already sRGB-encoded**; bytes represent encoded values. Shader decode completes the pipeline.

### Common authoring mistakes

| Mistake | Symptom |
|---------|---------|
| Albedo painted in linear in Substance | Washed out after `sparkSrgbToLinear` — double decode |
| Normal map saved as sRGB in exporter | Wrong lighting, purple/green shifts |
| ORM in sRGB | Metallic channel crushed |
| Skip decode on albedo | Dark, contrasty materials |

Author albedo in sRGB (default in art tools). Mark normal/ORM as **non-color data** in exporters.

### HDR sun and lights

`ubo.lightColor.rgb * ubo.lightColor.w` is **linear radiance** — not sRGB. Intensity can exceed 1.0. Do not run sun color through `sparkSrgbToLinear` unless you intentionally stored sun color as sRGB bytes on CPU (Spark does not).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### UNORM array, shader decode

**Concept:** Storage decision

**Decision:** `VK_FORMAT_R8G8B8A8_UNORM` for entire scene array

**Why:** One format for albedo, normal, MR, emissive layers; decode only where needed.

**In Spark:** `VulkanSceneTextureUploader`, header comment.

### color_space.glsl include

**Concept:** sparkSrgbToLinear

**Decision:** Shared function, not inline per sample

**Why:** Single correct implementation; include guard prevents duplicate symbols.

**In Spark:** `shaders/color_space.glsl`.

### Albedo and sky only

**Concept:** Where decode happens

**Decision:** `sparkSrgbToLinear` on base color and equirect sky

**Why:** Art textures are sRGB; data maps are linear.

**In Spark:** `scene.frag` albedo and sky branches.

### Linear HDR scene output

**Concept:** Linear workflow

**Decision:** No `pow(color, 1/2.2)` before render pass end

**Why:** Tonemap pass owns display encoding (Part VIII).

**In Spark:** HDR color attachment format.

### No CPU gamma on load

**Concept:** CPU side

**Decision:** stb bytes copied verbatim

**Why:** Bytes already represent sRGB encoding from file format.

**In Spark:** `Texture2D::SetPixels`.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `shaders/color_space.glsl`
- `shaders/scene.frag` (albedo resolve block)
- `include/spark/render/scene/VulkanSceneTextureUploader.hpp` (format comment)

### Step 1 — Visualize linear vs skip

Temporarily comment `sparkSrgbToLinear` on albedo — scene looks darker and harsher. Restore.

### Step 2 — Solid white albedo

`textureLayer = -1`, `albedoTint = (1,1,1)`. Lighting should behave predictably — base is linear 1.0.

### Step 3 — Compare normal map

Apply sRGB decode to normal sample — lighting breaks. Confirm normal path uses raw `.xyz * 2 - 1` without gamma.

### Step 4 — Gradient texture test

Upload a linear gradient (procedural, not from sRGB file) — if it looks wrong with decode, author as sRGB or skip decode for that layer.

### Step 5 — Document per-map policy

Build a table for your engine: albedo sRGB, normal linear, ORM linear, emissive linear.

## 5. How This Connects to the Rest of Your Engine

Chapter **6** multiplies linear albedo by linear radiance in `evalBRDF`. Part **VIII** tonemaps linear HDR to display. Part **V** IBL samples environment maps with similar linear conventions in `ibl.glsl`.

Color space mistakes surface as "lighting feels wrong" — fix gamma before tuning light intensities.

## 6. Pitfalls and Debugging

- **Everything too dark** — missing sRGB decode on albedo.
- **Everything too bright/washed** — decode on linear data textures.
- **Inconsistent between tools** — Substance linear export vs Blender sRGB.
- **Alpha channel** — sRGB decode on RGB only; straight alpha stays linear.
- **Using SRGB format for array** — would wrongly decode normal maps if whole array is SRGB.

## 7. Exercises

1. Plot `sparkSrgbToLinear(x)` for x in [0,1] — compare to `pow(x, 2.2)` approximation.
2. Load same albedo with and without decode — screenshot difference in shadow terminator.
3. Find all `sparkSrgbToLinear` call sites in Spark shaders (grep).
4. Explain why sun `lightColor` skips decode in one paragraph.
5. If switching to `SRGB8_ALPHA8` for albedo only, what happens to normal map layers in the same image? (Cannot share one array format — reinforces Spark design.)

## Summary

Spark stores scene textures as UNORM and decodes sRGB-encoded albedo and sky art in the shader with `sparkSrgbToLinear`. Normal, metallic-roughness, and emissive maps sample as linear data. Keep all BRDF accumulation in linear space; defer display gamma to the tonemap pass.

---

Next: [Diffuse Lighting](06-diffuse-lighting.md).
