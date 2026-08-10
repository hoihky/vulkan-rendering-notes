# 2D Game Pipeline

> **Build-Along Milestone:** TilemapShowcase2DDemo or Platformer2DDemo runs end-to-end with HUD, animation, and correct pass order.

**You will learn:**

- Assemble tilemap, sprites, animation, layers, particles, and screen UI in one demo loop.
- Submit a complete 2D snapshot via SubmitStandardLitSceneFromWorld without Vulkan in gameplay.
- Use Spark showcase and platformer demos as integration regression targets.

## 1. The Problem We Are Solving

Individual Part XI features work in isolation until you **integrate** them: one update order, one submit function, one HDR→tonemap→UI present chain. This chapter is the **integration milestone** — not new GPU code, but the production-shaped loop that ships in Spark's 2D demos.

Build along: run `TilemapShowcase2DDemo`, trace its `Render` submit call, then replicate the loop in your engine with your own map and player entity.
## 2. Core Concepts

### Integration snapshot model

The **2D game pipeline** is not a new renderer — it is the disciplined assembly of prior chapters into one `SceneRenderParams` fill and the existing HDR frame graph. One submit call gathers:

```text
viewProj + camera → frustum cull meshes (optional)
sprites (culled, sorted)
tilemaps (culled, layer sorted)
particles (optional)
screen UI rects/text
```

Gameplay **Update** runs physics, `SpriteAnimatorComponent`, FSM, AI; **Render/Submit** only reads components. Vulkan appears exclusively inside `PresentFrame`. Violating this boundary (calling `vkCmdDraw` from `OnUpdate`) reintroduces the coupling Part I eliminated.

SubmitStandardLitSceneFromWorld signature carries viewProj, light, fog, particle basis, sprite sort mode — 2D demos pass ortho matrix from camera helper.

The platformer loop fills **one SceneRenderParams snapshot** per frame: camera, lights, meshes, sprites, tilemaps, particles, screen UI. Gameplay writes components; submit flattens; renderer consumes. No subsystem calls `vkCmdDraw` directly. Integration bugs show up as empty arrays — log sizes at submit boundary before blaming GPU passes. TilemapShowcase2DDemo and Platformer2DDemo are reference loops to diff when your integration diverges. Treat snapshot immutability as a contract: record reads `pendingScene` copied at frame start, not live ECS.

Version your snapshot struct layout when adding fields — mismatched sizes between gameplay and renderer DLL boundaries cause subtle corruption in modular engines. A single `SubmitScene(GameWorld&, SceneRenderParams&)` function signature keeps all flatten logic discoverable for code review. Integration tests can assert snapshot counts after scripted spawn: N coins, M tiles, one player sprite with expected layer.


### Demo reference loops

Spark ships vertical slices:

| Demo | Highlights |
|------|------------|
| `TilemapShowcase2DDemo` | Layers, autotile, pathfinding HUD, Y-sort submit |
| `Platformer2DDemo` | Combat, bullets, health HUD, Kenney atlases |
| `RenderLayers2DDemo` | Lighting modes + layer overlap |

Both call `SubmitStandardLitSceneFromWorld` (or WithCamera variant) with 2D orthographic `viewProj`, particle basis vectors, and `SceneSpriteSortMode`. Use them as **regression targets** after engine changes — if showcase breaks, bisect submit vs renderer Record.

Shell demo EnterTilemapShowcase2DDemo loads entities then Simulate/Render split — follow same lifecycle in your game class.`TilemapShowcase2DDemo` stresses **tile layers, pathfinding, and ortho camera** without physics complexity. `Platformer2DDemo` adds **2D physics, character FSM, coins, HUD, and particles**. Read both `Render` methods side by side: showcase uses manual camera pan; platformer uses follow camera and health HUD submit. Copy the slimmer loop first, then add physics-driven sprite animation. Demos enable feature flags (`enableParticles`, shadow toggles) showing how to degrade gracefully on weak GPUs.

Diff Platformer2DDemo vs TilemapShowcase2DDemo `Render` functions line-by-line — shared submit helper extraction is your engine refactor target after reading both. Feature flags in demo params show minimal vs full integration — start from minimal and enable particles, shadows, and ImGui incrementally. Record video of both demos as golden reference when your fork claims Part XI parity.


### Camera and coordinate conventions

Side-scrollers use orthographic projection built from framebuffer aspect and world units per pixel. `TilemapShowcase2DDemo::Render` computes `camera.

ViewProjection(fbW, fbH)` and billboard basis for particles. Player follow reads `TransformComponent` each Update — camera entity or scripted offset — then submit uses the same viewProj for sprite cull and tile chunk visibility.

