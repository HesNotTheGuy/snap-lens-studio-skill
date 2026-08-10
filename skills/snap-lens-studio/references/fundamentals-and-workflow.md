# Lens Studio Fundamentals & Project Workflow

Lens Studio is Snap's free desktop authoring tool (Windows/macOS) for building **real-time AR experiences** — called **Lenses** — that run in Snapchat, on Spectacles, and (via Camera Kit) inside third-party mobile/web apps. It is not a "Filter" tool: Filters are static 2D overlays made on the web, while Lenses are code-driven AR with face/body/hand/world tracking, 3D, physics, and machine learning. This file orients you to what Lens Studio can do, its core architecture (SceneObjects + Components, cameras, render targets), and the full idea→published-Lens workflow. Getting the mental model right here — especially the entity/component composition model and the camera/render-target system — is what keeps everything downstream from breaking. ([overview](https://developers.snap.com/lens-studio/overview/getting-started/lens-studio-overview), [home](https://developers.snap.com/lens-studio/home))

## What is possible

Lens Studio covers AR tracking (face/body/hand/world), animation, audio, physics, scripting (JavaScript/TypeScript), **SnapML** (custom ML models), UI, Bitmoji/avatars, generative AI, and an **Asset Library** of reusable prebuilt assets. You publish to Snapchat as **Face Lenses** (front camera) or **World Lenses** (rear camera), or deploy through **Camera Kit** into your own apps. ([overview](https://developers.snap.com/lens-studio/overview/getting-started/lens-studio-overview))

**Current version: Lens Studio 5.23.1** (released 2026-08-05; 5.23.0 on 2026-07-28). The 5.23 line is framed around **SPECS 27**, so check the Spectacles firmware note before updating if you ship to glasses. Headline **5.23** additions: 3D Hand Mesh; Custom Texture Tracking for hands; **SVG import → Vector Composite** assets and **SVG Text** (5.23.1, per-run weight/italic/letter-spacing); Variable Fonts; Vertex Snapping (hold `V`) and 3D Grid Snapping (`G`); Mix To Snap on Video Texture; CRISP compression for Gaussian Splatting (~14x smaller); Copy/Paste Component Properties; a **Script Update Order** panel. Preceding 5.22 additions ([versioned release notes](https://ar.snap.com/lens-studio-v5)):
- **CLAD (Closed Loop Agentic Development)** — an agentic framework that can "one-shot an entire Lens" or invoke individual skills/agents to prototype, iterate, test, debug, move preview panels, check runtime state, optimize, and write Editor/Plugin code.
- **Binary graph files auto-convert to a new YAML format** on project load/import in 5.22+ (text-editable by external editors or AI); old binary graph-editor formats are deprecated.
- **Live JavaScript debugging** with breakpoints (experimental) — supported clients per the 5.22 notes: "VS Code built-in debugger, Cursor built-in debugger, or Any other VS Code fork" — **5.23 adds IntelliJ and WebStorm**, console forwarding, and editing variables while paused; scoped to SPECS contexts (SPECS 27 device, "No Simulation" and "Horizontal" preview panels). Plus **LEAF (Lens Evaluation & Automation Framework)** — a TypeScript integration-testing framework for SPECS Lenses: write test scenarios that simulate user actions, then assert against live scene state (see `assets-physics-and-collaboration.md`). ([v5 page](https://ar.snap.com/lens-studio-v5))
- **Particle Orient Node** (VFX/particle system) consolidates Billboard, Velocity Align, and Look At in one node, plus an Advanced mode for custom forward/up axes and a Rotation Angle input.

> **Spectacles (2024) version pin — critical gotcha, and Snap's own pages will mislead you.** To develop for Spectacles (2024) use the pinned **Lens Studio 5.15.x line** (currently **5.15.4**), *not* the 5.2x mainline: "Lens Studio 5.15.x is the last anticipated Lens Studio update series for Spectacles (2024) and should be used for all subsequent Spectacles releases until further notice."
>
> ⚠️ **The trap:** the Spectacles setup docs are headed **"Download latest Lens Studio"** and name no version — follow that literally and you land on 5.2x, which the 2024 hardware does not run. The pin is stated on the *download* surface, not the *docs* surface. Verified 2026-08-10: [ar.snap.com/download](https://ar.snap.com/download) carries "The following is for SPECS 27, and not Spectacles (2024). Spectacles (2024) users should continue using Lens Studio 5.15.xx," and lists 5.23.1's Supported Platforms as **Snapchat / SPECS 27 / Camera Kit** — Spectacles (2024) is absent. [ar.snap.com/spectacles](https://ar.snap.com/spectacles) separately serves 5.15.4 with Supported Platforms **Snapchat / Spectacles (2024) / Camera Kit**.
>
> Framework docs saying "Lens Studio v5.15 or later" are stating a **floor, not a ceiling** — the only compatibility chart in the Spectacles docs tops out at v5.15. Before upgrading past 5.15.x, confirm on [ar.snap.com/spectacles](https://ar.snap.com/spectacles) that Snap names a newer build for this device. ([compatibility](https://support.spectacles.com/hc/en-us/articles/27749036143380-Compatibility))

## Terminology (do NOT conflate)

- **Filters** = static/simple **2D overlays** on a Snap (frames, color, stickers, timestamp/temp/speed, Geofilters). No AR, no tracking, no code. Made on the web at create.snapchat.com, not in Lens Studio. On-Demand Geofilters were **discontinued in 2023** (submissions/support ended 2023-02-10; delivery stopped 2023-03-31). Never call Lens Studio a "Filter" tool. ([help](https://help.snapchat.com/hc/en-us/articles/7012334010772))
- **Lenses** = real-time **AR** built in Lens Studio; split into **Face Lenses** (front camera) and **World Lenses** (rear camera).
- **Spectacles Lenses** = a distinct target authored in Lens Studio with its own SDK (**Spectacles Interaction Kit / SIK**, **Sync Kit / Connected Lenses**, 6DoF) and the LS 5.15.x pin.

## Key components, assets & APIs

**Editor panels** (dockable; reset via **Window > Default Layout**; open any via the **Window** menu). Core: **Scene** (3D viewport), **Scene Hierarchy** (SceneObject tree; drag to reparent; per-object enable checkboxes; add via `+`), **Inspector** (add/edit/remove Components; Transform always present), **Asset Browser** (all project assets; the older "Resources" name is legacy), **Asset Library** (searchable prebuilt assets to drag in), **Preview**, **Logger** (open via **Window → Utilities → Logger**; log with `print('msg')`), plus component-specific editors (2D, Material, VFX, visual-scripting graphs). ([panels](https://developers.snap.com/lens-studio/lens-studio-workflow/lens-studio-interface/panels), [debugging](https://developers.snap.com/lens-studio/features/scripting/debugging))

**SceneObject + Component model.** Lens Studio uses a **composition (entity/component) model** familiar to Unity devs. ([SceneObject API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.SceneObject.html))
- A **SceneObject** is a node in the hierarchy. It always holds a **Transform** and may hold zero or more Components and child SceneObjects. Key scripting properties: `enabled` (bool), `name` (string), `layer` (LayerSet — determines which Camera renders it). Key methods: `getComponent(type)`, `getComponents(type)`, `createComponent(typeName)`, `getChild(i)`, `getChildrenCount()`, `getParent()`, `getTransform()`, `destroy()`, `copySceneObject()` (shallow) / `copyWholeHierarchy()` (deep). A script reaches its own object via `script.getSceneObject()`. ⚠️ **Deprecated on SceneObject — do not use:** `getComponentCount()`, `getAllComponents()`, `getComponentByIndex()`, `getFirstComponent()`, `getRenderLayer()`/`setRenderLayer()` (use the `layer` property), and the misspelled `isEnabledInHiearchy` (use `isEnabledInHierarchy`).
- **Limits (verified):** global cap of **100,000 SceneObjects**; no limit on children per object, but hierarchy depth cannot exceed **256 levels**.
- A **Component** attaches behavior/visualization to a SceneObject. Base props `enabled`, `sceneObject`; methods `getSceneObject()`, `getTransform()`, `destroy()`. In scripting, access components by **exact type string**, e.g. `sceneObject.getComponent('Component.RenderMeshVisual')`, `'Component.ScriptComponent'`, `'Physics.BodyComponent'`. Prefer the real `Component.*` / `Physics.*` names from the API reference — e.g. the physics body is `Physics.BodyComponent` (not "PhysicsBody"), mesh visuals are `Component.RenderMeshVisual` / `Component.MaterialMeshVisual`. Always confirm a component's exact type string before hardcoding it. ([Component API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.Component.html), [accessing components](https://developers.snap.com/lens-studio/features/scripting/accessing-components))

**Cameras.** The **Camera component** renders the scene using one of two projection types ([camera](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/camera)):
- **Perspective** — realistic depth; uses **Field of View** (Perspective only); has a **Device** property (**None / Physical Aspect / Physical Fov / All Physical**) to match the real device camera. Use for 3D content.
- **Orthographic** — no perspective distortion; uses a **Size** property (viewing-box height) instead of FOV. Standard for 2D Screen Image / Screen Text / UI; the Overlay Camera preset is orthographic.

Key Camera properties: **Layers** (a camera renders only objects whose `layer` is in its set), **Render Target**, **Render Order** ("The lower the Render Order, the earlier that this camera's output texture will be added to the Render Target" — i.e. lower renders first), **Mask Texture**, **Depth Mode**, **Near/Far**, **Clear Color**, **Clear Depth**.

**Render Targets:** **Live Target** (what the user sees live), **Capture Target** (recorded snap output), **Overlay Target** — "the only Render Target that renders at the device's native display resolution" and "Overlay Target can not be added to the final captured Snap." Multiple cameras can share one Render Target for perf, or chain (feed one camera's Render Target as another's input texture). Typical setup: a perspective camera for 3D + an orthographic camera for UI.

**Scenes, Scene Manager & Prefabs.** A **Scene is fundamentally an Object Prefab.** The **Scene Manager** partitions a Lens into logical scenes/prefabs, loads/unloads them, and stacks them via **additive loading**; access via `global.sceneManager`, and loaded scenes can take a `parent`. **Prefabs** are reusable object trees created by dragging to the Asset Browser or right-clicking "Save as Prefab." ([scene manager](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/scene-manager))

## How to build it

1. **Start from a template.** New Project opens a template chooser plus a blank project; each template maps to a tutorial and is the fastest start. (The exact current 5.22 template names could not be verified against a live chooser — treat any specific names as illustrative and check the live chooser.) Beginner path: Built-In AR Effects → Add 3D Content → Customize → Test on Device → Text/Tweens/Interactions → Scripting → Publish. ([first lens](https://developers.snap.com/lens-studio/overview/building-your-first-lens/built-in-ar-effects))
2. **Build the scene.** Add SceneObjects in the Scene Hierarchy, attach Components in the Inspector, set up cameras (perspective for 3D, orthographic for UI), and pull assets from the Asset Browser / Asset Library. Organize with Prefabs and the Scene Manager.
3. **Preview** using the four Preview modes (open more via **Window > Preview**) ([previewing](https://developers.snap.com/lens-studio/lens-studio-workflow/previewing-your-lens)):
   - **Photo mode** — built-in face/world stills or import.
   - **Video mode** — prerecorded clips *with tracking data*, or import. Convert clips with the exact FFMPEG command: `ffmpeg -i my_video.MOV -vcodec h264 -acodec mp2 my_video.mp4`.
   - **Webcam mode** — live webcam.
   - **Interactive Preview** — walk the scene like on-device. Controls: move **W/A/S/D + Q (down) / E (up)** or arrow keys; **rotate** = hold **Shift + directional keys** *or* hold **Right Mouse Button and drag**; **Ctrl + scroll** to zoom; scroll wheel moves the camera forward/back; Lens Reset = **Ctrl+R** (Win) / **Cmd+R** (Mac).
4. **Pair & push to a real device** (needs latest Snapchat + latest Lens Studio) ([pairing](https://developers.snap.com/lens-studio/lens-studio-workflow/pairing-to-snapchat)):
   - Click **Preview Lens** (top-right) → scan the Snapcode in Snapchat, accept, wait for "Device Paired."
   - Press **Preview Lens** again to push the Lens.
   - **Multiple devices/accounts:** down-arrow next to **Send to Snapchat** → **Pair New Account**.
   - **Unpair:** Snapchat **Settings → Additional Services → Manage → Lens Studio → Pair Status → Tap to Unpair**.
   - **On-device profiling:** tap the **Bug** button for **FPS**, **RAM**, **SIZE**, **LAT** (Lens Activation Time), plus a last-push timestamp.
5. **Publish.** Fill in **Project Info** (name, icon, hint, camera facing) → optimize (watch SIZE/RAM/FPS) → click **Publish** (top-right), which opens **My Lenses** (opens Project Info first if unconfigured). The Lens submits and shows **In Review**; on approval it's distributed via a **Snapcode + shareable Link** (download the Snapcode from the `...` menu; manage in the **My Lenses** portal). Sponsored/branded Lenses go through the commercial track. ([submitting](https://developers.snap.com/lens-studio/publishing/submitting/submitting-your-lens), [snapcodes](https://developers.snap.com/lens-studio/publishing/distributing/snapcodes), [publish](https://developers.snap.com/lens-studio/overview/building-your-first-lens/publish))

## Best practices

- **Match projection to content:** perspective for 3D; orthographic for UI/Screen Text/Image. Route crisp text/UI through the **Overlay Target** (native resolution) — remembering overlay content is excluded from captured snaps.
- **Use layers + Render Order deliberately** (lower = drawn first). Share a Render Target across cameras to save performance; chain cameras when one needs another's output as an input texture.
- **Test on real devices**, not just Preview, and watch the on-device **Bug** overlay (SIZE/FPS/RAM/LAT).
- **Organize with Prefabs + Scene Manager**; keep hierarchy depth modest (hard cap 256, object cap 100k).
- **Prefer unversioned `/home` doc URLs**; the canonical host is **developers.snap.com** (docs.snap.com/lens-studio/* issues a 308 redirect there). Older versioned URLs like `/lens-studio/4.55.1/...` are stale.
- **For Spectacles (2024), pin to LS 5.15.x (currently 5.15.4)** and use SIK / Sync Kit — do not chase the 5.2x mainline. ([Spectacles](https://ar.snap.com/spectacles), [compatibility](https://support.spectacles.com/hc/en-us/articles/27749036143380-Compatibility))

## Common pitfalls

- **Nothing renders / object invisible** → the object's `layer` isn't in the Camera's **Layers** set, the object/parent `enabled` is false, or it's on the wrong Render Target.
- **UI text blurry** → rendered through a perspective/live camera instead of the orthographic **Overlay Target**.
- **UI/2D shows in preview but missing from the captured snap** → it's on the **Overlay Target**, which is excluded from final captures by design.
- **Wrong draw order / z-fighting between 3D and 2D** → misconfigured **Render Order** (lower = first) or clashing **Clear Depth**.
- **Can't pair / push fails** → Snapchat or Lens Studio not on latest; re-scan the Snapcode; force-quit Snapchat.
- **Preview video shows no face effects** → the clip lacks tracking data, or isn't H.264 MP4 (convert with the FFMPEG command above).
- **Push to Spectacles fails after updating LS** → you left the 5.15.x pin; roll back.
- **Stale tutorials** → many older/4.55.1 and third-party tutorials predate LS 5's SceneObject / Scene Manager / YAML-graph changes; verify against current developers.snap.com docs.

## Driving Lens Studio over MCP / the Editor API — verified the hard way

- ⚠️ **`model.openProject(new Editor.Path("…/X.esproj"))` switches projects IN-PROCESS.** No app restart: the MCP server, its auth token, and loaded plugins all survive; ~40 s to import. That is the difference between a 40-second and a 3-minute loop when working across a family of lens projects. `setDefaultProject()` / `setEmptyProject()` also exist on `Editor.Model.IModel`.
- ⚠️ **Plugin "Trust and Load" dialogs gate MCP tool registration.** After each app launch the server answers HTTP but returns `Tool not found` until a human clicks through the per-plugin filesystem-permission prompts. The **"Trust External Plugins"** preference did *not* suppress them in 5.22, and grants aren't persisted in project prefs, global `PluginsPreferences`, or the registry. Mitigate by batching work per launch and preferring in-process `openProject` to restarting.
- The MCP auth token is **keychain-backed and stable across restarts** (log: `McpHttpServer::initializeAuthToken] Loaded existing authentication token from keychain`). Default port **50040**; a second concurrent instance takes **50041** with a *different* token — keep one instance open.
- ⚠️ **`metaInfo.lensName` has a working setter but `project.save()` reverts it.** Set the published name in the Publish dialog.
- Camera `renderLayer` rejects a raw layer-set mask through generic setProperty (use the Editor API `LayerSet`), while SceneObject layers do accept `setLayers(mask)`.
- Knowledge-base and generative tools (`QueryLensStudioKnowledgeBase`, `GenerateLensIcon`, …) require being **signed into Snapchat inside Lens Studio** (My Lenses → Login); otherwise they fail with auth/connection errors.
- `PreviewPanelTool action:screenshot` needs LS **5.23+**; on 5.22 use `CapturePanelScreenshotTool` with `pluginId: Snap.Plugin.Gui.PreviewPanel`. Changing the preview source can leave playback **paused** — send `resume` after `setConfig`, and prove motion by diffing two captures rather than trusting one frame.

## Go deeper

- [Lens Studio overview & getting started](https://developers.snap.com/lens-studio/overview/getting-started/lens-studio-overview)
- [Camera, render targets & render order](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/camera)
- [Scene Manager, scenes & prefabs](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/scene-manager)
- [SceneObject scripting API (limits, methods)](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.SceneObject.html)
- [Previewing your Lens (modes & controls)](https://developers.snap.com/lens-studio/lens-studio-workflow/previewing-your-lens)
- [Pairing & pushing to Snapchat](https://developers.snap.com/lens-studio/lens-studio-workflow/pairing-to-snapchat)
- [Submitting & publishing a Lens](https://developers.snap.com/lens-studio/publishing/submitting/submitting-your-lens)
- [Release notes (all 5.x versions)](https://ar.snap.com/lens-studio-v5) · [Spectacles compatibility (5.15.x pin)](https://support.spectacles.com/hc/en-us/articles/27749036143380-Compatibility)