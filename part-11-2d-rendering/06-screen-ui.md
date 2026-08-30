# Screen UI

> **Build-Along Milestone:** HUD health bar stays corner-anchored after window resize; UI passes appear after tonemap in RenderDoc.

**You will learn:**

- Draw screen-space rects and text after tonemap into the present pass.
- Use framebuffer pixel coordinates and paintOrder for stacking.
- Apply scissor clipping for scroll regions via VulkanScreenUiClip.

## 1. The Problem We Are Solving

HUD and menus are **not** world geometry. They must stay fixed on screen while the camera moves, use **display-referred** colors after tonemap, and batch efficiently. Spark's `VulkanScreenUiPass` records orthographic quads and text into the swapchain render pass — separate from HDR world sprites.

Milestone: full-screen dim overlay at 50% alpha after tonemap, plus corner-anchored health bar surviving window resize on Retina displays.

## 1b. Concept Deep Dive — Display-Referred UI After Tonemap

### Screen space uses framebuffer pixels, not viewProj

HUD rects live in `ScreenRectDraw` / `ScreenTextDraw` on `SceneRenderParams` — emitted during gameplay prep, never from Vulkan code. `VulkanScreenUiPass` builds an orthographic projection from swapchain pixel width/height; Y increases downward:

```text
World sprite:  model × viewProj → HDR pass (camera moves → sprite moves)
Screen UI:     pixel rect → ortho NDC → swapchain present pass (camera moves → HUD fixed)
```

Mixing the two by assigning screen pixel coords to a world `model` matrix breaks when the camera zooms in 3D hybrid scenes.

### Record order: tonemap first, then screenUi, then ImGui

UI colors are authored 0–1 for the monitor, not linear HDR radiance. Drawing before tonemap makes widgets respond to exposure sliders — wrong for shipping HUD:

```text
HDR scene → SSAO (optional) → RecordTonemap → screenUi.Record → RecordImGuiDrawData
```

Spark records both UI paths inside the **present render pass** targeting swapchain UNORM — not `hdrRenderPass`.

### paintOrder and scissor for stacking

Multiple rects sort by `paintOrder` before record. `VulkanScreenUiClip` applies scissor regions for scroll panels — same command buffer as rects/text, no separate render pass per widget. Use `GetFramebufferSize` on Retina, not window client size, for corner anchors.

## 2. Core Concepts

### Screen space vs world space

**Screen UI** — HUD, menus, health bars, dialogue — lives in **framebuffer pixel coordinates** with Y increasing downward, independent of world camera FOV or orthographic world scale. Spark stores draws in `ScreenRectDraw`, `ScreenSpriteDraw`, and `ScreenTextDraw` on `SceneRenderParams`, all emitted during gameplay/render prep — never from Vulkan code.

World sprites (Chapter 1) multiply by `viewProj` and respect scene depth. Screen rects multiply by an **orthographic UI projection** built from swapchain pixel width/height inside `VulkanScreenUiPass::Record`. Mixing the two spaces in one shader is a common beginner bug: HUD quads drawn in the HDR pass scale when the camera zooms.

ScreenRectDraw uses top-left origin Y down — opposite of some UI frameworks; convert when porting Dear ImGui layout math. World sprites use viewProj — never assign screen pixel coords to model matrix.

Screen UI uses **pixel coordinates** anchored to framebuffer corners, not world transforms. A health bar at (16, 16) stays fixed when the camera orbits terrain in hybrid demos. World sprites move with viewProj; screen rects multiply by inverse framebuffer size for NDC placement in the screen UI vertex shader. Mixing the two by parenting HUD entities to the player transform breaks in 3D scenes — attach HUD to screen space submit instead. DPI scaling on Retina displays requires `GetFramebufferSize` for layout, not window client size alone.

Safe-area insets on mobile notches apply as **padding offsets** when converting anchor rects — Spark desktop demos skip this but your port should extend layout helpers. Coordinate handedness for Y-down screen space vs Y-up world space is a common bug when copying rects from UI design tools — flip Y once at import to engine conventions. Debug-draw a one-pixel border around HUD rects in development to catch off-by-one Retina scaling errors early.


