# Diffuse Lighting

> **Build-Along Milestone:** You can trace the directional sun term from `ubo.lightDir` through `evalBRDF` to HDR `Lo`, including wrap diffuse and shadow modulation.

**You will learn:**

- Compute Lambert diffuse with correct normal and light direction.
- Combine diffuse and GGX specular in `evalBRDF`.
- Apply sun shadow and hemisphere fill in the forward path.
- Distinguish PBR, toon, and ambient-only contributions in `scene.frag`.

## 1. The Problem We Are Solving

Textured albedo tells you **surface color**; lighting tells you **how much** of each light reaches the camera. The simplest model is **Lambert diffuse**: brightness proportional to `max(dot(N, L), 0)`.

Spark's lit path goes further — Cook-Torrance specular, metallic energy split, wrapped diffuse for outdoor sun, cascaded shadow attenuation, hemisphere ambient, then punctual lights (Chapter 9). This chapter focuses on the **directional sun** and the `evalBRDF` core that punctual lights reuse.

## 2. Core Concepts

### Vectors for shading

At each fragment, build:

| Vector | Formula | Meaning |
|--------|---------|---------|
| `N` | `normalize(vNormal)` or TBN × normal map | Shading normal (world) |
| `V` | `normalize(cameraPos - vWorldPos)` | View direction |
| `L` | `normalize(-lightDir)` for directional | Toward light |
| `H` | `normalize(V + L)` | Half vector for specular |

Spark's UBO stores `lightDir` pointing **toward the light** for the sun — shader uses `Ld = normalize(ubo.lightDir.xyz)` and BRDF uses `L` toward light consistently in `evalBRDF(N, V, L, ...)`.

### Diffuse Lambert term

```glsl
float NdotL = max(dot(N, L), 0.0);
vec3 diffuse = kD * baseColor / PI;
// contribution: diffuse * NdotL * radiance
```

`kD = (1 - F) * (1 - metallic)` — metals have no diffuse albedo.

### Wrapped diffuse for sun

Outdoor scenes soften the terminator:

```glsl
const float sunDiffuseWrap = 0.22;
float NdotL = max((rawNL + wrapNL) / (1.0 + wrapNL), 0.0);  // when wrapNL > 0
```

Wrap redistributes light into grazing angles — fake subsurface / fill without a separate pass. **Punctual lights use wrapNL = 0** (harder falloff).

### Specular GGX (summary)

`evalBRDF` combines:

- `D_GGX` — normal distribution (roughness → highlight size)
- `G_Smith` — geometry masking
- `F_Schlick` — Fresnel (view-dependent specular color)

```glsl
vec3 specular = (D * G * F) / max(4.0 * NdotV * specNL, 1e-6);
return (diffuse * NdotL + specular * specNL) * radiance;
```

Roughness clamped to minimum 0.04 avoids singularities. Full microfacet theory extends Part IV — here you implement and call `evalBRDF`.

### Sun radiance from UBO

```glsl
vec3 sunRad = ubo.lightColor.rgb * ubo.lightColor.w;
vec3 sunBrdf = evalBRDF(N, V, Ld, base, met, rough, sunRad, sunDiffuseWrap);
Lo += sunBrdf * sunShadow;
```

`lightColor.w` is **intensity** scalar — can exceed 1 for bright sun.

### Shadow modulation

Before adding sun BRDF, Spark computes `sunShadow` from cascaded shadow maps (Part V preview):

```glsl
if (push.skyMode == 0 && ubo.shadowParams.w > 0.5 && (push.shadowFlags & 2) != 0) {
    sunShadow = blendSunShadowCascades(distCam, vWorldPos, N, Ld, sunGracing);
    // distance fade mixes shadow to 1.0 far from camera
}
```

`shadowFlags & 2` = receive shadows. Sky draws skip sun lighting.

### Skylight fill in shadow

When sun is occluded, hemispherical ambient tints shadowed diffuse:

```glsl
float skyFill = (1.0 - sunShadow) * (0.11 + 0.14 * sunGracing);
vec3 skyHue = mix(ambGround, ambSkyBase, 0.72);
Lo += base * (1.0 - met) * (1.0 / PI) * ndUp * skyHue * skyFill * max(length(sunRad), 1e-4);
```

Prevents pitch-black shadows outdoors while staying physically motivated.

### Hemisphere ambient base

```glsl
float hemi = clamp(N.y * 0.5 + 0.5, 0.0, 1.0);
vec3 ambTint = mix(ambGround, ambSkyBase, hemi);
vec3 ambient = ambTint * base * mix(1.0, 0.35, met);
ambient += ubo.ambientProbe.rgb * ubo.ambientProbe.w * base * mix(0.35, 0.12, met);
```

`ambientColor`, `ambientSky`, `ambientProbe` come from UBO — filled by lighting profile (Chapter 10).

### Toon shading branch