Align **physics** (Part XIV) and **render** tile size constants — `kTileWorld` in showcase vs `TilemapComponent::GetTileWorldSize()` must match or collision and draw misalign by subpixel gaps.

TilemapShowcase passes SortOrderThenWorldY — platformer may use same for character vs ground Y overlap. Mismatch tile world size vs physics grid causes float hover.2D games use **orthographic viewProj** with Y-up world space matching physics grids. Z is still meaningful for depth between sprites and optional 3D background meshes. Near/far planes must include all sprite Z offsets and background terrain if hybrid. Camera follow lerps position in gameplay; submit reads final camera matrices from `CameraComponent`. Mismatch between cull frustum and submit matrices pops tiles at screen edges — use the same viewProj for `SceneSpriteTileCull` and draw submission.

Orthographic size defines half-height in world units — tie to tile size so integer tile boundaries align with pixel-perfect art at target resolution. Z-fighting between coplanar sprites on same layer needs slight Z offset or sort keys — depth alone insufficient for alpha stacks. Hybrid 3D terrain plus 2D sprites may use two cameras — document which viewProj feeds submit to avoid culling with wrong matrix.


### Cross-part feature flags

Submit accepts toggles: `enableParticles`, shadow flags, SSAO on 3D backgrounds in hybrid scenes. Empty sprite/tile arrays skip record work in composite pass — early-out when counts zero. Platformer enables explosions (particles) and disables heavy 3D mesh lists; showcase enables tile animation time on water layer.

When profiling, disable particles or tile layers independently without recompile by guarding submit walks or clearing component lists in demo modes.

enableParticles false skips emitter collection — Platformer2D enables for explosions. Empty sprite array still valid submit — composite2D may early-out.

Part XI integrates flags from earlier parts: **HDR** (X), **textures** (IV), **particles** (VIII), **shadows on 3D** (IX), **frustum cull** (XII ch4). Centralize toggles in demo params or ImGui rather than scattering `#ifdef` in submit code. Disabling particles should skip both submit collection and record — not just GPU skip. Document default flag combinations for "2D only" vs "2.5D with terrain" test scenes so QA reproduces artist setups.

Central `DemoParams` or `EngineToggles` struct prevents fifty bools scattered in globals — mirror Spark demo header patterns. Disabling HDR while leaving sprite sRGB decode on still works but changes artist tuning baseline — document flag combinations for QA test matrices. Frustum cull disable flag helps isolate popping bugs vs sort bugs during integration triage.


### Collision vs render separation

Tile **render** comes from visible tile instances; **collision** from `TilemapGameplayGrid` or physics shapes derived from tile collision flags (`TileCollisionShape::FullCell`). Player position comes from `TransformComponent` + `Rigidbody2DComponent` — single source on player entity. Coin pickup hides `SpriteComponent` or disables object — does not remove tile layer.

Regenerate autotile neighbors on cell edits via `TilemapAutotileComponent::RebuildIfNeeded` before submit that frame so visuals match grid state after destructible terrain (future gameplay).

Coin pickup disables entity or SpriteComponent — does not remove tile layer. Autotile rebuild on destructible terrain edit before submit.

Physics grids and render tilemaps **share map data** but not runtime pools. Collision updates on fixed timestep; render collects visible tiles on render timestep. Editing collision without sync shows foot sliding on invisible pits. Platformer2DDemo separates `TilemapGameplayGrid` from visual tile layers — follow that split. Sprite hitboxes often use AABB from transform scale, not alpha silhouette — document for designers. Render sort layers do not affect physics layers.

Physics timestep may differ from render rate — render interpolates transform for smooth motion while physics grid uses fixed step. Tile collision import should validate every solid tile maps to gameplay grid cell — visual-only tiles must not block movement. Coin pickup uses trigger collider separate from sprite render layer — gameplay events do not require render sort changes unless adding collection VFX on higher layer.


### Playtest and profile checklist

Before calling Part XI complete, verify:

```text
☐ Idle/run/jump anim on move (Ch 3)
☐ Lit player responds to sun (Ch 2)
☐ Tile layers sort behind/in front (Ch 5)
☐ HUD locked after resize (Ch 6)
☐ ImGui exposure optional (Ch 7)
☐ RenderDoc pass order: HDR 2D → tonemap → UI
☐ Visible tile/sprite counts drop off-screen (Ch 4 + Part XII cull)
```

Log visible tile pool size vs sprite count when panning — CPU submit should stay flat for static maps.

