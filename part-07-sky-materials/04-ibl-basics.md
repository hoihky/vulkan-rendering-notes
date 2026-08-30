# IBL Basics

> **Build-Along Milestone:** Metallic sphere reflects HDR sky; toggling iblParams.w removes environment specular.

**You will learn:**

- Explain diffuse and specular IBL terms in Spark's ibl.glsl.
- Wire iblEnvironmentLayer from sky submit to scene UBO iblParams.
- Toggle IBL and observe metal vs dielectric response.

## 1. The Problem We Are Solving

Flat ambient terms make metals look like plastic. **IBL** supplies direction-dependent indirect light from an
environment — ideally the same equirect visible as the sky — so specular reflections match the background.

Build along: assign an equirect to sky MaterialComponent, let submit resolve `iblEnvironmentLayer`, and enable
`ubo.iblParams.w` on a metallic test sphere.

## 1b. Concept Deep Dive — Split-Sum IBL in Spark

### What IBL approximates

**Image-based lighting** replaces flat ambient with radiance sampled from an environment map. Spark implements the **split-sum approximation**:

```text
Specular IBL ≈ prefiltered_env(N, R, roughness) × (F0 × brdf_lut.x + brdf_lut.y)
Diffuse IBL  ≈ irradiance(N) × baseColor × (1 - metallic)
```

Diffuse samples the environment along the **surface normal**; specular along the **reflection vector** `R = reflect(-V, N)` with roughness-dependent blur.

### ibl.glsl entry points

```glsl
// shaders/ibl.glsl (concept)
vec3 sparkEvalSpecularIbl(sampler2DArray ldrTex, sampler2DArray hdrTex,
        vec3 n, vec3 v, vec3 baseColor, float metallic, float roughness, float occlusion) {
    if (ubo.iblParams.w < 0.5) return vec3(0.0);  // IBL disabled
    int layer = int(round(ubo.iblParams.x));
    // importance-sample GGX hemisphere, accumulate prefiltered radiance
    // sample BRDF LUT at (NdotV, roughness)
    return prefiltered * (f0 * brdf.x + brdf.y) * intensity * occlusion;
}

vec3 sparkEvalDiffuseIbl(...) {
    vec3 irradiance = sparkSampleEnvironment(ldrTex, hdrTex, n, layer, ...);
    return irradiance * baseColor * (1.0 - metallic) * intensity * occlusion;
}
```

### Auto-resolve environment from sky draw

```cpp
// src/spark/scene/submit/SceneSubmitMaterial.cpp (lines 126–147)
void ResolveIblEnvironmentLayer(SceneRenderParams& params) noexcept {
    if (params.iblEnvironmentLayer >= 0) return;  // explicit override
    for (each draw d in params.draws) {
        if (d.skyMode != SceneSkyMode::None && d.textureLayer >= 0) {
            params.iblEnvironmentLayer = d.textureLayer;
            params.iblEnvironmentIsHdr = d.textureIsHdrLinear;
            params.iblEnvironmentUvScale = d.textureUvScale;
            return;
        }
    }
    params.iblEnvironmentLayer = -1;  // procedural env in shader
}
```

Assign equirect once on sky `MaterialComponent` — metals on props automatically reflect the same HDR without duplicating layer setup.

### UBO packing each frame

```cpp
// src/spark/render/scene/VulkanSceneUniformWriter.cpp (concept)
u.iblParams[0] = (float)scene.iblEnvironmentLayer;
u.iblParams[1] = scene.iblIntensity * (scene.iblEnvironmentIsHdr ? 0.4f : 1.0f);
u.iblParams[2] = scene.iblEnvironmentIsHdr ? 1.0f : 0.0f;
u.iblParams[3] = scene.iblEnabled ? 1.0f : 0.0f;
```

### Offline BRDF LUT (binding 12)

`VulkanIblBrdfLut` bakes a 512×512 Hammersley + GGX integration texture at init — fragment shader samples it for the `F0 × A + B` split-sum term without runtime Monte Carlo per pixel.

## 2. Core Concepts

### Split-sum IBL preview

