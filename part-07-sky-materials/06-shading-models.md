# Shading Models

> **Build-Along Milestone:** PBR and toon spheres side-by-side under one sun and point light.

**You will learn:**

- Branch fragment shading on `SceneShadingModel::LitPbr` vs `ToonCel`.
- Configure toon band, rim, and wrap diffuse parameters from `MaterialComponent`.
- Explain which lighting features (IBL, punctual, skylight fill) each path uses.

## 1. The Problem We Are Solving

Realistic PBR and stylized cel shading share the same mesh, texture fetch, and shadow visibility — they diverge only in how **N·L** and specular are integrated. Duplicating render passes or pipelines per art style wastes maintenance and breaks batching.

Your engine should select the shading model **per material** with a single push constant branch inside `scene.frag`. Build along: place two identical spheres, set one `LitPbr` and one `ToonCel`, and tune bands until the cel read is crisp under the same sun and point lights.

## 1b. Concept Deep Dive — Cook-Torrance PBR in scene.frag

### evalBRDF — the LitPbr core

Spark's PBR path uses GGX normal distribution, Smith geometry, and Schlick Fresnel:

```glsl
// shaders/scene.frag (lines 203–237)
vec3 evalBRDF(vec3 N, vec3 V, vec3 L, vec3 baseColor, float metallic, float roughness,
        vec3 radiance, float wrapNL) {
    float rawNL = dot(N, L);
    float NdotL = (wrapNL > 1e-5) ? max((rawNL + wrapNL) / (1.0 + wrapNL), 0.0) : max(rawNL, 0.0);
    if (NdotL <= 0.0) return vec3(0.0);

    vec3 H = normalize(V + L);
    float rough = clamp(roughness, 0.04, 1.0);
    vec3 F0 = mix(vec3(0.04), baseColor, metallic);
    vec3 F = F_Schlick(max(dot(V, H), 0.0), F0);

    float D = D_GGX(max(dot(N, H), 0.0), rough);
    float G = G_Smith(max(dot(N, V), 0.001), max(rawNL, 0.0), rough);
    vec3 specular = (D * G * F) / max(4.0 * NdotV * max(rawNL, 1e-4), 1e-6);

    vec3 kD = (1.0 - F) * (1.0 - metallic);
    vec3 diffuse = kD * baseColor / PI;
    return (diffuse * NdotL + specular * max(rawNL, 0.0)) * radiance;
}
```

**Energy conservation:** `kD = (1-F) × (1-metallic)` — metals have no diffuse term; dielectrics use F0 ≈ 0.04.

### LitPbr output assembly

```glsl
// shaders/scene.frag (concept — after toon early-out)
vec3 sunBrdf = evalBRDF(N, V, Ld, base, met, rough, sunRad, sunDiffuseWrap);
Lo += sunBrdf * sunShadow;
accumulatePunctualPbr(N, V, base, met, rough, Lo);  // clustered point/spot lights
vec3 iblSpecular = sparkEvalSpecularIbl(...);
vec3 iblDiffuse = sparkEvalDiffuseIbl(...);
vec3 color = ambient + Lo + iblSpecular + emissive;
sparkApplyTransmission(...);  // KHR_materials_transmission
outColor = vec4(color, alpha);
```

### Toon path — quantized N·L before return

```glsl
// shaders/scene.frag (lines 504–527)
if (push.shadingModel == 1) {
    float b = clamp(float(push.toonDiffuseBands), 2.0, 8.0);
    float steppedNL = floor(ndl * b + 1e-4) / max(b - 1.0, 1.0);
    vec3 sunDiffuse = base * (1.0 - met * 0.92) * steppedNL * sunRad * sunShadow;
    // hard spec band + view rim ...
    Lo = sunDiffuse + specCol + rimCol;
    accumulatePunctualToon(...);
    outColor = vec4(ambient + Lo + emissive, alpha);
    return;  // skip IBL specular entirely
}
```

### Push constant wiring from CPU

```cpp
// VulkanSceneOpaquePass.cpp (concept)
push.shadingModel = static_cast<std::int32_t>(d.shadingModel);
push.toonDiffuseBands = std::clamp(d.toonDiffuseBands, 2, 8);
push.toonRimIntensity = d.toonRimIntensity;
push.toonRimPower = d.toonRimPower;
```

## 2. Core Concepts

### Single pipeline, shader branch

Spark keeps **one** lit graphics pipeline (`pipelineLit`) and switches shading inside `scene.frag` via `push.shadingModel`. PBR uses Cook-Torrance `evalBRDF`; toon uses banded wrap diffuse, stepped specular, and view rim.

