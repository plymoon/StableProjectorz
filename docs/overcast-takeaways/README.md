# What Overcast can take from StableProjectorz

Study of this fork (`plymoon/StableProjectorz`, HEAD `37fecc9`, upstream project by IgorAherne) for the
Overcast team. Read-only analysis, nothing was built or run. Written unattended; assumptions are listed
in §5 so they can be challenged in the morning.

Path shorthand used in every file of this report:

* `F/` = `Assets/_gm/Features/`
* `C/` = `Assets/_gm/_Core/`
* Everything else is a repo-root-relative path.

---

## 1. Licence assessment (read this first)

**Not legal advice. Copying code is an owner decision. Everything below is a risk map.**

| Question | Finding | Evidence |
|---|---|---|
| Licence of the project | **GNU AGPL v3** (not "or later" explicitly) | `LICENSE` is the verbatim "GNU AFFERO GENERAL PUBLIC LICENSE Version 3, 19 November 2007" text, 661 lines. GitHub shows AGPL-3.0, consistent. |
| Copyright holder / "or later" grant | **Not stated.** The "How to Apply" appendix at the end of `LICENSE` is the unfilled generic template. No per-file headers exist in the project's own `.cs/.shader/.compute/.cginc` files (grep for `copyright`/`license` finds only one hit: a `//MIT license` comment crediting a gist by yasirkula in `C/UI (reusable)/Widgets and Gadgets/ScrollRect/ScrollRect_ItemFocuser_UI.cs:23-24`). `README.md` credits IgorAherne and points at his repos (`IgorAherne/Shadow_R`, `IgorAherne/rembg-stable-projectorz`). | `LICENSE:621-650` (appendix), `LICENSE:540-558` (§13), `README.md`, `F/Repos/*` |
| Can we combine it with GPL-3.0 Overcast? | **Yes, as a combination.** AGPL-3.0 §13 (last paragraph) explicitly allows linking/combining AGPL-covered work with GPL-3.0-covered work. Each part keeps its own licence; the AGPL §13 *network-interaction* requirement then applies to the combination as a whole. | AGPL-3.0 §13 |
| What does "network clause applies to the combination" mean for us? | If Overcast contains AGPL-derived code and a user can interact with *a modified version* remotely over a network, we must offer them the Corresponding Source of that version. Overcast exposes an HTTP + MCP surface, even if usually on localhost; whether a localhost-only API counts as "remote interaction" is untested. Practical consequence: if code is copied, ship the whole combined source with every distribution and with any remote/hosted use. We already owe source under GPL when distributing binaries, so the *incremental* cost is mainly the hosted/remote scenario. | AGPL-3.0 §13 |
| Can AGPL code be relicensed to plain GPL-3.0? | **No.** The copied part stays AGPL-3.0; only the *combination* can be conveyed under both. Also we cannot assume "AGPL-3.0-or-later" because the repo has no "or later" notice. Record copied files as `AGPL-3.0-only` unless the copyright holder says otherwise. | `LICENSE` has no per-project notice |
| Are algorithms and ideas free to reuse? | Ideas, formulas, pipelines, parameter values and UX patterns are not protected by copyright; only the *expression* (the code text, identifiers, comments, structure) is. Re-implementing from a prose description is the safe path; do **not** translate C#/HLSL line by line. | general copyright principle |
| Third-party code vendored in the repo | MIT: `Assets/Plugins/SimpleFileBrowser` (`LICENSE.txt`), `Assets/Plugins/AssimpNetter` (`License.txt`), `Assets/UnityWindowsFileDrag-Drop` (`LICENSE`). Apache-2.0: `Assets/UnityAI-master/LICENSE`, `Assets/UnityNLP-master/LICENSE`. **No licence file at all:** `Assets/UnityUtil-main` (radix sort / prefix-scan compute shaders). Shader fragments credited in comments: Unity built-in `SpatialMappingWireframe` (MIT per its header) and a ShadowCaster snippet from `przemyslawzaworski/Unity3D-CG-programming` (licence not verified). Details in `11-libraries-and-licences.md`. | see file 11 |

**How each file in this report marks reuse need**

* `IDEA` – describe-and-reimplement only. No code from the repo is needed or recommended.
* `ALGO` – a specific formula/parameter set worth reproducing; still reimplemented from the description given here.
* `COPY?` – only if the owner decides to copy code, and then see the licence notes above. Nothing in this report *requires* copying; where a piece is unusually intricate we say so.

Recommended policy: **clean-room by default.** One person (or this report) describes; another implements in Python/numpy/OpenCV
or three.js. Keep the report as the written spec; do not paste repo code into Overcast.