**Image-based lighting** approximates infinite environment radiance from a texture. Spark's `ibl.glsl` implements diffuse irradiance via environment lookup along **N**, and specular via importance-sample prefilter + analytic BRDF approximation (split-sum).


### Equirect in texture array

Environment maps live in the same `sceneTextures` array as albedo. `sparkSampleEquirect` maps direction to `(phi, vEq)` — identical math to the sky equirect branch in `scene.frag`.


### Procedural fallback

When `layer < 0`, `sparkSampleProceduralEnv` mixes `ubo.ambientColor` and `ubo.ambientSky` by direction Y — scenes without HDR files still get directionally varying indirect light.


### iblParams UBO gate

`ubo.iblParams.w` enables IBL; `.x` is environment layer index, `.y` intensity. Submit sets `iblEnvironmentLayer` from the first sky draw with a bound texture (`ResolveIblEnvironmentLayer`).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Shared functions in scene.frag

**Concept:** ibl.glsl include

**Decision:** Shared functions in scene.frag

**Why:** One implementation for sky sampling and material IBL; reduces drift between visible sky and lighting.

**In Spark:** `shaders/ibl.glsl`

### Auto-bind sky equirect for IBL

**Concept:** ResolveIblEnvironmentLayer

**Decision:** Auto-bind sky equirect for IBL

**Why:** Artists assign texture once on sky MaterialComponent; metals pick up matching reflections.

**In Spark:** `src/spark/scene/SceneSubmitMaterial.cpp`

### mix(ambient, iblDiffuse, met) when IBL on

**Concept:** Replace part of hemispheric ambient

**Decision:** mix(ambient, iblDiffuse, met) when IBL on

**Why:** Avoid double-counting flat ambient and environment — metallic surfaces lean on IBL.

**In Spark:** `scene.frag` IBL integration (~569–574)

### kIblSampleCount importance GGX loop

**Concept:** 16-sample prefilter

**Decision:** kIblSampleCount importance GGX loop

**Why:** Runtime prefilter without offline mip chain — trade GPU for simpler asset pipeline early on.

**In Spark:** `ibl.glsl` sparkPrefilterEnvironment

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `shaders/ibl.glsl`
- `shaders/scene.frag`
- `src/spark/scene/SceneSubmitMaterial.cpp`

### Step 1 — Read ibl.glsl entry points

*Equirect sampling and procedural environment fallback.*

```glsl
// shaders/ibl.glsl (lines 10–30)
vec3 sparkSampleEquirect(sampler2DArray tex, vec3 dir, int layer) {
    float phi = atan(dir.z, dir.x) / (2.0 * SPARK_PI) + 0.5;
    phi = fract(phi);
    float mu = acos(clamp(dir.y, -1.0, 1.0)) / SPARK_PI;
    float vEq = 1.0 - mu;
    return sparkSrgbToLinear(textureLod(tex, vec3(phi, vEq, float(layer)), 0.0).rgb);
}

vec3 sparkSampleProceduralEnv(vec3 dir) {
    vec3 horizon = ubo.ambientColor.rgb;
    vec3 zenith = ubo.ambientSky.rgb;
    float t = clamp(dir.y * 0.5 + 0.5, 0.0, 1.0);
    return mix(horizon, zenith, pow(t, 0.65));
}

vec3 sparkSampleEnvironment(sampler2DArray tex, vec3 dir, int layer) {
    if (layer >= 0) {
        return sparkSampleEquirect(tex, dir, layer);
    }
    return sparkSampleProceduralEnv(dir);
}
```


*sparkEvalSpecularIbl and sparkEvalDiffuseIbl gated by ubo.iblParams.w.*

