# Interactivity, Audio/Text/UI & Advanced Integrations

This reference covers the layer of a Lens that turns a static effect into something a user can touch, hear, share, and connect to the outside world: screen-space UI, text (2D and 3D), audio and audio-reactive effects, animation and tweening, multiplayer via Connected Lenses, and the network paths (approved Remote APIs, InternetModule, Remote Service Gateway) plus the three delivery targets that change what's allowed (Snapchat, Camera Kit, Spectacles). The through-line worth internalizing: **the same Lens behaves differently depending on where it runs.** Open-internet `fetch` works on Spectacles and Camera Kit but not in a published Snapchat Lens; a long list of APIs silently no-op inside Connected Lenses; and Spectacles and Camera Kit are each pinned to specific Lens Studio versions. Design for the target, not just the feature.

Version baseline: latest Lens Studio is **5.23.2** (Aug 17, 2026), preceded by 5.23.0 (Jul 28, 2026) ([download](https://ar.snap.com/download)).

## Table of contents
- [UI: screen space, Screen Transform, widgets](#ui-screen-space-screen-transform-widgets)
- [Text and 3D Text](#text-and-3d-text)
- [Audio and audio-reactive effects](#audio-and-audio-reactive-effects)
- [Animation and tweening](#animation-and-tweening)
- [Connected Lenses (multiplayer / shared AR)](#connected-lenses-multiplayer--shared-ar)
- [Networking: Remote APIs, InternetModule, Remote Service Gateway](#networking-remote-apis-internetmodule-remote-service-gateway)
- [Delivery targets: Camera Kit and Spectacles](#delivery-targets-camera-kit-and-spectacles)
- [Bitmoji / GenAI Suite / Snap3D](#bitmoji--genai-suite--snap3d)
- [Hand-rolled UI, hints, persistence & capture-safety — verified the hard way](#hand-rolled-ui-hints-persistence--capture-safety--verified-the-hard-way)
- [Version cheat-sheet](#version-cheat-sheet)
- [Go deeper](#go-deeper)

## UI: screen space, Screen Transform, widgets

Screen-space UI is fundamentally separate from 3D content because it uses a different camera and coordinate system. **2D/UI content renders under an Orthographic Camera; 3D content under a Perspective Camera.** The bridge is the **Screen Transform** component, which positions and anchors a 2D element and must be a child of another Screen Transform or of an Orthographic Camera, with its render layer set to the Orthographic layer ([Screen Transform properties](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/2d/screen-transform-properties)). Anchors and Bounds are edge positions expressed in **normalized parent coordinates from -1 to 1**, with **Basic** and **Advanced** modes; the scripting class is [`ScreenTransform`](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.ScreenTransform). This normalized, camera-scoped model is why a widget that "should" be on screen renders nowhere when its hierarchy or render layer is wrong.

### Key components & APIs
- **Screen Transform** — anchoring/positioning of 2D elements; class `ScreenTransform`.
- **Canvas Component** — alternative screen-space container ([canvas component](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/2d/canvas-component)).
- **Screen Region + Device Simulation** — respect safe zones/notches across devices ([screen region](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/2d/screen-region-device-simulation)).
- **UI Widgets** (Custom Components; all require a valid Screen Transform hierarchy) ([UI widgets](https://developers.snap.com/lens-studio/features/ui/ui-widgets)):
  - **UI Button** — fires `onPressDown` (first frame pressed), `onPressUp` (first frame released), and `onPress` (every frame it is being pressed). Press animations: **Bounce, Squish, Tween, Transform**. Enable **`Interactable`** so the widget responds to input — practically, the widget won't react unless Interactable is on (the doc frames it as a setting to enable rather than a literal "required" flag).
  - **UI Toggle**, **UI Slider** (value range 0–1), **UI Color Picker**, **UI Panel**, plus **Camera Roll Widget** and **Image Carousel**.

### How to build it
1. Add an **Orthographic Camera** (or use the Screen Transform hierarchy the template provides).
2. Parent your UI SceneObjects so each has a **Screen Transform** whose parent is another Screen Transform or the Orthographic Camera; set the render layer to Orthographic.
3. Drop in a **UI Button** (or other widget), enable **`Interactable`**, and wire behavior to `onPressDown` / `onPress` / `onPressUp` in a script.
4. Use **Screen Region + Device Simulation** to keep controls out of notches/rounded corners on real devices.

### Common pitfalls
- **Widget does not respond to touch** — it is not under an Orthographic Camera / valid Screen Transform hierarchy, its render layer is not Orthographic, or `Interactable` is not enabled.
- **A UI Panel above a widget swallows touches** — the panel intercepts input before it reaches the widget beneath it. Reorder or resize the panel.

## Text and 3D Text

Lens Studio ships flat text (**2D Text**, **Rich Text**, and **Shareable Text Edits** — [2d-text](https://developers.snap.com/lens-studio/features/text/2d-text), [rich-text](https://developers.snap.com/lens-studio/features/text/rich-text), [shareable-text-edits](https://developers.snap.com/lens-studio/features/text/shareable-text-edits)) and true geometry via **Text3D**, which matters when text needs depth, lighting, and to sit convincingly inside a 3D scene.

**Text3D** converts text into a 3D object with built-in **extrusion** settings. Add it via **Scene Hierarchy `+` → Text3D**; it auto-generates a **`Text3D Default Material`** with distinct sections for **Front, Back, Out, and Inner edge** so each face can be styled independently. The **`Editable`** field enables keyboard editing and auto-adds an interaction component that opens the on-screen keyboard. Scripting class: [`Text3D`](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.Text3D) ([3d-text feature](https://developers.snap.com/lens-studio/features/text/3d-text)). Text3D supports non-Latin fonts, but treat the exact supported-language list (e.g., Arabic/Farsi/Chinese/Japanese) as **unconfirmed** against the current doc page — verify before relying on a specific script.

## Audio and audio-reactive effects

Audio in a Lens is both playback and a signal source. **`script.audio.play(loops)`** plays a clip — pass **`1`** to play once and **`-1`** to loop indefinitely. The recommended source format is **MP3, mono, under 15 seconds** (short mono clips keep Lens size and latency down) ([playing audio](https://developers.snap.com/lens-studio/features/audio/playing-audio)).

- **Spatial audio** — three effects shape how a sound is perceived in 3D: **Distance Effect** (attenuation vs. listener distance), **Directivity Effect** (orientation-based attenuation relative to an Audio Listener), and **Position Effect** (L/R stereo panning, best experienced on headphones).
- **Mix to Snap** records the Lens's audio and suppresses the microphone during capture, so recorded Snaps carry the Lens soundtrack instead of ambient mic noise.
- **Audio-reactive visuals** — the **Audio Analyzer** drives visuals from amplitude/frequency. Live microphone input comes from **`MicrophoneAudioProvider.getAudioFrame`** — but note this is **disabled inside Connected Lenses** (see the disabled-APIs list below). VoiceML / speech modules are also available.

## Animation and tweening

Lens Studio supports four animation types: **Transform**, **Skeletal** (keep it under **100 joints** for performance), **Blend Shape**, and **Vertex** ([3D animation](https://developers.snap.com/lens-studio/assets-pipeline/3d/animation/3d-animation)). The important historical shift: **prior to Lens Studio 5.0.10, animations used the Animation Mixer; from 5.0.10 on, the current system is the Animation Player.** **`AnimationPlayer`** consumes **AnimationClips** and exposes play/stop/resume plus animation events; the legacy **[`AnimationMixer`](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.AnimationMixer)** methods remain documented for old projects but are deprecated ([animation player](https://developers.snap.com/lens-studio/features/animation/animation-player)). The editor warns on deprecated APIs (the specific claim that warnings began at LS 5.8.0 is **unverified** — keep the concept, not the version number).

**Tweening** is for lightweight, code-driven property animation without authoring clips. Lens Studio bundles **Tween.js** with a wrapper: add **`+` → Scripts → Tween Manager** (`TweenManager`) once, then attach per-object **TweenType** scripts (move/scale/rotate/color) ([tween manager](https://developers.snap.com/lens-studio/lens-studio-workflow/adding-interactivity/tween-manager), [tween example](https://developers.snap.com/lens-studio/examples/lens-examples/tween)).

## Connected Lenses (multiplayer / shared AR)

Connected Lenses let multiple users share one AR experience. Modes are **Remote** and **Colocated**, supporting both synchronous and asynchronous play, with a hard **limit of 64 participants per session** ([Connected Lenses overview](https://developers.snap.com/lens-studio/features/connected-lenses/connected-lenses-overview)). State is kept in sync by the **Sync Framework**, a set of scripting helpers layered over a RealtimeStore ([sync framework](https://developers.snap.com/lens-studio/features/connected-lenses/connected-lenses-templates/sync-framework)):

- **`SessionController`** via **`global.sessionController`** — exposes `getLocalUserId()` and `getLocalConnectionId()`.
- **`SyncEntity`** via **`new SyncEntity(script)`** — wraps a RealtimeStore for one object.
- **Storage properties** — `addStorageProperty()`, `setPendingValue()`, the values `currentValue` / `pendingValue` / `currentOrPendingValue`, `notifyOnReady()`, and change callbacks `onAnyChange` / `onRemoteChange` / `onLocalChange`.
- **Events** — `sendEvent()` / `onEventReceived`.
- **Ownership** — `tryClaimOwnership()` / `doIOwnStore()`.
- **Instantiation** — **`Instantiator`** + **`InstantiationOptions`** for spawning synced objects.
- **Identity** — **`userId`** is tied to a Snapchat account and **persists**; **`connectionId`** is tied to a single device connection within the session. Use `userId` for cross-session persistence, `connectionId` for "this device, right now."

**APIs disabled inside Connected Lenses** (they silently do nothing, which is a frequent source of "works solo, breaks in multiplayer" bugs): `UserContextSystem.requestBirthdate` / `requestBirthdateFormatted` / `requestCity`; dynamic **Text** read-back; `ProceduralTextureProvider.getPixels`; `DepthTextureProvider.getDepth`; `DeviceTracking.hitTestWorldMesh` / `raycastWorldMesh` / `getPointCloud`; `DeviceLocationTrackingComponent.distanceToLocation`; `ScanModule`; `OutputPlaceholder.data`; **`MicrophoneAudioProvider.getAudioFrame`**; `TensorMath.textureToGrayscale`; `LocalizationSystem.getLanguage`.

## Networking: Remote APIs, InternetModule, Remote Service Gateway

Networking is where target choice bites hardest. **From Lens Studio 5.9, `fetch` / `performHttpRequest` / `createWebSocket` / `createWebView` moved from `RemoteServiceModule` to the new `InternetModule`.** Already-public Lenses keep working on `RemoteServiceModule` until they are re-published with 5.9+ ([remote service module](https://developers.snap.com/lens-studio/features/remote-apis/remote-service-module)).

- **`InternetModule`** — the API reference states it is **"Available only on Spectacles and Camera Kit"**, and the class was introduced at **Lens Scripting v305**. Methods: `fetch` (standard Fetch API — importantly, it **does NOT reject on 4xx/5xx**, so always check `response.status`), `createWebSocket`, `performHttpRequest`, `createWebView`, and `makeResourceFromUrl` / `makeResourceFromBlob` which produce a `DynamicResource` for use with RemoteMediaModule ([InternetModule](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.InternetModule.html)).
- **Version annotation detail** — on the `RemoteServiceModule` side, the moved methods are `@deprecated` ("This method has been moved to InternetModule") and annotated as of **Lens Scripting v308** (`fetch` / `createWebSocket` / `performHttpRequest` / `createWebView`), with **`createWebViewOptions` since v313** ([RemoteServiceModule](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.RemoteServiceModule.html)). So: the InternetModule *class* arrived at **v305 (LS 5.9)**, but cite **v308** for the RSM deprecations and **v313** for `createWebViewOptions` — not v305 for all of them.
- **Critical constraint** — open-internet `fetch` is **not** available in standard Snapchat-published Lenses (Spectacles + Camera Kit only). A Snapchat Lens must use the approved **Remote API** path.

**Approved Remote APIs via `RemoteServiceModule`** — the sanctioned way for a Snapchat Lens to talk to a backend ([remote service module](https://developers.snap.com/lens-studio/features/remote-apis/remote-service-module)):
1. `global.RemoteApiRequest.create()` → set its `endpoint` → `script.remoteServiceModule.performApiRequest(req, cb)`. `performApiRequest` exists **since Lens Scripting v164**; `subscribeApiRequest` (streaming) **since v288**; `createAPIWebSocket(endpoint, params)` opens WebSockets to authorized services.
2. APIs must be registered/approved via an **API Spec** at `my-lenses.snapchat.com/apis/` → get a **Spec ID** → import the generated RemoteServiceModule asset. Prebuilt specs (AccuWeather, OpenAI/ChatGPT, Snapchat Places, etc.) are in the Asset Library.

**Best practices for Remote APIs** (documented guidance): keep to **≤3 concurrent calls** (ideally 1); keep responses **under 800 KB**; expect high latency and show loading content; and **do not fetch images dynamically** — pre-author them as Remote Assets and load via RemoteMediaModule.

## Delivery targets: Camera Kit and Spectacles

### Camera Kit (Lenses in your own app/website)
**Camera Kit** is a cross-platform SDK that runs the Snapchat AR engine and your Lenses inside your own iOS/Android/Web app, with no Snapchat install required ([Camera Kit home](https://developers.snap.com/camera-kit/home)). Web SDK workflow (npm `@snap/camera-kit`): `bootstrapCameraKit({ apiToken })` → `createSession({ liveRenderTarget })` (an `HTMLCanvasElement`) → `session.setSource(mediaStream)` → `lensRepository.loadLens(lensId, lensGroupId)` → `session.applyLens(lens)` → `session.play()`. Lenses are uploaded via the Camera Kit dashboard and grouped by **Lens Group ID** ([web for beginners](https://developers.snap.com/camera-kit/integrate-sdk/web/guides/camera-kit-web-for-beginners)).

- **Remote APIs with Camera Kit** — the **host app answers the Lens's `RemoteApiRequest`s directly**, bypassing Snap servers; you implement a `RemoteApiService` keyed to the Spec ID. **Launch data (≤3 KB)** seeds Lens state on `applyLens`, and the Lens reaches endpoints via `script.api` ([remote API on Camera Kit](https://developers.snap.com/camera-kit/integrate-sdk/web/guides/remote-api)).
- **Version compatibility** — Camera Kit pins Lens Studio versions to SDK versions ([compatibility matrix](https://developers.snap.com/camera-kit/ar-content/lens-studio-compatibility)); verified rows include **LS 5.18.x → Mobile 1.46.x / Web 1.14.x**, **LS 5.17.x → Web 1.13.x (Mobile n/a)**, LS 5.16.x → Mobile 1.45.x / Web 1.12.x, LS 5.13.x → Mobile 1.43.x / Web 1.9.x, down to LS 4.36.x. Older-LS Lenses keep running in newer Camera Kit. **This matrix drifts — re-fetch it at build time.** ⚠️ **Re-verified 2026-09-07: the matrix still tops out at LS 5.18.x — there is no row for 5.19 through 5.23.** A Lens bound for Camera Kit should therefore be authored on **5.18.x or lower** until Snap publishes newer rows; a 5.23 Lens has no documented SDK pairing. Nor is there an LTS pairing to fall back on — every window on the [Camera Kit LTS page](https://developers.snap.com/camera-kit/getting-started/lts) had closed by 2025-09-16.
- **Unsupported on Camera Kit** (verified against the [compatibility matrix](https://developers.snap.com/camera-kit/ar-content/lens-studio-compatibility), 2026-08-10; unchanged 2026-09-07) — **Mobile:** Licensed Sounds, Remote Service-enabled Lenses, Scan, Spatial Persistence, VoiceML/Speech recognition/Text-to-speech; **Ray Tracing is unsupported on Android only — iOS IS supported**; **Multi-User Services** "requires inquiry". **Web:** all of the above plus Face True Depth, Location AR, Ray Tracing and Multi-User Services outright, with Lens Hints "coming soon".
- ⚠️ **Bitmoji IS available on Camera Kit — do not read its absence from the compatibility matrix as exclusion.** That matrix lists only *unsupported* features, so absence means nothing. The authoritative signal is the support badge on the [Bitmoji overview](https://developers.snap.com/lens-studio/features/bitmoji-avatar/overview): Snapchat ✔ and Spectacles ✔ outright, and Camera Kit **Android, iOS and Web all marked limited compatibility** — "This feature may have limited compatibility and may not perform optimally." Plan for it, but budget testing time and keep a fallback: "limited" is Snap's own word. Verified 2026-08-10, unchanged 2026-09-07.

### Spectacles (AR glasses)
Spectacles (2024) is **pinned to the Lens Studio 5.15.x line**, described as "the last anticipated Lens Studio update series for Spectacles (2024)." The 5.15.x series runs through **5.15.4** (live). **Do not target Spectacles 2024 with mainline 5.2x** ([Spectacles](https://ar.snap.com/spectacles), [v5.15.4](https://ar.snap.com/download/v5-15-4), [older versions](https://ar.snap.com/older-versions)).

- Frameworks unique to Spectacles: **Spectacles Interaction Kit (SIK)**, **Spectacles UI Kit**, and **Spectacles Sync Kit** (Connected Lenses for glasses), plus WebXR, ASR/speech, and enhanced camera access. Spectacles share the core LS API set, but some APIs are wearable-only and some mobile APIs don't run on-device.
- **`InternetModule` open-internet `fetch` works on Spectacles** (unlike standard Snapchat Lenses).
- **Remote Service Gateway (RSG)** — requires **Lens Studio v5.10.1+** and **Spectacles OS v5.062+** ([RSG](https://developers.snap.com/spectacles/about-spectacles-features/apis/remoteservice-gateway)). **Snap-hosted:** DeepSeek (Chat Completions with R1 reasoning) and **Snap3D** (text→3D). **Externally hosted:** OpenAI (Chat Completions, Image Gen/Edit, TTS, Realtime) and Google Generative AI (Gemini, Imagen, Lyria). Setup: install the **Remote Service Gateway** package + **Token Generator** plugin (Windows → Remote Service Gateway Token), then enter per-service tokens in **`RemoteServiceGatewayCredentials`**; tokens are account-scoped and non-expiring. Per LS 5.15.4 notes, RSG now supports **separate tokens per platform** (Snap/Google/OpenAI). Combined with the 5.15.x cap, the effective Spectacles-2024 working range is roughly **5.10.1–5.15.x**.

### Dual Camera (custom component, source-verified on LS 5.23.2, Sep 2026)

The Asset Library custom component **"Dual Camera"** (id `1LuqTdMCONaAiEUhzg36Dq`) installs a TypeScript
component and a **`Reverse Camera Texture`** asset. Read its source before designing around it: at `onAwake`
it swaps the *provider inside that texture asset* - a neutral proxy first, then the bundled **mock** texture
in the editor, the real reverse camera on a supporting device, or its fallback elsewhere - and assigns the
asset to each placeholder visual's `mainPass.baseTex`. `isSupported` is a Promise: in the editor it resolves
`!debug` (the "Fallback Preview" toggle), on device `deviceInfoSystem.supportsDualCamera`. `fallbackMode`
defaults to **MediaPicker, which pops a picker UI on unsupported devices** - set `None` (and handle the
absence yourself) unless that is what you want.

Measured limit: **binding the Reverse Camera Texture into a graph material's texture parameter and sampling
it in a code node produced NaN in the preview** - the whole pass dropped and the camera showed through -
while the same shader with the camera texture bound drew correctly (forced-open composite, mean 96/102/145
vs the untouched 147/137/135). Show the feed through the component's placeholder path (a standard Image
material) and mask or composite around that image; do not sample the asset in a code node, at least in
preview. Device behaviour was not tested.

**The placeholder path, measured (LS 5.23.2, Sep 2026).** A full-screen `Image` whose material
`baseTex` is the Reverse Camera Texture, parented under a `MaskingComponent` square whose
`cornerRadius` is half its size (a circle), drew **empty** in the editor - bound at start, bound again
after `isSupported` resolved, and re-bound a few frames later. The provider swap never reached the
material binding. The package's bundled **Mock** texture bound directly to the same material drew at
once. Recipe that works: expose the Mock as a script input and bind it when
`global.deviceInfoSystem.isEditor()` is true, the reverse texture otherwise; keep the mask object
disabled until `isSupported` resolves true and let the post-effect draw a fallback inside the hole;
drive the mask's `anchors` and `cornerRadius` from script (the Editor API exposes only `cornerRadius`,
in pixels, and children are clipped to the rect). Give that camera's `renderTarget` the main Render
Target with a `renderOrder` after the post-effect camera, not the Overlay Target, or the feed is
missing from the captured snap. The real reverse feed is still untested in this design: only a phone
can show it, so say so in the hand-off.

## Bitmoji / GenAI Suite / Snap3D

- **Bitmoji** — the **Bitmoji 3D** component is a no-code way to download an avatar for the user, friends, or My AI; **Bitmoji Head** is for face Lenses. Animate via clips, Mixamo, or the **Bitmoji Animation** plugin (which uses AnimationPlayer — the same driver behind the mixer→player migration). The **Bitmoji Suite plugin** adds outfits, **Props**, and animation ([bitmoji 3d](https://developers.snap.com/lens-studio/features/bitmoji-avatar/bitmoji-3d), [bitmoji suite](https://developers.snap.com/lens-studio/features/bitmoji-suite/overview)).
- **GenAI Suite** landed with **Lens Studio 5.0**: **Snap3D / 3D Asset Generation** (text/image→3D), **Head Morph**, material/texture generation, and **3D Capture**. Runtime **Snap3D** text→3D is exposed on **Spectacles via the Remote Service Gateway** ([3D asset gen](https://developers.snap.com/lens-studio/features/genai-suite/3dag-generation), [GenAI Suite blog](https://ar.snap.com/blog/genai-suite-lens-studio-5.0)).
- **Camera Kit caveat** — Bitmoji **is** available on Camera Kit (Android/iOS/Web), but Snap marks all three **"limited compatibility … may not perform optimally"** on the [Bitmoji overview](https://developers.snap.com/lens-studio/features/bitmoji-avatar/overview) badge. It is *not* in the Camera Kit unsupported list. Ship it if you want it, but test on device early and keep a non-Bitmoji fallback path. Verified 2026-08-10, unchanged 2026-09-07.

## Hand-rolled UI, hints, persistence & capture-safety — verified the hard way

Everything below was confirmed by building custom lens controls (a drag color wheel, a joystick color picker, sliders, toggles) against LS 5.22.1, Jul 2026. The UI Widgets above are the paved road; this is what you hit when you hand-roll controls out of `ScreenTransform` + `Image`.

### Native localized hints beat baked text
Snap ships **pre-localized hint strings** that auto-translate to the viewer's language — better than a text box you author in English, which is dead weight for a global audience.
- Add via script: `script.getSceneObject().createComponent("Component.HintsComponent")`, then `showHint("lens_hint_tap", -1)` (duration `-1` = until hidden) and `hideHint(id)`. Full ID table lives in the `HintsComponent` docstring (`lens_hint_tap`, `lens_hint_swap_camera`, `lens_hint_open_your_mouth`, …).
- A startup hint can also be set with **no scripting** in Project Info → Lens Hints.
- **Trade-off:** the wording is Snap's and fixed ("Tap!"). Custom copy means a Text component — English-only unless you localize it yourself.
- ⚠️ **Native hints render in Snapchat's own UI layer, NOT the lens render target.** A preview-panel screenshot will NOT show them even when they fire correctly. Verify with the boolean return of `showHint(...)` or a `print`, never by screenshot.

### Overlay camera = the only reliable way to keep UI out of the captured Snap
Controls that help the user (pickers, sliders) usually should **not** bake into the recorded photo/video. The dependable mechanism is the **Overlay Target**, which by design "can not be added to the final captured Snap".
- Fastest path: **Scene Hierarchy `+` → Overlay Camera** (`OverlayCameraObjectPreset` via the API) — it creates its own Render Target and wires the Scene's Overlay Target for you. Parent your UI under it.
- Do **not** hide UI inside `SnapImageCaptureEvent` / `SnapRecordStartEvent` handlers — that fails on device and the UI lands in the capture anyway.
- ⚠️ **The overlay renders at the device's NATIVE resolution, so its aspect differs from the lens render target.** Measured: a **large** control (a near-full-width color wheel) visibly distorted into an ellipse on the overlay, while a **small compact** control (a ~15%-width joystick) showed no perceptible distortion. Overlay is safe for compact HUD elements, risky for large UI unless you redo layout math against the overlay's resolution.
- If you do polar/aspect math on overlay-parented UI, read width/height from the **Overlay** Render Target, not the main one, or hit-testing will be skewed.

### Hit-testing and moving widgets by hand
- `ScreenTransform.containsScreenPoint(touchPos)` → is this touch on my control. Test the **highest-priority control first** (a toggle button before the panel under it) and latch which control was grabbed on `TouchStartEvent`, so a drag that wanders off the control keeps controlling it.
- `screenPointToLocalPoint(pos)` → normalized **-1..1 inside that transform**; `screenPointToParentPoint(pos)` → same in the *parent's* space, directly usable as anchor values.
- Move an element by rewriting anchors: `st.anchors = Rect.create(left, right, bottom, top)`. Parent an indicator (knob/handle) to the control it rides on so its anchor space *is* that control's space.
- ⚠️ **A newly created SceneObject defaults to layer 1 and will silently not render** under a UI camera on another layer. Set it explicitly (`setLayers(mask)`, e.g. `1048576`) — the object exists, is enabled, and draws nothing until you do.
- Aspect: local coords are normalized to a rect that is usually **not pixel-square**, so `atan2`/`length` on raw local coords yields skewed angle/radius. Correct with the rect's pixel aspect (anchor spans × render-target resolution) before any polar math.
- ⚠️ **Y-convention clash:** baked textures are authored **y-down** (image space) while `ScreenTransform` local space is **y-up**. Polar math against a baked wheel/dial needs `atan2(-y, x)` (and the inverse mapping negated) or the picked color won't match the pixel under the finger.

### Owning touch without breaking Snapchat
- `global.touchSystem.touchBlocking = true` makes the lens handle touches full-screen; pair with `enableTouchBlockingException("TouchTypeDoubleTap", true)` so double-tap camera-flip still works.
- Vertical drags are relatively safe to claim (the lens carousel is horizontal), making "swipe up/down anywhere" a good gesture for a 1-D parameter — often better UX than an on-screen slider, at zero screen cost.
- ⚠️ **A bare `TapEvent` under `touchBlocking = true` can receive nothing** — verified by injecting taps that produced no state change at all. If you claim touches, bind `TouchStartEvent` as well behind an idempotent handler so one physical tap toggles once. Better still, **do not claim touches on a tap-only Lens**: a tap has no direction, so there is nothing for the carousel to steal, and claiming costs the user their carousel navigation for no benefit.

### First-run tour: the carousel clip has to show the mechanic
A tap-cycled lens whose first frame is its default state looks inert in the auto-generated preview
clip (about 3 s), and a reviewer will send it back for that. Pattern that shipped (Sep 2026): when
the persistent `"<lens>_saved"` flag is absent, auto-advance the state on a timer (presets every
0.9 s; a window opening 0.3 s in and moving at 1.7 s and 3.0 s) with the native `lens_hint_tap`
hint showing, and end the tour on the first tap. Never write the saved flag during the tour: only
the user's own tap counts as a choice.

### Persist settings so a retake doesn't reset the lens
By default every retake re-runs the lens: parameters snap back to defaults and first-run hints re-appear. Fix with **persistent storage**, which survives retakes and sessions on that device.
- `global.persistentStorageSystem.store` is a `GeneralDataStore`: `putFloat/getFloat`, `putBool/getBool`, `putString/getString`, vec2/3/4, arrays.
- ⚠️ **There is no `has(key)`.** Getters return a zero-value when absent (`getFloat` → 0, `getBool` → false), so write an explicit `"<lens>_saved"` boolean flag and branch on it — that flag doubles as the "first-ever run" test for whether to show the hint at all.
- Save on `TouchEndEvent` rather than every frame of a drag.
- **Remember the last state the user chose, not only settings.** A recurring review ask: after a
  snap is taken and scrapped, the lens must come back where it was. Save the chosen preset index, or
  open/closed plus position, on every tap; on start restore it and skip the tour and the hint.
  Verified in the editor: a lens reset restored a preset index and `open at (0.75, 0.25)`.

### Input you do NOT get
- **No raw accelerometer.** No acceleration API for shake detection; the accessible motion signal is a `DeviceTracking` component in **Rotation** mode whose rotation you diff per frame. Untestable in Lens Studio (no gyro in the editor) — it only proves out on device.
- **No access to Snapchat's native camera zoom.** A lens cannot read or drive it, so any "zoom" must live in your own shader/transform.
- Lens Studio's bundled preview **videos carry embedded touch recordings** that fire real `TapEvent`s. If your lens cycles modes on tap it will appear to cycle by itself — switch the preview to a **static image** before concluding you have a bug.

## Version cheat-sheet
- **LS 5.0** — GenAI Suite (Snap3D / 3D Asset Gen, Head Morph).
- **LS 5.0.10** — AnimationMixer → AnimationPlayer.
- **LS 5.8.0** — deprecation warnings for old APIs *(exact version unverified; keep the concept)*.
- **LS 5.9** — `fetch` / `performHttpRequest` / `createWebSocket` / `createWebView` moved RemoteServiceModule → **InternetModule** (Spectacles + Camera Kit only). InternetModule class = Lens Scripting **v305**; RSM deprecations annotated **v308**; `createWebViewOptions` **v313**.
- **LS 5.10.1 / Spectacles OS 5.062** — Remote Service Gateway floor.
- **LS 5.15.x** (through 5.15.4) — Spectacles 2024 pinned line.
- **Camera Kit** — pin LS to the published matrix (top row 5.18.x ↔ Mobile 1.46.x / Web 1.14.x as of 2026-09-07; no rows for 5.19–5.23); re-fetch at build time.

## Go deeper
- [UI Widgets](https://developers.snap.com/lens-studio/features/ui/ui-widgets) and [Screen Transform properties](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/2d/screen-transform-properties)
- [Text3D](https://developers.snap.com/lens-studio/features/text/3d-text) · [Playing Audio](https://developers.snap.com/lens-studio/features/audio/playing-audio) · [Animation Player](https://developers.snap.com/lens-studio/features/animation/animation-player) · [Tween Manager](https://developers.snap.com/lens-studio/lens-studio-workflow/adding-interactivity/tween-manager)
- [Connected Lenses overview](https://developers.snap.com/lens-studio/features/connected-lenses/connected-lenses-overview) · [Sync Framework](https://developers.snap.com/lens-studio/features/connected-lenses/connected-lenses-templates/sync-framework)
- [Remote Service Module / Remote APIs](https://developers.snap.com/lens-studio/features/remote-apis/remote-service-module) · [InternetModule API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.InternetModule.html) · [RemoteServiceModule API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.RemoteServiceModule.html)
- [Camera Kit home](https://developers.snap.com/camera-kit/home) · [Camera Kit ↔ Lens Studio compatibility](https://developers.snap.com/camera-kit/ar-content/lens-studio-compatibility) · [Camera Kit Remote API](https://developers.snap.com/camera-kit/integrate-sdk/web/guides/remote-api)
- [Spectacles](https://ar.snap.com/spectacles) · [Remote Service Gateway](https://developers.snap.com/spectacles/about-spectacles-features/apis/remoteservice-gateway) · [Download / versions](https://ar.snap.com/download)
