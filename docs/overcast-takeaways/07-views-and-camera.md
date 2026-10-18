# 07 - Cameras, views, context maps and view planning

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Reuse marks as in README §1.

## 1. Camera rig (all under `F/Camera/`)

`UserCameras_MGR.cs` owns up to **6 "view cameras"** (`View_UserCamera`), each with child cameras that all copy its pose, FOV and principal-point shift every frame:

| Camera | Class | Renders | Used for |
|---|---|---|---|
| View | `View/View_UserCamera.cs` | what the user sees (HDR sky, wireframe, UV morph) | viewport; also depth for 3D clicking and brushing |
| Content | `Content/Content_UserCamera.cs` | colour image at generator resolution | the init/context image ("clay" render or textured render) |
| Depth | `Depth/Depth_UserCamera.cs` | black-and-white depth, **all views in one image** | depth control image + edge detection + contrast |
| Normals | `Normals/Normals_UserCamera.cs` + `Normals_ViewSpace.shader` | view-space normals | normal control image |
| Vertex colours | `VertexColors/*` | vertex-colour image | part/region control image |
| Projector | `Projections/ProjectorCamera.cs` | nothing visible; stores POV and renders depth/visibility | the camera that "shines" a result (one per generation) |

* Near/far: the projector camera uses a **tight near plane** (`tightNearPlane`, computed by `CameraTools.TightPlanes_around_meshes`) and a **fixed far plane of 1000** ("NOT TIGHT, so that all bad float imprecision is kept away"). Bringing the far plane closer is documented in code comments as harmful to depth precision. For a numpy renderer we render linear eye depth ourselves, so this is not a constraint, but the principle "tight near, never tight far for non-linear depth" matters if the three.js viewport renders depth.
* `ParamsBeforeRender` snapshots and restores camera state around each helper render, so helper renders never disturb the user's view. Worth mirroring in the Overcast render API (immutable view parameters per render call).
* Cameras are kept `enabled = false` and rendered manually (`Render()` / `RenderWithShader`) to control order; `OnPreCull` forces a huge culling matrix to avoid culling bugs when views are off-screen ("sneaky visual bug" comment).

## 2. Multi-view sheet and principal-point pins

SPZ's UI for composing a sheet (`F/Camera/Navigation/CamerasMGR_PinsZone_UI.cs`, `CamerasMGR_POVdefaults_UI.cs`, `F/Camera/Multi-View/MultiView_Ribbon_UI.cs`):

* A small rectangle represents the content image. **Each active view camera has a draggable "pin"; the pin position is that camera's principal point `(cx, cy)`** (`Set_ProjMatrixCenter_ofCamera`). Dragging a pin moves where the object appears inside the shared image; the camera itself does not move.
* Pins are clamped to the unit square ("must not allow the perspective center to be outside [0,1], otherwise it leads to issues with depth, visibility…").
* A button cycles through **pre-arranged layouts** for N = 1…6 pins (`_1_pinPlacementVariants` … `_6_…`, each a set of variants, advanced with `_placementVariant_ix++`), animated over 0.3 s. Only the first pin is active by default; extra cameras are added by a slider (1-6).
* Depth for the sheet: every view's `Depth_UserCamera` renders its own region (single-POV shadow-caster shader `Depth_ShadowcasterReplacement.shader`) and a blit **adds** it into one shared texture (`Depth/Shaders/BlitDepth_of_LatestCamera_Additive.shader`; the texture is cleared to black first, `UserCameras_MGR_CamTextures.OnUpdate_SD_ContentDepth_Started`). The shared texture is then contrast-normalised **once** (`OnDepthUpdate_End`; `Depth_UserCamera.renderCam` comment: "Later it will allow 1 contrast-compute invocation instead of 6 separate depths"). The *multi-POV* depth shader (`Depth_MultiPOV_ShadowcasterReplacement.shader`, up to 6 view-projection matrices, one triangle per front-facing view) is a different consumer: it feeds the projector-side visibility/edge pass (`Depth_Contrast_Helper.Get_BlackWhiteDepth_POVs`, see `01`).
* After generation, `GenData2D.povInfos` keeps every POV (including disabled ones, to keep indices aligned with masks), so the sheet can be re-projected exactly (`01`) and the camera set can be restored with one click (`UserCameras_MGR.Restore_CamerasPlacements`).
* FOV: one slider for all views; the camera moves along its axis to keep apparent size (`02`, `CameraTools.Calc_PosOffset_forFOVchange`: `newDist = dist·tan(fov0/2)/tan(fov1/2)`).