---

## 2. Executive summary

StableProjectorz (SPZ) is a Unity 6 desktop tool that textures meshes by projecting Stable-Diffusion images (via an external
A1111/Forge/ComfyUI server) from "projector cameras". Roughly 700 C# files, 74 `.shader`, 27 `.compute`, 15 `.cginc`. Its
engine is almost entirely **GPU render passes into UV-space texture arrays** (one array slice per UDIM tile). That makes it a good
source of *algorithms*, but its *implementation* (geometry shaders, `SV_RenderTargetArrayIndex`, Unity camera depth) does not carry
over to a numpy/OpenCV core. What carries over is the recipe for each pass.

**What is genuinely worth taking**

1. **Consistency by generation strategy, not by post-hoc colour matching.** SPZ lets one generation cover up to 6 views at once
   (each view camera renders into a sub-region of one image via an off-centre projection), and its "Where empty" inpaint mode
   generates only texels not yet covered, conditioned on the current render. This is the same idea as our context sheet and is the
   best answer to "palette divergence" in the repo. (`01`, `06`, `07`)
2. **A compact, well-tuned multi-view weighting scheme**: per-texel `dot(normal, view)` × true visibility, sharpened by repeated
   power+renormalise over the 4 best views, plus a separate edge-fade channel from blurred depth discontinuities. (`02`)
3. **A mature brush-on-3D design**: screen-space stamp evaluated per UV fragment, depth/visibility gated, capsule-interpolated,
   accumulated by `max` per stroke, applied to a persistent mask as a *difference* (prev→curr), with a "mask range [0,2]" trick so the
   brush can overcome edge fading. (`05`)
4. **Padding/dilation recipe** with a separate UV-chunk coverage mask, border pre-blur, export padding of `max(w,h)/16`. (`02`, `03`)
5. **Context-map conventions** (depth normalisation with margins and anti-banding noise, view-space normals, ID render). (`07`)
6. **UX patterns** that are cheap and effective: solo/hide hotkeys, projector-footprint checker highlight, greyscale "order of
   projections", hover-preview of a single view inside the brush cursor, UV morph view, non-selected meshes as fading wireframe. (`09`)

**What SPZ does *not* have (do not expect it)**

* No UV auto-unwrap/packing/island handling beyond reading existing UVs (grep for unwrap/xatlas/GenerateUV: nothing).
* No undo/redo or history for painting or layers (grep `Undo` finds only UI-animation code and an unrelated comment). "Redo" is regenerate.
* No layer blend modes; layers are ordered projections with opacity + HSV/contrast; no symmetry in brushes; no "flow" parameter (opacity + hardness stamp only).
* No colour matching between overlapping projections (no Reinhard/histogram/MVGD). Only per-projection HSV+contrast sliders and an optional delighter.
* No seam-aware blending beyond weights + dilation; no Poisson/gradient-domain blending.
* No view planner. Camera placement is manual; there are only default pin arrangements for N views (`F/Camera/Navigation/CamerasMGR_POVdefaults_UI.cs`).
* No Blender bridge. 3D exchange is import via Assimp, export via a FBX/OBJ helper, plus a JSON TCP "agent bridge".
* Large parts are SD-plumbing (A1111 payloads, ControlNet, Trellis/Hunyuan 3D WebUI): ignore `F/StableDiffusion/**` and `F/3D Generate/**`.

**Data flow, mesh import → final texture (as implemented)**

```
Assimp import (F/3D Models/AssimpLoader.cs)  -> normalise to 3-unit box (Objs3D_Container.RescaleModel_fitIntoVolume)
  -> scan UVs for UDIM tiles (UDIMs_Helper.CountUDIMsForMesh)           -> texture array, 1 slice per UDIM
view cameras (up to 6, each with pose, FOV, principal-point shift)
  -> content cam renders clay+depth+normals(+mask) into ONE image       -> sent to SD server
  -> result stored in GenData2D with the exact camera POVs + request + byproducts
per generation, per POV:
  ProjectionVisibility_R8G8   : UV-space (faded vis, real vis) via camera depth test      [F/Camera/Projections]
  Projections_Alignment       : UV-space (dot(n,v)*vis) for each POV, sorted top-4
  Alignment_to_Visibility.cs  : sharpen+renormalise -> weights summing to <=1 across POVs
every dirty frame (Objects_Renderer_MGR.OnUpdate):
  clear accumulation texture -> for each layer in UI order: Projection(Multi)Shader blends into it
  -> optional AO / inpaint colour layer -> AntiSeams (dilate 5 texels) -> display
export: dilate max(w,h)/16 texels -> PNG per UDIM (+ AO), FBX/OBJ, .spz project (JSON + PNG data folder)
```

