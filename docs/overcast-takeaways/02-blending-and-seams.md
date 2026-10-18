# 02 - Texel weighting, blending, dilation/padding, seams, colour consistency

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Reuse marks as in README §1.

## 1. Weighting of several views per texel

Files: `F/Camera/Projections/Shaders/Projections_Alignment.shader`, `F/Camera/Projections/Shaders/Alignment_to_Visibility.compute`,
orchestration in `F/Camera/Projections/ProjCam_HelpTextures_Init.cs` (`ImproveVisibility_byAlignments`). Runs only when a generation has
**more than one view** (`isMultiView`); single-view projections use the plain visibility from `01` instead.

### 1.1 Alignment pass (`Projections_Alignment.shader`)

For every UV texel and every enabled view `i` (max 6):

```
d_i = saturate( dot(n_smooth, normalize(cam_i - P)) ) * Vreal_i        # Vreal = G channel of the visibility texture (see 01)
id_i = (i+1) / numPOV                                                  # packed into [0,1], 0 means "none"
```

* `n_smooth` is the **interpolated vertex normal** here (the visibility pass used flat face normals for culling). Mixing the two is deliberate: culling must be hard and robust,
  blending wants a smooth field.
* The six `(d_i, id_i)` pairs are sorted descending with a fixed compare-swap network (`sortDotsArray_Descending`), and **only the best four** are stored in two RGBA8
  texture arrays (`alignDots_rgba8`, `alignIxs_rgba8`). Views ranked 5th and 6th get weight 0 at that texel.
* Rasterisation leaves holes along triangle edges; those holes would read `d = 0, id = -1` and show up as **bright seams** later. `Alignments_AntiSeam`
  dilates both textures by 2 texels, using the id texture (`Blit` of `alignIxs` into an R8 helper) as the coverage mask because dilating by the dot texture itself still showed seams.
  The comment explains why ids work well: "ids are packed into [0,1] and a -1 index is represented by 0, very convenient (0 = empty space)".

### 1.2 Sharpening + normalisation (`Alignment_to_Visibility.compute`)

```
p  = lerp(1.5, 3.5, DotFadePow)          # DotFadePow = the per-projection "blur pow" slider in [0,1]
repeat 5 times:   d = pow(|d|, p);  d = d / max(2e-6, sum(d))
visibility[best_k].R *= d_k               # k = 0..3; G (real visibility) is untouched
```

Mathematical remark (ours, not in the code): the loop equals one step `d_i^q / Σ d_j^q` with **`q = p^5`**, i.e. **q ∈ [7.6, 525]**. The code comment says the
repetition exists because raising to a large power in one go "would squash everything into zero and hit numerical precision" in low-precision storage. In float32 numpy
you can use `q` directly (divide by the max first for stability).

Behaviour: with `q ≈ 8` two views whose dots differ by 10 % get weights roughly 0.69 / 0.31, so there is a visible blend band; at `q ≈ 500` it is nearly a hard Voronoi cut.
So the slider is really "blend-zone width". `ALGO`: expose `q` (or `p`) as a parameter and keep the sum-to-≤1 property.

