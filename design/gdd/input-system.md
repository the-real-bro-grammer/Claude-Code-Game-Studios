# Input System

> **Status**: In Design
> **Author**: Robert Michael Watson + (specialist agents will be listed as consulted)
> **Last Updated**: 2026-04-21
> **Implements Pillar**: Pillar 5 (No Input Goes Silent — 100ms feedback floor) primary; Pillar 4 (Prep + React phase handoff); Pillar 2 (Readable at a Glance — hitbox + safe-area correctness); Pillar 1 (Cute Chaos — finger-occlusion slapstick offsets)

## Summary

Input System is Money Miner's touch dispatcher — the single authority that converts raw Unity touches into typed, tier-budgeted events for every consuming system. It owns the tap-vs-drag classifier, the safe-area-aware pointer state, the finger-occlusion offset that keeps tools visible above the thumb, and the input-suppression gate that prevents ghost taps during Prep→Rush phase transitions or modal dialogs. Every system that reads player intent (Tool System's drag-to-place, Collection System's tap-to-catch, Biome Map's swipe-to-pan, HUD's button taps) consumes Input's events — no system reads `UnityEngine.InputSystem` APIs directly.

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: none (zero-dep foundation) · Publishes on Event Bus: `input.tap.main`, `input.drag.start`, `input.drag.end` (all T1 Feedback)

## Overview

Mobile touch input looks trivial until you ship it. A "tap" is not a single Unity event — it is a `began` → `moved` → `ended` sequence that must be classified as tap vs drag vs hold within a bounded time/distance window. A "drag" that starts on a tool icon must not double-fire as a "tap" on the playfield beneath. A finger hovering over the bottom third of the phone must not occlude the critical tool it is placing. A tap during a Modal/Dialog overlay must be suppressed. A player whose phone is a Samsung Galaxy S-series at 20:9 aspect ratio must see the same hit-zones as a player on a 19.5:9 iPhone. These are the problems Input System owns — and every consuming system downstream trusts Input to have already solved them before its handler runs.