### Post-tonemap present pass

Display-referred UI draws **after tonemap** into the swapchain render pass:

```text
HDR scene → SSAO/bloom (optional) → tonemap → screenUi → ImGui
```

UI colors are authored in 0–1 sRGB-ish values meant for the monitor, not linear HDR radiance. A white HUD rectangle `{1,1,1}` after tonemap reads correctly; the same value in HDR scene pass would barely survive exposure changes. Spark records `screenUi.

Record` then `RecordImGuiDrawData` before `vkCmdEndRenderPass` on the present target (lines 355–371 in `VulkanRenderer.cpp`).

Tonemap writes swapchain image — screenUi reads same image as render target. UI colors authored 0-1 display referred — not HDR linear. Exposure slider does not affect UI drawn after tonemap.

Display-referred UI draws **after tonemap** so `#FFFFFF` text reads as white on screen regardless of HDR exposure. Drawing UI before tonemap makes widgets respond to exposure sliders — sometimes desirable for debug overlays, wrong for shipping HUD. `VulkanScreenUiPass` targets swapchain format directly with sRGB-aware blend if configured. Alpha premultiplication policy must match sprite pipelines or edges look dark on video capture. Record order: tonemap → screenUi → ImGui ensures debug panels paint atop HUD.

Swapchain color space may be sRGB non-linear — UI tints authored in Photoshop sRGB look correct without manual gamma pow in shader. HDR scene exposure changes do not affect HUD contrast when UI is post-tonemap — intentional for readable health bars during bright bloom moments. Video capture APIs that grab swapchain images include UI automatically; use capture flags from Part X to omit UI for marketing level shots.


### Draw types and paint order

Three UI draw streams batch separately but share **paintOrder** for interleaving:

| Type | Struct | Typical use |
|------|--------|-------------|
| Solid/gradient rect | `ScreenRectDraw` | Panels, dim overlays, health bar fill |
| Textured quad | `ScreenSpriteDraw` | Icons, nine-slice borders |
| Text | `ScreenTextDraw` | FPS counter, dialogue |

`paintOrder` monotonically increases as UI elements emit — rects and text merge into correct stacking. Multiple buckets exist (`screenRects`, `screenOverlayRects`, `screenLateRects`) so gameplay HUD, pause overlay, and tooltips can clear independently without rebuilding the entire tree each frame.

screenLateRects draw after standard rects — tooltips and cursor use late bucket. paintOrder merges rects and text in single sorted sequence per bucket.`ScreenRectDraw`, text draws, and nine-slice panels share a **paintOrder** integer sorted stably like sprites. Lower paintOrder draws first; higher draws on top within the UI pass. Blend mode selects alpha vs additive for glow effects on menus. Clip rectangles implement scroll views — scissor set during record per draw batch. Batch consecutive rects sharing texture and blend to minimize pipeline binds; text usually batches by font atlas texture from the same pass object.

Nine-slice panels expand one logical rect into nine quads with shared texture — implement as CPU-side expansion into multiple ScreenRectDraw entries with sub-UVs. Text draws interleave with rects in one sorted index array — stable sort by paintOrder preserves batching opportunities when consecutive entries share font atlas. Additive UI glow for menus uses separate blend pipeline slot like sprites — do not reuse alpha blend pipeline for additive HUD effects.


### Scissor clipping

Scroll lists and nested panels set `clipEnabled` with pixel rect on `ScreenRectDraw`. `VulkanScreenUiClip` intersects clip rects with framebuffer bounds and emits `vkCmdSetScissor` for affected draw batches — consecutive draws sharing the same clip region batch together to minimize scissor churn.

Depth test is **off** for UI — z-order is purely paint order and bucket sequence. Forgetting to disable depth causes HUD triangles to fail against a cleared depth buffer or leftover scene depth if sharing attachments incorrectly.

clipEnabled intersects clip rect with framebuffer — zero area clip skips draw. Consecutive draws share scissor state when clip rect unchanged — batch optimization in RecordSolidRects.