| Stage | Shared by both | PBR only | Toon only |
|-------|----------------|----------|-----------|
| Albedo sample | ✓ | | |
| ORM unpack | ✓ | | |
| Normal map TBN | ✓ | | |
| CSM sun shadow | ✓ | | |
| `evalBRDF` sun | | ✓ | |
| Stepped N·L bands | | | ✓ |
| `accumulatePunctualPbr` | | ✓ | |
| `accumulatePunctualToon` | | | ✓ |
| Hemispheric ambient | ✓ (different mix) | | |
| `sparkEvalSpecularIbl` | | ✓ | |
| Skylight penumbra fill | | ✓ | |

### Shared fetch stage

Both paths sample albedo, ORM, normal map, and build world **N** identically before branching — only light integration differs. Shadows multiply sun terms in both models; punctual lights call separate accumulate helpers.

### Toon parameters from material

| Field | Role |
|-------|------|
| `toonDiffuseBands` | Quantizes wrap diffuse (clamped 2–8 in push) |
| `toonRimPower` | View rim falloff exponent |
| `toonRimIntensity` | Rim brightness scale |

PBR ignores these fields; toon ignores IBL specular entirely.

### IBL on PBR only

The toon path uses hemispheric ambient fill and exits before `sparkEvalSpecularIbl`. Stylized characters stay clean without glossy environment reflections — by design, not oversight.

### Early return structure

```text
scene.frag main():
  sky early-out (push.skyMode != 0)
  fetch albedo, ORM, build N
  if (push.shadingModel == ToonCel) { … punctual toon … return; }
  // PBR continues: evalBRDF, skylight fill, punctual PBR, IBL
```

Keeping toon in an early block avoids nested `else` chains in the PBR path.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### uint8 on draw item → push constant

**Concept:** `SceneShadingModel` enum

**Decision:** Compact `uint8` per draw, copied to push block

**Why:** No SPIR-V specialization constants yet; per-draw switch is cheap.

**In Spark:** `SceneShadingModel.hpp`, `VulkanSceneOpaquePass.cpp`

### `if (shadingModel == 1) { … return; }`

**Concept:** Toon before PBR in fragment main

**Decision:** Early return after shared fetch

**Why:** PBR path stays linear; toon owns its ambient mix.

**In Spark:** `scene.frag` lines ~504–549

### `sunDiffuseWrap` on N·L

**Concept:** Wrap diffuse for toon

**Decision:** Soften terminator before band quantization

**Why:** Avoids harsh single-step cel on curved spheres.

**In Spark:** `scene.frag` toon sun block

### `sunShadow` on both paths

**Concept:** Shared shadow multiply

**Decision:** Cel characters still receive CSM

**Why:** Stylization applies to lit term, not visibility.

