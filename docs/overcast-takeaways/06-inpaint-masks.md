# 06 - Inpaint masks: how regions are chosen and turned into generator masks

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Reuse marks as in README §1. SD-specific payload fields (denoise strength, `inpainting_fill`, soft inpaint, ControlNet) are ignored.

This is the most transferable part of SPZ for Overcast's tile loop: **what to regenerate is decided in 3D/UV space, then rendered per view into a small screen-space mask**.

## 1. Workflow modes (`F/StableDiffusion/WorkflowToolsRibbon SD/WorkflowRibbon_UI.cs`)

| Mode | Meaning | Mask source |
|---|---|---|
| `ProjectionsMasking` | no generation; the brush edits the per-view reveal masks (`04`/`05`) | n/a |
| `Inpaint_Color` | user paints a **coloured** region on the mesh; the colour also seeds the image the generator edits | UV RGBA layer, alpha > 0 |
| `Inpaint_NoColor` | user paints a **colourless** region (checker overlay); only the mask is sent, colour is not altered | UV layer, smooth alpha |
| `TotalObject` | regenerate the whole object silhouette; optional "anti-edge" removes grazing/silhouette edges | full white inside silhouette |
| `WhereEmpty` | regenerate **only texels not yet covered** by existing projections | accumulated-art alpha < 0.95 |
| `AntiShade` | unused ("not used at the moment") | |

## 2. Drawing the region: in UV space, through the mesh

* The inpaint brush (`F/Paint/Inpaint/Inpaint_MaskPainter.cs`) paints into a **UV-space RGBA layer** with the 3D brush from `05`; it is not drawn on the screen image. Consequences: the region is **view-independent**, survives camera moves, can be reused from another view, and it is
  stored per project. This is the central difference to typical 2D inpaint tools.
* Each time it is needed, the region is **re-projected to the current content camera** by rendering the mesh with a shader that samples the UV layer
  (`F/Paint/Inpaint/Shaders/Inpaint_UV_Mask_to_screenMask.shader`): back faces culled by flat normal; `strict = alpha > 0 ? 1 : 0`, `smooth = alpha`, result `lerp(strict, smooth, isColorless)`; optional "fully white" override for the entire silhouette.
  Point sampling only ("sample via Point, not Bilinear, otherwise dilation will have issues").
* The preview shows the region live on the mesh in the viewport, fading in/out with hover over the viewport (`_hoverFadeIn_speed = 7`, `_hoverFadeOut_speed = 0.5`).

## 3. "Where empty": the coverage mask (highest-value idea here)

`F/Paint/Inpaint/Shaders/Inpaint_WhereEmpty_to_screenMask.shader` (+ `Inpaint_ScreenMasker_EmptyNothing.cs`):

```
render the mesh from the content camera (Cull Back), per fragment:
    a    = AccumulatedArt[uv, udim_slice].a          # point sampled
    mask = (a < 0.95) ? 1 : 0
```

* The threshold is **0.95, not 0.99**: "we want to apply the mask even if visibility is, say, 98 % - helpful around borders where we might have fading. 0.99 picks up minor brush imprecisions even though a projection might look fully opaque."
* The accumulated alpha already includes view fading, brush hiding and visibility, so the generator is asked to fill **exactly what the current layers do not cover** from this viewpoint.
* Generator conditioning in SPZ: the content image (render of the current textured/clay model) goes in as init image; the mask selects the empty parts. The result is projected back **only inside the mask** (hard-thresholded, `01` §1.3). So the new region copies the palette/lighting from already-textured neighbours.

`IDEA/ALGO` for Overcast (Stage 3): implement `coverage_mask(view)` = `rasterise(mesh, view) → look up accum alpha via the UV map → threshold 0.95 → screen mask`. Use it for:
(a) the **second and later tiles** of a sequential plan: generate only the uncovered part, with the partially textured render as the init/context image (palette continuity);
(b) a "fix gaps" op at the end;
(c) the seam-hiding loop: erode the covered area slightly, regenerate a ring around seams.
Effort M if the headless render already exposes the UV→screen map.

## 4. Turning the raw mask into the one sent to the generator

Pipeline in `F/Paint/Inpaint/Inpaint_ScreenMasker_Original.cs` (`RenderScreenMask`) and `…_EmptyNothing.cs`, constants: **mask max side = 512 px** (`INPAINT_MASK_MAX_PX_SIZE`; "this small size allows a wide blur efficiently, with a fixed number of blur iterations"), later resized for sending.