Scissor rects clip **fragment output** in framebuffer space, not NDC — convert layout rects using current viewport dimensions. Nested clips intersect; Spark applies inner clip during child layout in immediate-mode UI helpers. For scroll lists, update scissor offset each frame without rebuilding vertex data if using retained rect pools. Validation layers warn if scissor exceeds framebuffer — clamp on window resize events. Capture tools should note scissor state when HUD elements appear truncated after resolution change.

Scroll views update scissor each frame from layout — stale scissor after dynamic resize clips entire panel. Intersect parent and child scissors for nested menus; empty intersection should skip draw entirely rather than setting invalid scissor. Vulkan scissor Y origin is top-left in framebuffer coordinates matching Spark screen UI convention — verify against your window library docs when porting from D3D.


### Font atlas and text batching

Text draws reference `SceneRenderParams::uiFont` — a baked glyph atlas uploaded once (or when font changes). `VulkanScreenUiPass` records font upload commands when atlas dirty, then batches glyph quads into instanced draws similar to world sprites but in pixel space.

Platformer demos push health HUD via `Platformer2D::HealthHud` writing rects/text into snapshot arrays. Retina displays require **framebuffer size** from `GetFramebufferSize`, not window logical pixels — mismatch shifts click targets and anchor corners.

uiFont nullptr skips text draws silently — mount font in demo Load like TilemapShowcase TextOverlayComponent. Glyph quads batch into instanced draws in pixel ortho.

Text rendering uploads a **glyph atlas** once, then instances quads per string run during submit. Kerning tables live on CPU; GPU sees final positioned rects. Dynamic strings (score counters) rebuild rect arrays each frame — cache width for static labels. Font SDF extensions are outside Spark baseline but would reuse the same screen UI pass hooks. Ensure atlas sampling uses alpha channel for glyph coverage; RGB may be white-only for single-color fonts with tint multiplier on draw.

Dynamic score text changing every frame rebuilds glyph quads — cache string width for static labels like level titles. Font atlas regeneration on DPI change requires invalidating both GPU texture and CPU glyph metrics — hook framebuffer scale callback from window system. Missing glyph fallback to tofu square indicates codepoint not in atlas range — extend atlas build step in tooling pipeline.


### Retained layout vs immediate mode

Spark uses **retained** screen draws rebuilt each frame from components like `TextOverlayComponent` and demo HUD helpers — not Dear ImGui (Chapter 7). Game UI survives shipping builds; ImGui is developer overlay. Your engine may add a `UiCanvas` tree later; Spark's flat arrays prove the renderer boundary: gameplay pushes structs, pass sorts and records.

HealthHud mirrors HealthComponent each frame — presentation separate from simulation. Game shipping uses screenUi; ImGui stripped via macro.

Spark gameplay demos push **ScreenRectDraw** arrays each frame from simple layout functions — retained widget trees are optional sugar. Immediate layout computes positions at submit time from anchor enums (TopLeft, BottomRight). Retained mode helps complex menus; immediate mode suits debug overlays. Pick one style per project layer to avoid double layout bugs. Platformer2DHealthHud demonstrates minimal immediate anchors without Dear ImGui — compare with Chapter 7 for tooling UI.

Platformer2DHealthHud recalculates anchor rects from framebuffer size each submit — pattern works for simple HUD without retaining widget tree nodes. Complex RPG inventories may warrant retained layout with dirty flags — Spark teaching path stays immediate for transparency. Mixing ImGui debug panels with screen UI HUD in same pass requires paintOrder above gameplay HUD if debug should overlay everything.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Separate Screen*Draw structs

**Concept:** Screen space vs world space

**Decision:** Separate Screen*Draw structs

**Why:** Type system prevents accidentally assigning pixel coords to world sprite model matrices.

**In Spark:** `SceneRenderParams.hpp` lines 230–275.

### VulkanScreenUiPass after tonemap

**Concept:** Post-tonemap present pass

**Decision:** VulkanScreenUiPass after tonemap

**Why:** Correct display color space and avoids HDR exposure blowing out HUD alpha.

**In Spark:** `VulkanRenderer.cpp` lines 355–368.

### paintOrder merge key

**Concept:** Draw types and paint order

**Decision:** paintOrder merge key

**Why:** Rects and text interleave without requiring one mega-vertex buffer layout.

