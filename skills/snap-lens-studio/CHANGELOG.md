# Changelog — snap-lens-studio skill

**Stamp:** `2026-09-11.3`

This file is the sync key for every agent that loads the skill (Grok, Claude, and
anything cloning from GitHub). Read it **before** editing the skill. If two
copies disagree on the stamp, stop and merge — do not stack a new lesson on a
stale tree.

## Copies

| Location | Agent |
|---|---|
| `~/.grok/skills/snap-lens-studio/` | Grok |
| `~/.claude/skills/snap-lens-studio/` | Claude |
| GitHub | `https://github.com/HesNotTheGuy/snap-lens-studio-skill` (public; owner username only — no catalogue/revenue notes) |

GitHub is wired and is the third copy — the sync point for any agent that clones
rather than reads a home directory. All three must carry the same stamp: edit both
home directories **in the same turn**, then commit, tag with the stamp, and push.

## Before you edit

1. Read the stamp in **both** homes and the newest tag on GitHub.
2. If stamps differ, diff the two trees and merge into both before adding work.
3. The lesson you are adding has **one home** (usually a file under `references/`).
   Do not paste it into SKILL.md *and* a reference. SKILL.md only gets a router
   line if agents would otherwise miss the file.

## After you edit

1. Bump the stamp (`YYYY-MM-DD.N`) and add a dated entry below.
2. Write the **same files** to the other agent home.
3. Commit, tag with the stamp (`git tag YYYY-MM-DD.N`), and push `main` **with tags**.
4. Tell the user all three copies now share the new stamp.

## Known drift (do not ignore)

None as of `2026-09-11.3`. All three copies (Grok home, Claude home, GitHub `main`)
carry this stamp. Current live pin is LS **5.24.0** (10 Sep 2026, verified on ar.snap.com
2026-09-11); Spectacles (2024) remains **5.15.4**.

## Entries

### 2026-09-11.3 — 5.24 crash rule: never rewrite an open project on disk

- **Home:** `references/fundamentals-and-workflow.md` — two same-session crashes traced to rewriting the
  open project's graph shader on disk and then switching projects; park first, lock-file guard in the
  build, wait for the icon-compression/start print after `openProject` before touching the preview.
  `includeChrome:false` gives real 720x1280 frames in 5.24. The editor persistent store does not survive
  a crash. Injected taps within ~2.7 s of a reset land after the first-run tour stepped. Relaunch +
  ~40 s `Tool not found` window.
- `SKILL.md`: one router line.

### 2026-09-11.2 — what a 5.24 project migration does, MCP after the update

- **Home:** `references/fundamentals-and-workflow.md` — measured 5.23.2 → 5.24.0 migration over MCP:
  no dialog; `.esproj` rewritten with `iconHash` preserved; `//@component` prepended to a legacy JS
  script; packages re-stamped (`clientVersion` 14.21); scene, shader and icon files untouched; LS
  writes `BackUp/<old build>.zip`; lens behaviour identical; **Minimum Client Version 14.21.0** for
  lenses published from 5.24. MCP link survived the update; `ExecuteEditorCode` compiles as
  TypeScript (TS2339 on untyped object literals); screenshot size follows the panel.
- `SKILL.md`: one router line (silent migration + client floor).

### 2026-09-11.1 — Lens Studio 5.24.0 pin + release-note gotchas

- Pin: mainline **5.24.0** (released 10 Sep 2026; verified on ar.snap.com/download and
  ar.snap.com/lens-studio-v5 on 2026-09-11; the Custom Text2D and Connected Framework doc pages
  exist). Spectacles (2024) stays **5.15.4**; the 5.24.0 page's Spectacles line reads "The current
  latest Spectacles Firmware does not support this Lens Studio version". Intel Macs: 5.25 is the
  last Intel build, 5.26+ needs Apple silicon. Every per-file version line updated (SKILL.md table
  + seven references).
- **Home:** `references/fundamentals-and-workflow.md` — 5.24 headline list; the **2D Text blend
  default PremultipliedAlpha → Normal** migration (automatic on project update; other text blend
  modes may shift, so re-check lenses with text); Editor API callback `fetch` removed on
  AssetListService/MusicListService → `fetchAsync`; Leaderboard lenses from 5.24 need Snapchat 14.23.
- **Home:** `references/materials-rendering-vfx.md` — Custom Text2D materials (Text Data node) and
  the blend-default note.
- **Home:** `references/interactivity-audio-text-ui-integrations.md` — Connected Framework pointer
  (5.24 multiplayer matchmaking SDK on Connected Lenses); version cheat-sheet row.
- `references/snapml-and-generative-ai.md` — AI Video Transform + AI Photo Duo GenAI plugins.
- Not yet exercised: nothing in 5.24 was run here; the MCP / Editor API notes still describe 5.23.2
  behaviour until re-measured.

### 2026-09-09.2 — second-camera placeholder path, first-run tour + last-state memory, capture routing

- **Home:** `references/interactivity-audio-text-ui-integrations.md` — Dual Camera placeholder path
  measured: an Image material bound to the Reverse Camera Texture draws **empty** in the editor (the
  provider swap never reaches the binding); bind the package's Mock under `isEditor()`. First-run
  tour pattern (auto-advance until the first tap, hint showing, never save during the tour) and
  "remember the last state, not only settings" (restore verified after a lens reset).
