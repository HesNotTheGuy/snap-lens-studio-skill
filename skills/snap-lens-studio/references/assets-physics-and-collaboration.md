# Assets, Physics & Collaboration

This file fills four practical gaps the rest of the skill does not cover in depth: getting content **into** a Lens correctly (the asset pipeline), making things **move and collide** (physics), keeping a project **version-controlled and shareable** across a team, and **testing/debugging** what you built. These are the areas where a Lens either silently bloats past the 8 MB budget, behaves non-deterministically, becomes un-mergeable in git, or ships a bug that only appears on-device. Everything below is verified against live docs at [developers.snap.com](https://developers.snap.com); version-specific facts are flagged inline. Two facts up front, because they drive every decision here: a Snapchat Lens must be **< 8 MB** and use **< 150 MB RAM** ([Performance and Optimization](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide)).

## 1. Asset Pipeline & Import

### What is possible
You import images, 3D models, audio, materials, ML models, and prebuilt components into the **Asset Browser**, and Lens Studio auto-processes them (compression, mipmaps) toward the tight size/RAM budget. Heavy content that would blow the 8 MB cap can be pushed to **Remote Assets** or delivered as a **SnapML** model, neither of which counts against the Lens size.

### Key components, assets & APIs
- **Asset Browser** panel — mirrors the project's `Assets/` folder; import by dragging a file in ([3D Object Import](https://developers.snap.com/lens-studio/assets-pipeline/3d/importing-content/overview)).
- **Asset Library** — Snap's built-in repository (top-left button). Use **Import** for individual assets and **Install** for Custom Components / Plugins / packages, which install into Lens Studio rather than the current project ([Asset Library](https://developers.snap.com/lens-studio/assets-pipeline/asset-library/asset-library-overview)).
- **`.lspkg`** — a package of assets/components; installed via the Asset Library or the Asset Browser `+` button. Contrast with the project file (see §3). Right-click → **Unpack for Editing** to modify bundled resources.
- **Remote Assets** / **`RemoteReferenceAsset`** — downloaded on demand and **do not count toward Lens size** (up to 10 MB each; 500 MB org storage) ([Remote Assets](https://developers.snap.com/lens-studio/features/lens-cloud/remote-assets-overview)).

### How to build it
- **Images/textures:** import a JPG or PNG (JPG is smaller but lacks transparency). Select the texture in Asset Browser and set compression in the **Inspector**: **Optimize for performance** (default; faster load, lower RAM) vs **Optimize for size** (smaller download, lower quality). Enable **mipmaps** for FPS unless the texture is always 1:1/magnified ([Compression overview](https://developers.snap.com/lens-studio/publishing/optimization/overview), [Texture Optimization](https://developers.snap.com/lens-studio/publishing/optimization/texture-optimization)).
- **Right-size resolution:** no texture may exceed **2048×2048**; drop 2048→1024→512 based on on-screen size. Screen-space textures should match occupied pixels (max 720×1280) ([Performance guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide)).
- **3D models:** import **FBX** or **glTF** to carry meshes, materials, skeletons, animations, and textures; **OBJ** carries meshes/textures only (no animation). After import, the Scene Hierarchy shows the object, its **bones**, and an **Animation Player** component ([3D Import overview](https://developers.snap.com/lens-studio/assets-pipeline/3d/importing-content/overview)).
- **Audio:** drag the file into the Asset Browser to create an **Audio Track** asset ([Audio Tracks](https://developers.snap.com/lens-studio/features/audio/audio-track-assets)).

### Best practices
Prefer FBX/glTF for custom exports and OBJ for downloaded models (highest compatibility). Push large/optional content to Remote Assets and fetch it one asset at a time. Keep models under **100k triangles** (60k if skinned), rigs under ~100 joints, and animations < 10 s ([Performance guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide)).

### Common pitfalls
- **sRGB vs linear:** the live docs do not expose a per-texture sRGB/linear toggle by that name — color textures are treated as sRGB and data maps (normal/roughness) must be authored as linear/non-color in your DCC tool. Confirm the current Inspector options rather than assuming a checkbox ([Texture Optimization](https://developers.snap.com/lens-studio/publishing/optimization/texture-optimization)).
- Exact **audio formats** (mp3/wav/ogg) are not enumerated in the docs; only that stereo is downmixed and 1-channel is read. Verify before relying on a specific container ([Audio Tracks](https://developers.snap.com/lens-studio/features/audio/audio-track-assets)).
- Greyed-out (packaged) assets can't be edited until you **Unpack**.

## 2. Physics

### What is possible
Rigid-body simulation with gravity, collisions, constraints, layer-based collision filtering, and raycasts/shape-casts. Supported on **Snapchat, Spectacles, and Camera Kit** ([Physics Assets and Components](https://developers.snap.com/lens-studio/features/physics/physics-component)).

### Key components, assets & APIs
- **Physics World** + **WorldSettings** — each world runs its own simulation (gravity, default matter, collision matrix, 30–240 Hz rate). A non-customizable default world always exists.
- **`Physics.BodyComponent`** — turns a SceneObject into a collider. The **Dynamic** checkbox makes it a moving rigid body; unchecked = **static** collider.
- **Collision shapes** — Sphere, Box, Capsule, Cone, Cylinder, Mesh (optional convex hull); Levelset for hair/cloth only.
- **`Physics.ColliderComponent`**, **`Physics.Filter`** (layer-based collision filtering), **`Physics.Matter`** (friction/bounciness), **`Physics.ConstraintComponent`** (Fixed/Point/Hinge joints).
- **`Physics.createGlobalProbe()`** / **`Physics.createRootProbe()`** / `world.createProbe()`; then **`probe.rayCast(start, end, cb)`** or **`rayCastAll(...)`**; hit exposes `position`, `normal`, `distance`, `collider`, `t` ([Raycast](https://developers.snap.com/lens-studio/features/physics/raycast)).

### How to build it
A basic raycast:
```javascript
var probe = Physics.createGlobalProbe();
probe.rayCast(new vec3(0,100,0), new vec3(0,-100,0), function (hit) {
    if (hit) { print(hit.position); print(hit.normal); }
});
```
For interactions, add a dynamic `BodyComponent`, then subscribe to events on it — only **dynamic bodies** generate collision events ([Collision and Overlap](https://developers.snap.com/lens-studio/features/physics/collision-and-overlap)):
```javascript
body.onCollisionEnter.add(function (e) { /* e has the other collider/body */ });
```
Events include `onCollisionEnter`/`onCollisionStay` and (for intangible/overlap use) `onOverlapEnter`/`onOverlapStay`.

### Best practices
Use the **Intangible** checkbox for trigger volumes: they generate overlaps and are hit by raycasts but don't push objects. Filter collisions with **`Physics.Filter`** layers instead of checking pairs in script. Prefer primitive colliders over Mesh colliders for cost.

### Common pitfalls
Constraints **cannot be moved after creation** — transform edits are ignored. Two static colliders never produce collision events. `createRootProbe` only sees the root world; use `createGlobalProbe` to cross all worlds.

## 3. Version Control & Collaboration

### What is possible
Lens Studio projects are git-friendly but mostly **binary**, so diffing/merging is limited. Snap ships merge tooling to reduce conflicts, and **Lens Studio 5.22** converts binary graph files to **YAML**, which are text-diffable and even editable in an external editor ([5.22 release notes / v5 page](https://ar.snap.com/lens-studio-v5)).

### Key components, assets & APIs
- **Project file:** Lens Studio 5 saves **`.esproj`** (older versions used **`.lsproj`**) — commit it ([Project Structure](https://developers.snap.com/lens-studio/lens-studio-workflow/advanced/source-control)).
- **Folders:** `Assets/` (commit — the editable heart of the project), `Support/` (commit — `StudioLib.d.ts`, `jsconfig.json`), `Cache/` (**optional, safe to gitignore** — only speeds loading), `Backup/` (**exclude** — crash-recovery only).
- **Binary formats needing Git LFS:** `.lsmat`, `.lsvfx`, `.oprfb`, `.lso`, `.mesh`, `.t3d`, `.pvr`.

### How to build it
Run `git init` then save in Lens Studio **4.40+** — it auto-generates `.gitignore` and `.gitattributes` (don't edit them except to add project-specific ignores). Install Git LFS **early** and ensure every collaborator runs `git lfs install`/`git lfs pull`, or they'll get pointer files that won't open ([Project Structure](https://developers.snap.com/lens-studio/lens-studio-workflow/advanced/source-control)).

### Best practices
Keep script-only PRs separate from scene/resource PRs. Edit scripts (JS/TS) outside the editor to avoid re-saving binary assets. Break large projects into **prefabs**. Everyone must be on the **same Lens Studio version** — have one person upgrade first to test.

### Common pitfalls (collaboration traps)
- **Forward-only YAML conversion (5.22):** opening a project or importing a package in 5.22+ **permanently** rewrites binary graphs to YAML. Expect a large graph diff on first commit, and note older Lens Studio versions can't read it back — coordinate the upgrade.
- **Spectacles version pin:** Spectacles (2024) targets **LS 5.15.x**, so a team on 5.22 mainline can't share one project with Spectacles work — a real handoff trap (treat as ground truth from the skill; confirm the current pin on the [download page](https://ar.snap.com/download)).
- `.lso` files wrap content as binary and are undiffable; prefabs can't reference other prefabs.

## 4. Testing & Debugging

### What is possible
Log to a panel, watch TypeScript compile status, preview live with a webcam or recorded tracking video, push to a real phone, and read an on-device performance overlay — all without leaving Lens Studio.

### Key components, assets & APIs
- **Logger** panel (**Window → Utilities → Logger**) — `print(...)`, `console.log/debug/info/warn/error(...)` (supports `%s`/`%d`/`%f`); filter by **device or script** (top-right), **Clear**, right-click **Copy**. Paired-device logs also appear here ([Debugging with Logger](https://developers.snap.com/lens-studio/features/scripting/debugging)).
- **TypeScript Status** panel (**Window → Utilities → TypeScript Status**) — compile state/errors; TS components are `@component` classes extending `BaseScriptComponent`. The Preview may not update until compilation succeeds ([TypeScript](https://developers.snap.com/lens-studio/features/scripting/typescript)).
- **Preview** panel modes — **Webcam**, **Photo**, **Video** (pre-recorded clips with embedded tracking data; add your own via **+ From Files**), and **Interactive Preview** (walk the scene with WASD/arrows) ([Previewing Your Lens](https://developers.snap.com/lens-studio/lens-studio-workflow/previewing-your-lens)).
- **Preview Lens** — pair a phone via **Snapcode** (press-and-hold to scan in Snapchat), then push the Lens to it; the **dropdown next to Send to Snapchat** pairs additional accounts ([Pairing to Snapchat](https://developers.snap.com/lens-studio/lens-studio-workflow/pairing-to-snapchat)).
- **Bug overlay** — on the paired device, tap the **Bug** icon (top-left in Snapchat) to see **FPS**, **RAM**, **SIZE**, and **LAT** (Lens Activation Time) ([Pairing to Snapchat](https://developers.snap.com/lens-studio/lens-studio-workflow/pairing-to-snapchat)).

### How to build it
For breakpoint debugging, install the **Lens Studio Visual Studio Code Extension**, open the project root, then in VS Code's **Run and Debug** tab pick **"Attach to running Lens"** (or **"Debug Lens"** for `onAwake`) and press **F5**. Note: the full breakpoint workflow currently only appears on the **versioned** doc [/4.55.1/…/vscode-extension](https://developers.snap.com/lens-studio/4.55.1/references/guides/general/vscode-extension); the [unversioned page](https://developers.snap.com/lens-studio/features/scripting/vscode-extension) documents only IntelliSense/snippets — verify against the current build.

### Common pitfalls
Custom preview videos carry no tracking data, so **Rotation, Surface, and World tracking won't work** on them (Face/Marker/Object tracking do). If content isn't rendering, check the camera's **Render Order**, layer assignment, and z-position.

### Diagnostic table (symptom → likely cause)
| Symptom | Likely cause |
|---|---|
| Nothing renders | Object's layer not in the camera's rendered layers / wrong Render Target / z outside clip plane |
| UI missing from the **captured/recorded** Snap | It's on an **Overlay** render target — Overlay content is not recorded; use the Scene's **Capture Target** instead ([Camera](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/camera)) |
| Preview stale / TS changes ignored | TypeScript didn't compile — check **TypeScript Status**, then refresh Preview |
| World/Surface tracking dead in Preview | Using a custom video without tracking data — use a built-in Video preset |
| Lens crashes on low-end phones | RAM over budget (target < 150 MB; > 275 MB likely crashes) ([QA Troubleshooting](https://developers.snap.com/lens-studio/publishing/optimization/lens-qa-troubleshooting)) |
| No collision events fire | Both bodies static, or collider is **Intangible** — make one **Dynamic** |

### Settled and still-open items
- **Cursor breakpoint debugging: confirmed (Jul 2026).** The 5.22 experimental debugger's supported clients are "VS Code built-in debugger, Cursor built-in debugger, or Any other VS Code fork" ([v5 page](https://ar.snap.com/lens-studio-v5)).
- **SPECS integration testing: confirmed (Jul 2026) — LEAF (Lens Evaluation & Automation Framework).** Write TypeScript test scenarios that simulate user actions and assert against live scene state ([v5 page](https://ar.snap.com/lens-studio-v5)); deeper feature docs are still thin, so check the current [release notes](https://ar.snap.com/lens-studio-v5) for specifics.
- **Still open — how to reveal the Bug overlay** (Developer Mode / triple-tap / shake): docs only say "tap the Bug icon after pairing" — no gesture is documented.

## Cloning and templating lens projects — verified the hard way

An effective pattern for shipping a family of related lenses (used for ~15 post-effect lenses, LS 5.22.1):
1. Build one project to completion, then **strip it to a template** (camera + post-effect + material + shader; delete lens-specific scripts and UI).
2. Clone per effect with `robocopy /E`, excluding `Cache`, `Workspaces`, `PluginsUserPreferences`, `*.jpg`, `.virtual-scene.json`, and the source `.esproj`.
3. Copy the source `.esproj` to `<NewName>.esproj`, then set `lensName:` and mint a **fresh `documentId:`**.
   - ⚠️ A naive `documentId: [0-9a-f-]+` regex also matches **`originalDocumentId:`** — anchor it (e.g. `(?<!original)documentId:`) or you will rewrite provenance too.
4. Swap in that effect's `.graphShader` body and per-effect script; adjust material parameter defaults.
5. Delete `Cache/` before reopening so assets re-import cleanly.

Related file-level habits:
- Hand-authoring or overwriting assets is safest while Lens Studio has that project **closed** — park the editor on a different project first (`openProject`), edit on disk, then reopen.
- **Overwriting a PNG in place keeps its `.meta` GUID**, so every material still resolves and the texture simply hot-reloads on refresh — the cheapest way to iterate baked art (UI, atlases, overlays).
- Structural edits to a `.graphShader`/`.mat` are best staged on disk while the project is parked; editing them under a live project risks the material re-importing against a stale pass.

## Go deeper
- [Performance and Optimization for Lenses](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide) — the 8 MB / 150 MB / 2048² budgets.
- [Remote Assets](https://developers.snap.com/lens-studio/features/lens-cloud/remote-assets-overview) — offload heavy content off the budget.
- [Physics Assets and Components](https://developers.snap.com/lens-studio/features/physics/physics-component) · [Raycast](https://developers.snap.com/lens-studio/features/physics/raycast) · [Collision and Overlap](https://developers.snap.com/lens-studio/features/physics/collision-and-overlap).
- [Lens Studio Project Structure & Source Control](https://developers.snap.com/lens-studio/lens-studio-workflow/advanced/source-control) · [Migrating to Lens Studio 5](https://developers.snap.com/lens-studio/overview/migrating-to-lens-studio/migrating-to-lens-studio-5).
- [Previewing Your Lens](https://developers.snap.com/lens-studio/lens-studio-workflow/previewing-your-lens) · [Pairing to Snapchat](https://developers.snap.com/lens-studio/lens-studio-workflow/pairing-to-snapchat) · [Debugging with Logger](https://developers.snap.com/lens-studio/features/scripting/debugging).