**In Spark:** `StableSortRectDrawIndices` in `VulkanScreenUiPass.cpp` lines 19–49.

### VulkanScreenUiClip helper

**Concept:** Scissor clipping

**Decision:** VulkanScreenUiClip helper

**Why:** Shared between solid, sprite, and text record paths — one clip policy.

**In Spark:** `include/spark/render/ui/VulkanScreenUiClip.hpp`.

### Deferred font upload in Record

**Concept:** Font atlas and text batching

**Decision:** Deferred font upload in Record

**Why:** Atlas rebuild on same command buffer as draw — no mid-frame device loss on font change.

**In Spark:** `VulkanScreenUiPass::RecordFontUpload`.

### Flat arrays on snapshot

**Concept:** Retained layout vs immediate mode

**Decision:** Flat arrays on snapshot

**Why:** Simplest contract for Part XI; ImGui layered separately for debug.

**In Spark:** `Platformer2D::HealthHud` demo reference.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/ui/VulkanScreenUiPass.cpp`
- `include/spark/render/ui/VulkanScreenUiClip.hpp`
- `src/spark/render/core/VulkanRenderer.cpp`
- `src/spark/demo/platformer2d/Platformer2DHealthHud.cpp`

### Step 1 — Read ScreenRectDraw layout

Pixel rect, clip, paintOrder.

*ScreenRectDraw*

```cpp
// include/spark/engine/SceneRenderParams.hpp (lines 229–252)
/** Filled axis-aligned rectangle in framebuffer pixels (Y downward). */
struct ScreenRectDraw {
    float x = 0.0F;
    float y = 0.0F;
    float width = 0.0F;
    float height = 0.0F;
    Vector3 color{1.0F, 1.0F, 1.0F};
    float alpha = 1.0F;
    /** Second stop when gradient != None; ignored for solid fills. */
    Vector3 colorB{1.0F, 1.0F, 1.0F};
    ScreenRectGradient gradient = ScreenRectGradient::None;
    /**
     * When true, Vulkan scissor is set to the clip rect (intersected with the framebuffer) for this draw
     * and any consecutive draws sharing the same clip in a batch.
     */
    bool clipEnabled = false;
    float clipX = 0.0F;
    float clipY = 0.0F;
    float clipW = 0.0F;
    float clipH = 0.0F;
    /** Monotonic key so rects and texts can be merged for draw order (see VulkanRenderer UI pass). */
    std::uint32_t paintOrder = 0;
    SceneBlendMode blendMode = kSceneBlendModeDefault;
};
```

### Step 2 — Trace tonemap to screen UI order

Present pass after tonemap.

*screenUi ImGui*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 355–371)
    hdrTonemapPass.RecordTonemap(
            commandBuffer,
            imageIndex,
            frameIndex,
            presentRenderPass.vkPass,
            presentSwapchain().extent,
            presentationFramebuffers,
            resolvedLighting.exposure);
    screenUi.Record(
            commandBuffer,
            frameIndex,
            presentSwapchain().extent,
            pendingScene,
            sceneParamsValid);
    RecordImGuiDrawData(commandBuffer);

    vkCmdEndRenderPass(commandBuffer);
```

### Step 3 — Study rect stable sort

Blend mode and paintOrder.

*StableSortRectDrawIndices*

```cpp
// src/spark/render/ui/VulkanScreenUiPass.cpp (lines 19–49)
void StableSortRectDrawIndices(const Array<ScreenRectDraw>& rects, Array<std::size_t>& indices) noexcept {
    indices.Clear();
    for (std::size_t i = 0; i < rects.GetSize(); ++i) {
        indices.PushBack(i);
    }
    const std::size_t n = indices.GetSize();
    for (std::size_t i = 1; i < n; ++i) {
        const std::size_t key = indices[i];
        const std::uint8_t keyBlend = GetSceneBlendModePassOrder(rects[key].blendMode);
        const std::uint32_t keyPaint = rects[key].paintOrder;
        std::size_t j = i;
        while (j > 0) {
            const std::size_t prev = indices[j - 1];
            const std::uint8_t prevBlend = GetSceneBlendModePassOrder(rects[prev].blendMode);
            if (prevBlend > keyBlend) {
                indices[j] = prev;
                --j;
                continue;
            }
            if (prevBlend < keyBlend) {
                break;
            }
            if (rects[prev].paintOrder > keyPaint) {
                indices[j] = prev;
                --j;
                continue;
            }
            break;
        }
        indices[j] = key;
    }
```