When `push.shadingModel == 1`, sun uses **stepped diffuse bands** instead of `evalBRDF`:

```glsl
float steppedNL = floor(ndl * b + 1e-4) / denom;
vec3 sunDiffuse = base * (1.0 - met * 0.92) * steppedNL * sunRad * sunShadow;
```

Separate specular band and rim light — same `sunShadow`, different BRDF aesthetic.

### Material resolve before lighting

Albedo `base` combines tint, texture (sRGB decoded), and metallic scales roughness from maps — all before `Lo` accumulation. Order in `main()`:

1. Resolve `base`, `met`, `rough`, `N` (normal map)
2. Sun directional + shadow
3. Punctual loops (`accumulatePunctualPbr`)
4. Ambient + IBL specular
5. Emissive add
6. `outColor = vec4(ambient + Lo + emissive, alpha)`

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### evalBRDF shared by sun and punctual

**Concept:** Specular GGX

**Decision:** One function; radiance and wrap vary per light

**Why:** Consistent energy; punctual passes attenuated radiance in.

**In Spark:** `scene.frag` `evalBRDF`, called from sun and `accumulatePunctualPbr`.

### Wrap only on directional

**Concept:** Wrapped diffuse

**Decision:** `sunDiffuseWrap = 0.22`; punctual `wrapNL = 0`

**Why:** Sun fills outdoor scenes; local lights stay sharp.

**In Spark:** Sun call vs `evalBRDF(..., 0.0)` in punctual loop.

### Sun in UBO not SSBO

**Concept:** Sun radiance

**Decision:** `lightDir`, `lightColor` in frame UBO

**Why:** One sun per frame; no index loop needed.

**In Spark:** `scene_ubo.glsl`, `VulkanSceneUniformWriter`.

### Shadow as separate multiplier

**Concept:** Shadow modulation

**Decision:** Multiply BRDF result by `sunShadow`, not zero radiance

**Why:** Allows partial penumbra from PCF; skylight fill uses `(1 - sunShadow)`.

**In Spark:** Lines 492–558 in `scene.frag`.

### HDR linear output

**Concept:** Material resolve order

**Decision:** `outColor` can exceed 1.0 before tonemap

**Why:** Bright sun + specular highlights need headroom.

**In Spark:** HDR render target.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `shaders/scene.frag` (`evalBRDF`, sun block, ambient)
- `shaders/scene_ubo.glsl`
- `src/spark/render/scene/VulkanSceneUniformWriter.cpp`

### Step 1 — Diffuse only

Replace `evalBRDF` return with `diffuse * NdotL * radiance` (zero specular). Verify terminator rotates with light dir.

### Step 2 — Flip light direction

Negate `ubo.lightDir` on CPU — if shading inverts, your `L` convention was wrong.

### Step 3 — Normal debug

Use `N` from normal map vs vertex normal — compare specular highlight shape.

### Step 4 — Shadow on/off

Toggle `push.shadowFlags` receive bit — `sunShadow` should be 1.0 when off.

### Step 5 — Toon mode

Set `shadingModel = 1` on a draw — banded diffuse visible on sphere.

### Step 6 — Intensity sweep

Increase `lightIntensity` 1 → 10 → 100 — HDR buffer should clip only after extreme values; tonemap handles display.

## 5. How This Connects to the Rest of Your Engine

Chapter **7** adds point and spot lights using the same `evalBRDF`. Chapter **8** fills UBO from ECS. Chapter **9** loops punctual SSBO. Part **V** deepens shadow sampling. Part **VIII** tonemaps `Lo`.

Diffuse is the **first** lighting term to validate — if `N·L` is wrong, nothing else helps.

## 6. Pitfalls and Debugging

- **Flat shading** — forgot `normalize(N)` after interpolation.
- **Inside-out lighting** — `L` points away from light; check UBO sign convention.
- **Black everywhere** — `lightIntensity` zero or albedo zero after texture miss.
- **Specular fireflies** — denominator not clamped; Spark uses `1e-6` floor.
- **Metallic still diffuse** — `kD` should go to 0 as metallic → 1.

## 7. Exercises

1. Visualize `NdotL` as grayscale for directional light only.
2. Plot wrapped vs hard diffuse for `wrapNL = 0, 0.22, 0.5`.
3. Disable specular in `evalBRDF` — measure FPS change (often small).
4. Explain why metals use `F0 = baseColor` in `evalBRDF`.
5. Match sun direction to a debug arrow drawn in editor (ECS +Z toward light).

## Summary

Spark shades the directional sun with `evalBRDF` — Lambert diffuse plus GGX specular, metallic energy conservation, optional wrap for outdoor softness, and shadow modulation. Hemisphere ambient and probe fill shadowed regions. Toon mode replaces BRDF with stepped bands. Resolve linear albedo and normals first, then accumulate `Lo` in HDR before tonemap.

---

Next: [Light Types](07-light-types.md).
