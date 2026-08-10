# Face Tracking & Face Effects

Face tracking is the backbone of most Snapchat Lenses: it locates a user's head, eyes, mouth, and 93 landmark points in real time, then lets you attach 3D objects, warp geometry, drive expression-based effects, retouch skin, and swap or generate whole faces. This reference covers the Lens Studio primitives (Head Binding, Face Mesh, Face Expressions), the no-code and scripted trigger paths, the 2D/3D attachment and distortion effects, retouch/beautify, multi-face handling, and the newer ML/GenAI face features. Because Snap ships effects with **non-uniform cross-surface support**, always check the per-feature support badge (Snapchat / Spectacles / Camera Kit) on each doc page before you rely on an effect.

> **Version & platform framing (important).** Lens Studio **5.23.1** (Aug 5, 2026) is current; like 5.22 it targets **SPECS 27 (the new-generation Spectacles), explicitly NOT Spectacles (2024)**. **Spectacles (2024) is pinned to Lens Studio 5.15.x** ("5.15 will be the last anticipated Lens Studio update for Spectacles (2024)"). Because 5.22 auto-converts binary graph files to YAML, **5.22 projects are not backward-compatible to the 5.15 line**. See the [release notes](https://ar.snap.com/lens-studio-v5). The canonical docs host is **developers.snap.com** (`docs.snap.com` and `lensstudio.snapchat.com` redirect there); prefer the unversioned `/lens-studio/features/ar-tracking/face/...` paths.

## What is possible

- **Attach 2D and 3D objects to the head** — glasses, hats, masks — that follow head position and rotation, with an occluder that hides geometry as the head turns.
- **Query 93 face landmarks** and per-face metadata from script for custom logic.
- **Drive a 3D Face Mesh** that mimics the user's expression, and read **51 face expression weights** (blendshapes) to auto-drive rigged characters or trigger events.
- **Fire events** on facial actions (mouth open, brows raised, smile, kiss) with no code (Behavior) or in script.
- **Texture, distort, retouch, and recolor** the face — Face Mask, Face Inset, Face Liquify/Stretch, Face Retouch, Eye Color.
- **Support up to two faces** simultaneously.
- **ML/GenAI effects** — Face Swap, Face Morph, and the GenAI Suite (Face Generator, Head Generator, Face Animator).

## Key components, assets & APIs

- **Head Binding** (`Component.Head`) — the foundation of face tracking; attaches child objects to the tracked head and exposes landmark data. Ships with a child **Face Occluder** (a 3D face model that hides attached geometry as the head turns) and a **Face Index** property (0-based) selecting which detected face to follow. **Attachment Point Type** offers the 13-option set, verbatim: **Head Center, Candide Center, Triangle Barycentric, Face Mesh Center, Left Eyeball, Right Eyeball, Mouth Center, Chin, Forehead, Left Forehead, Right Forehead, Left Cheek, Right Cheek**. *Supported on Snapchat, Spectacles, and Camera Kit.* [Head-attached 3D objects](https://developers.snap.com/lens-studio/features/ar-tracking/face/head-attached-3d-objects)
- **Face Landmarks** — **93 tracked points**, each referenced by an index, accessed through the Head Binding script API. Positions are returned in **screen space** ("Like Object tracking, the position of the points are in screen space"); depth/rotation come from the Head Binding. **Current API: `Head.onLandmarksUpdate`.** ⚠️ `getLandmark(id)` and `getLandmarks()` are **deprecated since Lens Scripting 305**, and `getFacesCount()` is **deprecated since Lens Scripting 306** (verified against the bundled `StudioLib.d.ts`, not the web docs — the live doc page still shows the old pattern). For presence, use `SceneObject.isEnabledInHierarchy` on the object holding the Head component; assign Face Index on the Head component to check a specific face. [Face landmarks](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-landmark)
- **Face Mesh** (`Assets.FaceMesh`) — a 3D mesh asset mimicking the user's expression in real time, and the **sole provider of Face Expression weights**. Geometry toggles: **Face, Eye, Mouth, Skull, Ear**. *Supported on Snapchat, Spectacles, and Camera Kit.* [Face mesh](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-mesh) · [Face expressions](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-expressions)
- **Face Expressions** — the system tracks **51 expressions**; read them via `faceMesh.control.onExpressionWeightsUpdate`, which passes a `NamedValues` object with parallel `names`/`values` arrays. Full enum: [`Built-In.Expressions`](https://developers.snap.com/lens-studio/api/lens-scripting/enums/Built-In.Expressions.html).
- **Scripted Face Events** — all inherit `FaceTrackingEvent` (carries `faceIndex`): `FaceFoundEvent`/`FaceLostEvent`, `MouthOpenedEvent`/`MouthClosedEvent`, `BrowsRaisedEvent`/`BrowsLoweredEvent`/`BrowsReturnedToNormalEvent`, `SmileStartedEvent`/`SmileFinishedEvent`, `KissStartedEvent`/`KissFinishedEvent`. [Events reference](https://developers.snap.com/api/lens-studio/Classes/Events)
- **Behavior component** — no-code Trigger = **Face Event** → event type → Response. [Behavior](https://developers.snap.com/lens-studio/lens-studio-workflow/adding-interactivity/behavior)
- **Face Swap component** — installable ML component (Asset Library → Custom Components); API `start(): Promise<Texture>`, read-only `sourceFacePresent`/`targetFacePresent`, events `onSwapStarted`/`onResultReady`/`onError`/`onSourceFacePresent`/`onTargetFacePresent`. [Face swap](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-swap)

## How to build it

**Attach a 3D object to the head.** Add a **Head Binding** component, then parent your 3D Mesh Visual under it and pick an **Attachment Point Type** (e.g. `Forehead` for a hat). Keep the child **Face Occluder** active so the object is hidden correctly as the head turns. For a second face, duplicate the object and increment **Face Index**.

**Read landmarks from script.** Subscribe to `onLandmarksUpdate` rather than polling — the callback only fires while a head is tracked, which removes the "valid data" guard entirely:

```javascript
//@input Component.Head headBinding
script.headBinding.onLandmarksUpdate.add(function (landmarks) {
  const noseTip = landmarks[30]; // screen-space vec2
});
```

For presence, bind `FaceFoundEvent` / `FaceLostEvent` (both carry `faceIndex`), or read `SceneObject.isEnabledInHierarchy` on the Head component's object.

<details><summary>Deprecated polling pattern (still in the live docs — do not copy)</summary>

```javascript
// getFacesCount() deprecated @ Lens Scripting 306; getLandmark()/getLandmarks() @ 305
if (script.headBinding.getFacesCount() < 1) return;
const landmarksPosition = script.headBinding.getLandmark(30);
```
</details>

**Add a Face Mesh and read expressions.** Add via Scene Hierarchy `+ > Face Mesh` (auto-binds to the head) or Asset Browser `+ > Face Mesh`. Keep the Face Mesh object **always active** — it is the only source of expression data. Read weights via the mesh control:

```javascript
// @input Assets.FaceMesh faceMesh
script.faceMesh.control.onExpressionWeightsUpdate.add(function (expWeights) {
  var idx = expWeights.names.indexOf(Expressions.MouthClose);
  if (idx >= 0) print('MouthClose weight: ' + expWeights.values[idx]);
});
```

Rig blendshapes whose names match tracked expressions are **auto-driven**, so a correctly named character rig animates with no extra code.

**Apply 2D texture effects.** Use **Face Mask** to map a texture onto the face: assign a Texture plus an Opacity Texture (white shows, black hides), edit anchor points with Symmetrical Mode / Detect Face / Reset Points, and tune Draw Mouth / Use Original Face. Use **Face Inset** to copy a face region (Left Eye, Right Eye, Mouth, Nose, Face) and re-map it with Inner/Outer Border Radius, Subdivision Count, Source Scale/Offset, Flip, and a Blend Mode. [Face mask](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-mask) · [Face inset](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-inset)

**Recolor eyes.** Add **Eye Color** in Color mode (change iris color) or Texture mode (overlay a square texture for cat-eyes/reflections); settings include Main Material, Blend Mode, Alpha, Face Index, and **Eye To Render**. From script: `script.eye.mainPass.baseColor = new vec4(r,g,b,a)` (RGBA 0–1). *Snapchat + Camera Kit only — NOT Spectacles.* [Eye color](https://developers.snap.com/lens-studio/features/ar-tracking/face/eye-color)

**Distort the face.** From the Distort template: **Face Liquify** (spherical bulge/pucker; Radius, Intensity; multiple points), **Face Stretch** (drag mapped points with a strength slider), plus **Face Inset**. [Distort template](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-templates/distort)

**Drive distortion from script** (the doc page lists Inspector properties only, but both components are fully scriptable — verified in `StudioLib.d.ts`). This is what turns a commodity static warp into an expression-driven toy:

```javascript
// LiquifyVisual — radius and intensity are writable.
// BOTH default to 1. intensity is a COEFFICIENT, not an amount:
// 1.0 = neutral, <1 warps inward, >1 warps outward. Authoring 0 as "off"
// ships a lens that sucks every resting face inward.
liquify.intensity = 1.0 + expressionWeight * GAIN;
liquify.radius = 1.0;

// FaceStretchVisual — named features, intensity strictly in (-0.5, 2)
stretch.setFeatureWeight('Wide', w);
const cur = stretch.getFeatureWeight('Wide');   // getter exists
const names = stretch.getFeatureNames();        // @since 193 — resolve, never hardcode
stretch.addFeature(name); stretch.removeFeature(name); stretch.clearFeatures();
```

- The editor default-names features `Feature 0`, so a hardcoded string is a silent no-op — always resolve via `getFeatureNames()` at `OnStart`.
- **`faceIndex` lives in two different places.** `FaceStretchVisual` carries it directly. **`LiquifyVisual` has no `faceIndex` field** — a Liquify Point is a Head Binding *plus* a LiquifyVisual on one object, so the second-face duplicate means editing the **Head Binding**.
- **Hard crash, not a slowdown: more than 10 liquify effects per render order per face index crashes.** Assert at startup.
- Liquify and Stretch **warp whatever renders behind them**, so any text you want in the capture needs a `renderOrder` above every warp visual or the lens drags its own caption through the distortion.

**Retouch/beautify.** Add **Face Retouch** for Soft Skin, Teeth Whitening, Eye Sharpening, Eye Whitening, and ML Retouch; the Inspector exposes Face Index, Auto Mode, Auto Type, and per-feature Intensity sliders ("use low intensities for a more subtle effect"). *Snapchat + Camera Kit only.* [Face retouch](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-retouch)

**Swap a face (ML).** Install **Face Swap** from Asset Library → Custom Components; place it in a Scene Object with a Screen Transform under the Orthographic Camera. **Swap On** modes: Start (testing), **Face Present** (default; fires after ~5 frames of detection), Manual (`start()`). Optional Real-Time Mode is resource-heavy. *Supported on Snapchat + Camera Kit only — the [Face Swap page](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-swap) badge does not list Spectacles.*

## Best practices

- **Check the support badge per feature.** Head Binding & Face Mesh cover Snapchat + Spectacles + Camera Kit; Face Swap, Face Retouch, and Eye Color are Snapchat + Camera Kit only (per each feature's [doc](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-swap) badge).
- **Keep the Face Mesh object active** whenever you need expressions or auto-driven blendshapes — it is the only provider ("the face mesh object itself must always remain active to receive information about the face").
- **Subscribe to `Head.onLandmarksUpdate` instead of polling landmarks**, and use `FaceFoundEvent`/`FaceLostEvent` for presence — the polling API is deprecated (305/306) and the callback only fires while a head is tracked, so the stale-data guard disappears.
- **Always reset your rig on `FaceLostEvent`.** Nothing fires a "finish" event when tracking drops mid-effect, so a warp driven by an expression envelope stays pinned at maximum forever until the face returns.
- **Keep every deformation round, above the jawline, and taut** if you want a charming read rather than an insulting one — round-and-inflating parses as cartoon physics, jowly or squinting parses as a weight joke.
- **Use the built-in Face Mesh UV map for External/custom meshes** (External Face Mesh drives Face Morph and exposes External Scale, Expression Multiplier, Mapping UV, added blend shapes, and a built-in↔custom transition).
- **Prefer static/one-shot ML modes and test on older devices.** ML effects (Face Swap Real-Time, Face/Head Generator, ML Retouch) are the heaviest.
- **Match rig blendshape names to expression names** to get free auto-drive instead of manual wiring.

## Common pitfalls

- **Skull/Ear geometry appears untextured** — those Face Mesh geometry parts require the **UV1** channel in the material; UV0 is for the face-only mesh.
- **`getLandmark` returns garbage or errors on the first frame** — it is also deprecated (305). Move to `Head.onLandmarksUpdate`; if you must keep the old call, guard it with `SceneObject.isEnabledInHierarchy` on the Head object rather than the also-deprecated `getFacesCount()` (306).
- **A neutral face is subtly sucked inward** — `LiquifyVisual.intensity` was authored to `0` as "off". It is a coefficient whose identity is **`1.0`**; below 1 warps inward.
- **`setFeatureWeight` silently does nothing** — the feature name is wrong. The editor default-names features `Feature 0`; resolve real names with `getFeatureNames()` at `OnStart`. Weight must also sit strictly inside `(-0.5, 2)`.
- **All 51 expression weights read zero with no error** — the Face Mesh object was deactivated. It is the sole provider and must stay active; hide it with a fully transparent Face-only material instead of disabling it.
- **Expression weights are always zero** — the Face Mesh object was deactivated; it must stay active.
- **Only one face is affected in a two-person shot** — the cap is **two faces**; duplicate the object and increment **Face Index** (0 = first, 1 = second). The two-face limit and its supported platforms are documented on the page below. [Multiple faces](https://developers.snap.com/lens-studio/features/ar-tracking/face/working-with-multiple-faces)
- **Assuming an effect ships everywhere** — Eye Color, Face Retouch, and Face Swap all have no Spectacles support (Snapchat + Camera Kit only). Read the badge.
- **Assuming version compatibility across Spectacles generations** — the 5.2x mainline is SPECS 27 only; Spectacles (2024) stays on 5.15.x, and 5.22's YAML conversion is forward-only.
- **Unverified tip:** a "chain Render Orders sequentially with Face Swap to avoid one-frame lag" note was **not found** on the live Face Retouch page — do not treat it as documented.

## ML & GenAI face effects

- **Face Morph** (template) — uses External Face Mesh; ships Face Mesh Geometric Shapes (Cube/Sphere + Weight slider + Face Reprojection script) and Face Reprojection - Character (Pin to Mesh, texture transition, Expression Controller). [Face morph](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-templates/face-morph)
- **Face Generator** (GenAI Suite) — text prompts up to **500 chars** (Enhanced/Advanced) or **200 chars** (Original; Original cannot combine text + image). Preview times: Enhanced ≤5 min, Advanced ≤20 min, Original ≤15 min; training ≤~2 hrs. Covers age/gender/animal/character morph/accessories/hairstyles/beauty/food/shapes. Snapchat and (limited) Spectacles. [Face ML generation](https://developers.snap.com/lens-studio/features/genai-suite/face-ml-generation)
- **Head Generator** (GenAI Suite) — AI-generates a 3D head driven by expressions; an image reference yields the highest quality. [Head morph generation](https://developers.snap.com/lens-studio/features/genai-suite/head-morph-generation)
- **Face Animator** (GenAI) — animates a face image with the user's expressions. [Face animator](https://developers.snap.com/lens-studio/features/genai-suite/face-animator)

## Go deeper

- [Face effects overview (catalog)](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-effects-overview) — 2D (Face Image, Face Texture, Face Image Picker Texture) and 3D attachment starting points.
- [Head-attached 3D objects](https://developers.snap.com/lens-studio/features/ar-tracking/face/head-attached-3d-objects) — Head Binding, anchor points, occluder, Face Index.
- [Face landmarks](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-landmark) and [Face mesh](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-mesh) / [Face expressions](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-expressions) — the tracking + expression pipeline.
- [Working with multiple faces](https://developers.snap.com/lens-studio/features/ar-tracking/face/working-with-multiple-faces) — the 2-face cap and Face Index workflow.
- [Face swap](https://developers.snap.com/lens-studio/features/ar-tracking/face/face-swap) and [Face ML generation](https://developers.snap.com/lens-studio/features/genai-suite/face-ml-generation) — the version-sensitive ML/GenAI features.
- [Release notes](https://ar.snap.com/lens-studio-v5) — confirm the LS version and Spectacles pinning before shipping.
