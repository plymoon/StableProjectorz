# 04 - Layers, masks, history/undo

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Reuse marks as in README §1.

## 1. SPZ's "layer stack" is a list of generations

There is no Photoshop-style layer panel. The unit is a **`GenData2D`** (`F/StableDiffusion/GenData/GenData2D_Archive.cs`, class `GenData2D`) that bundles everything about one generation or one imported texture:

| Field | Meaning |
|---|---|
| `kind` | `SD_ProjTextures` (a projection), `SD_Backgrounds`, `UvTextures_FromFile` (flat UV texture), `UvPaintedBrush` (colour painted in UV, additive blend), `UvNormals_FromFile`, `BgNormals_FromFile`, `AmbientOcclusion`, `TemporaryDummyNoPics` |
| `povInfos` | up to 6 `CameraPovInfo` (position, rotation, fov, principal point, was-enabled) |
| `_projCamera` | the `ProjectorCamera` that "shines" it (`F/Camera/Projections/ProjectorCamera.cs`) |
| `_byproductsOfRequest` | the context images used for the request: view, depth, normals, vertex colours, screen mask with and without anti-edge |
| `_generatedTextures` | result images (possibly several: a batch) with GUIDs and UDIM sectors |
| `_masking_utils` (`GenData_Masks`) | **per-POV UV masks** (R8) and **visibility** (R8G8), see §3 |
| request payloads | the original txt2img/img2img/extra payloads (SD-specific; ignore) |