Sum-to-one property: the dots already contain `Vreal`, so occluded views contribute 0 and the remaining views are normalised among themselves. A texel seen by a single view gets weight 1 (times
that view's own faded visibility).

Dead parameter: `_BestPov_DotMinRange` is set from `edgeBlurStride_01` (`RenderAlignments_POVs_apply`) and described in the compute file as "best view gets 100 % everywhere in this range",
but the kernel never reads it (only declared). Do not assume that behaviour exists.

### 1.3 Final multi-view colour (recap from `01`)

```
w_i      = saturate(2*M_i - (1 - Vfaded_i)) * Vreal_i       # Vfaded_i already contains the alignment weight d_i
colour   = Σ w_i * art_i / Σ w_i                            # normalised by weights only
alpha    = Σ opacity_i * screenMask_i * w_i
```

`M_i` is the per-view brush mask in [0,2]-encoding (see `04`, `05`). `Σ w_i` is not forced to 1; instead the **brush operator keeps the sum of apparent masks ≤ 1**
(`ApplyBrushStroke_IntoMaskPOVs.compute`, shrinks the others proportionally).

### 1.4 What to take for Overcast Stage 3-6

| Item | Mark | Effort | Notes |
|---|---|---|---|
| Weights ∝ `(n·v · Vreal)^q`, top-4 only, q configurable | ALGO | S-M | direct numpy; benchmark against our current weighting on the 4 reference props |
| Hole-fill the weight/id maps before use | ALGO | S | if our UV raster has 1-texel gaps; use the same chunk-aware dilation as for colour |
| Smooth normals for weights, flat normals for culling | ALGO | S | cheap, removes faceting in blend bands on low-poly props |
| Restrict to 4 views per texel | IDEA | S | memory/time cap; fine for 6-view rigs |
| Edge fade from blurred depth discontinuity (single view) | ALGO | S | see `01` §1.2 |

## 2. Layer composition order and blend states

Driver: `F/Render/Objects_Renderer_MGR.cs`, `OnUpdate`/`ProcessMeshes`/`cycle_through_generations`. Nothing is cached incrementally: when anything is dirty
(`ReRenderAll_soon()` or the user is in multi-view edit mode) the whole accumulation texture is rebuilt:

1. Clear accumulation texture to transparent (or **opaque white** if an AO layer exists: `ApplyStartingColor`), draw UV-chunk starting colour (vertex colours optional).
2. For each generation in the **UI icon order** (drag-and-drop in the grid, `Art2D_IconsUI_List.guid_ordered_in_grid_refDontAlter()`):
   * `UvTextures_FromFile` → blit with UV mask (`Blit_uvTexMasked.shader`): `alpha *= 2·mask`.
   * `UvPaintedBrush` → blit with `Blend One OneMinusSrcAlpha, One One` (`Blit_uvBrush_ADDITIVE.shader`).
   * `SD_ProjTextures` → projection pass (`01`).
   * `AmbientOcclusion` → remembered, applied last.
   * Skips layers according to **solo/hide** state (`Q` / `Ctrl+Q`, see `04`).
3. Optional highlight (`ProjectorCameras_MGR.HighlightProjCamera_maybe`): draws a checker texture of the selected projector's footprint on top (alpha `0.4`) so the user sees where it shines even if hidden by later layers.
4. Inpaint colour layer overlay (`Inpaint_MaskPainter.ApplyColorLayer_To_UV_Textures`), then AO (`AO_BlitApply.shader`, see `03`).
5. **Anti-seam dilation** by 5 texels using the UV-chunk mask (`AntiSeams_maybe`). The comment says it "used to be 2 px, requested to be 5 by user 'obsc' on Discord 25/04/2024".

Blend state worth noting for Overcast: layers accumulate with `RGB = lerp(dst, src, w)`, `A = dst.A + w`. That means **RGB is stored as a "premultiplied-like" value** on texels with partial alpha
over empty space (`dst = 0`): colour is darkened by `w`. The repo has no pre/un-premultiply handling (grep for "premult" finds nothing), and later dilation averages that dimmed colour into the padding.
We could not run it to confirm visual impact, but treat this as a known trap; in numpy keep colour straight (unpremultiplied) and keep coverage in a separate channel. See `12`.

## 3. Dilation / padding

Files: `F/TextureTools/Dilation/TextureDilation_MGR.cs`, `TextureDilation.shader`, `TextureDilation_DetectUVchunkEdges.shader`, `TextureDilation_BlurUVchunkEdges.shader`.

### 3.1 Algorithm

Iterative, one texel per iteration (ping-pong between target and helper with **point sampling**; the code stresses "sample via Point, not Bilinear, otherwise dilation will have issues"):

```
for i in range(N):                       # N = number of texels to grow
    helper = copy(target)                # point filtered
    for each texel t with coverage(t) <= 0.01:
        collect neighbours n with coverage(n) > 0.01
        rule One_of_4_samples:   take the first covered of (left, right, up, down)      # keeps technical data crisp
        rule NineSamplesAveraged: average all covered texels in the 3x3 block            # colour, slightly softens
    if a separate chunk mask is used: grow it the same way in lock-step
```

* **Coverage source** (`DilateByChannel`): R, G, B, A of the same texture, **or a separate R8 "UV chunks" texture** (`Use_Separate_UVChunksR8`). Using the separate mask decouples
  "this texel belongs to an island" from the colour alpha, so partially transparent colour still counts as covered. SPZ renders that mask once per import and per resolution change
  (`Objects_Renderer_MGR.RenderUVChunksTex_maybe`, shader `F/TextureTools/UV Chunks/Color_UV_Chunks.shader`, constant white inside islands).
* Thresholds: grow where coverage `≤ 0.01`; border detection uses `> 0.1`.
* **Erosion mode** (`isExpand=false`, keyword `SHRINK`): a covered texel is cleared if any 3×3 (or 4-neighbour) texel is uncovered. Used for screen-space masks (`06`).
* **Border pre-blur** (`findUVchunkBorders_thenBlurThem`, `bordersWiderBlur`): (1) mark border texels = covered, with 1-7 covered neighbours of 8 (`DetectUVchunkEdges`), (2) replace each border texel's RGB with the mean of its non-border covered neighbours in 3×3 (or 5×5), keep its alpha (`BlurUVchunkEdges`), (3) then dilate. This removes the dark/odd fringe produced by rasterisation and bilinear sampling at island borders so it is not smeared outward.
* Cost model: each iteration is two full blits over the whole array. Radius is clamped to the texture size.

### 3.2 Where and how many texels

| Use | Texels | File |
|---|---|---|
| Visibility texture after UV raster | 2 (channel G, 9-sample) | `ProjCam_HelpTextures_Init.DilateVisibility` |
| Alignment dots + ids | 2 (separate chunk mask from ids, 9-sample) | `Alignments_AntiSeam` |
| Live anti-seam of the accumulation texture | 5 (separate chunk mask) | `Objects_Renderer_MGR.AntiSeams_maybe` |
| Bucket fill of a mask | **1**, "2 is already too much, it would creep through seams of nearby uv islands" | `F/Paint/MaskPainter.cs OnBucketFill_orDelete_button` |
| **Export** | `max(width,height)/16` (2048 → 128) with border pre-blur "wider" | `Save_MGR.Get_ProjectionsDict` |
| Screen masks | 1 dilate / 1 erode pairs | `Inpaint_ScreenMasker_*.cs` |

### 3.3 What to take

* `ALGO`: **separate coverage mask + padding width as a fraction of resolution (1/16)**. The 1/16 default is an empirical, generous number; an artist-friendly default for Overcast exports.
* `ALGO`: **pre-blur the island border texels before padding**; cheap, removes visible rims in mipmapped renders.
* `ALGO`: **two flavours of padding**: "first hit" for non-colour maps (ids, weights, normals) vs averaged for colour.
* **Do not copy the iterative GPU loop.** In OpenCV: `cv2.distanceTransformWithLabels(~covered, cv2.DIST_L2, 3, labelType=cv2.DIST_LABEL_PIXEL)` gives the nearest covered texel for every empty texel in one pass; fill from it, then (optionally) a small blur restricted to the new band. Effort S. Be aware that nearest-fill can reach across a thin gutter into a neighbouring island (SPZ has the same leak and mitigates it only by capping radii); limit radius to the island gutter or use island ids from a connected-components pass.

## 4. Seam handling beyond padding: what SPZ actually does

SPZ has **no dedicated seam-welding** (no cross-island colour averaging along matched edges, no gradient-domain blending). Its seam strategy is the sum of:

1. smooth weights between views (§1) so there are no hard view-to-view cuts,
2. edge fade of projections near silhouettes and grazing angles (`01`),
3. padding (§3),
4. hand fix-up: the per-view mask brush (`05`) and "Where empty" regeneration (`06`),
5. generator-side consistency: multi-view sheet (`01`/`07`).

This means the Stage 3 requirement "no visible seams" is only partly supported; use SPZ's weights/padding and add our own seam work (texture-space blur along UV edges, Poisson fill) rather than expecting something to copy.

## 5. Colour consistency / palette divergence

Present in SPZ:

* **Per-layer colour controls**: `EffectsPostProcess` in `C/Shader_Includes/ShaderEffects.cginc`, driven by the layer's HSV + contrast sliders (`ProjectorCameras_RenderHelper.SetupMatVars_HSVC_Usual`):
  hue shift added to H (wrapped), saturation multiplier `s' = s·(sat + 2·max(0, sat-1))` (slider 0→1→3), value multiplier, then contrast about mid-grey 0.5, final `saturate`.
  Plus a tint colour and total visibility (opacity) slider per projection.
* **Delighting** (`03`), manual.
* **Generation-side**: multi-view sheet (≤ 6 views from one sampling run) and "Where empty" inpaint conditioned on the current render (`06`).

Absent: any automatic matching between overlapping projections (no mean/std transfer, histogram matching, or Poisson). So for Stage 3 (4 props, no palette divergence) SPZ contributes **strategy**
(generate together / generate conditioned on what exists) rather than algorithms; our Reinhard/histogram/MVGD matching stays the fallback.

## 6. Risks and licence notes

* The weighting scheme is simple and unpatented as far as can be seen from the repo; reimplement from §1 numbers. No code needs to be copied.
* Edge cases that SPZ handles by hand-tuned constants (thresholds 0.3/0.15 for the angle ramp, `3e-5` depth bias, 5 or 128-texel paddings) should be re-tuned on our four reference props; they were tuned for SD 512-1024 px content on Unity 3-unit meshes.
