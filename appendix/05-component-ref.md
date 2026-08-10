---
title: Component Quick Reference
order: 5
---
# Component Quick Reference

> **Build-Along Milestone:** Document data path for any rendering component in three grep steps.

**You will learn:**

- Find rendering-related ECS components and submit consumers.
- Distinguish gameplay components from editor-only tags.
- Trace component → SceneRenderParams → GPU path in three greps.

## 1. The Problem We Are Solving

Spark's ECS has dozens of components — for rendering study, a **working subset** matters. This appendix lists transforms, meshes, materials, cameras, lights, sky, particles, terrain, sprites, tilemaps, and audio cues with **submit consumer** hints.

Use it when extending the engine: if a component lacks a SceneSubmit line, it will not appear on screen.


Appendix chapter 05 supports active development — bookmark it. When stuck mid-chapter on Parts XV–XVI, grep appendix first before re-reading entire prior part. Keep local notes column for your renamed types alongside Spark names.

Schedule 30-minute appendix refresh after any PR touching `SceneSubmit`, descriptor layouts, or `RecreateSwapchain`.


Complete the five-component trace lab once with RenderDoc paused on the skinned draw — future component bugs will take minutes instead of hours because the path muscle memory exists.

## 2. Core Concepts

### ECS vs snapshot

**Components** are authoring/simulation state on `GameObject`. **SceneRenderParams** is derived **each frame** — not stored on disk.

```text
TransformComponent  ─┐
MeshComponent       ─┼→ SceneSubmit → SceneDrawItem in params.draws
MaterialComponent   ─┘
```

Serialization (Part XIII) saves components; GPU never reads JSON — it reads params built at submit.

### Component domains

Spark organizes under `include/spark/ecs/components/`:

| Folder | Examples |
|--------|----------|
| rendering/ | Mesh, Material, Sprite, Tilemap, Terrain |
| lighting/ | PointLight, SpotLight, DirectionalLight |
| camera/ | Camera, Camera2D |
| physics/ | Rigidbody2D, BoxCollider3D |
| audio/ | SoundCue, AudioListener |
| animation/ | Animator, SpriteAnimator |

Grep `ComponentKind` enum for authoritative list — reference docs may lag code.

### Required component pairs

| Visible result | Required components |
|----------------|---------------------|
| Lit static mesh | Transform + Mesh + Material |
| Skinned character | Transform + SkinnedMesh + Material (+ Animator) |
| Sprite | Transform + Sprite (+ SortingGroup optional) |
| Tilemap layer | Transform + Tilemap |
| Point light | Transform + PointLight |

Submit skips incomplete entities — missing Material may use debug defaults with warning in dev builds.

### Submit gather pattern

`SceneSubmit.cpp` uses `ForEachActiveGameObject` + component getters — rigid sink, skinned walk, sprite/tilemap collectors. Frustum cull dispatches into sink when spatial policy enabled (Part XII).

Adding component = add `FillMyComponent*` gather + optional serializer registry entry (Part XIII).

### Editor-only vs runtime

Some components serialize for editor metadata but never reach SceneSubmit — distinguish via registry docs. Runtime render components always have grep hit in `SceneSubmit.cpp` or dedicated fill module.

### Cross-reference workflow

Bug: "entity invisible" — checklist:

```text
1. Component present on active GameObject?
2. Gather hook in SceneSubmit?
3. Field in SceneRenderParams array non-empty?
4. Pass record draws that item type?
5. Descriptor/texture ready?
```

Appendix ties steps 1–3; descriptor map step 5; params ref step 4.

### Skinned mesh component chain

Skinned draw requires `TransformComponent`, `SkinnedMeshComponent`, `MaterialComponent`, animated `jointPalette` filled before submit, and skin SSBO upload for current frame index. Missing palette → T-pose or validation error on skin SSBO range.

AnimatorComponent drives palette; submit reads matrices after animation update in Update order.

### Non-render components

`AiAgentComponent`, `SoundCueComponent`, `Rigidbody2DComponent` never appear in SceneRenderParams directly — they influence simulation that **later** fills params (transform, velocity) or parallel subsystems (audio). Grep negative: component not in SceneSubmit.cpp → not a rendering component.

### Light component mapping

`PointLightComponent` + Transform → `ScenePointLight` in params → cluster SSBO → scene.frag loop. `SpotLightComponent` adds cone angles — verify radians in component match shader expectation (full cone, not half-angle).

Directional sun often from params defaults or `DirectionalLightComponent` if present — grep submit for light direction source in your fork.

### Physics components

Collider and rigidbody components do not appear in SceneRenderParams — physics debug draw would be separate line overlay pass if added. Do not grep SceneSubmit for BoxCollider2D expecting draw item.