Display order = **order of the UI icon grid** (`Art2D_IconsUI_List.guid_ordered_in_grid_refDontAlter()`); the compositing loop is in `Objects_Renderer_MGR.cycle_through_generations` (`02` §2).
Groups of icons (batch results) can be **hidden** (`Q`) or **soloed** (`Ctrl+Q`) (`Art2D_IconsUI_List.IconGroupVisibility_byShortcuts`; `A`/`D` select previous/next group and also restore that generation's camera placement).

Findings that matter for Overcast:

* **Blend model**: one normal "over" blend per projection with an opacity; no multiply/screen/overlay modes, no per-layer blend selection. AO is a separate fixed multiply stage; painted colour is a fixed additive-premultiplied stage.
* **Per-layer adjustments** live in a small circular-slider menu on each icon (`F/Icons/IconUI/IconUI_Art2D_ContextMenu.cs`): visibility (opacity), edge blur stride, edge blur power (also the multi-view blend sharpness, `02` §1.2), "multi-projection blend strength", background blur, hue/saturation/value/contrast, plus buttons: delete (double-click to confirm), upscale ×2/×4 (double-click to confirm), save, load custom image into the projection, **clone**, **restore camera**, show depth, copy seed, delight. Changing a blend/HSVC slider re-renders the whole stack live (`ProjectorCamera.OnIcon_Some_*`).
* **Reordering is disabled.** `Art2D_IconsUI_List.ChangePriority_IconGroup_maybe` returns immediately with the comment "NOT USED NOVEMBER 2024. Preventing movement of icons, because Save/Load gets broken". So order = creation order in the current build, a user-hostile limitation. See `12`.
* **Clone** (`GenData_Masks.Clone`, `RenderUdims.Clone`) copies a generation including its masks: cheap manual branching/versioning of a layer.
* **No layer names or groups beyond a generation's batch.** Layers are thumbnails.

`IDEA` for Overcast's layer model:

| SPZ concept | Overcast mapping |
|---|---|
| `GenData2D` = generation + its camera(s) + context images + masks | `Layer(kind=projection)` with a `provenance` block (see `08`) |
| Solo/hide group + hotkeys | `layer.visible`, `layer.solo`, single-key toggles in the React layer panel; solo should hide everything else in the viewport *and* in the exported preview |
| Per-layer HSVC + opacity + edge blur/pow | layer adjustments applied at composite time, not baked into the stored image |
| Double-click-to-confirm on destructive buttons | consistent in React (or press-and-hold) |
| "Restore camera" on a layer | `layer.camera_set` → one click to return to the viewpoint that produced the layer; very useful with the view planner |
| Footprint highlight + order view | debug overlays (`09`) |

## 2. Masks

Three different mask families, all UV-space unless stated:

| Mask | Format | Where | Purpose |
|---|---|---|---|
| **Per-view brush mask** `_ObjectUV_brushedMaskR8[pov]` | `R8_UNorm` array, one slice per UDIM, bilinear, initial value 0.5 (gray) | `GenData_Masks` | reveal/conceal one view's contribution; encoding below |
| **Per-view visibility** `_ObjectUV_visibilityR8G8[pov]` | `R8G8_UNorm` | `GenData_Masks` | (faded, real) visibility from `01`; rebuilt whenever blend sliders change |
| **Inpaint colour layer** `_ObjectUV_brushedColorRGBA` | RGBA8 | `F/Paint/Inpaint/Inpaint_MaskPainter.cs` | region the user wants regenerated (see `06`) |
| Screen-space masks (`screenSpaceMask_WE/NE`) | R8, longest side 512 px | `Inpaint_ScreenMasker_*` | what the generator receives, derived from the UV mask |

Resolution of masks comes from a user setting (`Settings_MGR.get_uv_brushPrecision_res()`), independent of the accumulation texture (`GenData_Masks.MASK_RESOLUTION`); the brush shader uses `_UV_Chunks_scale` to
reconcile different resolutions between mask and chunk texture.

### 2.1 The [0,2] mask encoding (key trick)

The stored byte `m ∈ [0,1]` is interpreted as `M = 2m ∈ [0,2]` and combined with the view's "invisibility" `inv = 1 − Vfaded`:

```
apparent = saturate(M − inv) * Vreal
```

* default `m = 0.5` → `M = 1` → `apparent = Vfaded·Vreal` (the automatic blend, no user intent);
* painting up to `m = 1` → `M = 2` → `apparent = Vreal` for any `inv ≤ 1` (override automatic fading, "I want this view here even though it is at a grazing angle");
* erasing to `m = 0` → hidden.

When writing (`ApplyBrushStroke_IntoMask.compute`): `M' = clamp(2m + diff, 0, 1 + inv)` and the stored value is `M'/2`. The upper clamp `1+inv` stops the mask from accumulating more than what is needed to cancel
the fade, so a later erase does not need to "undo" excess paint. `ALGO`; precision caveat: 8 bits split into two ranges leaves 128 steps for each; acceptable for brush work, but use 16-bit in numpy.

### 2.2 Multi-view sum-to-one rule

`F/Paint/Shaders (Brushing)/ApplyBrushStroke_IntoMaskPOVs.compute`: when the user paints view A up by `diff > 0`, the compute computes how much "apparent visibility" the other views may keep
(`1 − apparent_A`), and if their total exceeds it **shrinks each other view proportionally** to its share (`vis = maxAllowed · apparent_i / Σ apparent`), keeping the portion that is "pure override" (`min(M_i, inv_i)`). Painting a negative value never affects other views.
Result: **user edits stay normalised**, without a global renormalisation pass that would undo other edits. `ALGO`, effort M, relevant for Stage 5 where the user adjusts which view wins at a texel.

### 2.3 Legacy atlas code (ignore)

`AdditiveMasks_ToOverlayMasks.compute`, `MaskAtlas_*.compute`, `Fill_OrderAtlas.compute`, `AtlasSort.cginc`, `BrushMasks_BitwiseTools.cginc` implement an older design: up to 128 masks bit-packed as 8-bit values in `uint4` atlas
texels, with an order array and separate "additive vs overlay" weights. A search for each kernel's Unity GUID (`.meta`) across prefabs, scenes, assets and C# finds **zero references** for `AdditiveMasks_ToOverlayMasks`, `Fill_OrderAtlas`, `MaskAtlas_Blit_uint2`, `MaskAtlas_ExtractSector_uInt4` and `MaskAtlas_Fill` (only `MaskAtlas_Blit_uint4` has one reference); they look like abandoned experiments (dead code). Good to know: SPZ moved from "N masks sorted per texel" to "≤ 6 per-view masks with sum-to-one".

## 3. History / undo

* **There is no undo/redo**, and the app says so itself: confirmation texts read "There is no CTRL+Z yet." (`F/3D Models/UI/SD_subMesh_IconUI.cs:68`, `ModelsHandler_3D_UI.cs:43,121`), "Careful, there is no CTRL+Z!" (`F/Icons/IconUI_List_Art/IconsUI_List.cs:206,230,246`, `Art2D_IconsUI_List.cs:77`). This applies to brush strokes, bucket fills, mask inversion, layer deletion or flattening. Grep for `Undo`/`undo` in `F/Paint`, `F/Camera`, `F/StableDiffusion`, `F/Save Load Import Export`
  finds only a comment about eroding after dilating and an unrelated animation-restore comment. `ProjectorCamera.OnIcon_Some_Blending_Slider` etc. just recompute.
* What *does* exist as history: every **generation is kept** (archive of `GenData2D`) with its request, camera and context images, and **clone** exists. the inpaint UI has a "ReDo" control (`F/Paint/Inpaint/UI/ReDo holder (scaled) - PIVOT AT BOTTOM.prefab`) which, judging by its name and context, re-runs the generation rather than reverting edits (not verified in code).
* Strokes are applied progressively to the mask during dragging (diff of current vs previous stroke buffers) and cannot be reverted.

`IDEA` for Overcast Stage 5: do the opposite. Strokes are small and well-defined (`prev`/`curr` buffers plus parameters), so record each stroke as a command `(layer, mask_id, stroke_params, tile_bbox, before_patch)` and keep `before_patch` crops for a bounded undo stack, flushing to disk when it exceeds a memory cap.
Also snapshot masks before destructive operations (invert, fill, flatten).

## 4. Take / effort / risks

| Item | Mark | Effort | Risk |
|---|---|---|---|
| [0,2] mask encoding and clamp rule | ALGO | S | confusing for users if exposed; hide behind "reveal/hide/auto" semantics |
| Proportional shrink of other views on paint | ALGO | M | must be vectorised in numpy; per-stroke dirty rectangle |
| Hide/solo + hotkeys, clone, restore camera, double-click confirm | IDEA | S each | none |
| Ordered stack with free reordering | IDEA | S | SPZ disabled it because the save format depended on order; use stable layer ids |
| Undo via stroke commands | IDEA | M | not in SPZ; our design |