```glsl
// shaders/ibl.glsl (lines 78–112)
vec3 sparkEvalSpecularIbl(
        sampler2DArray tex,
        vec3 n,
        vec3 v,
        vec3 baseColor,
        float metallic,
        float roughness,
        float occlusion) {
    if (ubo.iblParams.w < 0.5) {
        return vec3(0.0);
    }
    int layer = int(round(ubo.iblParams.x));
    float intensity = ubo.iblParams.y;
    vec3 f0 = mix(vec3(0.04), baseColor, metallic);
    vec3 r = reflect(-v, n);
    float nDotV = max(dot(n, v), 0.001);
    vec3 prefiltered = sparkPrefilterEnvironment(tex, r, roughness, layer);
    vec2 brdf = sparkEnvBrdfApprox(roughness, nDotV);
    return prefiltered * (f0 * brdf.x + brdf.y) * intensity * occlusion;
}

vec3 sparkEvalDiffuseIbl(
        sampler2DArray tex,
        vec3 n,
        vec3 baseColor,
        float metallic,
        float occlusion) {
    if (ubo.iblParams.w < 0.5) {
        return vec3(0.0);
    }
    int layer = int(round(ubo.iblParams.x));
    float intensity = ubo.iblParams.y;
    vec3 irradiance = sparkSampleEnvironment(tex, n, layer);
    return irradiance * baseColor * (1.0 - metallic) * intensity * occlusion;
}
```

### Step 2 — Trace environment layer resolve

*ResolveIblEnvironmentLayer — first sky draw with textureLayer wins.*

```cpp
// src/spark/scene/SceneSubmitMaterial.cpp (lines 59–70)
void ResolveIblEnvironmentLayer(SceneRenderParams& params) noexcept {
    if (params.iblEnvironmentLayer >= 0) {
        return;
    }
    for (std::size_t i = 0; i < params.draws.GetSize(); ++i) {
        const SceneDrawItem& d = params.draws[i];
        if (d.skyMode != SceneSkyMode::None && d.textureLayer >= 0) {
            params.iblEnvironmentLayer = d.textureLayer;
            return;
        }
    }
}
```

### Step 3 — See IBL added in lit path

*PBR path mixes iblDiffuse into ambient and adds iblSpecular.*

```glsl
// shaders/scene.frag (lines 569–574)
    vec3 iblSpecular = sparkEvalSpecularIbl(sceneTextures, N, V, base, met, rough, occlusion);
    vec3 iblDiffuse = sparkEvalDiffuseIbl(sceneTextures, N, base, met, aoAmb * occlusion);
    if (ubo.iblParams.w > 0.5) {
        ambient = mix(ambient, iblDiffuse, mix(0.55, 0.92, met));
        ambient *= mix(1.0, 0.12, met);
    }
```

### Step 4 — Enable IBL on a metal sphere

```cpp
params.iblParams = {layerIndex, 1.0f, 0.f, 1.0f}; // w > 0.5 enables
```

Place a chrome `MaterialComponent` (metallic ≈ 1, roughness ≈ 0.05) under the same sky equirect.

### Step 5 — Toggle iblParams.w off

Compare metal sphere — should fall back to hemispheric ambient only. Confirms gate works before tuning intensity.

### Step 6 — Milestone — sky and IBL match

With HDR sky texture, rotating camera should update **both** visible background and specular highlights on metals consistently.

## 5. How This Connects to the Rest of Your Engine

**Sky Rendering Modes** registers the equirect layer sky IBL reuses. **MaterialComponent** metallic/roughness
feeds split-sum specular. Part **VI** ambient UBO fields become procedural env when no texture is bound.

## 6. Pitfalls and Debugging

- **sRGB environment without linearize** — dull metals; Spark uses sparkSrgbToLinear on sample.
- **IBL on + full ambient** — blown HDR; scene.frag mixes down ambient when IBL active.
- **Layer index mismatch** — iblParams.x must match sceneTextures index; rely on ResolveIblEnvironmentLayer.
- **No environment texture** — procedural env still works; verify gradient matches sky tint art direction.

## 7. Exercises

1. Disable IBL on one draw item only — not supported per-draw yet; toggle global `iblParams.w` and note effect.
2. Visualize reflect vector R as color on a sphere.
3. Document which ambient UBO terms `scene.frag` reduces when IBL is enabled.
4. Change sky equirect tint — confirm metals and background shift together when layer shared.
5. Set `iblIntensity` to 0.5 — compare specular highlight strength on chrome sphere.

## Summary

ibl.glsl provides diffuse and specular environment sampling; submit resolves iblEnvironmentLayer from sky texture for consistent metal reflections.

---

Next: [MaterialComponent](05-material-component.md).