### Component gather reference table

| Component | Gather function | Params destination | Required pair |
|-----------|-----------------|-------------------|---------------|
| TransformComponent | All gathers | modelMatrix / position | Always |
| MeshComponent | GatherMeshes | draws[] | + Material |
| SkinnedMeshComponent | GatherMeshes | draws[] + skin offset | + Material |
| MaterialComponent | GatherMeshes | material layers | + Mesh |
| SpriteComponent | GatherSprites | sprites[] | Transform |
| SpriteAnimatorComponent | GatherSprites | UV rect via animator | Sprite |
| TilemapComponent | SceneTilemapSubmitter | tilemapTiles[] | Transform |
| PointLightComponent | GatherLights | pointLights[] | Transform |
| SpotLightComponent | GatherLights | spotLights[] | Transform |
| CameraComponent | TryFillSceneCamera | camera block | — |
| ParticleEmitterComponent | GatherParticles | particles[] | Transform |
| TerrainComponent | GatherMeshes | draws[] | Material |
| SoundCueComponent | ProcessSoundCues | (not params) | SoundSubsystem |
| AudioListenerComponent | ProcessAudioListeners | (not params) | — |
| AiAgentComponent | SimulateGameAi | (not params) | — |
| Rigidbody2DComponent | Physics simulate | (not params) | Physics |

**Trace exercise:** pick SkinnedMesh — grep `GatherMeshes` → `SceneDrawItem` fields → `RecordSceneOpaque` push constants → `scene.vert` skin SSBO fetch.

### Five-component trace lab

Hands-on appendix lab — complete in 45 minutes:

**Components:** Transform + SkinnedMesh + Material + Animator + PointLight on one character.

**Step 1 — Serializer:** grep component in scene JSON schema — confirm save/load round-trip in editor demo 10.

**Step 2 — Gather:** breakpoint or log in `GatherMeshes` — confirm one SceneDrawItem with skinPaletteOffset >= 0.

**Step 3 — Params:** inspect params.draws[0] fields — modelMatrix, textureLayer, flags shadow cast.

**Step 4 — GPU:** RenderDoc capture — one draw, binding 2 SkinSSBO non-empty, binding 1 texture layer matches material albedo.

**Step 5 — Light:** params.pointLights[0] position matches light entity transform; cluster SSBO receives upload.

**Failure triage:**

| Missing | Check |
|---------|-------|
| No draw | Mesh gather skip — missing Material |
| T-pose | Animator not updating skin palette |
| Black mesh | textureLayer -1 or upload pending |
| No light | Light outside cluster grid or not gathered |

Write trace results in notebook — same format for every rendering bug during production.

### Component serialization vs runtime gather

Part XIII saves components to scene JSON — not all components serialize equally:

| Component | Serialized | Gathered | Notes |
|-----------|------------|----------|-------|
| TransformComponent | yes | yes | Authority for pose |
| MeshComponent | path ref | yes | Reload mesh from path |
| MaterialComponent | asset refs | yes | Texture paths |
| SkinnedMeshComponent | path + skin | yes | Palette runtime |
| SpriteComponent | atlas + rect | yes | UV from animator |
| TilemapComponent | TMX path | yes | Reimport on load |
| SoundCueComponent | clip path | no (audio subsystem) | ProcessSoundCues |
| AiAgentComponent | partial | no (AI sim) | Blackboard runtime |
| Rigidbody2DComponent | yes | no (physics) | Simulate2D |

Editor places components; runtime gather builds params — **desync** if editor shows mesh but gather skips disabled flag. Grep `enabled` checks in gather functions when "invisible in play mode" bugs appear.

Adding custom component: implement serializer (Part XIII), gather (SceneSubmit), and document in this table same PR.

### Subsystem parallel paths diagram

Not everything flows through SceneRenderParams — parallel CPU subsystems:

```text
                    ┌─ SoundSubsystem → SoundEngine → PumpFrame
GameWorld / ECS ────┼─ PhysicsWorld2D → Simulate2D
                    ├─ AiSubsystem → SimulateGameAi
                    └─ SceneSubmit → SceneRenderParams → VulkanRenderer
```

Audio, AI, physics mutate world state **before** submit snapshot. Submit reads final poses for frame — order in Engine Update matters.

C# interop must call exports for each parallel path — not only render-related APIs. Script that moves Transform but skips physics step sees floating rigidbodies until native sync.

When debugging "component exists but no effect," first classify: rendering path (params), physics path, audio path, or AI path — appendix component table column "Params destination" vs "(not params)" answers immediately.

### Editor-only vs runtime components