Take for Overcast (value high, effort M): treat **sheet layout** as first-class data: `Sheet{ size, tiles:[{cam, cx, cy, fov, rect}] }`. The default layouts for N = 2..6 (grid-like) can be generated; keep views from overlapping in the image; keep a stable view order. See README takeaway #1.

## 3. Context maps (what is sent to the generator)

| Map | SPZ implementation | Take |
|---|---|---|
| **Depth** | Raw depth → black/white where **close = white, background = 0**. Then min/max normalisation (below). | ALGO |
| **Normals** | `Normals_ViewSpace.shader`: world normal = TBN · (optional normal-map sample, else `(0,0,1)`); `viewN = mul((float3x3)V, worldN)`; colour `= (viewN + 1) · 0.5`; clear colour is `(0.5, 0.5, 1)`. README: "Red = Right, Green = Up". Honors an imported tangent-space normal map per UDIM (`UvNormals_FromFile` layer). | ALGO |
| **Vertex colours** | straight vertex colour render | IDEA |
| **Mesh IDs** | `MeshID_Depth_Replacement.shader`: pass 1 depth, pass 2 writes `_UniqueMeshID` colour (a `ushort` packed in a `float4` via `MaterialPropertyBlock`) | IDEA (picking, `09`) |
| **Screen mask** | see `06` | ALGO |

### 3.1 Depth normalisation (worth matching in our context maps)

`F/Camera/Projections/Depth_Contrast_Helper.cs` + `F/Camera/Depth/Shaders/ReduceFind_MinMaxDepth.compute` + `Improve_DepthContrast.compute`, driven for the generator image by `F/Camera/UserCameras_MGR_CamTextures.cs OnDepthUpdate_End` with user sliders from `LeftRibbon_UI` (contrast, brightness, blur step, "sharp blur", final blur, "final blur inside"):

1. Find `dmin, dmax` over the **non-zero** pixels (zero = background) by a hierarchical reduction: each thread handles a 4×4 block, repeated on the shrinking intermediate texture until one value remains (`R16G16_SFloat` ping buffers, an atomic-free pyramid).
2. **Active path (`ApplyExact`, kernel `CSMain_4x4_float1_strict`; the only mode any caller uses)**: `d' = lerp(d, (d − dmin)/(dmax − dmin), contrast)` for `d > 0`, then `d' ·= brightness`; background stays exactly 0 ("we want black depth background for controlnet").
   (An unused `ApplyMild` kernel also exists: it pretends `dmin' = max(0, dmin − 0.03)` and `dmax' = dmax + 0.05` "so none of the portions of the model are ever fully black / prevents white blob on the forehead". No caller selects it, but the margins are a reasonable idea when a mesh part would otherwise map to pure black.)
3. Add **subtle noise** after normalising: `noise = (hash(pixel)·2 − 1) · 0.011 · sqrt(width/1024) / sqrt(max(brightness,0.1))`, multiplied by `(1 − 0.8·saturate((dmax − dmin)·100))`, so it is strongest when the depth range is tiny. Purpose (comments): small depth variation would band when stretched; flat surfaces parallel to the camera should not be contrasted hard.
4. Optional blur passes: box (half size up to 12, 7 for depth) with `farSteps_amplification = 0.05·(minSide/512)²`, `skipSample_if_differenceGrtr = 1/2^sharpBlur` (do not blur across large depth jumps), then a smaller final blur that can ignore zero-depth samples (stays inside the silhouette).
5. Rendering note: the "contrast" depth texture is made with the camera's near plane tightened to the mesh and far = 1000 (non-linear depth, "edge-detection algorithms work better with it"); a separate **linear 0-1 depth** (`_SD_depthCam_RT_R32_linear`, `ENSURE_LINEAR_01_DEPTH`, `Linear01Depth_float`: `1/((far−near)/near · z + 1)`) is kept for distance comparisons (mask blur limiter, `06`). The comment says Unity's `Linear01Depth()` differs between Editor and Build because of `_ProjectionParams`, hence the manual near/far.

