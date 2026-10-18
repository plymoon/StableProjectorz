# 09 - Viewport features, UX patterns, hotkeys, HDRI, agent bridge

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`, `L/` = `Assets/_gm/Layouts/`. Reuse marks as in README §1.

## 1. Viewport features

| Feature | How SPZ does it | Take |
|---|---|---|
| **Unlit textured view** | The mesh shows the accumulated UV texture with an unlit material (`VisualizeFinalMat_Helper`; the `_Final Mat` material in `F/Camera/View/` uses `PreviewWithWireframe.shader` itself, which samples the texture array and has no lighting code); there is **no scene lighting model** in the viewport. | IDEA: for texture review an unlit mode is a useful default; Overcast already has HDRI/AA so offer "unlit / lit" toggle |
| **Wireframe on the textured mesh** | Single pass, geometry shader: for each triangle compute the distance of the fragment to the nearest edge from the 2D triangle (area/base = height), `t = exp2(-2·d²)·wireColor.a`, `colour = lerp(texture, wire, t)`. Colour and alpha configurable ("make it barely visible"). Credits NVIDIA "Solid Wireframe" and Unity's `SpatialMappingWireframe` (MIT). | ALGO, S (a three.js shader: barycentric coordinates attribute) |
| **Non-selected meshes as transparent wireframe** | Hidden by default; shown as wireframe with a fade (`_selectMode_wireOpacity01` ramps at `_selectMode_wireSpeed`, slower while panning) when the user is in click-select mode (holding Ctrl) or panning for > 0.25 s. | IDEA, S |
| **UV inspection mode** | A global `_GLOBAL_WarpIntoUVSpace01` morphs every fragment from its 3D screen position to its UV position (flat "unfolded" mesh) with animation speed adapted to FOV (`UserCameras_UV_warp_Helper`); pan/zoom of the UV plane through `_GLOBAL_InspectUV_Navigate` (offset xy, scale zw); an overlay grid (`UV_Preview_bg.shader`) marks UDIM sectors. Modes are `dim_sd` (generation), `dim_uv`, `dim_gen_3d`. | IDEA, S-M: animate between the 3D mesh and the UV layout (lerp positions) in three.js |
| **Mesh picking** | Dedicated ID render at viewport size; each mesh has a `ushort` id encoded in the colour (`MeshID_Depth_Replacement.shader`); click reads the pixel (`ClickSelect_Meshes_MGR`). `Ctrl`+click selects; `Ctrl+A`/`Ctrl+D` all/none. | IDEA (use GPU ID buffer instead of CPU raycast) |
| **Surface pivot picking** | MMB click reads the viewport depth to place the orbit pivot on the surface (`CameraOrbit_ClickPivot`). | IDEA, S |
| **Background / sky** | Gradient colours, static skybox, or an image; `SkyboxBackground_MGR`. A 2D background layer can be generated and masked like a projection. | IDEA |
| **Texture filtering toggle** | Point vs bilinear in the viewport (`SAMPLER_POINT` keyword) for pixel-art checks; resolution ladder from 32 px. | IDEA, S |
| **Selection gizmos** | Bounding box gizmo (`BoundingBoxGizmo.cs`), pivot sphere; no transform gizmos for the mesh (the mesh is normalised on import and cannot be moved/rotated by the user). | |
| **Screenshot capture** | `Screenshot_MGR`: drag a rectangle in the viewport to capture (async GPU read-back; magnified eyedropper shares the code). | IDEA |
| **Layer previews inside the viewport** | Checker footprint of the selected projector, greyscale "order of projections" while holding `R` (also switches to projection-mask mode), per-view image shown inside the brush cursor with "WRONG SIDE" text when the cursor is over a reversed face. | IDEA, S each |

Order-of-projections view (`MainViewport_UI.ShowOrderOfProjections_maybe`, `ProjectorCameras_RenderHelper.SetupMatVars_HSVC_forOrder`): each layer is drawn as a flat grey with `value = lerp(0.25, 0.8, index/(n−1))` so later layers look brighter and the user sees the draw order without any text. Cheap and effective.

## 2. UX patterns that are well solved (and cheap to copy)

* **Everything has a hotkey and a visible hover cue**; modifiers are consistent: `Ctrl`/`Shift` for "bigger" variants (fill, delete), `Alt` for navigation, `Q` visibility, `X` flip add/subtract, `H` hardness, `[ ]` size, number keys opacity, `F` focus, `A/D` previous/next layer (also restores that layer's camera), `Shift+1…4` switch right-panel tabs (art list / backgrounds / mesh / controlnet; plain `1…0` are brush opacity).
* **Hover-gated keys** (most brush keys act only while the cursor is over the viewport) and a global "is an input field focused" guard (`KeyMousePenInput.isSomeInputFieldActive`) so typing a prompt never triggers tools. Unify this guard in a single hotkey registry in React.
* **Right-mouse picker at cursor** (`Space` or RMB release opens the colour wheel centred on the cursor, `F/MouseWorkbench/`) keeps the pointer where the work is.
* **Double-click-to-confirm** instead of modal popups for destructive buttons (delete layer, upscale), and a **modal confirm only for truly global changes** (see pitfalls).
* **Status line with progress** (`Viewport_StatusText.ShowStatusText(msg, isError, seconds, showProgress)`), used for every long operation ("Baking Ambient Occlusion…", "Importing, scanning UVs…").
* **Slide-out panels** that open on hover for rarely used actions (export, save/load, brush options) so the main view stays clean (`SlideOut_Widget_UI`).
* **Circular sliders with snapping** for per-layer parameters (`CircleSlider_Snapping_UI`): compact, in the icon's context menu.
* **First-run flows**: splash, welcome screens for "novices" and "cmd" users, tooltips on hover (`F/Tooltips`), a dedicated "Help Tips" canvas that shows context hints, and one-time tooltips when a feature is first meaningful (e.g. when the cameras slider goes from 1 to 2 views, the "blend cameras" button animates to attract attention).
* **Attention animation** on controls that become relevant (e.g. `_BlendCams_button_anim.Play()` when entering multi-view edit mode).
* **Tablet pen** is a first-class input: pressure to size/opacity, with a status message when a tablet is detected.
* **Frame-rate hygiene**: target 70 fps, 2 fps when the window is unfocused (`Settings_MGR`), reduced resolution and optional FPS throttle while the generator runs (`Performance_MGR`: texture resolution is dropped to 256 while generating and restored afterwards, except in the final preparation frames before a request so that low-res images are never sent as control inputs). Relevant when Qwen and the renderer share one GPU.
* **Settings that wipe data** (see `12`) are the counter-example.

## 3. HDRI from a chrome ball (viewport lighting idea)

Files: `F/Skybox + Background/SkyboxReconstruction.compute`, `MakeHDR_FromTextures2D.compute`, `HDR_PanoSkybox_MGR.cs`, `MergeTextures_intoHDR.cs`. Based on DiffusionLight (the shader header cites arXiv 2312.09168 §3.3).

1. Generate (with the image model) a render of a **chrome ball** in the target environment at several exposures; project depth of an inserted sphere onto the estimated scene depth so the model draws a ball (`PutSphere_onto_Depth.compute`, `HDR_PanoSkybox_MGR`).
2. Convert the ball image to an equirectangular panorama by inverting the mirror-ball mapping. For each output pixel `(u,v)`: `lon = (2u−1)π`, `lat = (v−0.5)π`, direction `d = (−sin(lon)cos(lat), sin(lat), cos(lon)cos(lat))`; ball texture coordinate from an **angular fisheye** mapping: `denom = sqrt(2(1 + d.z))`, `uv = 0.5 + 0.5·(d.xy/denom)`; outside the unit disc or `denom < 1e-4` (directly behind the ball) → black. The code comment explains why the plain `1 + d.z` denominator was replaced: the edge of the ball reflects what is *behind* the ball, so the mapping must stretch beyond the hemisphere.
3. **Exposure merge** (`MakeHDR_FromTextures2D.compute`): take several images at EV 0, −1, …; use EV0 for chroma; where EV0 luminance (Rec.709 weights, linear light) exceeds 0.9, look at darker exposures until one is below the threshold and use its luminance scaled by `2^(−EV)`, keeping EV0's chrominance (avoids ghosting). Alpha stores final luminance. Max 8 exposures.

`ALGO`, effort M: a "generate HDRI for this prop" button for Stage 6 is plausible with Qwen-Image if a chrome-ball prompt works; compare with stock HDRIs first. Value medium.

## 4. Agent bridge (external control)

Files: `F/AgentBridge/{README.md,SPZ_Agent_Bridge.cs,SPZ_Agent_Protocol.cs,SPZ_Agent_Tools.cs}`; separate MIT project `spz-mcp` by redDwarf03 (referenced in the root README).

* **Protocol**: line-delimited JSON over TCP on 127.0.0.1, default port 8765, `{"id","tool","params"}` → `{"id","ok","result"|"error"}`. `describe` returns protocol version and the tool catalogue with parameters; clients must not hard-code the list.
* **Security**: off by default (`--agent-bridge` in `spz.config`); loopback only; every request needs a token; token auto-generated on first launch and stored in a per-user file (`%LOCALAPPDATA%/StableProjectorz/agent-bridge.token`), or pinned in config. The README is honest about the threat model ("not there to keep out an attacker who already runs code as you; it removes the mode where merely enabling the bridge let any local process or a web page … drive the application").
* **Threading**: listener threads, tool bodies run on the **main thread**, drained from `Update()` (explicitly not `LateUpdate()`, where render passes are scheduled); tools may answer several frames later (screenshot).
* **Tools**: `describe`, `get_app_state`, `get_viewport_screenshot`, `list_generations`, `list_events`, `invoke_event` (fires any registered UI event id with typed parameters). The `StaticEvents` registry gained introspection and `TryInvokeDynamic` that reports failures instead of ignoring them.
* **Known limit** (their README): overlapping screenshots cancel each other (shared `StopAllCoroutines()`), so the busy latch expires after 10 s.

Overcast already exposes one operation registry over HTTP and MCP, which is the better design (operations first, UI second). `IDEA` items worth checking against ours: `describe` with parameter schemas, token file with random secret and loopback binding, async tool results, and a screenshot tool that returns a region.

## 5. Take / effort

| Item | Mark | Effort | Value |
|---|---|---|---|
| Order-of-layers greyscale view, footprint checker highlight, cursor-peek of a view | IDEA | S each | medium-high |
| Barycentric wireframe overlay; non-selected fading wireframe | ALGO/IDEA | S | medium |
| 3D↔UV morph viewport and UDIM grid | IDEA | S-M | medium |
| ID-buffer picking, depth-based pivot picking | IDEA | S | medium |
| Hotkey registry with hover/typing guards | IDEA | S | medium |
| Reduced-resolution mode while the generator runs | IDEA | S | medium (shared GPU) |
| Chrome-ball HDRI and EV merge | ALGO | M | medium |
| Agent-bridge conventions | IDEA | S | medium |