Some components exist for editor gizmo / debug — skip gather in shipping:

| Component | Editor | Runtime gather |
|-----------|--------|----------------|
| SelectionGizmoComponent | yes | skipped DEVELOPER_TOOLS |
| Collider debug draw | optional | physics debug flag |
| ImGuiPanelComponent | yes | never in params |

`#ifdef SPARK_DEVELOPER_TOOLS` wraps gather branches — retail build must not reference editor components or link editor translation units.

When porting editor component to runtime (e.g., trigger volume visualization), add explicit gather to debug pass — not opaque scene pass.

Component `enabled` flag default true — disable instead of remove for seasonal props. Gather checks enabled before append to params arrays.

### Physics vs render transform sync

`Rigidbody2DComponent` simulation writes back to `TransformComponent` — single authority rule:

```text
Physics step → updates Transform
SceneSubmit → reads Transform for modelMatrix
```

Gameplay code should not write Transform and velocity inconsistently same frame — platformer sets velocity, physics integrates, transform follows. Direct Transform teleport must wake rigidbody or reset velocity to avoid impulse explosion next step.

3D mesh with kinematic motion bypasses physics — steering chapter writes Transform directly; do not attach Rigidbody2D on same entity.

Desync symptom: mesh floating offset from collider debug draw — grep who last wrote Transform before submit.

### Component Kind enum grep workflow

Authoritative component list:

```bash
rg 'enum class ComponentKind' include/spark/ecs/
rg 'case ComponentKind::' src/spark/ecene/
```

Each Kind maps to serializer, optional gather, optional interop in SparkInteropComponents.cpp — three grep hits expected for rendering components.

Adding component requires:

```text
1. ComponentKind value
2. C++ component class header
3. Serializer in SceneDocumentIO
4. Gather in SceneSubmit (if renders)
5. Optional SparkComponentKind interop
6. Row in this appendix table
```

Missing step 4 → invisible in play mode with no compile error — most common student fork bug.

### Transform hierarchy and world matrix

`TransformComponent` world matrix combines parent chain:

```text
world = parentWorld * local
```

Gather uses world matrix for modelMatrix — local only incorrect for child props parented to moving platform. glTF import sets parent links; runtime spawn must `SetParent` explicitly.

Scale zero on axis breaks normal matrix for lighting — guard in gather or assert in debug when MeshComponent present. Non-uniform scale on skinned mesh may need normal recompute — Spark baseline assumes uniform or small skew for learning assets.

Rotate parent 90° — child collider and mesh both rotate if physics reads same Transform after step — verify debug draw uses world space.

### Material and mesh component interaction table

Rendering requires **consistent material/mesh pairing** — gather skips or uses defaults when pairing breaks:

| Mesh type | Material fields used | Skip condition |
|-----------|---------------------|----------------|
| Static MeshComponent | albedo, normal, ORM, emissive layers | meshId invalid |
| SkinnedMeshComponent | same + skin palette | skeleton mismatch |
| SpriteComponent | atlas layer, lighting mode | zero alpha clip |
| TerrainComponent | grass/rock material preset | empty height buffer |

MaterialComponent without MeshComponent on same entity — no draw (component orphan). Mesh without Material — gather may use default gray material index 0 — visible but wrong; assert in debug builds.

**Multi-material glTF** — submesh index on draw item selects material slot from vector on MaterialComponent. Wrong submesh index shows correct geometry with wood texture on metal submesh — trace glTF import submesh table.

When debugging PBR dark albedo, verify ORM layer bound — missing ORM defaults metallic/roughness to values that kill specular. Cross-check RenderDoc binding 1 layer indices against MaterialComponent gather log.

### Light component gather quick ref

| Component | Params array | Shadow contribution |
|-----------|--------------|---------------------|
| DirectionalLightComponent | sun block fields | CSM enable flags |
| PointLightComponent | pointLights[] | Punctual slot assignment |
| SpotLightComponent | spotLights[] | Spot shadow atlas tile |

Lights without Transform — skip gather with debug LOG. Zero intensity lights should cull before cluster upload to save SSBO bandwidth. Directional overrides time-of-day when both present — see lighting precedence table in appendix ch02.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SceneSubmit translation layer

**Concept:** ECS vs snapshot

**Decision:** SceneSubmit translation layer

**Why:** Single gather module — renderer O(draws) independent of entity count scaling tricks.

**In Spark:** `src/spark/scene/SceneSubmit.cpp`.
### Folder per domain

**Concept:** Component domains

**Decision:** Folder per domain

**Why:** Scales to 40+ component types without flat include chaos.

**In Spark:** `include/spark/ecs/components/`.
### Submit skips incomplete

