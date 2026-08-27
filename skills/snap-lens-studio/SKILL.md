---
name: snap-lens-studio
description: >-
  Guide for building Snapchat Lenses (AR) in Snap's Lens Studio — the desktop
  tool for face/world/body/hand tracking, 3D, materials & VFX, scripting
  (JavaScript/TypeScript), SnapML custom ML models, generative-AI authoring,
  UI/audio, physics, and publishing. Use this for ANY Snapchat AR / Lens /
  "filter" work in Lens Studio: creating or debugging a Lens, face filters,
  world effects, SnapML, performance or file-size-budget issues, publishing /
  submission / review, Spectacles Lenses, or Camera Kit integration — and
  whenever you need to know what is possible or find the right Snap doc. Trigger
  on "Snapchat filter/lens", "Lens Studio", ".lsproj"/".esproj", "SnapML", "Spectacles
  Lens", "Camera Kit", face/world tracking in Snapchat, or lens publishing —
  even when the user says "filter" loosely. Prefer this over memory: Lens Studio
  changes fast across 5.x versions, so verify specifics against the official docs
  this skill points to.
license: MIT
---

# Building Snapchat Lenses in Lens Studio

Lens Studio is Snap's free desktop tool for authoring **Lenses** — real-time AR
experiences with tracking, 3D, scripting, and ML. This skill is a router plus a
method: it holds distilled, verified reference material in `references/`, and it
teaches you how to confirm the fast-moving specifics against the official docs
rather than guessing. Lens Studio ships a new version roughly monthly and many
online tutorials are stale, so **the habit of verifying against a live doc or
build is part of the skill, not an afterthought.**

## First, get the words right (they are different products)

- **Filters** = static **2D overlays** on a Snap (frames, color, stickers,
  timestamp/temp/speed, Geofilters). No AR, no tracking, no code. Made on the web
  at create.snapchat.com — **not** in Lens Studio. (On-Demand Geofilters were
  discontinued in 2023.) When a user says "filter," they usually mean a Lens.
- **Lenses** = real-time **AR** built in Lens Studio; **Face Lenses** (front
  camera) and **World Lenses** (rear camera). This is what this skill is about.
- **Spectacles Lenses** = Lenses for Snap's AR glasses — authored in Lens Studio
  but a **distinct target** with its own SDK (Spectacles Interaction Kit, Sync
  Kit) and a **version pin** (see below).

## Rule #1 — pick your delivery target before anything else

The target determines the Lens Studio version, which APIs exist, the performance
model, and which features silently do nothing. Decide this first:

