# WorldRendering and Lit Scene Submit

> **Build-Along Milestone:** You replace a manual `FillStandardLitSceneFromWorld` call with `world.Rendering().SubmitLit` and optional frustum culling via `LitSceneSubmitOptions`.

**You will learn:**

- Use `GameWorld::Rendering()` as the blessed lit-submit API.
- Bundle sun, ambient, particles, and cull scene in `LitSceneSubmitOptions`.
- Understand how `LitSceneSubmitter` fills `SceneRenderParams` internally.
- Know when to use `FillLit` vs `SubmitLit` vs `SubmitLitWithCamera`.

## 1. The Problem We Are Solving

Chapter 10 taught ECS → `SceneRenderParams` with free functions in `SceneSubmit.hpp`. Spark refactored that path: demos now call **`world.Rendering()`**, a thin facet on `GameWorld` that forwards to `LitScene::Fill` / `LitScene::Submit` and the internal **`LitSceneSubmitter`**.

Long positional argument lists (sun direction, ambient, particle basis, sprite sort mode) are error-prone. **`LitSceneSubmitOptions`** groups per-frame inputs. Frustum culling hooks through `WithSceneForCulling(&scene)` instead of ad-hoc flags scattered in demos.

Build along: open any current 3D demo `OnRender`, trace `world.Rendering().SubmitLit` or `SubmitLitWithCamera`, and compare to the legacy helper name still in older book excerpts.

## 1b. Concept Deep Dive — Blessed API vs Legacy Helpers

### WorldRendering facet

```cpp
// include/spark/scene/core/world/WorldRendering.hpp
class WorldRendering final : public WorldView {
public:
    void FillLit(IEngineContext& context, const Matrix4& viewProjection,
            const Vector3& cameraPositionWorld, const LitSceneSubmitOptions& options,
            SceneRenderParams& outParams);

    void SubmitLit(IEngineContext& context, const Matrix4& viewProjection,
            const Vector3& cameraPositionWorld, const LitSceneSubmitOptions& options);

    [[nodiscard]] bool SubmitLitWithCamera(IEngineContext& context, const LitSceneSubmitOptions& options);

    [[nodiscard]] bool ResolveCamera(float fbW, float fbH, SceneRenderParams& outParams) const noexcept;
};
```

Access: `gameWorld.Rendering().SubmitLit(...)`. Same pattern as other world facets — zero extra storage on `GameWorld`.

### Typical demo OnRender (current Spark)

```cpp
LitSceneSubmitOptions opts{};
opts.WithDirectionalLight(sunDir, sunColor, sunIntensity)
    .WithAmbient(ambientColor)
    .WithSpriteSortMode(SceneSpriteSortMode::SortOrderThenWorldY);
if (Scene* scene = context.TryGetScene()) {
    opts.WithSceneForCulling(scene);
}
world.Rendering().SubmitLit(context, viewProjection, cameraPositionWorld, opts);
```

`SubmitLit` fills `SceneRenderParams` and calls `context.SetSceneRenderParams` — you usually **do not** call `SetSceneRenderParams` yourself after `SubmitLit`.

### FillLit — patch snapshot before GPU

Use when editor tools or custom passes tweak `SceneRenderParams` after gather but before record:

```cpp
SceneRenderParams params{};
world.Rendering().FillLit(context, viewProjection, cameraPositionWorld, opts, params);
params.iblIntensity = editorOverride;  // custom patch
context.SetSceneRenderParams(params);
```

### LitSceneSubmitter — what actually walks ECS

Internal orchestration (you rarely call directly):

```text
LitSceneSubmitter::Fill
  → ResetSceneParams (clears draws, textures, instancing arrays, …)
  → lighting resolve + camera fields
  → CollectMeshDraws (rigid + skinned collectors, optional frustum cull)
  → CollectWater / sprites / tilemaps / particles / foliage / instancing
  → SceneDrawListSorter (opaque sort + transparent partition)
  → ResolveIblEnvironmentLayer
```

Partitioning transparent draws moved to **`SceneDrawListSorter`** (replaces older standalone partition helpers).

### Legacy compatibility (still in tree)