### Step 4 — Push a dim pause overlay

Full-screen alpha 0.5 after tonemap.

### Step 5 — Corner-anchored HUD bar

Use GetFramebufferSize on Retina.

### Step 6 — Inspect Platformer2D HealthHud

Demo HUD without Vulkan in gameplay.

*HealthHud*

```cpp
// src/spark/demo/platformer2d/Platformer2DHealthHud.cpp (lines 1–40)
#include "spark/demo/platformer2d/Platformer2DHealthHud.hpp"

#include <algorithm>
#include <cmath>
#include <cstdio>

namespace Spark::Platformer2D {

namespace {

Spark::SharedPtr<Spark::Texture2D> MakeWhitePixelTexture()
{
    Spark::Texture2D tex(Spark::Utf8String("PlatHudWhitePixel"));
    Spark::Array<std::uint8_t> px;
    px.Resize(4U);
    px[0] = 255;
    px[1] = 255;
    px[2] = 255;
    px[3] = 255;
    tex.SetPixels(1U, 1U, Spark::MoveTemp(px));
    return Spark::MakeShared<Spark::Texture2D>(Spark::MoveTemp(tex));
}

}  // namespace

void HealthHud::Initialize(
        Spark::GameWorld& world,
        const Spark::SharedPtr<Spark::Texture2D>& whitePixelTex,
        Spark::DemoRootCollection& roots)
{
    Shutdown(world);
    Spark::SharedPtr<Spark::Texture2D> tex = whitePixelTex;
    if (tex.Get() == nullptr) {
        tex = MakeWhitePixelTexture();
    }

    root = world.CreateGameObject();
    root->GetName() = Spark::Utf8String("PlatHealthHud");
    rootTr = root->AddComponent<Spark::TransformComponent>();
    roots.Track(root);
```

### Step 7 — Resize stress test

HUD corners pinned after resize.

### Step 8 — Milestone — HUD after tonemap

RenderDoc: UI after tonemap pass.

## 5. How This Connects to the Rest of Your Engine

Screen UI sits at the **end of the frame graph** after Part X tonemap. **World sprites** (Chapter 1) are earlier in HDR. **ImGui** (Chapter 7) records immediately after `screenUi`. **Capture tools** (Part X ch6) may exclude UI layer for level screenshots via flags.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

Screen UI consumes swapchain after Part X tonemap. World sprites never use ScreenRectDraw — different coordinate spaces. ImGui (Chapter 7) paints after screenUi in same present pass.

## 6. Pitfalls and Debugging

- **HDR UI by mistake** — white text blows out or vanishes with exposure; keep UI in present pass.
- **Depth test left on** — HUD fragments fail depth and disappear.
- **Logical vs framebuffer size** — mouse and layout misalign on Retina; always use `GetFramebufferSize`.
- **Forgetting paintOrder** — text renders under panels drawn later in separate buckets incorrectly.
- **World sprite for HUD** — scales with camera zoom; use ScreenRectDraw instead.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

Retina resize without relayout leaves HUD off-screen — listen to framebuffer resize callback every frame. Premultiplied alpha mismatch shows dark fringes on recorded video exports.

## 7. Exercises

1. Draw full-screen dim overlay alpha 0.5 after tonemap.
2. Render FPS counter from `uiFont` glyph atlas.
3. Resize window — confirm HUD stays in top-left corner pixel-aligned.
4. Enable clip on a scroll panel — verify contents truncate at panel edge.
5. List three differences between `screenTexts` and `screenOverlayTexts` paint order.

## Summary

Screen UI draws orthographic rects, icons, and text into the swapchain pass after tonemap via VulkanScreenUiPass, using pixel coordinates, paintOrder, and optional scissor clips.

---

Next: [Dear ImGui](07-imgui.md).