- **Home:** `references/materials-rendering-vfx.md` — `MaskingComponent` circle: pixel-square rect,
  `cornerRadius` = half its size (the only property the Editor API exposes); animate a hole from script.
- **Home:** `references/fundamentals-and-workflow.md` — content under a UI camera that must be in
  the snap: `renderTarget` = main Render Target, `renderOrder` after the post-effect camera.
- `SKILL.md`: two router lines (second-camera feed; the two review asks for tap-cycled/timed lenses,
  plus: a timed effect must end on a look worth keeping).
- Still untested: the real reverse feed on a phone in the masked-placeholder design.

### 2026-09-09.1 — six-lens batch: three silent code-node deaths, verification harness, Dual Camera

- **Home:** `references/materials-rendering-vfx.md` — three dead-pass signatures with their
  fixes: missing `output_vec4 result;` (white frame, `Port_FinalColor…` props), reserved word
  `cast` (`["mainColor"]`), and **NaN from a stale texture parameter mixed at weight 0** (healthy
  props, nothing drawn). Diagnose-by-painting method. `pow(neg)` / reversed `smoothstep`.
- **Home:** `references/fundamentals-and-workflow.md` — `includeChrome:false` screenshots can be
  white stubs; ~1.2 s tap-to-capture latency; `holdProgress` / `debugOpen` debug inputs set via
  `scene-graphql setProperty` for exact frames; `getTapPosition()` y-down vs `screenUV` y-up.
- **Home:** `references/interactivity-audio-text-ui-integrations.md` — Dual Camera component,
  source-verified: provider swap inside the Reverse Camera Texture asset, `isSupported`
  semantics, MediaPicker default; **a code node cannot sample that texture in preview (NaN)** —
  use the placeholder path.
- `SKILL.md`: two router lines in cross-cutting gotchas.
- Not a lesson: helper functions before `main()` were inlined during the hunt; they were not the
  cause and remain untested either way.

### 2026-09-07.1 — docs re-verification + review fixes

- Snap docs re-checked 2026-09-07: 5.23.2 still current (no 5.24); Spectacles (2024)
  still 5.15.4; Camera Kit matrix unchanged and **still tops out at LS 5.18.x**;
  Bitmoji badge unchanged; submission thresholds unchanged.
- **Home:** `references/performance-and-publishing.md` — Lens Creator Rewards: the two
  programmes, permanent one-way enrolment, and Snap's **ineligible lens types**
  (basic filters, beautification tools, static 2D images, frames, copies); growth
  hacks named as an explicit rejection reason; the 320×320 icon hedge rewritten.
- **Home:** `references/fundamentals-and-workflow.md` — the MCP token **rotates on
  server restart inside LS** and `.mcp.json` is written to the open project's
  directory; three-way triage for `401` / `Tool not found` / no tools. SVG Text
  corrected to **5.23.1** (5.23.2 is bug fixes only). Stale "5.22" template hedge
  dropped.
- **Home:** `references/interactivity-audio-text-ui-integrations.md` — the Camera Kit
  matrix has no rows for 5.19–5.23 (author Camera Kit Lenses on ≤ 5.18.x); all LTS
  windows closed.
- `SKILL.md`: 320×320 removed from the "unconfirmed" list (it is LS's default icon
  size and the recipe is verified at it); router line for the MCP triage; the
  "contra the note above" hot-reload bullet reworded — it agreed with the bullet it
  argued with.
- `references/face-tracking-and-effects.md`: duplicate "expression weights zero"
  pitfall removed.
- Tags: `2026-09-05.1` applied retroactively to the union commit; this stamp tagged
  on its own commit; `main` pushed with tags.
- **Open, not done:** SKILL.md (~440 lines) still carries five full lesson blocks
  that the one-home rule says belong in `references/` — the materials reference even
  points back to SKILL.md for the shader-failure table. Moving them is a separate
  pass.

### 2026-09-05.1 — union merge + 5.23.2 pin

- Union of Grok and Claude homes (not a wholesale overwrite of either).
- Kept Grok editor/MCP lessons: `setEmptyProject`+`openProject` hard-crash,
  320×320 `iconHash` recipe, no `importExternalFile` of in-project paths,
  GLSL-string hot-reload vs structural graph breakage.
- Kept Claude product/API lessons: Camera Kit Bitmoji = limited (not unavailable),
  Ray Tracing iOS yes / Android no, `Head.onLandmarksUpdate`, `TurnOnEvent`
  deprecated, silent-shader detector, dead-control tests, testing traps.
- Version pin: mainline **5.23.2** (17 Aug 2026). Spectacles (2024) **5.15.4**.
- Release-notes surface is [ar.snap.com/download](https://ar.snap.com/download);
  `developers.snap.com/lens-studio/download/release-notes` does not carry the
  current version.
- Hand Mesh (LS 5.23) noted in `references/world-body-and-tracking.md`.
- Hung-object lesson kept in world-body + SKILL.md testing traps.

### 2026-08-27.1 — hung objects are not UV boxes

- **Home:** `references/world-body-and-tracking.md` (best practice + pitfall).
- **Also:** testing trap on Idle Person 1–6.

### 2026-08-27.2 — split audit (no merge)

- File-by-file diff of Grok vs Claude homes. Merge completed in `2026-09-05.1`.