| Target | Lens Studio version | Notes |
| --- | --- | --- |
| **Snapchat** (Face / World Lens) | current mainline — **5.23.x** (5.23.1, Aug 5 2026) | file-size/RAM/FPS budgets; `RemoteServiceModule` for approved APIs |
| **Spectacles (2024)** | **pinned to 5.15.x** (5.15.4) — *not* mainline | thermal/power budget plus a [≤ 25 MB published-Lens cap](https://developers.snap.com/spectacles/get-started/start-building/publishing-lens); SIK/Sync Kit; `InternetModule` (open `fetch`) |
| **Camera Kit** (your own iOS/Android/Web app) | pinned per a **drifting LS↔SDK matrix** — re-check at build | some Lens features are unavailable: Remote Service–enabled Lenses, Licensed Sounds, Scan, VoiceML/TTS, Multi-User, Spatial Persistence. **Ray Tracing: unsupported on Android, supported on iOS.** **Bitmoji IS available**, marked "limited compatibility" — see `interactivity-audio-text-ui-integrations.md` |

> Verify the current version and the Spectacles pin at
> [ar.snap.com/download](https://ar.snap.com/download) (mainline) and
> [ar.snap.com/spectacles](https://ar.snap.com/spectacles) (the 2024 device)
> before relying on either — these numbers move, and the *docs* surface does not
> carry them. ⚠️ The Spectacles setup docs say "Download latest Lens Studio" and
> name no version; that does **not** lift the 5.15.x pin. Updating mid-project
> can break Spectacles-2024 pushes (roll back to 5.15.x), and the binary→YAML
> graph conversion introduced in 5.22 is **forward-only**.

## Build-a-lens methodology

1. **Choose the delivery target** (above) — it sets the version and constraints.
2. **Set performance budgets before importing anything.** The cheapest
   optimization is the asset you never imported. (`references/performance-and-publishing.md`)
3. **Start from the closest template**, not a blank project.
4. **Build the scene** on the SceneObject + Component model. Cameras: perspective
   for 3D, orthographic for UI; mind layers + render targets.
   (`references/fundamentals-and-workflow.md`)
5. **Import and right-size assets;** push heavy assets to SnapML / Remote Assets —
   they do **not** count against the ~8 MB Lens budget.
   (`references/assets-physics-and-collaboration.md`)
6. **Add the lightest tracking that works** for the effect.
   (`references/face-tracking-and-effects.md`, `references/world-body-and-tracking.md`)
7. **Script behavior:** `OnAwake` = self-configuration only; `OnStart` =
   cross-component wiring (this ordering prevents the most common null-reference
   bug). Prefer TypeScript class components; keep per-frame `Update` lean.
   (`references/scripting.md`)
8. **Add interactivity, UI, audio.** (`references/interactivity-audio-text-ui-integrations.md`)
9. **Preview**, then **test on real devices** via Preview Lens pairing; watch the
   on-device Bug overlay (SIZE / FPS / RAM / LAT).
10. **Optimize to budget** and run the QA stress tests (rapid tapping,
    hold-record, carousel select/deselect, camera swap).
11. **Configure Project Info, publish,** pass the 3-stage review, distribute via
    Snapcode / Lens Link.

## Reference map — load the specific file for exact APIs, budgets, property names

Keep this SKILL.md as the router; open the reference for the facts (exact type
strings, numeric budgets, component names) rather than answering from memory.

| Developer intent | Open |
| --- | --- |
| Project setup, editor panels, cameras/render targets, preview, pairing, publish workflow | `references/fundamentals-and-workflow.md` |
| JS/TS scripting, events, lifecycle, modules, Behavior, Tween | `references/scripting.md` |
| Face attachment/mesh/**face landmarks**/expressions/retouch, face ML effects | `references/face-tracking-and-effects.md` |
| World/surface, body, hand, marker, Landmarker/location tracking, segmentation | `references/world-body-and-tracking.md` |
| Materials/shaders, particles/VFX, lighting, the render pipeline | `references/materials-rendering-vfx.md` |
| Custom ML models (SnapML) + generative-AI authoring tools | `references/snapml-and-generative-ai.md` |
| UI, text, audio, tween/animation, multiplayer (Connected Lenses), networking, Camera Kit, Spectacles | `references/interactivity-audio-text-ui-integrations.md` |
| Asset/texture import pipeline, physics, version control/collaboration, testing & debugging | `references/assets-physics-and-collaboration.md` |
| File-size/RAM/FPS budgets, profiling, optimization, publishing, review policy | `references/performance-and-publishing.md` |

## Snapchat screen furniture — lens-UI safe zones (device-verified, Jul 2026)

Coordinates below are ScreenTransform anchors (x, y in -1..1, y up); screen %
measured from the top of a portrait phone frame.

- **Left rail (carousel open — the state where users first try a lens)** —
  measured from a real device screenshot (iPhone, Jul 2026): Bitmoji + search
  (y 8–12%), the active lens's icon tile (y ~14–18%), favorite heart (y ~20–24%),
  share arrow (y ~24–28%), view-count eye (y ~28–31%); all x < ~12%. With the
  carousel closed this collapses to the lens-info chip (y 8–18%, x 0–40%).
  **Left-side controls must start at ≥ ~33% from top (anchor top ≤ 0.34).**
- **Right rail** — flip/add-friend on top, then expandable tools (flash,
  adjustments, music, HD, chevron): y 8–34%, x > ~85%. Right-side controls
  (sliders) should start below ~35% from top (anchor top ≤ 0.3) if x > 0.85.
- **Bottom** — lens carousel tiles y ~73–83%, category tabs ~85%, action bar
  ~92%: keep everything above anchor y ≈ −0.45.
- **Safe interactive band** — anchors y ∈ [−0.45, 0.34], full width except
  x > 0.85 which is safe only below y ≈ 0.3.
- **Tool UI must never bake into captures** — render it via a camera whose
  renderTarget is a RenderTarget assigned to `scene.liveOverlayTarget`
  (runtime Scripting API, on `ScriptScene` via `global.scene`). The overlay output is excluded from captured photos/videos by
  design. Do NOT rely on hiding UI in `SnapImageCaptureEvent` /
  `SnapRecordStartEvent` handlers — it fails on device and the UI ends up in
  the captured photo.

## How to search the Snap docs effectively

- **Canonical host is `developers.snap.com`.** `docs.snap.com/lens-studio/*`
  **308-redirects** there. `lensstudio.snapchat.com` no longer points at the docs —
  it 308-redirects to [easylens.snapchat.com](https://easylens.snapchat.com/)
  (Snap's consumer Easy Lens tool), so old links on that host are dead ends.
  Hardcode `developers.snap.com`.
- **Prefer unversioned `/lens-studio/...` paths.** Versioned copies like
  `/lens-studio/4.55.1/...` are frozen and stale.
- **Scripting API:** `developers.snap.com/lens-studio/api/lens-scripting/...` —
  confirm exact type strings (e.g. `Component.RenderMeshVisual`,
  `Physics.BodyComponent`) before hardcoding them.
- **Check the per-feature cross-surface support badge** (Snapchat / Spectacles /
  Camera Kit) before relying on an effect — many features are surface-gated.
- **Confirm the current version and the Spectacles pin** in the release notes.
- Treat any specific **template name, numeric budget, or count** as
  version-sensitive and verify it against a live page or build.

## Project hygiene & code quality

Lens projects punish clutter more than most codebases, because the deliverable has
a hard ~8 MB cap and a strict RAM budget — dead weight isn't just untidy, it costs
reach and frame rate.

- **Delete dead assets and files.** Unused textures, meshes, scripts, and imported
  packages bloat the project, slow loads, and confuse collaborators — and
  hidden-but-present SceneObjects **still load into memory at runtime** even when
  disabled. If it's not used, remove it; source control means you can always get it
  back.
- **Don't import "just in case."** Bring assets in when they're needed, at the
  resolution they're needed. Right-sizing at import beats compressing later.
- **One job per script; reuse over copy-paste.** Keep Script Components small and
  focused; put shared logic in Script Modules (`require`) and reusable object trees
  in Prefabs instead of duplicating them across the hierarchy.
- **Name things for what they do.** "SceneObject (14)" and "Material (3)" copies rot
  fast; a stranger (or you in a month) should navigate the hierarchy by names alone.
- **Strip scaffolding before shipping:** debug `print`s, the Text Logger, test
  objects, dead code paths, and commented-out blocks.
- **Keep the repo clean:** commit `Assets/`, `Support/`, and the project file;
  gitignore `Cache/`; exclude `Backup/`; use Git LFS for binary formats. (Details in
  `references/assets-physics-and-collaboration.md`.)
- **Do a cleanup pass before every submission**, same as the QA stress tests — walk
  the Asset Browser and hierarchy and justify each item's existence.

## Verification & trust discipline

The reference files are verified against live docs, but a few facts are
**explicitly flagged as unconfirmed** (e.g. RAM 120 vs 150 MB across pages,
hand-gesture count, a 320×320 icon claim, a Face Retouch render-order tip). Treat
flagged items as open questions — re-confirm against the official doc before
asserting them, and prefer the doc link over a remembered number. When installing
third-party Custom Components / Asset Library items, read the component's real
source before trusting it.

## Cross-cutting gotchas (the highest-value traps)

- **Nothing renders** → the object's `layer` isn't in the camera's **Layers** set,
  the object/parent is disabled, or it's on the wrong **render target**.
- **UI shows in preview but is missing from the captured Snap** → it's on the
  **Overlay Target**, which is excluded from captures by design.
- **A feature silently does nothing** → it's unsupported on your target
  (`fetch`/InternetModule on Snapchat Lenses, some Connected-Lenses APIs, Ray
  Tracing / Classification on unsupported devices, Remote Service–enabled Lenses
  / Licensed Sounds / Scan / VoiceML on Camera Kit). Check the support badge on the
  feature's own doc page — **absence from Camera Kit's unsupported list is not
  evidence of exclusion**, that list only enumerates what is broken. Bitmoji is the
  worked example: absent from the matrix, but its badge marks Camera Kit
  Android/iOS/Web "limited compatibility".
- **Null reference on startup** → you wired cross-component references in
  `OnAwake` instead of `OnStart`.
- **Spectacles push breaks after a Lens Studio update** → you left the 5.15.x pin.

## Render-to-texture (off-screen camera) traps — verified the hard way, Jul 2026

Any pipeline where a camera renders into its own RenderTarget that another pass
samples (paint masks, feedback loops, mirrors) hits ALL of these:

- **Off-screen cameras are dependency-pruned.** A camera whose RenderTarget only
  feeds a *custom material texture parameter* (bound in editor OR via runtime
  `pass.tex = rt`) is silently SKIPPED — no clear, no draws; the RT reads as
  stale GPU garbage that looks like old frames. The scheduler only traces
  FIRST-CLASS references: another camera's `maskTexture`, an RT `inputTexture`,
  or the scene output chain. Route the RT through one of those (e.g. compositing
  via `camera.maskTexture` — which is usually the better architecture anyway).
- **Off-screen cameras do NOT adopt the device aspect.** `Device Property: All`
  is ignored for cameras rendering to a texture — the ortho viewport stays at
  the authored `Aspect` (typically 1.0, square), so a "Full Frame" ScreenRegion
  under it covers only a centered square of the RT. On a real phone the effect
  appears "locked between two y values." FIX: copy the aspect from a
  screen-rendering camera at runtime, every frame (`offCam.aspect =
  screenCam.aspect;` — runtime `Camera.aspect` is writable). Test in the
  device-simulation preview (19.5:9), not just the default 9:16 — the bug is
  invisible at aspects close to square coverage.
- **`camera.maskTexture` aspect-FITS the mask** (letterboxes, never stretches),
  and runtime `camera.aspect` reads the PREVIEW aspect, not the target device's
  — so camera-mask compositing of an RT is aspect-fragile end to end. For
  full-screen masks, composite in the SHADER instead: sample the mask RT at raw
  `screenUV` (raw UV sampling stretches mask space onto screen space by
  construction, immune to every aspect mismatch). If nothing else references
  the mask RT, add a scheduling anchor: a tiny ortho camera on its own layer
  rendering ONE invisible drawable (e.g. a corner ScreenImage at alpha 0.004)
  into the main target with `maskTexture` = the mask RT. A camera with zero
  drawables is itself pruned — the invisible drawable is load-bearing.
- **A "full-frame" quad writing into an off-screen RT may not cover the RT.**
  ScreenRegion sizing and the camera projection can disagree about the frame at
  device aspect, leaving unwritten RT regions — which raw-UV compositing then
  exposes as garbage rectangles (uninitialized memory that can't be painted or
  erased). If the quad's shader works purely from `screenUV` (screen-position
  input), geometry size is free: OVERSIZE the quad's anchors hugely (e.g.
  ±4 instead of ±1) so it rasterizes across the entire RT under any aspect
  disagreement. Screen-position-driven shaders are immune to quad geometry.
- **Editing a .graphShader while Lens Studio runs breaks its materials.** The
  material re-imports against a stale pass: editor property panels still LOOK
  fine, but the pass renders nothing (effect silently dead). Recovery: save the
  project and restart Lens Studio (project open re-imports in dependency order);
  alternatively make a real content change to the .mat after the graph
  recompiles. Prefer staging shader edits on disk while LS is closed.
- **Shader math that depends on aspect** (round brushes, circles) should take
  the aspect as a script-fed uniform from a screen camera, not derive it from
  the RT's `texSize` — the RT aspect and the display aspect can differ.
- **Adding a feedback/RT pass via the MCP editor API — three traps (Jul 2026):**
  (1) **`asset-graphql duplicateAsset` is broken for graph (code-node) materials** —
  it does NOT reassign the GUID, producing two `.mat` files that share the source's
  GUID (a collision that renames the in-memory asset and breaks references). Don't
  clone code-node materials this way. Instead HAND-AUTHOR the `.mat`: copy the
  source, give it a fresh Material GUID + PassInfo GUID, keep the SAME
  `Pass: !<reference> <id>` (the shader pass is shared across materials), set your
  texture/param bindings. Writing the file auto-imports it; LS assigns its own GUID
  in the auto-generated `.meta` — then edit the `.mat` body's Material GUID to MATCH
  the `.meta` before any reload. A second material can even reuse ONE shader for two
  modes via a sentinel uniform (e.g. `paramB >= 9` selects a stabilize branch).
  (2) **Camera `renderLayer` rejects `scene-graphql setProperty(valueType:LAYER_SET_MASK)`**
  ("Invalid layer set mask value"). Set it via Editor API:
  `cam.renderLayer = Editor.Model.LayerSet.fromMask(N)`. New layer bits must be
  REGISTERED first (`scene.layers.add(freeUserLayerId)`, find via
  `LayerId.forEachUser` + `!layers.contains`). SceneObject layers, by contrast, DO
  accept raw masks via `setLayers`.
  (3) **Duplicating a camera or adding a layer can silently corrupt a post-effect
  camera's `renderLayer`** to a render-almost-everything mask — it then renders your
  new RT pass too and crushes the scene background to black. After any scene surgery,
  re-assert each post-effect camera's `renderLayer` to EXACTLY its own effect layer.
- **Contra the note above: GLSL-string edits to a code node DO hot-reload** via
  `PreviewPanelTool refresh` (no restart) — verified repeatedly. Live
  `asset-graphql`/Editor-API property edits also apply on refresh. Only new asset
  *files* and final persistence need `project.save()`, and any hand-authored assets
  or feedback-loop wiring MUST be re-verified after a full restart (publish loads
  from disk, so a working live preview is necessary but not sufficient).

## Silent failures in custom code-node shaders — verified the hard way, Aug 2026

A custom code node that fails to compile produces **no magenta, no console error
and no thrown exception**. The material silently falls back and the Lens keeps
rendering while quietly doing nothing.

**The detector — run it after every scripted shader edit:**

```js
material.passInfos[0].getPropertyNames()
// healthy → the real parameter list
// dead    → exactly ["mainColor"]
```

A failed compile is **cached**, so refreshing will not clear it. Open a different
project and reopen this one.

**What kills a shader silently**

| cause | note |
|---|---|
| A **reserved GLSL keyword** used as a variable name | `flat`, `sample`, `filter`, `input`, `output`, `layout`, `varying`, `attribute`, `uniform`, `precision`, `invariant`, `discard`, plus the memory qualifiers `coherent`, `volatile`, `restrict`, `readonly`, `writeonly` |
| A **backslash anywhere** in a spliced body | scripted edits that write `//` comments into a YAML block scalar can emit `\` and break the parse |
| Calling a helper the harness does not declare | a helper present in one project's preamble is not present in another's |
| int/float mixing | `pow(x, 3)` instead of `pow(x, 3.0)` |

### mediump precision — legal code, wrong pixels

A custom code node is **not guaranteed `highp`**. Desktop preview is, so none of
this reproduces on a workstation; assume it on device.

- **A literal outside the precision's range is undefined.** `float best = 1.0e9;`
  as a nearest-neighbour sentinel: GLSL ES guarantees only about 2^14 and fp16
  caps at 65504. Derive the sentinel from the metric's real upper bound instead.
- **Non-representable constants shift `floor()`.** `floor(k * 0.2)` for an
  every-fifth test: 0.2 is not representable in binary, so at `k = 25` the product
  is 4.9987793 in fp16, `floor` drops to 4, and the case silently never fires at
  exactly the setting a user reaches by turning the control up. Add a small
  epsilon and justify why it cannot promote a wrong case.
- **Hash helpers overflow into NaN.** `fract(sin(dot(p, vec2(127.1, 311.7))) * k)`
  with `p` in pixel coordinates reaches ~660,000 — far past mediump — so
  `sin(inf) = NaN`, and because `NaN * 0 = NaN` the result poisons the entire
  frame, not just the intended region. Bound the input inside the helper itself:
  `dot(mod(p, 289.0), vec2(127.1, 311.7))`. One edit fixes every call site, and
  inputs already below the modulus are bit-identical.

## Controls that are alive in code and dead on screen

A control can read perfectly in source and still do nothing a user can perceive.
Three distinct causes, none of which code review finds:

**1. A threshold against an ABSOLUTE value has a subject-dependent usable range.**
An effect that swept a threshold against a per-pixel selector built from raw
luminance worked on bright subjects and was dead on dark ones: the selector never
rose high enough for the threshold to enter range until the control was ~58% of
the way up, so the bottom half did nothing — and *how much* was dead depended on
the subject. Normalising luminance through a fixed window is **not** a fix; a
subject occupying a narrow slice of the range still compresses into a fraction of
0..1. Drive selection from a quantity that is uniformly 0..1 for every subject (a
hash is), and use the absolute value only as a small bias.

**2. A MULTIPLY has luminance-proportional authority.** An effect applied as
`col * tint` produces a delta that scales with how bright the subject already is.
Measured across subject luminance 0.06–0.67, one such effect varied 11× in
strength and fell below the visible threshold at the dark end — it simply did less
for some users than others. Blend toward an **additive** term as the input
darkens; additive authority does not vanish as colour approaches zero. Keep that
term luma-neutral (+R, ~0 G, −B) where it must add warmth without reading as
lightening.

**3. min/max over a THRESHOLDED scan is a discrete measurement.** Locating a
subject by scanning N samples and taking min/max quantises the answer to 1/N, and
produces three symptoms at once:
- a **cliff** — the subject leaves the scan rows and the effect vanishes entirely
- **banding** — a whole range of subject sizes returns the identical answer
- **popping** — a sub-1% subject movement flips the result by tens of percent, so
  the control cannot be held at a setting and the effect jitters on live video

Fix all three with **weighted moments**: use the mask value itself as the weight
and compute a centroid and standard deviation over one grid.

```glsl
// continuous in the input: no threshold, no seed row, no lattice
float w = 0.0, sx = 0.0, sxx = 0.0;
for (int i = 0; i < N; i++) {          // constant bounds, always
    float m = clamp(mask.sample(uv_i).r, 0.0, 1.0);
    w += m;  sx += m * x_i;  sxx += m * x_i * x_i;
}
float cx = sx / max(w, 0.0001);                       // centre
float sd = sqrt(max(sxx / max(w, 0.0001) - cx * cx, 0.0));
float halfWidth = sd * 1.85;   // sd of a filled ellipse ≈ radius/2
```

**The test that finds all three: render the control's extremes and diff them.**
Reading the shader catches none of them, because the code looks reasonable in
every case. A mean absolute difference below roughly **5/255** between a control's
two ends means the control is decorative — one real example measured 4.8/255
between its extremes and was rewritten to drive a different quantity entirely.

## Testing traps that produce false results

- **A still-image preview source freezes time and redraw.** Shader-clock
  animation, runtime parameter writes and animated grain all appear **dead** while
  a script readback shows the writes landing correctly. Verify anything animated
  or interactive against a **video source or a device**, never a still.
- **"TypeScript is not compiled" renders as a perfect no-op.** A freshly opened
  project can log this; the Lens then never runs, so no material parameters are
  ever set and the preview looks exactly like a broken shader. Recompile first,
  and give every Lens a `print()` start line naming its bound inputs — **no prints
  means the Lens is not running, and the shader is not the suspect.**
- **A bare `TapEvent` under `touchBlocking = true` can receive nothing.** Bind
  `TouchStartEvent` as well behind an idempotent handler, or do not claim touches
  on a tap-only Lens (a tap has no direction, so there is nothing to protect from
  the carousel).
- **Log collection that resets the Lens** wipes interaction state before it
  collects. To observe output after an injected gesture, read the log file
  directly instead.
- **Editing a project file while the project is OPEN loses the edit.** The editor
  writes its in-memory copy back on save, silently discarding the change. Edit
  project-level files only while the project is closed, then reopen.

## Glossary

- **Face vs World Lens** — front-camera vs rear-camera AR.
- **SnapML** (bring-your-own ML model runtime) vs **Generative-AI Suite** (hosted
  generators, which often emit SnapML assets).
- **Perspective vs Orthographic** cameras — 3D depth vs flat UI.
- **Live / Capture / Overlay** render targets — seen live / recorded into the Snap
  / native-res overlay excluded from the Snap.
- **RemoteServiceModule** (approved Remote APIs, Snapchat) vs **InternetModule**
  (open `fetch`, Spectacles / Camera Kit only).