| Legacy | Status |
|--------|--------|
| `FillStandardLitSceneFromWorld` | Delegates to `LitSceneSubmitter::Fill` — prefer `FillLit` |
| `SubmitStandardLitSceneFromWorld` | Fill + `SetSceneRenderParams` — prefer `SubmitLit` |
| `LitScene::Builder` | Deprecated fluent style |

New game code should cite **`docs/LIT_RENDER_API.md`** in Spark, not add new call sites on `SceneSubmit::*` free functions.

### LitSceneSubmitOptions fields

```cpp
// include/spark/scene/submit/LitSceneSubmitOptions.hpp (excerpt)
struct LitSceneSubmitOptions {
    Vector3 lightDirectionWorld;
    Vector3 lightColor;
    float lightIntensity;
    Vector3 ambientColor;
    bool enableParticles;
    Vector3 particleCameraRight, particleCameraUp;
    float sceneTimeSeconds;
    SceneSpriteSortMode spriteSortMode;
    const Scene* sceneForCulling;  // null = no mesh frustum cull
};
```

Fluent `With*` methods return `*this` for chaining.

## 2. Core Concepts

### SubmitLitWithCamera

When the main camera lives on ECS (`CameraComponent` / `Camera2DComponent`), `SubmitLitWithCamera` resolves view/projection and camera position, then submits. See Part V Chapter 8; implementation is `LitScene::SubmitWithCamera`.

### sceneForCulling

Pass `context.TryGetScene()` when the active level uses `Scene` spatial policy (BVH, quadtree, etc.). Without it, lit submit still runs but may visit every rigid drawable — fine for small scenes, costly for open worlds (Part XII).

### Relationship to manual learning path

Chapter 10 manual `SceneDrawItem` assembly remains valid for **understanding** the snapshot. Production demos should converge on **`world.Rendering().SubmitLit`** once the contract is clear.

## 3. Design Decisions

**Facet on GameWorld** — keeps rendering entry discoverable (`world.Rendering()`, `world.LoadMesh()`, …) without bloating `GameWorld` method count.

**Options struct vs positional args** — adding a new per-frame flag (e.g. foliage wind time) extends `LitSceneSubmitOptions` once instead of breaking every demo signature.

**Single submitter class** — one place to add mesh instancing, LOD resolve, and layer sort (Parts VI and XI).

## 4. Implementation Walkthrough

### Spark files

- `include/spark/scene/core/world/WorldRendering.hpp`
- `include/spark/scene/submit/LitSceneSubmitter.hpp`
- `src/spark/scene/submit/LitSceneSubmitter.cpp`
- `include/spark/scene/submit/LitSceneSubmitOptions.hpp`
- `docs/LIT_RENDER_API.md`

### Milestone check

1. Grep a demo for `Rendering().SubmitLit`.
2. Breakpoint `LitSceneSubmitter::Fill` and confirm `params.draws` size matches visible entities.
3. Toggle `WithSceneForCulling` and log draw count on TerrainDemo when spinning camera away.

## 5. How This Connects to the Rest of Your Engine

Part IV lighting profiles still merge inside submit. Part VI skinned draws use `SkinnedDrawableSubmitCollector`. Part VII materials apply in mesh collectors. Part XI sprite sort mode is set on `LitSceneSubmitOptions`. Part XII frustum policy requires `WithSceneForCulling`. Part XVI profiling should time `LitSceneSubmitter::Fill`, not individual legacy helpers.

## 6. Pitfalls and Debugging

- **Double SetSceneRenderParams** — `SubmitLit` already sets; do not call again unless you used `FillLit` only.
- **Null sceneForCulling** — expecting BVH cull but never passed `Scene*` — draw count stays at full scene size.
- **Stale book snippets** — `FillStandardLitSceneFromWorld` in older chapters maps to the same pipeline; update your code to `world.Rendering()`.

## Summary

Prefer **`world.Rendering().SubmitLit`** with **`LitSceneSubmitOptions`**; internal **`LitSceneSubmitter`** fills `SceneRenderParams`; legacy `SceneSubmit::*` helpers remain compatibility wrappers.

---

Previous: [Submitting a Scene from Code](10-submitting-scene.md). Next: [Part III — Shader Essentials](../part-03-shaders/index.md).