Log tilemapTiles size when panning — confirms cull. RenderDoc event order documents HDR 2D before tonemap before UI.

Profile **visible sprite count, visible tile count, stable sort ms, and SSBO upload size** each build week. Spikes in tile collection often mean broken chunk cull after camera refactor. Ten-minute playtest catches sort flicker and animation hitches from delta time spikes. Capture one RenderDoc frame per milestone verifying pass order: HDR composite before tonemap, HUD after. Checklist: spawn/death without sort artifacts, coin pickup layer bump, pause overlay alpha, ImGui exposure live tune.

Ten-minute soak test catches memory leaks in snapshot arrays if PushBack without Clear — verify Clear at submit start each frame. Tracy or similar markers on submit subphases (cull, sort, tile collect) localize regressions after integration merges. Milestone checklist printed in chapter milestone block should be ticked literally before declaring Part XI complete in your engine fork.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SubmitStandardLitSceneFromWorld entry

**Concept:** Integration snapshot model

**Decision:** SubmitStandardLitSceneFromWorld entry

**Why:** One function signature covers 2D and 3D demos — reduces duplicate submit code paths.

**In Spark:** `src/spark/scene/SceneSubmit.cpp` — sprite/tilemap blocks lines 329–371.

### Shell demo mode switching

**Concept:** Demo reference loops

**Decision:** Shell demo mode switching

**Why:** Isolated Load/Simulate/Render per demo — clean regression matrix.

**In Spark:** `SparkShellDemo.cpp` EnterTilemapShowcase2DDemo / EnterPlatformer2DDemo.

### ViewProjection from 2D camera helper

**Concept:** Camera and coordinate conventions

**Decision:** ViewProjection from 2D camera helper

**Why:** Centralizes ortho math — submit and cull share identical matrix.

**In Spark:** `TilemapShowcase2DDemo::Render` lines 647–677.

### enableParticles param

**Concept:** Cross-part feature flags

**Decision:** enableParticles param

**Why:** Platformer2D uses particles; pure tile showcase skips collection overhead.

**In Spark:** `SceneSubmit.hpp` FillStandardLitSceneFromWorld signature.

### Tile definitions own collision shape

**Concept:** Collision vs render separation

**Decision:** Tile definitions own collision shape

**Why:** Art pipeline marks walls vs decor — one tileset drives both physics and UVs.

**In Spark:** `TilemapShowcase2DDemo` tileset setup lines 76–88.

### Document milestone gates

**Concept:** Playtest and profile checklist

**Decision:** Document milestone gates

**Why:** Prevents shipping partial 2D stack with broken sort or wrong pass target.