1. Allocate mask buffers sized to the content image aspect, longest side 512, R8, point filtered.
2. Optional: **depth edges** of the view (`ScreenDepth_EdgesDetector`: threshold + 7× box blur, see `01`) with thresholds from the UI (`edgeThresh`, `edgeBlur`, `edgeThick`, default thick 0.24).
3. Render the mesh with the mask shader (§2/§3) into `mask` (`RenderContentCameras(..., CameraClearFlags.SolidColor)`; "Clear color, NOT SKYBOX, else the masking shader will flood the whole screen").
4. **Shrink then expand by exactly 1 texel** to produce `noBlur` (an eroded copy remembers where the mask truly is; used to look up depth later).
5. `CalcDownscaledDepths_of_Mask`: downsample linear depth, only from masked texels, 4× smaller (`Downsample_depthOfMask.compute`).
   Observed oddity: the loop in the shader is `for offset = +2; offset <= +2` so it samples only one texel per 4×4 block (the comment says 5×5); harmless but not what the comment promises.
6. **Dilate by an amount derived from the blur** (`kernelHalf × iterations × stepLength`, with `/1.5` outside the colourless mode) so that blurring does not eat the inside of the region; never less than the explicit dilation.
7. **Blur** (4 iterations of a 12-half-size box with exaggerated far steps, `BlurTextures_MGR`) for brushed masks (`Inpaint_ScreenMasker_Original`). The where-empty masker (`Inpaint_ScreenMasker_EmptyNothing`, which `Inpaint_ScreenMasker` routes `WhereEmpty` to) still dilates by the blur-derived amount but **skips the blur** (comment: "EmptyNothing doesn't use soft inpaint. Blur is irrelevant, so only use dilate (v2.0.4)"), has a fixed edge thickness of 0.24, and does not feather.
8. **`PreventBlurredMask_whereFar`** (`F/Camera/Depth/PreventBlur_if_DistanceFromMask.cs`, `Camera/Depth/Shaders/PreventMaskBlur_byMaskDepth.compute`): for each texel *outside* the original mask, search a ±16 box on the downsampled mask-depth; discard candidates if any of 16 samples along the line to them crosses a depth edge; keep the blurred value only if some valid masked texel lies within `zThresh` in depth (`0.15 units / farClip`, i.e. 0.15 world units on the 3-unit model). In plain terms: **blur must not leak across depth discontinuities onto a different surface**.
9. **`FeatherBlurredMask`**: fade blurred values toward the nearest empty texel using the nearest-original and nearest-zero distances (±12 box), so the cut made by step 8 does not leave a hard edge. Skipped for EmptyNothing.
10. **Anti-edge**: subtract the depth-edge map from the mask with `Blend Zero OneMinusSrcColor` (`Blit_RemoveEdges_ScreenMask.shader`): mask = mask · (1 − edge). Avoids asking the generator to paint steep side surfaces and silhouettes where projections stretch.
11. Keep two outputs: **`skipAntiEdge` (dilated by 1 more texel; sent to the generator)** and **`antiEdge` (eroded by 1; used internally when projecting)**. Rationale in the comments: the generator should see a slightly *larger* region (no unpainted halo), the projection should apply to a slightly *smaller* region (no stretched silhouette pixels).

`ALGO` summary for Overcast, independent of the GPU details:

| Step | numpy/OpenCV |
|---|---|
| render view mask from UV layer / coverage | via UV→screen map (`01` §1.1) |
| depth edges | `|d − d_neigh|max over 3×3 > t`, t ≈ lerp(0.005, 0.1, sensitivity) on min/max-normalised depth, background neighbour = 1 |
| blur not crossing depth jumps | **joint/bilateral blur** guided by depth (`cv2.ximgproc.jointBilateralFilter`, or a blur masked by `abs(d_i − d_j) < 0.15·model_size/…`) |
| anti-edge | multiply by `1 − blurred_edges` |
| out-set vs in-set | `mask_send = dilate(mask, 1)`, `mask_project = erode(mask, 1)` |
| soft vs hard | the generator receives a soft mask only if it supports it; projection uses `mask_project > 0` as a hard gate |

## 5. What is generator-specific (ignore) vs reusable

* SD-specific: soft inpaint (differential diffusion), inpainting fill modes, `mask_blur = 0` ("to force SD to render the ball inside the requested zone"), `inpaint_full_res`, padding. Qwen-Image editing takes an image + instruction (and optionally a mask) so the mask API differs; keep the **region definition and the dual dilate/erode logic**.
* `HDR_PanoSkybox_MGR` uses the same machinery for a chrome ball (see `09`).

## 6. Effort, risk, licence

| Item | Effort | Risk |
|---|---|---|
| `coverage_mask(view)` with 0.95 threshold | M | depends on how faithfully our accum alpha reflects fade/brush |
| 512 px working resolution for mask post-processing | S | resize back with nearest/linear carefully |
| Depth-aware blur limiter | M | tune distance threshold relative to model size |
| Dual masks (send larger, project smaller) | S | easy to forget at one call site; encode as a typed pair |
| Region painted in UV, re-rendered per view | M (Stage 5) | requires UV paint layer + brush (`05`) |

Licence: algorithms only; none of the compute/shader code is needed.
