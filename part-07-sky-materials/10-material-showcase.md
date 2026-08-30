# Material Showcase Demo

> **Build-Along Milestone:** Part VII complete — showcase sphere validates maps, scalars, submit, and shading under fixed lights.

**You will learn:**

- Run MaterialShowcase3DDemo and toggle maps/scalars with keyboard.
- Use the demo as regression after scene.frag or submit edits.
- Explain how procedural textures keep the demo self-contained.

## 1. The Problem We Are Solving

Shader bugs hide in full game levels. A **material showcase** isolates one shaded sphere under fixed lights with
live toggles for every map Part VII introduced — the fastest feedback loop before merging BRDF changes.

Finish Part VII by running MaterialShowcase3DDemo and documenting your baseline screenshot set.

## 1b. Concept Deep Dive — MaterialShowcase3DDemo as Regression Harness

### Demo purpose

```cpp
// include/spark/demo/MaterialShowcase3DDemo.hpp
/**
 * Single LitPbr sphere whose material is edited at runtime (texture toggles, metallic/roughness,
 * emissive presets). Directional + point + spot lights. F1 fly, ESC menu.
 */
```

Uses `Mesh::CreateSkySphere` (high segment count) as showcase geometry — smooth specular for normal map validation.

### ApplyMaterialState — runtime toggles

```cpp
// src/spark/demo/MaterialShowcase3DDemo.cpp (lines 169–187)
void MaterialShowcase3DDemo::ApplyMaterialState() {
    showcaseMaterial->SetShadingModel(SceneShadingModel::LitPbr);
    showcaseMaterial->SetBaseColorTexture(useBaseMap ? baseColorTex : SharedPtr<Texture2D>{});
    showcaseMaterial->SetNormalTexture(useNormalMap ? normalTex : SharedPtr<Texture2D>{});
    showcaseMaterial->SetMetallicRoughnessTexture(useOrmMap ? ormTex : SharedPtr<Texture2D>{});
    showcaseMaterial->SetMetallic(metallic);
    showcaseMaterial->SetRoughness(roughness);
    showcaseMaterial->SetTint(TintForPreset(tintPreset));
    ApplyEmissiveToMaterial(*showcaseMaterial, emissivePreset, useEmissiveMap, emissiveIntensity, emissiveTex);
}
```

Every toggle calls setters → `MeshDirty` → next frame submit re-registers texture layers.

### Key bindings for validation

| Key | Effect | Chapter validated |
|-----|--------|-------------------|
| **1** | Toggle base color map | Part IV texture binding |
| **2** | Toggle normal map | Chapter 07 normal mapping |
| **3** | Toggle ORM map | Chapter 08 ORM |
| **Q/E** | Metallic ± | PBR scalars |
| **R/F** | Roughness ± | PBR + IBL specular blur |
| **Emissive presets** | HDR headroom | Part X tonemap |

### What to capture for baseline screenshots

1. LitPbr sphere, base map only, single directional light.
2. Normal map on — specular ripples visible.
3. ORM on — chrome vs matte regions from B/G channels.
4. Emissive intensity > 1 — HDR bloom headroom before tonemap.
5. Side-by-side with `ToonShadingDemo` for shading model comparison.

## 2. Core Concepts

### Interactive material sandbox

`MaterialShowcase3DDemo` spawns a PBR sphere + ground with point/spot lights — keyboard toggles base, normal, emissive maps and adjusts metallic/roughness/emissive scalars live.


### Procedural fallback textures

When brick asset missing, checkerboard base; normal and emissive maps generated on CPU (BuildTangentNormalRipples, BuildEmissiveLayerMap) — demo runs without external files.


### Fixed light rig

Directional fill from submit defaults plus authored point/spot — shader edits isolate to material changes, not moving sun.


### Regression harness

After every `scene.frag` edit, run showcase and verify: specular footprint, normal ripples, emissive HDR, ORM response — faster than full game level.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### One MaterialComponent target

**Concept:** Single showcase sphere

**Decision:** One MaterialComponent target

**Why:** ApplyMaterialState centralizes all toggles — keys 1–5 and Q/E/R/F/Z/X map to parameters.

**In Spark:** `MaterialShowcase3DDemo::ApplyMaterialState`

### High-segment sphere

**Concept:** CreateSkySphere for preview mesh

**Decision:** High-segment sphere

**Why:** Shows normal and specular variation across curved surface — better than cube for BRDF debug.

**In Spark:** `MaterialShowcase3DDemo::Load` sphereMesh

