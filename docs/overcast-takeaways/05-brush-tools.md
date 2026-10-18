# 05 - Brush tools (2D and 3D)

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Reuse marks as in README §1.

## 1. Inventory

SPZ has **one brush engine** (`F/Paint/MaskPainter.cs`, abstract) with three concrete uses:

| Class | Paints | Target | Notes |
|---|---|---|---|
| `Projections_MaskPainter` (`F/Paint/ProjectionsMasking/`) | per-view **reveal/hide mask** of a projection | UV mask R8 per view (`04` §2) | the main "fine-tune a projection" tool; strokes land through the mesh |
| `Inpaint_MaskPainter` (`F/Paint/Inpaint/`) | **colour layer** that marks where to regenerate (and, in coloured mode, actual colour) | UV RGBA8 layer | see `06`; colour is real paint, mask is alpha |
| `Background_Painter` (`F/Paint/BG painter/`) | 2D mask for the background image | screen-space 2D | uses `Screen2d_BrushShader` (no mesh) |

There is **no general-purpose texture painting** (no clone, smudge, fill with pattern, eraser of arbitrary layer pixels) and **no symmetry** (grep for "symmetr" finds only a delighter heuristic and a splash hint). The brush has
**size, opacity, hardness (stamp texture), colour, add/subtract direction**; no "flow", spacing, scatter, rotation, jitter, or smoothing parameters.

## 2. Stroke engine (viewport input → mask)

Per frame while the mouse/pen is down (`MaskPainter.OnUpdate/PaintOnTexture`):

1. Gather: `pointInViewport01`, previous point, `brushSize` (slider value mapped through `_brushSizeScale` curve), previous size, strength (opacity), first-frame flag.
2. Pen pressure (`Pen.current.pressure`) modifies size, opacity or both through two animation curves (`TabletPressureMode.AffectSize/AffectOpacity/AffectBoth/AffectNone`). On release, size/opacity are forced to 0 to prevent a tail dab.
3. Render the **stroke segment** into a *current stroke buffer* (R8 array), see §3. The buffer for the previous frame is kept to compute differences.
4. Apply the **difference** `curr − prev` to the persistent mask (`ApplyBrushStroke_ToUvMask`, compute shaders `ApplyBrushStroke_IntoMask(.compute|POVs.compute)`), every frame. Buffers are cleared on the next stroke start.
5. Mark everything dirty (`Objects_Renderer_MGR.ReRenderAll_soon()`), the display updates immediately.

Guard conditions: no painting while Alt (orbit) or Ctrl (zoom) is held, while an image import is in progress, while the eyedropper magnifier is active, in select-mesh mode, or under a global click blocker. Painting *starts* only when the cursor is over the viewport, but *continues* even if the cursor leaves it (explicit design: "useful when adjusting the backgrounds near the viewport border"). `Shift`+click straight lines are disabled (comment, Jan 2025: "doesn't look good, and might mess up someone's painting").

## 3. The 3D brush: rasterise the mesh into UV, evaluate the stamp in screen space

Shader: `F/Paint/Shaders (Brushing)/Object3d_BrushShader.shader` (and `_NoConservativeRaster` copy for old GPUs), math in `C/Shader_Includes/BrushEffects.cginc`.

Pipeline for one dab-segment `A → B` (screen coordinates in the main viewport, size and strength interpolated):

```
for every UV fragment of the selected meshes (mesh drawn with position = UV, as in 01):
    clip = camera_viewport_VP * P ; screen = clip.xy / clip.w          # where this texel appears in the viewport
    gate 1: texel visible to the projector camera of this view      (ProjVisibility.G > 0)
    gate 2: in front of the viewport camera (not behind)            (isOutsideScreen_or_behind)
    gate 3: not occluded in the viewport:  Linear01Depth(z) - 3e-5 <= viewportDepth(screen)
    if any gate fails: discard (write nothing)
    # nearest point on the segment, aspect-corrected
    t       = clamp( dot(p - A, B - A) / dot(B - A, B - A), 0, 1 )   (computed in aspect-corrected space)
    nearest = A + t(B - A);  size = lerp(sizeA, sizeB, t);  strength = lerp(strA, strB, t)
    uv_stamp = clamp( (screen - nearest + 0.5*size) / size , 0, 1 )   (size.x /= aspect)
    stamp    = tex(brushStamp, uv_stamp).r                           # hardness texture
    stamp    = 1 - pow(1 - stamp, 1 + stronger)                      # stronger = 2 when erasing a mask, else 0
    nd       = saturate( invLerp(0.2, 0.5, dot(normalize(n), normalize(view))) )    # fade grazing angles; only when adding in single view
    wanted   = stamp * strength * nd
    out      = max(wanted, previousStrokeBuffer[uv])                  # accumulate by max → no build-up inside one stroke
```