**Concept:** Required component pairs

**Decision:** Submit skips incomplete

**Why:** Fail-soft in debug logs — avoids drawing garbage with null mesh.

**In Spark:** RigidDrawableSubmitSink guards.
### ForEach + typed getters

**Concept:** Submit gather pattern

**Decision:** ForEach + typed getters

**Why:** No reflection — explicit switches keep compile-time safety.

**In Spark:** SceneSubmit.cpp mesh walk.
### Programming guide component ref

**Concept:** Cross-reference workflow

**Decision:** Programming guide component ref

**Why:** Spark maintains extended markdown list — link from your wiki.

**In Spark:** `docs/programming-guide/.../07-game-component-reference.md`.
### Five-component trace lab in onboarding

**Concept:** Component gather reference table

**Decision:** Five-component trace lab in onboarding

**Why:** New engineers trace Transform→GPU in 45 min — faster than re-reading Parts IV–VII.

**In Spark:** SceneSubmit.cpp gather functions.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/SceneSubmit.cpp`
- `include/spark/ecs/Ecs.hpp`
- `docs/programming-guide/1-overview-architecture/07-game-component-reference.md`

### Step 1 — Grep five components in SceneSubmit

Transform, MeshComponent, MaterialComponent, SpriteComponent, TerrainComponent — note line of gather.

*Submit gather and frustum cull region*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 242–260)
    Array<SceneDrawItem> drawList;
    drawList.Reserve(48);

    RigidDrawableSubmitSink rigidSink{drawList, params, findOrAddTexture, defaultShadowFlags};
    if (sceneForCulling != nullptr) {
        DispatchDrawableFrustumCull(
                *sceneForCulling, viewProjection, sceneForCulling->GetSpatialPartitionKind(), rigidSink);
    } else {
        world.ForEachActiveGameObject([&](GameObject* o) {
            if (o == nullptr) {
                return;
            }
            const MeshComponent* mc = o->GetComponent<MeshComponent>();
            if (mc == nullptr || !mc->GetMesh()) {
                return;
            }
            rigidSink.OnDrawable(o, *mc, o->GetComponent<MaterialComponent>(), o->GetWorldMatrix());
        });
    }
```
### Step 2 — Trace MaterialComponent

Fields → SceneDrawItem textureLayer, metallic, roughness → push constants / scene.frag sample.
### Step 3 — Trace ParticleEmitterComponent

Part VIII — emitter sim → params.particles → VulkanParticlePass record.
### Step 4 — Trace PostProcessVolumeComponent

Part X — volume resolves exposure/SSAO flags into params — may not create draw item.
### Step 5 — Trace SoundCueComponent

Part XV — audio path bypasses SceneRenderParams — example of non-render component.
### Step 6 — Add new component workflow

Template: ECS component → Update logic → SceneSubmit gather → params field → GPU path → serializer registry.
### Step 7 — Skinned character checklist

List components for skinned draw: Transform, SkinnedMesh, Material, Animator, skeleton palette upload path.
### Step 8 — Build personal quick-ref table

Five columns: Component, Key fields, Submit function, Params field, Pass — pin above desk.

## 5. How This Connects to the Rest of Your Engine

Part **XIII** serializes components. **SceneRenderParams reference** lists GPU fields they fill. **Demo catalog** shows which components each demo spawns. **Descriptor map** completes GPU bind step.


Part **XV** audio components and Part **XIII** editor components appear in domain table — "(not params)" column prevents false grep in SceneSubmit when debugging SoundCue. Serialization chapter defines which rows persist to disk.

## 6. Pitfalls and Debugging

- **Component without gather** — exists in ECS but invisible.
- **Wrong Kind() in switch** — debug exhaustive Kind assert catches in dev.
- **Duplicate transform sources** — single TransformComponent authoritative; physics writes back.
- **Expecting audio in params** — SoundCue uses SoundSubsystem, not snapshot.
- **Editor gizmo component in submit** — guard with `#ifdef DEVELOPER_TOOLS` gather skip.


- **SoundCue expected in SceneRenderParams** — audio uses SoundSubsystem parallel path.
- **Duplicate transform from physics and component** — single TransformComponent authoritative.

## 7. Exercises

1. Complete five-component trace lab; document each step pass/fail.
2. Copy component gather table; add row for one custom component.
3. List components required for skinned character draw from table.

## Summary

Core ECS components feed SceneSubmit gather functions that build SceneRenderParams. Spark organizes components by domain; audio and AI use parallel subsystems outside the snapshot. Component gather, serialization, subsystem parallel paths, and editor/runtime split document every ECS-to-GPU route in Spark.