### On-screen key legend

**Concept:** TextOverlay help HUD

**Decision:** On-screen key legend

**Why:** Documents toggles without external docs — pattern from other Spark shell demos.

**In Spark:** helpText TextOverlayComponent

### FillStandardLitSceneFromWorld

**Concept:** Standard submit in Render()

**Decision:** FillStandardLitSceneFromWorld

**Why:** Same path production scenes use — showcase validates real gather, not a fork.

**In Spark:** `MaterialShowcase3DDemo` Render path

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/demo/MaterialShowcase3DDemo.cpp`
- `include/spark/demo/MaterialShowcase3DDemo.hpp`
- `src/spark/scene/SceneSubmit.cpp`

### Step 1 — Study Load() spawn

*Ground + showcase sphere + procedural textures + MaterialComponent.*

```cpp
// src/spark/demo/MaterialShowcase3DDemo.cpp (lines 268–312)
void MaterialShowcase3DDemo::Load(GameWorld& w, IEngineContext& /*context*/) {
    roots.Clear();
    roots.Reserve(8);

    sphereMesh = MakeShared<Mesh>(Utf8String("MatShowSphere"));
    *sphereMesh = Mesh::CreateSkySphere(0.72F, 18, 32);
    w.RegisterMesh(sphereMesh, "spark/demo/matshow_sphere");

    groundMesh = MakeShared<Mesh>(Utf8String("MatShowGround"));
    *groundMesh = Mesh::CreateGroundPlane(kSceneGroundHalfExtent);
    w.RegisterMesh(groundMesh, "spark/demo/matshow_ground");

    baseColorTex = MakeShared<Texture2D>(Utf8String("MatShowBase"));
    if (!DemoAssets::TryLoadBrickTexture(*baseColorTex)) {
        *baseColorTex = Texture2D::CreateCheckerboard(128, 16, Vector3{0.92F, 0.35F, 0.28F}, Vector3{0.18F, 0.22F, 0.55F});
    }
    w.RegisterTexture(baseColorTex, "spark/demo/matshow_base");
    normalTex = MakeShared<Texture2D>(Utf8String("MatShowNormal"));
    *normalTex = BuildTangentNormalRipples(128, Utf8String("MatShowNormalData"));
    w.RegisterTexture(normalTex, "spark/demo/matshow_normal");
    emissiveTex = MakeShared<Texture2D>(Utf8String("MatShowEmissive"));
    *emissiveTex = BuildEmissiveLayerMap(128, Utf8String("MatShowEmissiveData"));
    w.RegisterTexture(emissiveTex, "spark/demo/matshow_emissive");

    GameObject* ground = w.CreateGameObject();
    ground->GetName() = Utf8String("Ground");
    ground->AddComponent<TransformComponent>();
    ground->AddComponent<MeshComponent>(groundMesh, SceneMeshSlot::GroundPlane, Vector3{0.38F, 0.4F, 0.42F});
    if (MaterialComponent* m = ground->AddComponent<MaterialComponent>()) {
        m->SetShadingModel(SceneShadingModel::LitPbr);
        m->SetMetallic(0.05F);
        m->SetRoughness(0.88F);
    }
    roots.PushBack(ground);

    showcaseSphere = w.CreateGameObject();
    showcaseSphere->GetName() = Utf8String("MatShowcaseSphere");
    showcaseTransform = showcaseSphere->AddComponent<TransformComponent>();
    if (showcaseTransform != nullptr) {
        showcaseTransform->SetTranslation({0.0F, 0.72F, 0.0F});
    }
    showcaseMesh = showcaseSphere->AddComponent<MeshComponent>(sphereMesh, TintForPreset(tintPreset));
    showcaseMaterial = showcaseSphere->AddComponent<MaterialComponent>();
    ApplyMaterialState();
    roots.PushBack(showcaseSphere);
```

### Step 2 — Read input → material mapping

*ApplyMaterialState — toggles maps, tint, metallic, roughness, emissive.*

```cpp
// src/spark/demo/MaterialShowcase3DDemo.cpp (lines 168–186)
void MaterialShowcase3DDemo::ApplyMaterialState() {
    if (showcaseMaterial == nullptr) {
        return;
    }

    showcaseMaterial->SetShadingModel(SceneShadingModel::LitPbr);
    showcaseMaterial->SetBaseColorTexture(useBaseMap ? baseColorTex : SharedPtr<Texture2D>{});
    showcaseMaterial->SetNormalTexture(useNormalMap ? normalTex : SharedPtr<Texture2D>{});
    showcaseMaterial->SetMetallic(metallic);
    showcaseMaterial->SetRoughness(roughness);

    const Vector3 tint = TintForPreset(tintPreset);
    showcaseMaterial->SetTint(tint);
    if (showcaseMesh != nullptr) {
        showcaseMesh->SetAlbedo(tint);
    }

    ApplyEmissiveToMaterial(*showcaseMaterial, emissivePreset, useEmissiveMap, emissiveIntensity, emissiveTex);
}
```


*HandleMaterialInput — keys 1–5, Q/E, R/F, Z/X emissive.*

```cpp
// src/spark/demo/MaterialShowcase3DDemo.cpp (lines 226–265)
    if (in.IsKeyPressedThisFrame(GLFW_KEY_1)) {
        useBaseMap = !useBaseMap;
    }
    if (in.IsKeyPressedThisFrame(GLFW_KEY_2)) {
        useNormalMap = !useNormalMap;
    }
    if (in.IsKeyPressedThisFrame(GLFW_KEY_3)) {
        useEmissiveMap = !useEmissiveMap;
    }
    if (in.IsKeyPressedThisFrame(GLFW_KEY_4)) {
        emissivePreset = (emissivePreset + 1) % 4;
        if (emissivePreset == 1) {
            emissiveIntensity = 4.5F;
        } else if (emissivePreset == 2) {
            emissiveIntensity = 4.0F;
        } else if (emissivePreset == 3) {
            emissiveIntensity = 3.2F;
        } else {
            emissiveIntensity = 0.0F;
        }
    }
    if (in.IsKeyPressedThisFrame(GLFW_KEY_5)) {
        tintPreset = (tintPreset + 1) % 4;
    }
    if (in.IsKeyPressedThisFrame(GLFW_KEY_0)) {
        useBaseMap = true;
        useNormalMap = false;
        useEmissiveMap = false;
        emissivePreset = 0;
        tintPreset = 0;
        metallic = 0.22F;
        roughness = 0.48F;
        emissiveIntensity = 0.0F;
    }

    adjustScalar(metallic, GLFW_KEY_Q, GLFW_KEY_E);
    adjustScalar(roughness, GLFW_KEY_R, GLFW_KEY_F);
    adjustEmissive();

    ApplyMaterialState();
```

### Step 3 — Launch demo

Run Spark shell with MaterialShowcase3DDemo. Press **1/2/3** for base/normal/emissive maps, **Q/E** metallic, **R/F** roughness, **Z/X** emissive intensity.

### Step 4 — Break-test normal mapping

Temporarily force `normalMapLayer = -1` in shader — rerun demo; ripple specular should vanish (regression signal).

### Step 5 — Capture HDR emissive row

Preset **4** with emissive map — linear values > 1 before tonemap; confirms emissive path not clamped early in shader.

### Step 6 — Milestone — Part VII capstone

Sky from prior chapters optional backdrop; showcase sphere demonstrates PBR maps, toggles, and submit path end-to-end. Screenshot baseline for future shader diffs.

## 5. How This Connects to the Rest of Your Engine

Caps **Material Submit**, **Normal Mapping**, **ORM Textures**, and **Shading Models**. Part **X** tonemap
processes emissive HDR from this demo. Part **IX** can add shadow-casting columns to the same layout later.

## 6. Pitfalls and Debugging

- **Moving camera/lights during compare** — invalidates A/B screenshots; use fixed camera snap.
- **Missing tonemap** — emissive clips white; distinguish from shader bug.
- **Editing ApplyMaterialState without updating HUD** — keys undocumented; keep help text in sync.
- **Forking custom submit in demo** — invalidates regression value; always use FillStandardLitSceneFromWorld.

## 7. Exercises

1. Record key map table (1–5, Q/E, R/F, Z/X) in your engine notes.
2. Add a second sphere with `ToonCel` shading beside showcase — compare under same lights.
3. Capture RenderDoc on chrome preset — inspect ORM and normal layers bound.
4. Break `FindOrAddSceneTexture` dedup — predict layer count explosion.
5. Screenshot baseline set: base only, +normal, +emissive HDR, chrome ORM.

## Summary

`MaterialShowcase3DDemo` is an interactive PBR sandbox with procedural texture fallbacks — run it after every material or shader change in Part VII.

---

Continue to [Part VIII — Particles](../part-08-particles/index.md).
