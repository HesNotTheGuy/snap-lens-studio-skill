# Changelog — snap-lens-studio skill

**Stamp:** `2026-10-09.1`

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

1. Read the stamp in **both** homes and the newest tag on GitHub, then `git fetch` and
   compare `main` with `origin/main`: a PR merged on GitHub can land with no stamp bump
   and no tag (PR #1 did, 2026-10-07).
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

None as of `2026-10-09.1`. All three copies (Grok home, Claude home, GitHub `main`)
carry this stamp. Current live pin is LS **5.24.1** (7 Oct 2026, verified on ar.snap.com
2026-10-09); Spectacles (2024) remains **5.15.4**.

## Entries

### 2026-10-09.1 — Lens Studio 5.24.1, and PR #1 brought into the homes

- **PR #1 (`0698b60`, 2026-10-07):** a copy/tone pass on README, SKILL.md and four reference
  headings ("verified the hard way" → "field notes"); no facts changed. It was merged on GitHub
  with no stamp bump or tag, so neither home had it; this stamp carries it into both. It gets no
  tag of its own: a tag names the stamp inside its tree, and that tree still reads `2026-09-11.4`.
- Pin: mainline **5.24.1** (released 7 Oct 2026; verified on ar.snap.com 2026-10-09). Spectacles
  (2024) stays **5.15.4**; the Intel-Mac timeline is unchanged. Every per-file version line
  updated: the SKILL.md table, eight reference headers (two still said 5.23.2 — missed by the
  5.24.0 pass, which said all were done) and README (also still 5.23.2). The SKILL.md Camera Kit
  row now names the matrix's 5.18.x ceiling.
- **Home:** `references/materials-rendering-vfx.md` § Custom Text2D materials — Batching Enabled
  is required on the Shader node (the 5.24.0 preset shipped without it, so every preset-based text
  material was ignored; materials made in 5.24.0 need it ticked by hand); the YAML key; the full
  eight-value pass list (the old six-name list hid decoration outline/shadow); ports; code-node
  accessors.
- **Home:** `references/interactivity-audio-text-ui-integrations.md` § Spectacles — SPECS 27:
  `CursorVisualMode` renumbered (full order listed; compare by name), `createWorldAnchor` now
  declared to return a Promise (found by a typings diff; absent from the release notes and the API
  reference; runtime not tested), batched-shader device fixes. Version cheat-sheet row; Camera Kit
  matrix re-checked (still no rows past 5.18.x) and reconciled with the download page, which lists
  Camera Kit as a platform but defers to the matrix.
- **Home:** `references/fundamentals-and-workflow.md` — the 5.24.1 paragraph (client 14.21 and the
  MCP tool list unchanged), the typings-diff method, and a pitfall: a system-requirements warning
  at launch means no working GPU driver.
- `references/performance-and-publishing.md` — Batching Enabled listed as a toggle to profile, not a
  proven lever (no doc page explains it; the 5.24.1 notes only say batched shaders got cheaper).
- `SKILL.md`: one router line (ignored Custom Text2D material).
- Docs re-checked 2026-10-09: the Custom Text2D page gained its Batching Enabled line; the API
  reference, the Detailed Changelogs page (stops at 5.18), the Camera Kit matrix and the 2D Text
  page were not updated; `developers.snap.com/lens-studio/download/release-notes` now redirects to
  the docs home. All 160 cited URLs resolve.
- Functional test before the push: six developer questions answered by an agent restricted to the
  skill, then graded. It caught the two stale headers above and seven wording gaps (the 14.21
  floor stated as fact in one place, "typings only" readable as a type-only change, no
  CursorVisualMode names, the Camera Kit download page vs the matrix, Leaderboard status after
  5.24.1, Batching Enabled oversold, a heading implying an update fixes existing text materials).
  All fixed before the push. Pre-existing contradictions it also found (the RAM limit stated three
  ways, the hand-gesture count, "SPECS 27" vs "SPECS / 2026 glasses") are left for a separate pass.
- Not measured: opening a 5.24.0 project in 5.24.1.

### 2026-09-11.4 — editor quirks from a five-lens review loop

- **Home:** `references/fundamentals-and-workflow.md` — `PreviewPanelTool` screenshot does not create
  the output directory (reports saved, writes nothing); scene-graphql `setProperty` on a script input
  resets the lens; the persistent store loads late (first run after an open prints `restored false`,
  the next reset `restored true`); Publish does not save the project; the editor lens clock ran ~1 h
  behind the system clock; how to move a project folder safely; clean-quit log signature and MCP
  auto-reconnect after a relaunch.
- `SKILL.md`: one router line.

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