---

## 3. Ranked top-15 takeaways for Overcast Stages 3-6

Effort: **S** ≤ 2 days, **M** 3-7 days, **L** > 1 week (one engineer who knows the Overcast core). Value is for the stated Stage 3-6 goals.
"Reuse" refers to the marks defined in §1.

| # | Takeaway | Value | Effort | Reuse | One-line next step |
|---|---|---|---|---|---|
| 1 | **Multi-view "sheet" generation**: each view renders into a sub-rectangle of one square image by shifting the projection's principal point; one generated image is then projected back through all those per-view matrices. One generation = one palette. | Very high (Stage 3 palette divergence) | M | IDEA | Make the context-sheet layout emit, per tile, a `(fov, cx, cy)` off-centre projection and project the single result through all of them. (`01`, `07`) |
| 2 | **"Where empty" coverage mask**: render the mesh from the view, mark screen pixels whose UV texel has accumulated alpha < 0.95, erode/dilate by 1 px, blur with a depth-aware limiter, cut at silhouette edges; generate/inpaint only there, with the current render as init image. | Very high (seams + consistency, tile loop) | M | IDEA | Add an operation `coverage_mask(view, accum)` returning a screen-space mask from our existing UV→screen mapping. (`06`) |
| 3 | **Per-texel view weights**: `w_i = dot(n,v_i)·real_vis_i` over at most 4 best views, then 5× `pow(p)` + renormalise with `p = lerp(1.5, 3.5, k)`; sums to ≤ 1. | High (seam quality) | S-M | ALGO | Implement `sharpen_weights(dots, p)` in numpy and A/B it against the current weighting on the 4 reference props. (`02`) |
| 4 | **Two-channel visibility**: R = faded (grazing-angle ramp × blurred depth-edge map), G = real (front-facing & unoccluded). Fade shapes blends, G gates writes/brush and detects back faces. | High | S-M | ALGO | Store `vis_faded`, `vis_real` per view in the UV domain; use G as the only gate for projection writes. (`01`, `02`) |
| 5 | **Padding recipe**: dilate by **separate UV-chunk mask** (not alpha), pre-blur chunk borders, 4-sample "first hit" for technical maps vs 9-sample mean for colour, export padding = `max(w,h)/16` px. | High (cheap, visible quality) | S | ALGO | Replace our padding with `cv2.distanceTransform` + nearest-fill driven by a chunk mask; keep the 16th-of-resolution default. (`02`, `03`) |
| 6 | **Mask model for manual edits**: one R8 UV mask per view stored in [0,2] ("above 1 fights edge fade"), strokes applied as `curr - prev`, other views' masks shrink proportionally so the sum stays ≤ 1. | Med-high (Stage 5) | M | ALGO | Specify the mask encoding and the "apply stroke diff" op in the layer model before building the brush. (`04`, `05`) |
| 7 | **3D brush recipe**: screen-space stamp evaluated per UV fragment, occlusion + visibility gated, capsule between samples, `max`-accumulated stroke buffer, normal fade `invLerp(0.2, 0.5, n·v)`, gutter texels steal from neighbours. | High (Stage 5) | M-L | ALGO | Prototype on CPU/OpenCV via a per-view UV→screen map instead of rasterising; keep the same math. (`05`) |
| 8 | **Context-map conventions**: depth min/max normalised over non-background pixels (one normalisation for the whole sheet), background exactly 0, contrast/brightness sliders, tiny resolution-scaled noise vs banding; view-space normals `(n+1)/2`; ID render for picking; vertex colours. | Medium-high | S | ALGO | Match depth normalisation in our `context maps` generator and compare Qwen output quality. (`07`) |
| 9 | **Generation provenance record**: camera POVs (pos, rot, fov, principal point, enabled), request, byproduct images (view, depth, normals, vertex colours, masks) stored with each result so it can be reloaded, reprojected or re-run. | Medium-high (Stage 4/6, reproducibility) | M | IDEA | Add a `generation.json` + PNG bundle to the project format with exactly these fields. (`08`) |
| 10 | **UDIM rules**: tile index = `ceil(coord)-1` clamped ≥ 0 (so exactly 1.0 stays in tile 0), looped UV with 1e-6 epsilon pull-in, filename parse `10xx`, 24-tile cap, empty-UV fallback to one tile. | Medium (correctness) | S | ALGO | Add unit tests with UVs at exactly 0.0, 1.0, 2.0 and negative values. (`03`) |
| 11 | **Layer UX**: solo (`Ctrl+Q`) / hide (`Q`), per-layer HSV+contrast+visibility sliders re-rendering live, "show order" greyscale, checker footprint highlight of the projector, hover-preview one view inside the brush cursor. | Medium (Stage 5-6) | S each | IDEA | Add the footprint highlight and order view as debug overlays first (cheapest, most useful). (`04`, `09`) |
| 12 | **AO bake + de-light**: accumulate screen-space AO into UV from random view directions with an exponential running average (0.97-0.99 retention), blur inside chunks, dilate. Delight mask from LAB analysis. | Medium | S-M (AO), L (delight) | ALGO / IDEA | Try the AO accumulation as a multiplicative shading layer; defer delight. (`03`) |
| 13 | **HDRI for the viewport**: chrome-ball render → equirect via angular fisheye mapping; multi-exposure EV merge (DiffusionLight, arXiv 2312.09168). | Medium (Stage 6 viewport) | M | ALGO | Spike: Qwen chrome-ball + the mapping formula in `09`, compare with a stock HDRI. |
| 14 | **Agent-bridge design**: self-describing `describe` catalogue, token file with random secret, loopback only, main-thread dispatch, async screenshot tool. | Medium | S | IDEA | Add token + `describe` + `get_viewport_screenshot` to our HTTP/MCP surface if missing. (`09`) |
| 15 | **Viewport tooling**: 3D→UV morph with pan/zoom, barycentric wireframe, non-selected meshes as fading wireframe, ID-buffer picking, UDIM grid overlay. | Medium | S-M | IDEA | Port the UV-morph and wireframe overlays into the three.js viewport. (`09`) |