Design points worth keeping (all `ALGO`):

* **Capsule between samples** (nearest point on the segment with interpolated size/strength) gives continuous lines at any mouse speed; no stamp spacing parameter is needed.
* **`max` accumulation inside a stroke** means an overlapping scribble never exceeds the chosen opacity (like a Photoshop stroke with "build-up" off). The persistent mask changes only by the *increase* of this stroke buffer.
* **Visibility gate with the projector's own G channel**: you cannot paint a view's mask where that view does not see the surface, which avoids "invisible edits" and dead data. The shader comment explains the second reason: painting on the back would otherwise change the equalisation of *all* views' weights at that texel.
* **Normal fade** `invLerp(0.2, 0.5, n·v)` damps strokes on surfaces turned away from the viewport camera "to prevent accidentally painting sides of the mesh, leaving ugly stretches". It is switched off when erasing and in multi-view editing (`Projections_MaskPainter`: `fadeByNormal = !isMultiView && isPositive`), because users would not understand why nothing is painted. Requires smoothed normals ("your mesh needs 180° auto-smoothed normals").
* **Erase is stronger than add** (`brushStampStronger = 2` → `1 − (1 − s)^3`), because the colour path uses a different blend and erasing felt thinner. The colour-layer erase additionally uses an accelerating curve: `accel(x) = saturate(sin(x/max · π/2) · max)`, multiplied by `1 + 2(1 − alpha)`, and snaps alpha to 0 below 0.02 (`C/Shader_Includes/Brush_ApplyFinalStroke_ToMask.cginc`).
* **Conservative rasterisation** (`Conservative True`) guarantees that thin UV triangles still receive fragments; a copy of the shader without it exists for GPUs that malfunction with it. In a CPU/OpenCV design the equivalent is dilating the texel→triangle coverage by one texel, or sampling by inverse mapping (below).
* **Gutter handling** (`ApplyBrushStroke_IntoMask.compute`): a texel outside every UV island (per the chunk mask) copies the stroke difference from a valid 3×3 neighbour, so painted values spill 1 texel into the padding and do not leave seams when mip-mapped. Radius deliberately fixed at 1 ("don't use more than 1, it doesn't make much difference").

### 3.1 Colour strokes are baked once, at mouse-up

In coloured inpaint mode the stroke buffer is **applied live as an overlay** (`EntireColorLayer_BlitApply.shader` with `APPLY_LATEST_BRUSH_TOO`: `lerp(layer, brushColor, strokeBuffer)`) and **baked into the colour layer only on release**
(`Inpaint_MaskPainter.OnFinal_ApplyIncomingVals_intoMask`). The comment: repeated `lerp` across frames makes the path non-smooth and unstable, so the lerp must be applied once with the final stroke strength. `ALGO`, important for any colour brush.

### 3.2 Translating to Overcast (CPU / numpy / OpenCV)

The GPU trick (rasterise mesh at UV positions, evaluate screen-space function per fragment) can be replaced by an **inverse map** that the view/projection code already needs:

1. For the current viewport camera, maintain a `uv_to_screen` buffer for every covered UV texel (we compute exactly this for projection) with an occlusion/visibility flag (`01`).
2. A stroke segment becomes a function of screen position; evaluate it at `uv_to_screen` for texels inside the segment's screen bounding box only (bounding-box culling → tiny cost per stroke).
3. Keep `stroke_buffer` (float or uint16, same size as the mask) and `mask` (float16/32). Apply `delta = max(0, stroke_new − stroke_old)` per tile.
4. Alternative with identical visuals but different cost: render the stamp into a screen-space image and *gather* via `cv2.remap` with the UV→screen map (nearest or linear). It makes brush shape/hardness trivial (any stamp image) and is easy to GPU-accelerate later in three.js (project in a fragment shader).