`ALGO`, effort S. For Qwen-Image the preferred depth convention may differ, so make *invert, margin, noise, contrast, brightness* parameters and A/B them. The invariants worth keeping: **one normalisation shared by all tiles of a sheet** (same distance → same grey in every tile), **background exactly 0**, and **dither noise** instead of banding.

### 3.2 Which map goes where

`SD_GenRequestArgs_byproducts` (`F/StableDiffusion/Serialization/SD_JSON_Payloads.cs:283`) stores per generation: `usualView`, `depth`, `viewNormals`, `vertexColors`, `screenSpaceMask_NE`, `screenSpaceMask_WE`. These are saved with the project (`08`) so that the request can be reproduced. `IDEA` for Overcast: store the same set per tile.

## 4. Navigation (UX patterns that are well solved)

| Action | Input | Source |
|---|---|---|
| Orbit around the *selected meshes' bounds centre* | `Alt` + LMB drag | `Navigation/CameraOrbit.cs UsualOrbit` |
| Snap view to the nearest of **26 directions** (6 faces, 12 edges, 8 corners) and look at bounds centre | `Alt` + `Ctrl` + LMB | `CameraOrbit.SnapCameraDirection` / `nearest45DegreeDir` |
| Pan | MMB drag (wide screens normalise x by aspect) | `CameraPanning.cs` |
| Set orbit pivot on the surface under the cursor | MMB click (no movement within 0.002, press < 0.4 s); depth is read from the viewport depth texture; the mesh fades briefly to show the pivot sphere | `CameraOrbit_ClickPivot.cs` |
| Dolly | scroll or `Alt` + RMB drag | `CameraDolly.cs` |
| Fly | RMB held + `W A S D`, `Q`/`E` for down/up (0.8 up/down gain) | `CameraMove.cs` |
| Focus selection | `F` (animated "selfie-stick" lerp of pivot and camera) | `CameraFocus.cs` |
| UV view pan/zoom/focus | MMB, scroll or `Alt`+RMB, `F` | `Camera_UV_NavigateHelper.cs` |

Notes: `Ctrl`+click selects meshes through the ID buffer (`F/3D Clicking/ClickSelect_Meshes_MGR.cs`); `Ctrl+A` and `Ctrl+D` select/deselect all (guarded so "ignore Ctrl when click-selecting" setting does not swallow them). In multi-view edit mode non-active view cameras are re-parented under a disabled object (`_editMode_disabledGO`) and orbit centres on the click pivot instead of bounds.

### 4.1 Auto-framing by bisection (small, reusable)

`Camera_KeepBoundingBoxVisible.Ensure_BoundsVisible`: place the camera on a "selfie stick" at the bounds centre, bisect the stick length for **12 iterations** (start at `4×|bounds.size|`) until all **8 bounding-box corners** project inside **[0.1, 0.9]²** of the viewport and are in front of the camera. Used for framing the AO bake camera. `ALGO`, effort S; handy for per-view auto-framing in the view planner (use the mesh's convex hull instead of the bbox for tighter fits).

## 5. View planning

SPZ has **no view planner**: no coverage optimisation, no candidate sampling, no occlusion-aware scoring. What exists is manual placement + pin layout presets + the "Where empty" workflow (`06`), which acts as an **iterative coverage-driven planner done by the user**: look at the model, see what is still empty, orbit, regenerate. That loop is automatable and fits Overcast's planner:

```
repeat:
    candidates = canonical directions (the 26 SPZ directions are a decent seed set) + user views
    for each candidate view: score = Σ over uncovered texels visible from view of weight(n·v) / texel_area
    pick best, generate (sheet or single tile), project, update coverage
until uncovered area < ε
```

`IDEA` for the planner: reuse (a) the **26-direction set** as the default candidate set, (b) the weight function from `02` as the benefit term, (c) the "fully covered if alpha ≥ 0.95" rule from `06`, (d) bbox-corner bisection for framing.

## 6. Take / effort

| Item | Mark | Effort | Value |
|---|---|---|---|
| Sheet data model with principal-point pins and default layouts | IDEA | M | very high |
| Joint depth normalisation across sheet, margins, noise, background 0 | ALGO | S | high |
| View-space normal encoding and normal-map-aware normals | ALGO | S | medium |
| 26-direction snap + candidate set | ALGO | S | medium |
| Bisection auto-framing | ALGO | S | medium |
| Restore-camera-per-layer | IDEA | S | high (UX) |
| Dolly-zoom compensation | ALGO | S | low-medium |
