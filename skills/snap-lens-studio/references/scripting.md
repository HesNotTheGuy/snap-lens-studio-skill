# Scripting (JavaScript / TypeScript) & the Scripting API

Scripting is how a Lens becomes interactive: it wires runtime behavior — responding to touches, faces, timers, and frame updates — to the objects and assets in your scene. Lens Studio runs a JavaScript engine supporting the **ES2019** spec plus CommonJS-style `require`, and (since Lens Studio 5.0) supports **TypeScript natively**; you can mix JS and TS in the same project. Understanding the script *lifecycle*, the *event framework*, and how scripts *reference* other scene objects is the core of nearly everything beyond a static filter. This file targets **Lens Studio 5.23.x** (current mainline: 5.23.2, Aug 2026); note that **Spectacles (2024) development is pinned to Lens Studio 5.15.4**, but the same core Script API applies there. Docs live at [developers.snap.com](https://developers.snap.com) — hardcode that host (old `docs.snap.com/lens-studio/*` URLs 308-redirect to it) and prefer unversioned feature/API paths so your knowledge tracks the current release.

## What is possible

- **Author in JavaScript, TypeScript, or both.** The JS engine supports ES2019 + CommonJS `require`; TypeScript compiles to JS automatically. ([Script Overview](https://developers.snap.com/lens-studio/features/scripting/script-overview), [TypeScript](https://developers.snap.com/lens-studio/features/scripting/typescript))
- **React to events** — per-frame updates, taps/touches, face tracking, camera flips, keyboard, recording, and manipulation — through a unified event framework.
- **Create and modify the scene at runtime** — spawn `SceneObject`s, add/remove `Component`s, walk and re-parent the hierarchy, tween transforms and colors.
- **Expose tunable parameters** to the Inspector via `@input` (JS directives or TS decorators) so non-coders and designers can configure your logic.
- **Decouple logic from code** using the **Behavior** asset (Trigger → Response) and drive named animations with the **Tween Manager**.
- **Debug** with the Logger panel (`print`, `console.*`) and on-device with the Text Logger V2.

## Key components, assets & APIs

- **Script Asset** — the reusable code file in the Asset Browser. **Script Component** — that asset attached to a `SceneObject`; only a *component* participates in the scene and receives `script`/lifecycle events. ([Script Components](https://developers.snap.com/lens-studio/features/scripting/script-components))
- **Script Module** — a code file consumed via `require()`. A module has **no** `script` property but **does** have access to the global `scene` object. ([Script Modules](https://developers.snap.com/lens-studio/features/scripting/script-modules))
- **Global objects** — `global` (the pre-defined global scope; [Scripting Introduction](https://developers.snap.com/lens-studio/features/scripting/scripting-introduction)), `global.scene` (the runtime [`ScriptScene`](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.ScriptScene.html): `createSceneObject(name)`, `getRootObjectsCount()`, `getRootObject(i)`), plus helper globals `global.behaviorSystem`, `global.tweenManager`, `global.textLogger`.
- **[`SceneObject`](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.SceneObject.html)** — components (`getComponent(type)`, `getComponents(type)`, `createComponent(type)`, `copyComponent(c)`, `getComponentInAncestors/InDescendants`), hierarchy (`getChild(i)`, `getChildrenCount()`, `getParent()`, `setParent(o)`, `removeParent()`, `children[]`, `getTransform()`), and props (`enabled`, `name`, `layer`, `isEnabledInHierarchy`, `destroy()`, `copySceneObject()`, `copyWholeHierarchy()`). Component types are dotted strings, e.g. `'Component.RenderMeshVisual'`, `'Component.ScriptComponent'`.
- **`BaseScriptComponent`** — the class a TS `@component` extends (directly or indirectly).
- **Event types** — `OnAwakeEvent`, `OnStartEvent`, `UpdateEvent`, `LateUpdateEvent`, `OnEnableEvent`/`OnDisableEvent`, `OnDestroyEvent`, `TurnOffEvent` (⚠️ not `TurnOnEvent` — deprecated, use `OnStartEvent`), `DelayedCallbackEvent`, `TapEvent`, `TouchStart/Move/EndEvent`, and face/camera/keyboard/recording/manipulation events.
- **`InteractionComponent`** — scopes input to a specific object via EventWrappers (`.add(cb)`): `onTap`, `onTouchStart`/`onTouchMove`/`onTouchEnd`, `onDoubleTap`, scroll/pan/pinch/long-press, hover/focus/select. ([API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.InteractionComponent.html))
- **Helper packages** (bundled, not core runtime): **Behavior** and **Tween Manager** — must be added via the Scene Hierarchy `+` menu.

## How to build it

**Two authoring styles.** Legacy JavaScript exposes inputs on the global `script` object:
```javascript
//@input float intensity = 1.0
//@input string hello = "Hello, World!"
print(script.hello);        // inputs land on the global `script` object
```
Modern TypeScript uses a class with decorators (Lens Studio 5.0+):
```typescript
@component
export class NewScript extends BaseScriptComponent {
  @input('float') intensity: number = 1.0;
  onAwake() { print('Hello World'); }
}
```
Rules confirmed by the docs: *"Only one component decorator is allowed per file"*; the class must extend `BaseScriptComponent` (directly or indirectly); after inputs are set, `onAwake()` is called. ([TypeScript](https://developers.snap.com/lens-studio/features/scripting/typescript))

**Understand the lifecycle** — this ordering is *why* your code should be structured a certain way. Events are created with `script.createEvent("EventName")` (JS) / `this.createEvent("EventName")` (TS) and wired with `.bind(callback)`. ([Script Events](https://developers.snap.com/lens-studio/features/scripting/script-events))
1. **OnAwakeEvent** — fires *"before any other event in the Lens (including `OnStart` and `Update`)"*. Configure **self only**; do not reach into other components yet. In TS, `onAwake()` is the entry point.
2. **OnStartEvent** — once, *"after all `OnAwake` events have triggered on the first frame"* and before the first Update. The correct place to access other components and inputs.
3. **UpdateEvent** — every frame (keep this lean).
4. **LateUpdateEvent** — end of frame, pre-render.
5. **OnEnableEvent / OnDisableEvent**, **OnDestroyEvent**, **TurnOffEvent** (⚠️ `TurnOnEvent` is **deprecated** — use `OnStartEvent`) (`TurnOffEvent` fires once on Lens exit — ideal for cleanup).

**Time things** with a one-shot `DelayedCallbackEvent` instead of counting frames in Update ([API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.DelayedCallbackEvent.html)):
```javascript
var ev = script.createEvent("DelayedCallbackEvent");
ev.bind(function(){ print("delay over"); });
ev.reset(2);            // reset(time): triggers in `time` seconds
// also: ev.cancel(), ev.getTimeLeft(), ev.getDelayTime()
```

**Handle input.** Scene-level touch events are *"simple, full-screen touch events"* — an object without an Interaction Component receives touch across the whole screen. To scope touches to a 2D/3D object, add an **`InteractionComponent`** and subscribe via the EventWrapper `.add()` pattern: `onTouchStart.add(cb)`, `onTouchMove.add(cb)`, `onTouchEnd.add(cb)`. The Interaction Component **also exposes `onTap`** (*"Triggered when the user taps on the screen"* — `onTap.add(cb)`), plus `onDoubleTap`, scroll/pan/pinch/long-press, hover/focus/select, and `onTriggerPrimary`; the guide page only demos the touch trio, so check the API page for the full list. Scene-level `TapEvent` is for full-screen taps on objects without an Interaction Component. ([InteractionComponent API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.InteractionComponent.html)) ([Touch and Interactions](https://developers.snap.com/lens-studio/features/scripting/touch-input), [InteractionComponent API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.InteractionComponent.html)) Other event families bind identically: face (`FaceFoundEvent`, `FaceLostEvent`, `MouthOpened/ClosedEvent`, `BrowsRaised/Lowered/ReturnedToNormalEvent`), camera (`CameraFront/BackEvent`), keyboard (`KeyPress/ReleaseEvent`), recording (`SnapRecordStart/StopEvent`, `SnapImageCaptureEvent`), and manipulation (`ManipulateStart/EndEvent`).

**Expose parameters.** In JS use `//@input <type> name [= default]` (also `//@ui`, `//@typedef`); values appear on `script.<name>`. In TS use decorators confirmed on the page: `@component`, `@typedef`, `@input`, `@showIf`, `@hint`, `@label`, `@widget`, `@ui.separator`, `@ui.label`, `@ui.group_start`, `@ui.group_end`, `@allowUndefined`, `@typename`. Override type/default with `@input()` / `@input('int')` / `@input('mat2','{{1,2},{3,4}}')`. The docs give type strings as *examples* only, so treat the fuller list (`float`, `bool`, `string`, `vec2/3/4`, `mat2/3/4`, `SceneObject`, `Component`, `Material`, `Texture`, `ObjectPrefab`, `Asset`, custom `@typedef`) as generally correct but version-sensitive — verify exact literals against the current page. ([TypeScript](https://developers.snap.com/lens-studio/features/scripting/typescript))

**Reference other scripts** ([Accessing Components](https://developers.snap.com/lens-studio/features/scripting/accessing-components)): JS→JS, put shared members on `script` (`script.numberVal = 1`); TS→TS, use a typed input (`@input refScript: TSComponentA`); JS→TS, use `@input('Component.ScriptComponent')` typed as `any` (no completion) or provide a `.d.ts` declaration.

**Reuse code via modules** — CommonJS only ([Script Modules](https://developers.snap.com/lens-studio/features/scripting/script-modules)): *"Export a module using `module.exports` and import a module using the `require` function."* `require()` resolves by path (`"./MathModule"`, relative to the caller) or by name (searches current folder, then upward) — and *"All versions of require must be used with string literals not by passing a const or var argument."* Related loaders: `requireAsset()` (Textures/Materials without `@input`), `requireType()` (a ScriptAsset handle for `createComponent`), and native modules via the `LensStudio:` prefix, e.g. `require("LensStudio:BitmojiModule")`. (TS still uses `import` for compile-time type/class resolution; *runtime* module loading is CommonJS `require`.)

**Manipulate the scene at runtime:**
```javascript
var so   = script.getSceneObject();
var comp = so.createComponent('Component.ScriptComponent');
var got  = so.getComponent('Component.ScriptComponent');
var fresh = global.scene.createSceneObject("name");
```

**Wire no-code logic with Behavior** (add via Scene Hierarchy `+` → Scripts → Behavior; [Behavior](https://developers.snap.com/lens-studio/lens-studio-workflow/adding-interactivity/behavior)). Trigger modes: **Always / Once / After Interval**. Script bridge: locally on a Behavior script's `api` — `api.trigger()`, `api.addTriggerResponse(cb)`, `api.removeTriggerResponse(cb)`; globally — `global.behaviorSystem.sendCustomTrigger(name)`, `addCustomTriggerResponse(name, cb)`, `removeCustomTriggerResponse(name, cb)`.

**Animate with Tween Manager** (add via `+` → Scripts → Tween Manager; [Tween Manager](https://developers.snap.com/lens-studio/lens-studio-workflow/adding-interactivity/tween-manager)). Each tween needs a **Tween Name** to be triggered from script. Types: `TweenTransform`, `TweenScreenTransform`, `TweenColor`, `TweenAlpha`, `TweenValue`, `TweenChain`. `global.tweenManager` methods: `startTween(sceneObject, "tweenName" /*, onComplete, onStart, onStop */)`, `stopTween`, `pauseTween`, `resumeTween`, `setStartValue`, `setEndValue`, `getGenericTweenValue`.

## Best practices

- **Prefer TypeScript class components** (`@component extends BaseScriptComponent`) for new code — it is the modern typed idiom. The older `//@input` global-`script` JS style remains fully supported. (The docs do *not* officially crown a "recommended" language; "prefer TS" is guidance, not a mandate.)
- **OnAwake = self-configuration only; OnStart = cross-component wiring.** This mirrors the guaranteed ordering and avoids touching components that may not be initialized.
- **Keep reusable, scene-independent logic in Script Modules** (`require`) — remembering a module has `scene` but not `script`.
- **Expose config via typed `@input`** with `@ui`/`@widget`/`@hint`; use `@allowUndefined` for optional refs and null-check with the `isNull` property before use.
- **Use `DelayedCallbackEvent` for one-shot timers** and keep `Update` work minimal to protect frame rate.
- **Decouple designers from code** with Behavior custom triggers (`global.behaviorSystem`), and use **named Tween Manager tweens** instead of hand-rolled lerping.
- **Clean up in OnDisable/OnDestroy/TurnOff** — cancel `DelayedCallbackEvent`s and remove trigger responses.
- **Use VS Code for TypeScript.** The built-in editor's autocompletion is **JavaScript-only**: *"While Lens Studio provides auto-completion support for JavaScript-based scripts, it lacks similar support for TypeScript."* ([TypeScript](https://developers.snap.com/lens-studio/features/scripting/typescript))

## Common pitfalls

- **`TypeError: cannot read property 'X' of undefined/null`** — an `@input` was never assigned, or you accessed another component during OnAwake. Fix by assigning the input or moving the access to OnStart, and null-guard with the `isNull` property on SceneObjects/Components.
- **`X is not defined`** — a missing `require`/export, or a scope error.
- **Silent "nothing happens"** — the event was never `.bind()`-ed, *or* a `TapEvent` is firing full-screen because the object has no Interaction Component (to scope a tap to a specific object, use the Interaction Component's `onTap.add(cb)` — [API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.InteractionComponent.html)).
- **TS "changes not reflected"** — TypeScript compilation is not instantaneous. Compilation may be unfinished (check **Window > Utilities > TypeScript Status** and the Logger) or the Preview isn't refreshed.
- **`require` with a variable** — fails; the argument must be a string literal.
- **Assuming TS autocomplete in-editor** — it doesn't exist; use an external editor.
- **`Studio.log()`** — not documented; use `print()` / `console.*` instead.

**Debugging tools** ([Debugging with Logger](https://developers.snap.com/lens-studio/features/scripting/debugging)): open the Logger via `Window → Utilities → Logger`. Use `print('msg')`; `console.log(...)` with substitutions (`%s`, `%d`/`%i`, `%f`); `console.debug/info/warn/error`; `console.time/timeLog/timeEnd`; `console.trace()`. For on-device logs (Text Logger V2): `global.textLogger.log(...)`, `.setLoggingEnabled(false)`, `.setTextColor(new vec4(1,0,0,1))`, `.setLogLimit(n)`, `.setTextSize(50)`, `.clear()` — and **disable/remove the Text Logger before submitting** a Lens.

## Go deeper

- [Script Overview](https://developers.snap.com/lens-studio/features/scripting/script-overview) — engine capabilities (ES2019 + CommonJS), JS/TS coexistence.
- [TypeScript](https://developers.snap.com/lens-studio/features/scripting/typescript) — decorators, `@component`/`BaseScriptComponent`, `@input` types, autocomplete caveat.
- [Script Events](https://developers.snap.com/lens-studio/features/scripting/script-events) — lifecycle ordering and the full event catalog.
- [Script Modules](https://developers.snap.com/lens-studio/features/scripting/script-modules) — `require`, `requireAsset`, `requireType`, `LensStudio:` native modules.
- [Accessing Components](https://developers.snap.com/lens-studio/features/scripting/accessing-components) — cross-script reference patterns.
- [Touch and Interactions](https://developers.snap.com/lens-studio/features/scripting/touch-input) — full-screen touch vs. `InteractionComponent`.
- [Debugging with Logger](https://developers.snap.com/lens-studio/features/scripting/debugging) — Logger panel, `console.*`, Text Logger V2.
- [SceneObject API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.SceneObject.html) · [ScriptScene API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.ScriptScene.html) · [DelayedCallbackEvent API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.DelayedCallbackEvent.html) — exact signatures.
- [Behavior](https://developers.snap.com/lens-studio/lens-studio-workflow/adding-interactivity/behavior) · [Tween Manager](https://developers.snap.com/lens-studio/lens-studio-workflow/adding-interactivity/tween-manager) — bundled no-code helpers and their script bridges.
- [Release Notes](https://ar.snap.com/lens-studio-v5) — confirm the current version and the Spectacles 5.15.x pin.