**In Spark:** `scene.frag` `sunShadow` computation

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `shaders/scene.frag`
- `include/spark/render/scene/SceneShadingModel.hpp`
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp`
- `src/spark/demo/ToonShadingDemo.cpp` (optional comparison)

### Step 1 — Locate shading branch

*Toon path — banded diffuse, spec edge, rim highlight.*

```glsl
// shaders/scene.frag (lines 504–527)
    /** Toon/cel: banded wrap diffuse, hard spec band, view rim (push.toon*). */
    if (push.shadingModel == 1) {
        float b = clamp(float(push.toonDiffuseBands), 2.0, 8.0);
        float denom = max(b - 1.0, 1.0);
        float rawNL = dot(N, Ld);
        float ndl = max((rawNL + sunDiffuseWrap) / (1.0 + sunDiffuseWrap), 0.0);
        float steppedNL = floor(ndl * b + 1e-4) / denom;

        vec3 sunDiffuse = base * (1.0 - met * 0.92) * steppedNL * sunRad * sunShadow;

        vec3 Hs = normalize(V + Ld);
        float nh = max(dot(N, Hs), 0.0);
        float gloss = mix(120.0, 14.0, rough);
        float specTerm = pow(nh, gloss);
        float specEdge = smoothstep(0.22, 0.30, specTerm);
        vec3 specTint = mix(vec3(0.06), vec3(1.0), met);
        vec3 specCol = specTint * specEdge * max(rawNL, 0.0) * sunRad * sunShadow * 0.7;

        float nv = clamp(dot(N, V), 0.0, 1.0);
        float rimAmt = pow(1.0 - nv, max(push.toonRimPower, 0.35));
        vec3 rimCol =
                rimAmt * clamp(push.toonRimIntensity, 0.0, 4.0) * mix(vec3(0.32, 0.48, 1.0), sunRad, 0.45) * steppedNL;

        Lo = sunDiffuse + specCol + rimCol;
```

### Step 2 — Compare PBR path continuation

*PBR `evalBRDF` + punctual accumulate + IBL mix.*

```glsl
// shaders/scene.frag (lines 552–574)
    vec3 sunBrdf = evalBRDF(N, V, Ld, base, met, rough, sunRad, sunDiffuseWrap);
    Lo += sunBrdf * sunShadow;
    float skyFill = (1.0 - sunShadow) * (0.11 + 0.14 * sunGracing);
    vec3 skyHue = mix(ambGround, ambSkyBase, 0.72);
    float ndUp = max(dot(N, vec3(0.0, 1.0, 0.0)), 0.0);
    Lo += base * (1.0 - met) * (1.0 / PI) * ndUp * skyHue * skyFill * max(length(sunRad), 1e-4);

    accumulatePunctualPbr(N, V, base, met, rough, Lo);

    float hemi = clamp(N.y * 0.5 + 0.5, 0.0, 1.0);
    vec3 ambTint = mix(ambGround, ambSkyBase, hemi);
    vec3 ambient = ambTint * base * mix(1.0, 0.35, met);
    ambient += ubo.ambientProbe.rgb * ubo.ambientProbe.w * base * mix(0.35, 0.12, met);
    float aoAmb = mix(0.74, 1.0, pow(clamp(dot(N, V), 0.0, 1.0), 0.58));
    ambient *= aoAmb * occlusion;

    vec3 iblSpecular = sparkEvalSpecularIbl(sceneTextures, N, V, base, met, rough, occlusion);
    vec3 iblDiffuse = sparkEvalDiffuseIbl(sceneTextures, N, base, met, aoAmb * occlusion);
    if (ubo.iblParams.w > 0.5) {
        ambient = mix(ambient, iblDiffuse, mix(0.55, 0.92, met));
        ambient *= mix(1.0, 0.12, met);
    }
```

### Step 3 — Set models on two cubes

```cpp
left->GetComponent<MaterialComponent>()->SetShadingModel(SceneShadingModel::LitPbr);
right->GetComponent<MaterialComponent>()->SetShadingModel(SceneShadingModel::ToonCel);
```

Use identical albedo textures and light rig — only shading differs.

### Step 4 — Tune toon bands live

```cpp
mat->SetToonDiffuseBands(3); // sharper cel
mat->SetToonRimIntensity(0.6f);
mat->SetToonRimPower(3.0f);
```

### Step 5 — Verify push constant wiring

```cpp
// src/spark/render/scene/VulkanSceneOpaquePass.cpp
        push.shadingModel = static_cast<std::int32_t>(d.shadingModel);
        push.toonDiffuseBands = std::clamp(d.toonDiffuseBands, 2, 8);
        push.toonRimIntensity = d.toonRimIntensity;
        push.toonRimPower = d.toonRimPower;
```

Capture RenderDoc for both draws in the same pass — `shadingModel` should differ, pipeline handle identical.

### Step 6 — Milestone — side-by-side spheres

Two spheres, same maps: PBR shows smooth specular falloff; toon shows 3–4 discrete bands plus blue-tinted rim. Rotate point light — both respond, but toon spec stays a hard band.

## 5. How This Connects to the Rest of Your Engine

**MaterialComponent** owns `shadingModel` and toon fields. **Normal Mapping** perturbs **N** before either branch. **IBL Basics** feeds PBR only. **Material Showcase Demo** toggles models interactively for regression. Part **X** tonemap processes HDR from both paths.

## 6. Pitfalls and Debugging

- **Toon without shadows** — characters float; keep `sunShadow` multiply on cel path.
- **Too many bands** — approaches PBR; 2–4 bands read best for cel.
- **HDR emissive on toon** — can clip before tonemap; cap `emissiveIntensity` for stylized presets.
- **Expecting IBL spec on toon** — by design Spark skips it.
- **Wrong push field** — `toonDiffuseBands` clamped on CPU; values below 2 still become 2 in shader.

## 7. Exercises

1. Graph stepped N·L for 3 bands vs smooth Lambert on paper.
2. Animate `toonDiffuseBands` at runtime — note spec edge unchanged.
3. Capture RenderDoc push constants for PBR vs toon draw in same pass.
4. Run `ToonShadingDemo` and list which keys change bands vs rim.
5. Temporarily force `push.shadingModel = 1` in C++ for a metal sphere — describe the visual.

## Summary

`LitPbr` and `ToonCel` share texture fetch and shadows; `scene.frag` branches on `shadingModel` for BRDF vs banded cel integration. Toon exits before IBL; PBR continues through split-sum environment lighting.

---

Next: [Normal Mapping](07-normal-mapping.md).