---

## 4. Index

| File | Contents |
|---|---|
| [01-projection.md](01-projection.md) | Camera projection, occlusion, visibility textures, multi-POV passes, principal-point shifting |
| [02-blending-and-seams.md](02-blending-and-seams.md) | Weights, blending order, dilation/padding, seam handling, colour consistency (what exists / what does not) |
| [03-uv-and-baking.md](03-uv-and-baking.md) | UDIMs, mesh import and scale, chunk masks, bake/export, AO, delight |
| [04-layers-masks.md](04-layers-masks.md) | Layer model, masks, ordering, solo/hide, absence of undo/history |
| [05-brush-tools.md](05-brush-tools.md) | 2D/3D brush math, pressure, hardness, bucket fill, hotkeys, missing features |
| [06-inpaint-masks.md](06-inpaint-masks.md) | How regions are drawn and turned into generator masks, "Where empty", anti-edge logic |
| [07-views-and-camera.md](07-views-and-camera.md) | Camera rig, multi-view sheet, context maps, navigation, view-planning ideas |
| [08-project-format.md](08-project-format.md) | `.spz` JSON + data folder, provenance, asset handling, export naming |
| [09-viewport-and-ux.md](09-viewport-and-ux.md) | Viewport features, hotkeys, panels, HDRI, agent bridge, UX patterns |
| [10-performance.md](10-performance.md) | GPU tricks, resolution ladder, dirty-flag rendering, how to translate to numpy/OpenCV |
| [11-libraries-and-licences.md](11-libraries-and-licences.md) | Third-party code and algorithms with licences |
| [12-pitfalls.md](12-pitfalls.md) | Bugs, limits and user-hostile design to avoid |

## 5. Assumptions made (nobody to ask overnight)

* "Overcast" capabilities are taken exactly from the brief; nothing in the Overcast repo was read.
* Line numbers and behaviour refer to commit `37fecc9` on the fork's `main` lineage; the work branch is `overcast-takeaways`.
* Static reading only: values such as "retention 0.97" are defaults found in code/serialized fields and may be overridden in prefabs or at runtime (Unity serialises many values in `.prefab/.unity/.mat`, which were skimmed, not exhaustively read).
* Effort sizes assume the Overcast core already has: UV rasterisation, visibility/occlusion, per-texel weights, dilation, layer stack, view planner, context maps. Where something is described as "we already have" the effort is for the *delta*.
* Statements of absence ("no undo", "no symmetry", "no auto-unwrap") come from grep over `*.cs`, `*.shader`, `*.compute`, `*.cginc` and a read of the relevant modules; prefab-only behaviour could escape that, which is unlikely for such features.
* The report names `IgorAherne` as upstream author only because the repo's own URLs do; the copyright holder is not declared in the licence file.
* The harness for this session designated the working branch `claude/focused-shannon-eu2ddx`; the task asked for branch `overcast-takeaways`. Both receive the same commits.