**In Spark:** Part XI summary milestone list.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/demo/TilemapShowcase2DDemo.cpp`
- `src/spark/demo/Platformer2DDemo.cpp`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Run TilemapShowcase2DDemo end-to-end

Pathfind, layers, animated water.

### Step 2 — Read showcase Render submit

Ortho viewProj and sort mode.

*Render submit*

```cpp
// src/spark/demo/TilemapShowcase2DDemo.cpp (lines 647–677)
void TilemapShowcase2DDemo::Render(
        Spark::Scene& /*scene*/,
        Spark::GameWorld& world,
        Spark::IEngineContext& context) {
    int fbW = 0;
    int fbH = 0;
    context.GetFramebufferSize(fbW, fbH);
    if (fbW <= 0) {
        fbW = 1;
    }
    if (fbH <= 0) {
        fbH = 1;
    }
    const Spark::Matrix4 viewProj = camera.ViewProjection(static_cast<float>(fbW), static_cast<float>(fbH));
    Spark::Vector3 pr{};
    Spark::Vector3 pu{};
    camera.BillboardBasisWorld(pr, pu);
    Spark::SubmitStandardLitSceneFromWorld(
            world,
            context,
            viewProj,
            camera.position,
            Spark::Vector3{0.42F, 0.78F, 0.38F}.Normalized(),
            Spark::Vector3{0.94F, 0.97F, 1.0F},
            0.75F,
            Spark::Vector3{0.06F, 0.09F, 0.12F},
            false,
            pr,
            pu,
            0.0F,
            SceneSpriteSortMode::SortOrderThenWorldY);
```

### Step 3 — Compare Platformer2DDemo submit

WithCamera variant.

*Platformer Render*

```cpp
// src/spark/demo/Platformer2DDemo.cpp (lines 485–496)
void Platformer2DDemo::Render(Spark::Scene& /*scene*/, Spark::GameWorld& world, Spark::IEngineContext& context)
{
    (void)Spark::SubmitStandardLitSceneFromWorldWithCamera(
            world,
            context,
            Spark::Vector3{0.28F, 0.88F, 0.38F}.Normalized(),
            Spark::Vector3{1.0F, 1.0F, 1.0F},
            0.9F,
            Spark::Vector3{0.18F, 0.20F, 0.26F},
            false,
            sceneTime);
}
```

### Step 4 — Trace full submit sprite/tile path

Cull, sort, push 2D draws.

*sprite tile submit*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 329–371)
    const SceneSpriteTileCull spriteTileCull(viewProjection);
    const SceneTilemapSubmitter tilemapSubmitter{};

    world.ForEachActiveGameObject([&](GameObject* o) {
        if (o == nullptr) {
            return;
        }
        const SpriteComponent* sc = o->GetComponent<SpriteComponent>();
        if (sc == nullptr) {
            return;
        }
        if (params.sprites.GetSize() >= SceneRenderParams::MaxSprites) {
            return;
        }
        const Matrix4 spriteModel = o->GetWorldMatrix();
        if (!spriteTileCull.IsSpriteVisible(spriteModel)) {
            return;
        }
        SceneSpriteDraw sd{};
        sd.model = spriteModel;
        sd.tint = sc->GetTint();
        sd.uvRect = sc->GetUvRect();
        const ResolvedDrawableSort resolved = DrawableSortResolver::Resolve(*o, sc->GetSortOrder());
        sd.sortOrder = resolved.key.sortingOrder;
        sd.sortingLayerOrder = resolved.key.sortingLayerOrder;
        sd.sortWorldY = resolved.worldYAnchor;
        if (sc->GetTexture()) {
            sd.textureLayer = findOrAddTexture(sc->GetTexture());
        } else {
            sd.textureLayer = -1;
        }
        if (const SpriteLighting2DComponent* lit = o->GetComponent<SpriteLighting2DComponent>()) {
            sd.lightingMode = lit->GetMode();
            sd.lightingParam0 = lit->GetParam0();
            sd.lightingParam1 = lit->GetParam1();
        }
        sd.blendMode = SceneSubmitDetail::ResolveSpriteBlendMode(*o);
        params.sprites.PushBack(sd);
    });

    tilemapSubmitter.Submit(world, params, spriteTileCull, findOrAddTexture, SceneSubmitDetail::ResolveSpriteBlendMode);

    SceneSubmitDetail::StableSortSprites(params.sprites, params.spriteSortMode);
```

### Step 5 — Wire camera follow on player

Same viewProj for cull and submit.

### Step 6 — Coin pickup and particle burst

Part VIII integration test.

### Step 7 — Profile visible counts

Tile and sprite pool sizes.

### Step 8 — Milestone — full Part XI playthrough

End-to-end demo stable 10 minutes.

## 5. How This Connects to the Rest of Your Engine

Pipeline touches Parts **IV** (lights/textures), **V** (camera), **VIII** (particles), **IX** (shadows on 3D backgrounds), **X** (HDR/tonemap), **XII** (frustum cull), **XIV** (2D physics). Part **XIII** editor would edit TMX and spawn points serialized into these demos.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

Full integration touches IV (textures/lights), V (camera), VIII (particles), IX (3D shadows), X (HDR), XII (cull), XIV (physics). This chapter is the capstone checklist — no new GPU features, only orchestration discipline.

## 6. Pitfalls and Debugging

- **Vulkan from gameplay** — breaks architecture; keep GPU in renderer only.
- **Forgetting stable sprite sort after spawn** — new entities flicker alpha until sorted.
- **Tile collision misaligned** — verify tile world size matches physics grid constants.
- **Submit viewProj != cull matrix** — pop-in or over-cull at screen edges.
- **Skipping tonemap when judging look** — always evaluate final swapchain after post chain.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

Most integration bugs are empty snapshot arrays or stale sort after spawn — log at submit boundary. Never call Vulkan from gameplay update loops.

## 7. Exercises

1. Build minimal platformer: one tilemap layer, player with anim FSM, floor collision.
2. Add coin pickup hiding sprite + particle burst.
3. Profile tile visible count vs sprite count vs GPU time panning across map.
4. Checklist all Part XI milestones in one RenderDoc capture.
5. Trace `Platformer2DDemo::Render` submit call end-to-end.

## Summary

The 2D game pipeline fills one `SceneRenderParams` snapshot with culled/sorted tilemaps and sprites, optional particles, and screen UI, then uses the standard HDR frame graph through tonemap and present.

---

Continue to [Part XII — Terrain](../part-12-terrain/index.md).