The system provides four capabilities: (1) **typed event publication** — Input subscribes to `UnityEngine.InputSystem.EnhancedTouch.Touch.activeTouches` on the main thread and fires typed events (`TapPayload`, `DragPayload`) on Event Bus Feedback channels, all at T1 tier (≤ 16.67 ms `Fire` → last subscriber return) so consumers can gate same-frame feedback; (2) **pointer state query** — a pure `InputState.CurrentPointer` struct (position, pressed, age) that any gameplay system can read in `Update` / `FixedUpdate` without subscribing to events, for systems that need continuous state (Tool System's drag-to-place preview) rather than discrete events; (3) **phase-aware suppression** — Input queries the current Level phase (Prep / Rush / Paused) and Modal/Dialog stack depth, suppressing or routing events accordingly so ghost taps cannot fire during phase transitions, pause, or modal overlays; (4) **platform safe-area and hit-inflation contracts** — every published screen coordinate is clipped to the device safe area (notch + home indicator + punch-hole camera islands), and the 44pt iOS / 48dp Android minimum hit-target rule plus the 1.4× visual-hitbox inflation rule (per `.claude/docs/technical-preferences.md`) are enforced at the Input layer so every consumer inherits correct hitboxes without duplicate logic.

The system enforces five rules: **no direct `UnityEngine.InputSystem` access outside Input** (CI-enforced grep — consumers read only `InputEvent.*` events and `InputState.CurrentPointer`); **no input during `LoadingFromDisk`** (Save/Load holds the pause gate; Input's suppression mirrors the `SaveManager` state); **no tap synthesis from drag** (a drag that ends within the tap-classification window still fires `input.drag.end`, never `input.tap.main` — downstream systems that want either must subscribe to both); **no main-thread blocking** (Input's `Fire` → handler chain completes synchronously within the T1 16.67 ms budget; any consumer that would overrun must defer via `Task.Run` or the next frame via `EventBusPlayerLoopSystem`); **no gameplay rules in Input** (Input publishes raw pointer intent; whether a tap *hits* a coin is Collection System's concern, whether a drag *places* a ramp is Tool System's concern, whether a tap *advances* a dialog is Modal/Dialog's concern).

This GDD defines the design contract — events, states, suppression rules, tuning thresholds. Unity Input System package configuration (Input Actions asset structure vs `EnhancedTouch` direct polling; dispatch mode choice between `PlayerInput` component, generated C# class, or `InputAction` callbacks) is an implementation detail scoped for a future ADR when MVP implementation begins. Consumers designing against Input System may rely on the contract described here.

## Player Fantasy

Input System has no player-facing fantasy — it is the channel through which every other system's fantasy arrives. No player will ever say "I love this game's touch handler." They will say "that ramp snapped exactly where I wanted it" or "the gopher went for the bait the instant I tapped," and this is the system that made those sentences true.

The fantasy Input serves is Pillar 5's promise — **No Input Goes Silent**. Every tap and drag must reach its consuming system (Tool, Collection, HUD, Biome Map) within a single frame so that audiovisual feedback can fire inside the 100ms window the player's nervous system treats as "instant." Input is the first 16.67ms of that budget. Everything downstream — the tool spawn, the coin pop, the squeak — depends on this system getting out of the way fast enough that the other systems have time to be satisfying.

Picture the moment a player cares about most: Rush phase, 18 seconds on the clock, a gopher stream bending toward the left ledge, and the player drags a ramp from the tray with their right thumb. Their finger is directly over the ramp. They lift. The ramp is exactly where they meant it — offset cleanly above the point of contact, snapped to the grid, with a satisfying *clack* inside 100ms. They never thought about their finger. They thought about the gophers.

That is the fantasy Input protects — and it is not Input's fantasy. It belongs to Tool System, to Collection, to the Prep→Rush handoff. Input's job is to be the channel that delivers the tap and drag so faithfully that the player never notices Input exists. The tap-vs-drag classifier, the 1.4× hitbox inflation, the spawn-above-finger offset, the suppression gate that kills stale taps crossing the Prep→Rush boundary — every one of these exists to keep the player's attention on the board and off their hand.

The fantasy is NOT "my tap was registered." The fantasy is "I never had to wonder whether my tap was registered." Silence when working, invisible when working. When Input is doing its job, the player is thinking about gophers, not fingers.

## Detailed Design

### Core Rules

**R1. Single input authority.** No code outside `InputDispatcher.cs` and `InputState.cs` accesses `UnityEngine.InputSystem.*` namespaces (including `EnhancedTouch`, `LowLevel`). Consumers read only `InputEvent.*` typed events on Event Bus and the `InputState.CurrentPointer` struct. Enforcement — CI grep pattern `^using UnityEngine\.InputSystem(\.EnhancedTouch|\.LowLevel)?;` applied to all `.cs` files with allowlist: `src/Input/InputDispatcher.cs`, `src/Input/InputState.cs`, `tests/unit/Input/**/*.cs`, `tests/integration/Input/**/*.cs`. Any match outside the allowlist fails CI.

**R2. First-touch only at MVP.** Input tracks the single active touch identified by the finger ID of the first touch to enter `began` phase. Any simultaneous second-or-higher-finger contact is silently ignored (no event published, no log). The tracked finger is released on `ended` or `canceled`. Multi-touch (pinch-to-zoom on Biome Map, dual-thumb gestures) is explicitly post-MVP and requires an ADR to repeal R2. Enforcement — PlayMode test: simulated two-finger simultaneous `began` produces exactly one event sequence (from the first finger only).

**R3. Gesture classification.**

- **R3.a Tap.** Movement delta `< 10pt` (iOS) / `< 10dp` (Android) for the full touch duration AND total duration `≤ 300ms`. Published as `input.tap.main` on the `ended` phase — **never on `began`**. A tap-that-becomes-a-drag generates zero taps. Late-lift forgiveness: a touch lifting at 301ms+ with `< 10pt` movement and no velocity is still classified as a tap (motor-impaired accommodation).
- **R3.b Drag.** Movement delta `≥ 10pt` / `10dp` at any time during the touch, OR delta `> 6pt` / `6dp` after 150ms (slow-deliberate-drag early trigger, critical for Rush-phase ramp placement where waiting 300ms for classification wastes 18% of the 100ms feedback window). On the frame the threshold is crossed, `input.drag.start` fires with the original `began` position as `StartScreen` and the current position as `EndScreen`. On `ended`, `input.drag.end` fires with the original `began` position and the final position.
- **R3.c Long-press — reserved, unused at MVP.** Duration `> 300ms` with `< 10pt` movement is classified as long-press and **silently discarded**. No event published. The classification slot exists to prevent a future system from accidentally receiving a tap on what should have been a hold. Post-MVP systems that require long-press must repeal R3.c.
- **R3.d Continuous drag state via polling, not events.** No `input.drag.update` event exists. Systems needing per-frame drag position (Tool System's drag-to-place preview) read `InputState.CurrentPointer` in `Update`. Rationale: per-frame bus traffic at 60fps would blow T1 subscriber-handler budgets; polling is cheaper and self-throttles to each consumer's own frame cadence.

Enforcement — unit tests for each branch (at-threshold, one unit below, one unit above); integration test confirming no `input.tap.main` fires after a drag.

**R4. Hit-target resolution order.** For `TapPayload.Target`, Input performs a two-pass resolution on the main thread:

1. **UI raycast** — `EventSystem.current.RaycastAll(pointerEventData, results)` against a pre-allocated `List<RaycastResult>` (zero-alloc). If `EventSystem.current.IsPointerOverGameObject(touchId)` returns true (using the **touchId overload**, NOT the default -1 overload), the first raycast result is published as `Target` and Pass 2 is SKIPPED.
2. **World raycast** — `Physics2D.OverlapPointNonAlloc(worldPoint, resultsBuffer, layerMask)` with pre-allocated 8-element buffer and layer-filtered collision matrix. First hit wins. `Physics2D.OverlapPoint` (allocating) is BANNED — CI grep enforces this.

If both passes return null, `Target = null` (playfield tap). Rationale — reversing the order would let a tap pass through a modal overlay and reach the playfield beneath (ghost tap bug). ScreenSpaceOverlay canvases are outside the camera frustum and cannot be detected by Physics2D queries; EventSystem raycast is the only correct UI path. Enforcement — PlayMode test: UI layer over a world collider; published `Target` is the UI object.

**R5. Coordinate contract — safe-area-local pixels.**

- **R5.a Safe-area clipping.** Touches whose `began` position lies outside `Screen.safeArea` (iOS notch, home indicator, Android punch-hole exclusion zones, back-gesture edge) are suppressed at source — no event published, no `CurrentPointer` update. Publishing-and-filtering at the consumer layer creates a race where fast taps reach T2 subscribers before their own filter runs.
- **R5.b Every published `Vector2` is safe-area-local.** Origin is `Screen.safeArea.bottomLeft`. Consumers do NOT add their own safe-area offset. The safe-area rect is cached once on scene load (it does not change during a session on any supported iOS/Android device). A touch that begins inside safe area and moves outside remains tracked; positions are clamped to the safe-area boundary on each `Update`.

Enforcement — unit test with mock safe-area rect: touch position outside → no event; edge-crossing touch → coordinates clamped.

**R6. Hitbox inflation is consumer-authored, not Input-enforced.** Input publishes raw finger contact coordinates. Consuming systems (Tool System, Collection System, HUD, Biome Map, Modal/Dialog) are responsible for authoring their colliders and UI RectTransforms with the 1.4× visual-sprite-bounds inflation and the 44pt iOS / 48dp Android minimum hit-target floor (per `.claude/docs/technical-preferences.md`). Rationale — a centralized `InflatedBoundsRegistry` at the Input layer would require every dynamic spawn/despawn (gophers mid-Rush, consumables mid-deploy) to register/deregister with Input, impacting the 60fps frame budget and adding state coupling that Unity's native collider model does not require.

- **R6.a `DeviceDisplayMetrics` singleton.** Input exposes a read-only `DeviceDisplayMetrics` struct (pixels-per-point, pixels-per-dp, safe-area rect, screen aspect ratio) populated in `Initialize()` before any touch event fires. Consumers reference this to derive correct sizes without platform `#ifdef`s. Registered as a cross-system constant in `design/registry/entities.yaml` (post-authoring Section 5b).
- **R6.b Debug-build floor assertion.** In `UNITY_EDITOR` and `DEVELOPMENT_BUILD`, every consuming system's UI element / collider registered through the generic `IInteractable.RegisterHitBounds(Rect)` helper asserts that the bounds width AND height meet the accessibility floor. Violations produce a blocking assertion (not a silent clamp — silent clamps hide authoring errors). Release builds omit the assertion.

**R7. Finger-occlusion offset for spawn-above-finger.** When a consuming system (Tool System) requests a spawn position for an object dragged under the thumb, Input exposes `InputState.GetOcclusionOffset(Vector2 touchPoint) → Vector2`. Default offset: `+72dp` on the Y axis (≈1.5× a 48dp average thumb-pad diameter). If `touchPoint.y + 72dp > Screen.safeArea.top`, clamp to `Screen.safeArea.top - 8dp` (breathing room from ceiling). Fixed-offset over screen-relative — at 19.5:9 vs 21:9, screen-relative offsets feel inconsistent. Tuning range in Section G.

**R8. Input produces zero feedback.** No sounds, visuals, animations, haptics, or particle spawns originate from `InputDispatcher`. Input publishes a typed event on Event Bus and returns. All feedback is the subscribing systems' concern (HUD for button flash, VFX/Juice for tap pop, Audio Bus for tap SFX, Haptics for rumble). Enforcement — design-review gate: `InputDispatcher.cs` imports zero audio/visual/haptics namespaces; no `Play*()`, `Instantiate()`, `UnityEngine.Rendering.*`, or `UnityEngine.InputSystem.XInput.Haptics.*` calls.

**R9. MVP accessibility floors — configurable, UI deferred to VS.** Input reads an `AccessibilitySettings` struct from `PlayerPrefs` on `Initialize()` with three configurable values:

| Setting | MVP Range | Default | VS UI |
|---|---|---|---|
| Tap duration ceiling (R3.a) | 300–600ms | 300ms | Yes — slider in Accessibility menu |
| Drag-distance slop (R3.b) | 10–20pt/dp | 10pt/dp | Yes — slider |
| Long-press threshold (R3.c) | 300–1000ms | 300ms (suppressed at MVP) | Deferred — feature gated |

These paths exist at MVP so the VS Accessibility Service adds UI configuration without re-architecting Input.

**R10. Suppression gate — no events in any Suppressed state.** When `InputDispatcher.State` is `Uninitialized`, `Disposed`, or any `Suppressed.*`, all touches are discarded: no `input.*` events fire, `InputState.CurrentPointer.IsPressed == false`, `InputState.CurrentPointer.ScreenPosition` returns its last valid value (NOT zero — avoids consumers snapping to origin). Any in-flight gesture (begun but not classified) at suppression entry is silently abandoned; its eventual `ended` fires no event.

- **R10.a Phase suppression.** Input subscribes to `gameplay.level.phase_changed` on Event Bus (Gameplay namespace, T2 tier). On any `Prep→Rush`, `Rush→Paused`, or `Rush→Complete` transition, Input enters `Suppressed.PhaseTransition` for `PhaseTransitionSuppressDuration` ms (Section G knob). Input does NOT query Level Runtime directly — Level Runtime is a downstream consumer of Input; subscribing to the bus preserves zero-upstream-dependency.
- **R10.b Modal suppression.** Modal/Dialog calls `InputDispatcher.PushModalSuppression(modalId)` and `.PopModalSuppression(modalId)` directly (not via Event Bus — modal open/close must guarantee suppression before the next frame's touch processing). Depth-counted — nested modals do not prematurely lift suppression.
- **R10.c Load suppression.** Save/Load calls `InputDispatcher.SetLoadSuppression(bool)` at `LoadingFromDisk` enter/exit. Mirrors Save/Load's state machine (see save-load.md Section C, States and Transitions).
- **R10.d Priority.** `Suppressed.Loading` > `Suppressed.Modal` > `Suppressed.PhaseTransition`. All three are independent flags; `Active` is entered only when all three are simultaneously inactive. Debug-log state name is the highest-priority active flag.

**R11. Pause gesture cancellation (show-stopper fix).** On `OnApplicationPause(true)`: Input immediately abandons any in-flight gesture state, sets `_activeTouchId = -1`, and transitions to whatever suppression state is appropriate (Loading if Save/Load signals it, else a temporary resume-guard state). No `input.drag.end` fires on pause even if a drag was active — the pause itself is the cancel signal. On `OnApplicationPause(false)`: Input resets `_activeTouchId = -1` before the first post-resume `began` is processed. This prevents the Unity Input System's post-resume synthetic `ended` event (the OS reports finger-up state on resume; Unity fires `ended` against the pre-pause touch state) from firing a stale `drag.end` on a gesture the player already abandoned. Enforcement — PlayMode integration test: begin drag → trigger `OnApplicationPause(true)` → `OnApplicationPause(false)` → verify no `drag.end` fires; verify new touch after resume fires cleanly.

**R12. No gameplay logic in Input.** `InputDispatcher.cs` and `InputState.cs` do not `using` any of `MoneyMiner.Gameplay`, `MoneyMiner.Tools`, `MoneyMiner.Collection`, `MoneyMiner.Economy`, `MoneyMiner.Progression`. Input does not inspect hit-target components to decide event payload content (e.g., Input does NOT check "is this a coin?" to route to a different event). Whether a tap hits a coin, a ramp, a gopher, or a menu button is entirely the subscriber's concern. Enforcement — CI grep on `using` directives.

**R13. Input Pipeline Budget.** The full Fire→handler chain operates under a layered wall-clock budget:

- **R13.a Pre-Fire pipeline ≤ 8 ms.** From `EnhancedTouch.Touch.activeTouches` read through tap/drag classifier through `EventSystem.RaycastAll` + `Physics2D.OverlapPointNonAlloc` through `TapPayload` / `DragPayload` construction to the `EventBus.Fire(channel, payload)` entry. Measured inside `InputDispatcher.Update()`.
- **R13.b T1 subscriber chain ≤ 8 ms.** From `Fire` entry through last subscriber return. This is Event Bus's existing T1 budget (see event-bus.md T1_latency_max = 16.67 ms; Input accepts half as its slice).
- **R13.c Total ≤ 16.67 ms.** R13.a + R13.b must sum to the Pillar 5 input→render frame budget on the ADR-003 reference device.
- **R13.d T1 subscriber cap.** No `input.*` channel may have more than 8 T1 subscribers. Any subscriber whose handler measures >2 ms on the reference device MUST defer heavy work to `Task.Run` (off-main) or to the next frame via `EventBusPlayerLoopSystem` deferred queue.
- **R13.e CI gate tiers.** Development Build (with profiler markers enabled) asserts ≤ 20 ms total (R13.a + R13.b); Release Build on ADR-003 reference device asserts ≤ 16.67 ms. The 20 ms Dev threshold absorbs `Stopwatch.GetTimestamp` profiler-marker overhead (~0.1 ms × multiple markers) and IL2CPP stripping differences between Dev (None) and Release (High).

Enforcement — CI PlayMode instrumentation on `InputDispatcher.Update()` measures R13.a wall-clock; Event Bus AC-F4-equivalent measures R13.b; both reported per-run.

### States and Transitions

| State | Entry Condition | Exit Condition | Behavior |
|---|---|---|---|
| **Uninitialized** | Default on `InputDispatcher` instantiation | `Initialize()` completes (EnhancedTouch enabled, safe-area cached, DeviceDisplayMetrics populated, Event Bus subscription for `gameplay.level.phase_changed` registered) | Discards all touches. `CurrentPointer.IsPressed = false`. No events published. Must transition to Active before the first gameplay frame. |
| **Active** | `Initialize()` completes successfully OR any `Suppressed.*` state exits and all three suppression flags are false | Any suppression entry below OR `Dispose()` called | Classifies touches. Publishes `input.tap.main`, `input.drag.start`, `input.drag.end`. Updates `CurrentPointer` every `Update`. Pre-Fire pipeline operates per R13. |
| **Suppressed.PhaseTransition** | `gameplay.level.phase_changed` received on Event Bus | `PhaseTransitionSuppressDuration` timer elapses AND modal depth = 0 AND load suppression = false | In-flight gesture abandoned. `CurrentPointer.IsPressed = false`. No events published. Timer begins on entry. |
| **Suppressed.Modal** | `PushModalSuppression(modalId)` called, depth goes from 0 to N≥1 | `PopModalSuppression(modalId)` brings depth back to 0 AND load suppression = false AND no phase-transition timer active | In-flight gesture abandoned. `CurrentPointer.IsPressed = false`. No events published. Depth-counted — nested modals each push and pop cleanly. |
| **Suppressed.Loading** | `SetLoadSuppression(true)` from Save/Load on `LoadingFromDisk` enter | `SetLoadSuppression(false)` on `LoadingFromDisk` exit AND modal depth = 0 AND no phase-transition timer active | In-flight gesture abandoned. `CurrentPointer.IsPressed = false`. No events published. Boolean (non-stacking) — only one Save/Load operation in progress at a time. |
| **Disposed** | `Dispose()` called (scene unload, app quit) | Terminal — no exit | Unsubscribes from Event Bus. Calls `EnhancedTouchSupport.Disable()`. `CurrentPointer.IsPressed = false`. All future `GetState`/method calls throw `InvalidOperationException`. |

### Interactions with Other Systems

**Upstream (Input reads from / subscribes to):**

| System | Reads | Interface | Ownership |
|---|---|---|---|
| **Event Bus** | `gameplay.level.phase_changed` subscription for R10.a phase suppression | `SubscribeAutoManaged<PhaseChangedPayload>(channel, OnPhaseChanged, this)`; lifecycle auto-managed | Event Bus owns dispatch; Input is a subscriber. Input itself is never a publisher to the Gameplay namespace. |
| **Save/Load** | `SaveManager.SetLoadSuppression(bool)` direct call at `LoadingFromDisk` transitions | Direct MonoBehaviour method call (no Event Bus — must be synchronous with `LoadingFromDisk` state enter/exit) | Save/Load notifies Input; Input trusts the signal. |
| **Modal/Dialog** | `InputDispatcher.PushModalSuppression(modalId)` + `.PopModalSuppression(modalId)` direct calls at modal open/close | Direct method call; depth-counted; non-Event-Bus | Modal/Dialog notifies Input synchronously to guarantee suppression before the next touch frame. |
| **Scene/App State Manager** | `SceneManager.sceneLoaded` Unity callback for Initialize on scene entry | Unity built-in event | Scene Manager owns lifecycle; Input `Initialize()` hooks the callback. |

**Downstream (systems that subscribe to Input's Event Bus channels):**

| Subscriber | Channel | Tier | Payload Consumed | Purpose |
|---|---|---|---|---|
| **VFX/Juice** (VS) | `input.tap.main`, `input.drag.start`, `input.drag.end` | T1 | `TapPayload`, `DragPayload` | Tap pops, drag trail, deploy flash |
| **Haptics** (VS) | `input.tap.main` | T1 | `TapPayload` | Light tap on success, heavier on deploy |
| **HUD** (MVP) | `input.tap.main`, `input.drag.start`, `input.drag.end` | T1 | Both | Button highlight, drag-from-tray preview, consumable deploy indicator |
| **Tool System** (MVP) | `input.drag.start`, `input.drag.end` + `InputState.CurrentPointer` polling | T1 + poll | `DragPayload` for start/end; `CurrentPointer` for per-frame preview | Drag-to-place, drop-zone validation, tool snapping |
| **Collection System** (MVP) | `input.tap.main` | T1 | `TapPayload` (with `Target` hit-test result) | Tap-to-catch airborne loot |
| **Biome Map** (MVP) | `input.tap.main`, `input.drag.start`, `input.drag.end` | T1 | Both | Level-select tap, swipe-to-pan map |
| **Main Menu / Settings / Pause** (MVP) | `input.tap.main` | T1 | `TapPayload` | Menu navigation |
| **Modal/Dialog + Level End** (MVP) | `input.tap.main` | T1 | `TapPayload` | Modal dismiss, level-end continue |

**Data contracts — `InputState.CurrentPointer`:**

```csharp
public readonly struct InputPointerState {
    public Vector2 ScreenPosition;                    // safe-area-local pixels
    public bool IsPressed;                             // true from began to ended/canceled
    public float AgeSeconds;                           // seconds since began; -1 if !IsPressed
    public PointerClassification CurrentClassification; // Unknown | Tapping | Dragging
}
```

`TapPayload` and `DragPayload` are declared on Event Bus (see event-bus.md Section C Interactions table); Input is the publisher. Payload shapes are NOT duplicated here.

**Not routed through Input:**
- Platform-specific system events (volume buttons, home gesture, notification tray) — handled by Unity's native lifecycle callbacks, not Input
- Keyboard/mouse at MVP — no support (touch-only per technical-preferences.md); post-MVP expansion would require an ADR to repeal R2 and add device abstraction
- In-Editor rapid-iteration keyboard shortcuts for playtesters — live in `DEVELOPMENT_BUILD`-gated `EditorPlaytestInput.cs`, which IS on Input's allowlist but does not publish on Event Bus (writes directly to playtest log per CD-SYSTEMS ownership rule)

## Formulas

Input System is a dispatch layer with minimal continuous mathematics. Most threshold values (safe-area clipping, suppression durations, occlusion constants) are tuning knobs defined in Section G and referenced by rule in Section C. Two calculations require formal treatment because they have non-trivial boundary conditions or produce outputs that downstream CI tests key on.

### D.1 InputPipelineBudget

**Purpose.** Formalizes the latency constraint from R13 so that CI assertions and per-handler budget decisions have a single symbolic source to reference.

**Expression.**

```
TotalLatency_ms = PreFire_ms + T1Chain_ms

where:
  PreFire_ms   ≤ BudgetCeiling_ms / 2
  T1Chain_ms   ≤ BudgetCeiling_ms / 2
  TotalLatency_ms ≤ BudgetCeiling_ms

  BudgetCeiling_ms = 16.67   if DeviceProfile = Release   (= registry t1_latency_max)
  BudgetCeiling_ms = 20.00   if DeviceProfile = Dev
```

**Variable table.**

| Symbol | Type | Range | Description |
|--------|------|-------|-------------|
| `PreFire_ms` | float | 0 – 8.0 (Release) / 0 – 10.0 (Dev) | Wall-clock ms from `EnhancedTouch.activeTouches` read through `TapPayload` / `DragPayload` construction to `EventBus.Fire()` entry. Measured inside `InputDispatcher.Update()`. |
| `T1Chain_ms` | float | 0 – 8.0 (Release) / 0 – 10.0 (Dev) | Wall-clock ms from `EventBus.Fire()` entry through last T1 subscriber return on any `input.*` channel. Each subscriber's contribution is additive and serial. |
| `BudgetCeiling_ms` | float | {16.67, 20.00} | Total latency ceiling, selected by build profile. 16.67 = one 60fps frame (identical to registry `t1_latency_max` from event-bus.md). 20.00 = Development Build allowance for profiler-marker overhead (~0.1 ms × markers + IL2CPP-None vs IL2CPP-High stripping differences). |
| `DeviceProfile` | enum | {Release, Dev} | Build profile at CI time. Release = ADR-003 reference device with IL2CPP High stripping. Dev = editor or development build with profiler instrumentation enabled. |
| `TotalLatency_ms` | float | 0 – 20.0 (bounded by ceiling) | Sum of PreFire and T1Chain. Must not exceed BudgetCeiling for the active profile. Output is clamped by CI assertion — an overrun fails the test, it does not silently continue. |

**Output range.** Bounded above by `BudgetCeiling_ms`; bounded below by 0 (theoretical; minimum measured overhead is ~0.4 ms for empty subscriber chain). The ceiling is a CI hard gate, not a runtime clamp.

**Worked example — nominal Release build, 3 subscribers.**

```
PreFire_ms   = 4.2   (classifier + two raycasts + payload alloc)
T1Chain_ms   = 3.1   (Tool System 1.8ms + HUD 0.8ms + VFX 0.5ms)
TotalLatency = 4.2 + 3.1 = 7.3 ms   →   7.3 ≤ 16.67   PASS

DeviceProfile = Dev, same chain:
BudgetCeiling = 20.00 ms
7.3 ≤ 20.00   PASS (comfortable margin)
```

**Rationale.** The half-and-half split (PreFire ≤ half, T1Chain ≤ half) is a load-balancing contract, not a physics law. It prevents one side from crowding out the other: a slow raycast pass cannot steal budget from subscribers, and a slow subscriber cannot justify loosening the pre-fire cost. The asymmetric Dev ceiling absorbs Stopwatch overhead without changing the Release target — the production experience is unchanged.

**Validation.** CI PlayMode test on ADR-003 reference device: 1000 synthetic touch events across 60 frames; 99th-percentile `TotalLatency_ms < 16.67`; mean `PreFire_ms < 6.0` and mean `T1Chain_ms < 6.0`. Dev build gate: 99th-percentile `TotalLatency_ms < 20.0`.

### D.3 GestureClassification

**Purpose.** Defines the piecewise mapping from raw touch input `(d, t)` to gesture class. Replaces an implicit decision tree with explicit, gap-free boundary conditions so implementers and reviewers can verify completeness and testers can construct boundary-value cases mechanically.

**Expression.**

Let `d` = cumulative movement distance in platform-independent units (pt on iOS, dp on Android), and `t` = touch duration in milliseconds from `began` phase to current frame. `v` = finger lift velocity at `ended` (magnitude of last-frame delta / last-frame delta-time).

```
GestureClass(d, t, ended, v) =

  InProgress   if  t ≤ t_tap_max  AND  d < d_slop  AND  NOT ended
  InProgress   if  t ≤ t_early    AND  d < d_early AND  NOT ended

  Tap          if  ended  AND  d < d_slop  AND  t ≤ t_tap_max
  Tap          if  ended  AND  d < d_slop  AND  t > t_tap_max  AND  v < v_lateLift
               (late-lift forgiveness — motor-impaired accommodation, R3.a)

  Drag         if  d ≥ d_slop              (threshold crossing triggers immediately, any t)
  Drag         if  d ≥ d_early  AND  t ≥ t_early

  LongPress    if  ended  AND  d < d_slop  AND  t > t_tap_max  AND  v ≥ v_lateLift
               (reserved; silently discarded at MVP per R3.c)
```

**Variable table.**

| Symbol | Type | Range | Description |
|--------|------|-------|-------------|
| `d` | float | 0 – unbounded (pt/dp) | Cumulative movement distance since `began`. Computed as sum of per-frame deltas, not straight-line origin distance (handles circular motion correctly). |
| `t` | float | 0 – unbounded (ms) | Duration since the touch entered `began` phase on the current classifier frame. |
| `d_slop` | float | 10 – 20 (pt/dp) | Tap movement tolerance. Default 10. Accessible range 10–20 (Section G). Below 10: unintended drag promotions from normal motor jitter. Above 20: intended drags misclassified as taps on short swipes. |
| `d_early` | float | 6 – d_slop (pt/dp) | Early-drag trigger distance used only when `t ≥ t_early`. Default 6. Invariant: `d_early < d_slop` enforced by `OnValidate`. |
| `t_tap_max` | int | 300 – 600 (ms) | Tap duration ceiling. Default 300 ms. Accessible range 300–600 ms (Section G). |
| `t_early` | int | 100 – 200 (ms) | Early-drag trigger time. Default 150 ms. At `t ≥ t_early`, the smaller `d_early` threshold activates to catch deliberate slow Rush-phase drags before the full tap window closes. |
| `v` | float | 0 – unbounded (pt/ms) | Finger lift velocity at `ended` phase. Used only for late-lift forgiveness branch. |
| `v_lateLift` | float | 0 – 0.1 (pt/ms) | Velocity ceiling for late-lift Tap forgiveness. Default 0.02 pt/ms. Touches above this velocity at lift are LongPress (discarded), not Tap. |
| `GestureClass` | enum | {InProgress, Tap, Drag, LongPress} | Output classification. `InProgress` is transient (never published). `Tap` → `input.tap.main`. `Drag` → `input.drag.start` on threshold crossing. `LongPress` → silently discarded at MVP. |

**Output range.** Categorical, four values. Every `(d, t, ended, v)` combination maps to exactly one class — the piecewise definition is exhaustive and mutually exclusive. `InProgress` is never published to Event Bus; it is an internal state. The late-lift forgiveness branch and the `LongPress` branch are the only cases that require the `ended` signal to resolve — all `Drag` promotions fire on the frame the threshold is crossed, not on lift.

**Worked examples — three paths.**

```
Path A — clean tap:
  d=3pt, t=180ms, ended=true, v=0.01 pt/ms
  d < d_slop(10), t ≤ t_tap_max(300), v < v_lateLift(0.02)   →   Tap
  Publish input.tap.main

Path B — slow deliberate drag (Rush-phase ramp placement):
  d=7pt, t=160ms, ended=false
  d ≥ d_early(6) AND t ≥ t_early(150)   →   Drag
  Publish input.drag.start this frame
  (without early trigger: d=7 < d_slop(10); touch would remain InProgress until t=300ms — 140ms late)

Path C — long press (motor-impaired, slow lift):
  d=2pt, t=420ms, ended=true, v=0.05 pt/ms
  d < d_slop, t > t_tap_max(300), v ≥ v_lateLift(0.02)   →   LongPress
  Discard silently (MVP — slot reserved per R3.c)
```

**Rationale.** The early-drag trigger (Path B) is the load-bearing case for Rush-phase usability: without it, a deliberate 150 ms slow drag sits in `InProgress` until the 300 ms tap window closes, wasting half of the Tool System's 100 ms response window. The two-variable threshold `(d_early, t_early)` lets tuning tighten or loosen the slow-drag sensitivity without touching the standard slop, keeping tap and drag classifier sensitivities independent. The piecewise form makes boundary tests mechanically derivable — every variable has an exact threshold, so QA writes `(threshold − 1, at-threshold, threshold + 1)` cases without guessing.

**Validation.** Unit tests covering all eight boundary conditions: `d = d_slop − 1`, `d = d_slop`, `d = d_slop + 1`, `t = t_tap_max`, `t = t_early − 1`, `t = t_early` (with `d = d_early`), late-lift forgiveness path, long-press discard path. Integration test: simulated slow Rush-phase drag (`d = 7pt` at `t = 160ms`) confirms `input.drag.start` fires at frame ≈ 160 ms, not frame 300 ms.

### Formula Candidates Moved to Tuning Knobs

The following geometric and capacity constants are tuning knobs (Section G), not formulas: `occlusion_offset_dp` (default 72) and `edge_padding_dp` (default 8) from R7's spawn-above-finger clamp `min(touch.y + offset, safeArea.top − padding)`; `PhaseTransitionSuppressDuration_ms` from R10.a; `SubscriberCountMax` (8) and `PerHandlerCostThreshold_ms` (2) from R13.d. Each is a constant with a tuning range — no interacting variables, no emergent output. Treating them as formulas would be padding.

## Edge Cases

### OS / Platform

- **If `_activeTouchId` is tracked but the ID vanishes from `EnhancedTouch.Touch.activeTouches` without firing `ended` or `canceled`** (OS-level touch steal — Android OEM gesture navigation, notification tray swipe, Samsung edge panel): `InputDispatcher.Update()` performs an absence-detection pass — if the tracked ID is not present in `activeTouches` AND no `ended`/`canceled` was received, treat as an implicit cancel: clear `_activeTouchId`, abandon the in-flight gesture, publish nothing. Log `[InputDispatcher] Tracked touch {id} vanished — implicit cancel`. This is independent of R11's pause cancellation, which only covers `OnApplicationPause`.

- **If `ended` or `canceled` arrives for a finger ID that does not match `_activeTouchId`** (spurious OS ghost event; also the aftermath of an implicit cancel): discard silently. No event published, no state change, no log in release builds. Rationale — these events are noise at the current abstraction; matching IDs is the only authoritative signal that a tracked gesture ended.

- **If `Screen.safeArea` changes mid-session** (Android foldable unfold, split-screen activation): out of MVP scope. Money Miner is portrait-locked per `technical-preferences.md` and foldables are not a supported form factor for v1.0. If a foldable unfolds during play, the cached safe-area rect becomes stale and edge-adjacent touches may misbehave until scene reload. Post-launch consideration: add a `Screen.safeArea` change check in `Update` and re-cache + emit a `DeviceDisplayMetrics` changed event.

- **If the OS recycles a touch ID after a tracked finger vanished implicitly** (EC-OS1 fired, new touch arrives with the same finger ID the OS just recycled): a `began` matching `_activeTouchId` while `IsPressed` is already true (compound failure: EC-OS1's absence-detection did not run yet) is treated as an implicit cancel of the prior gesture before starting fresh — NOT silently skipped. Without this rule, the prior gesture's state would leak into the new touch. Enforcement — PlayMode test simulating OS ID recycling.

### State Machine / Lifecycle

- **If `SetLoadSuppression(true)` fires mid-drag**: R10.c abandons the in-flight gesture and clears `_activeTouchId`. When load suppression lifts and the Unity Input System delivers the physical `ended` event for the touch that was still down during load, the arriving `ended`'s finger ID does NOT match `_activeTouchId` (cleared during abandonment). The ghost `ended` is discarded under the EC-OS2 rule — no event published. This is why EC-OS2's ID-mismatch rule is load-bearing for suppression recovery.

- **If `PushModalSuppression(modalId)` is called during a T1 subscriber handler that was triggered by Input's own `Fire`** (Modal/Dialog opens in response to a tap, calling into Input's suppression API while Input is still inside its dispatch stack): `PushModalSuppression` sets the suppression flag atomically but gesture abandonment is DEFERRED to the next `InputDispatcher.Update()` entry. Abandonment code does NOT run from inside `PushModalSuppression` — re-entering the state machine from an arbitrary subscriber call stack depth would corrupt state.

- **If `InputDispatcher.Initialize()` is called before `EnhancedTouchSupport.Enable()` completes, or called a second time without an intervening `Dispose()`** (scene reload, additive scene load): `Initialize()` asserts `State == Uninitialized` at entry and throws `InvalidOperationException` with message `[InputDispatcher] Already initialized — call Dispose() before re-initializing`. Double-registration of the `gameplay.level.phase_changed` subscription would cause every phase change to trigger two suppression entries. Enforcement — integration test.

- **If a T1 subscriber mutates Input's suppression state inside its own handler** (e.g., Tool System places a tool → fires `gameplay.level.quota_progress` → Level Runtime emits `gameplay.level.phase_changed` → Input's subscription wants to enter `Suppressed.PhaseTransition`): Input snapshots suppression flags at `Update()` entry and uses the snapshot for the entire frame's processing. Suppression mutations during the T1 chain take effect the FOLLOWING frame, not mid-dispatch. Prevents re-entrancy hazards in the dispatcher.

### Classifier Boundary

- **If a frame drops to ≥ 200 ms (severe thermal throttle or background work stall)**: D.3 uses `t = elapsed since began`, sampled per `Update`. If `began` arrives at frame N, frame N takes 220 ms, finger lifts mid-frame — `t` at `ended` is 220 ms ≤ `t_tap_max(300)`, fires as Tap correctly. **Dangerous case**: if `began` is processed in frame N (220 ms duration) and finger lifts at the START of frame N+1 (also 220 ms), `t` at `ended` = 440 ms with near-zero `v` → classified as LongPress and silently discarded (R3.c). A player-intended quick tap is lost to frame-rate collapse. Accepted for MVP — Pillar 5 assumes a healthy 60 fps / 30 fps floor; sustained sub-5 fps is a thermal emergency, not a classification concern. Flag in Section I (Open Questions) if post-MVP telemetry shows late-lift taps disproportionately lost on specific devices.

- **If a second finger's `began` and the first finger's `ended` arrive in the same `Update`** (rapid multi-touch): Input processes the tracked finger's lifecycle event FIRST (publish tap / end drag, clear `_activeTouchId`), THEN evaluates any new `began`. If the new `began` is from a second finger, it is ignored per R2 (first-touch only). Processing in the wrong order would leave the first finger's gesture dangling and the second finger's `began` silently consumed. Enforcement — unit test with synthetic activeTouches buffer containing both phases.

- **If `t_tap_max` is raised to the accessibility ceiling (600 ms)** and a player holds a stationary finger for 580 ms with a slow lift (`v < v_lateLift`): classified as Tap per D.3 late-lift forgiveness. **UX discontinuity to document for playtesters**: under the 600 ms setting, extended stationary holds the player intended as "pause to think" fire as taps. This is a cost of motor-impaired accommodation. Flag in Section I for VS Accessibility Service UX review — the VS settings UI should describe this trade-off in the accessibility menu copy.

### Hit-Test Resolution

- **If the hit-tested GameObject (UI or world) is destroyed between the raycast and `EventBus.Fire`**: after `OverlapPointNonAlloc` or `RaycastAll`, Input performs a null-and-destroyed check (`target == null || !target`) before populating `TapPayload.Target`. If destroyed, `Target = null` (treat as playfield tap). Failing this check would propagate a phantom-object reference to 8 T1 subscribers, causing cascading `MissingReferenceException`. Enforcement — PlayMode test destroying the target object synchronously inside an earlier handler.

- **If multiple UI canvases share the same `sortingOrder` at the tap point** (authoring error): `EventSystem.RaycastAll` returns results in sort-then-distance order; at identical sort order the result order is Unity-implementation-defined. A `DEVELOPMENT_BUILD` scene-validator asserts every active canvas in the scene has a unique sort order. Runtime recovery is NOT attempted — Input cannot guess authoring intent. Release builds accept whatever Unity returns; the scene validator in Dev catches this before release.

- **If the tap falls on a world collider whose layer is excluded from Input's layer-filtered mask**: `OverlapPointNonAlloc` returns zero hits; `Target = null`; the player perceives the tap as "missing" a visible object. This is intentional — Input's layer mask defines which world objects are tap-eligible, and excluded objects are visual-only per design (Data Registry + Tool System decide which ToolConfigs register on tap-eligible layers). If a consuming system needs expanded hit eligibility, it updates its collider's layer, not Input's mask. Documentation handoff to consumer GDDs.

- **If a touch begins at exactly the `Screen.safeArea` boundary pixel**: inclusive of the safe area. The rule is `touch.position IN safeArea` (closed interval on all four edges). Only touches strictly OUTSIDE the safe-area rectangle are suppressed. Edge-of-screen taps are a common player affordance (reaching a button with the edge of the thumb) and must not be lost to a strict-inequality bug.

### Configuration / Tuning Extremes

- **If a consuming system queries `InputState.DeviceDisplayMetrics` before `InputDispatcher.Initialize()` completes** (Unity script execution order violation): the query throws `InvalidOperationException` with message `[InputDispatcher] DeviceDisplayMetrics not populated — InputDispatcher must execute before consumers in Script Execution Order`. Silent return of a default value would mask authoring errors in the scene setup. `InputDispatcher` is pinned to a negative Script Execution Order number (e.g., -1000) so it always initializes before default-order MonoBehaviours. Enforcement — integration test that instantiates a consumer MonoBehaviour at default order and asserts the InvalidOperationException fires if Input is mispinned.

## Dependencies

[To be designed]

## Tuning Knobs

[To be designed]

## Visual/Audio Requirements

[To be designed]

## UI Requirements

[To be designed]

## Acceptance Criteria

[To be designed]

## Open Questions

[To be designed]