For the 3D viewport in the React/three.js UI, evaluating the brush in a shader (as SPZ does) is natural: `Effort L` including the occlusion pass, but only `M` if we do it on the Python side per stroke event and stream back tile deltas.

## 4. Pressure, hardness, size, opacity, fill, eyedropper

| Feature | Behaviour | Source |
|---|---|---|
| Size | slider 0-1 mapped through an animation curve; `[` / `]` change by 0.01 on press, 0.03 per repeat tick (0.03 s) after a 0.2 s delay; `Shift`+RMB-drag resizes by the dominant axis of mouse delta (× 5), clamped to [0.001, 1] | `BrushRibbon_UI_Size.cs`, `MaskPainter.CursorPreviewUI_Reposition` |
| Opacity | keys `1…9,0` → 0.1…1.0 (only while hovering the viewport and with no Ctrl/Shift held); forced to 1.0 when switching to the colourless inpaint mode | `BrushRibbon_UI_Opacity.cs` |
| Hardness | 3 stamp textures (`brush_alpha_0/1/2.png`); key `H` cycles; `Ctrl+1/2/3` sets it exactly; entering multi-view edit mode auto-switches to the medium brush because the softest preview is barely visible | `BrushRibbon_UI_Hardness.cs` |
| Direction | `X` toggles add/subtract (positive/negative); a visible cursor colour shows it | `SD_BrushRibbon_UI_Direction.cs` |
| Bucket fill | `Ctrl/Shift+F` (button shows a short "confirm" text, second press within the window confirms): renders the selected meshes' islands (`Fill_UV_Chunks.shader`), dilates 1 texel, writes into the mask (white for add, black for subtract; for projections it respects the view's visibility) | `MaskPainter.OnBucketFill_orDelete_button` |
| Delete/clear | `Ctrl/Shift+E` (colour layer only; disabled for projection masks "to avoid confusion, user should use fill") | `BrushRibbon_UI_DeleteButton.cs`, `Projections_MaskPainter.OnDelete_button` |
| Invert mask | button; compute `InvertBrushMask.compute` over the view's mask with its visibility | `ApplyBrushStroke_ToUvMask.InvertMask` |
| Eyedropper | toggle or **Alt** hold: magnified preview (32 px, ×4) around the cursor; on click reads the *composited screen* via async GPU read-back; locks all other clicks while active | `BrushRibbon_UI_EyeDropperTool.cs` |
| Colour | swatches; **`Space` or releasing the right mouse button opens a colour picker centred on the cursor** (`MouseWorkbench_Zone`), a good pattern for fast pickers | `BrushRibbon_UI_Colors.cs`, `F/MouseWorkbench/` |
| View-mask peek | in multi-view edit mode the brush cursor shows **the selected view's art inside the cursor circle** (`CURSOR_COLOR_WHITE` variants), with "wrong side" text on reversed faces | `MultiProjectionShader.shader` (`MaskByCursor`) |

## 5. What is missing in SPZ and worth designing for Stage 5

* **Symmetry** (mirror stroke in object space across X/Y/Z): not present. Easy for us: evaluate the stamp also at the mirrored screen position through the mirrored UV texel (needs a mirror-UV map or object-space mirrored raycast).
* **Flow / spacing / smoothing / stabiliser**: not present; implement flow as accumulate-on-top instead of `max`, and a pulled-string stabiliser on the cursor path.
* **Per-stroke undo**: not present (`04` §3).
* **Layer-targeted painting**: the brush only edits masks/the colour layer. An Overcast "paint colour on layer" brush could reuse §3 with the colour deferral in §3.1.
* **Projective vs surface-following brush**: SPZ's stamp is projective (screen-space footprint); on strongly curved surfaces it smears, hence the normal fade. A surface-distance brush (geodesic or 3D-sphere falloff) is a possible improvement.
* **Tablet support**: SPZ reads pen pressure but not tilt; fine to copy the idea.

## 6. Risks and licence

* The 3D brush shader is original SPZ code; take the **math** above, not the HLSL.
* Precision: stroke buffers and masks are 8-bit (`R8_UNorm`, 256 levels; the screen-space brush shader even says so in a comment), so soft low-opacity strokes can band; use float16/uint16 in numpy.
* `Time`-based key repeat and hover checks are scattered across classes (see `12`); define Overcast's hotkeys in one registry.
