# Event Bus

> **Status**: Approved (pending independent `/design-review` in a fresh session)
> **Author**: Robert Michael Watson + systems-designer / unity-specialist / creative-director / qa-lead
> **Last Updated**: 2026-04-20
> **Implements Pillar**: Foundation for Pillars 1 (Cute Chaos), 2 (Readable at a Glance), 5 (No Input Goes Silent). Indirect support for 3 and 4 via decoupling.
>
> **Creative Director Review (CD-GDD-ALIGN)**: CONCERNS → RESOLVED 2026-04-20 (3 fixes applied: AC-F6 end-to-end Feedback Floor added; R8 bootstrap buffer locked; R9 Analytics codegen locked)

## Summary

The Event Bus is Money Miner's decoupled publish/subscribe dispatch mechanism — the thin layer that lets gameplay, UI, audio, and feedback systems communicate without holding direct references to each other. Gameplay systems fire typed events (`CollectionSuccess`, `BombDetonated`, `DogCaughtCat`) onto named channels defined by the Data Registry's `AudioEventTaxonomy` and `VisualEventTaxonomy`; feedback systems (Audio Bus, VFX/Juice, HUD, Modal/Dialog) subscribe to channels by ID and react without the publisher knowing they exist. The Event Bus itself owns zero gameplay logic — it is the wire, not the signal.

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: `None (pairs with Data Registry taxonomies)`

## Overview

The Event Bus exists to make the Feedback Floor contract (CD-SYSTEMS) physically possible without coupling Core gameplay systems to VFX, audio, or UI. Without it, Collection System would need a direct reference to Audio Bus, VFX/Juice, AND HUD just to fire one "coin collected" beat — and every Pillar 5 feedback moment would compound that tangle across 11+ systems. With it, Collection System fires one event; anyone who cares subscribes. The decoupling is what makes MVP fun-test validity possible: swapping placeholder VFX for full VFX/Juice at Vertical Slice requires zero Core changes, only new subscribers.

The Event Bus provides three capabilities: (1) **typed event dispatch** — events carry strongly-typed payload structs, never raw `object` or string blobs; (2) **channel routing** — event IDs from the taxonomies (e.g., `audio.tool.place.ramp`, `vfx.collect.coin.splash`) route events to subscribers registered on matching channels; (3) **lifecycle safety** — subscriptions auto-unregister on MonoBehaviour destruction to prevent dangling references and stale-closure memory leaks. The bus itself enforces four rules: no allocation in the dispatch hot path (Mali-G52 budget), no synchronous cross-frame recursion (events fired during dispatch queue for next frame), no string-typed channels at call sites (use generated string constants from the taxonomy), and no silent swallowing of unhandled events in debug builds.

This is the Pillar 5 enabler. Every tap, drag, collection, and hazard event routes through Event Bus to its feedback subscribers within the 100ms latency budget. It is also the Pillar 1 enabler indirectly: slapstick failure feedback (dog-cat cloud, bomb puff) only fires because the detection system published an event that VFX/Juice subscribed to. Like Data Registry, Event Bus succeeds by being invisible — if a player ever notices it, it has failed.

## Player Fantasy

Event Bus has no player-facing fantasy. No player will ever think "I love this game's pub/sub architecture" — and if they do, something has gone catastrophically wrong. This is plumbing. The fantasy it serves is everyone else's.

What Event Bus actually delivers is **the speed of the wink back**. In Money Miner, the player is the brain of a clever little farm heist — they tap, they prep, they react, and the farm answers. Pillar 5 ("No Input Goes Silent") is non-negotiable: every tap must get a reply inside 100ms, and that reply usually wants two or three voices speaking at once — a coin chirp, a number pop, a haptic thump, a subtle camera nudge. Event Bus is the switchboard that lets all of those voices answer the same tap without any of them knowing about each other. Publishers fire and forget; subscribers listen and respond. The farm feels alive because the wiring underneath doesn't care who's talking — it just makes sure everyone hears in time.

Without the bus, the failure mode is ugly and specific: either the tap handler grows a tangle of direct calls into audio, VFX, HUD, economy, and analytics (brittle, slow, and guaranteed to drop a response the first time someone refactors), or feedback channels start racing each other and the 100ms Feedback Floor quietly erodes into 180ms of jitter. Players won't be able to name it, but they'll feel the farm go slack. Event Bus is what keeps the heist feeling sharp — fast, coordinated, and a little smug about how effortless it looks.

## Detailed Design

### Core Rules

**R1. No hot-path allocation.** Dispatch must complete with zero heap allocations. Pre-pooled subscriber lists, no LINQ on dispatch path, no boxing of value-type payloads, no `params` keyword in `Fire<T>` signatures. Enforced by CI profiler assertion on a Rush-Phase stress scene (≤ 0 bytes/frame allocated by the bus).

**R2. No synchronous cross-frame recursion.** Events fired from inside a subscriber handler are queued and dispatched at the start of the next frame, not inline. A depth guard (int, frame-local) enforces this: depth > 0 triggers the deferred queue. The queue is a pre-allocated ring buffer (capacity: 64 events — tuning knob).

**R3. No string-typed channels at call sites.** All channel IDs are typed constants generated from the Data Registry taxonomies at editor/build time (an `AssetPostprocessor`-triggered codegen script writes `AudioEventIds.ToolPlaceRamp` / `VisualEventIds.CollectCoinSplash` const strings). Publishers reference these constants; passing a bare string literal to `Subscribe` or `Fire` is a CI static-analysis failure. This resolves Data Registry OQ-4 in favor of generated constants.

**R4. No silent swallowing in debug builds.** If an event is fired on a channel with zero active subscribers, a `Debug.LogWarning` fires with the channel SO as context (clickable in Unity Console). In release builds this check is stripped via `#if UNITY_EDITOR || DEVELOPMENT_BUILD`. A custom Editor Window also accumulates unhandled-event counts per channel for post-session audit (mirrors Data Registry's Balance Audit window pattern).

**R5. Taxonomy events and internal events are distinct channel namespaces.** Taxonomy events (whose IDs appear in `AudioEventTaxonomy` or `VisualEventTaxonomy` and match Data Registry R7 regex `^(audio|vfx)\.[a-z_]+\.[a-z_]+\.[a-z_]+$`) live in `ChannelNamespace.Feedback`. Internal gameplay events (e.g., `gameplay.level.phase_changed`) live in `ChannelNamespace.Gameplay` (format `^gameplay\.[a-z_]+\.[a-z_]+$`). A channel from one namespace cannot be used on the other — a runtime assertion enforces this in debug builds. Prevents gameplay subscribers from accidentally registering on audio-routed channels.

**R6. Subscriber dispatch order is undefined.** The bus makes no guarantee about which subscriber is invoked first. Any system requiring ordering (e.g., HUD must update before a post-frame screenshot) uses a two-phase pattern: subscribe to the event, then post a local update to a sorted execution queue it owns. Ordering dependencies between subscribers are a smell — the subscribing systems own resolution, not bus insertion order.

**R8 (locked from CD-GDD-ALIGN Concern 2): Bootstrap buffer for events fired before Ready.** The bus maintains a pre-allocated bootstrap buffer of up to 16 events. Any `Fire` call during the `Uninitialized` state (after the bus awakens but before all subscribers have completed registration — typically the first `Awake → Start` window) is queued to this buffer rather than throwing. On the first `Update` after the bus transitions to `Ready`, the bootstrap buffer drains in FIFO order via normal dispatch. Bootstrap buffer overflow (17th event) drops the newest event and logs `[EventBus] Bootstrap overflow` — this is strictly bounded; any system firing >16 events at boot is buggy by design. This replaces the original "throw on Uninitialized" behavior for the narrow scene-load window only; mid-session `Fire` on a Disposed bus still errors.

**R9 (locked from CD-GDD-ALIGN Concern 3): Analytics codegen path.** A companion `AssetPostprocessor` generates `AnalyticsSubscribers.cs` from the `gameplay.*` entries of every taxonomy SO at build time. Each generated entry is a per-channel explicit subscription that forwards the typed payload to the Analytics service. Matches the R3 codegen pattern; preserves R1 zero-allocation (each subscription is typed, no reflection); preserves R7 (no LINQ on payloads). Analytics consumers never manually subscribe to individual gameplay channels — the generated file is the single source of truth. Resolves OQ-3; preserves Pillar 3's analytics tuning loop.

**R7 (from Unity-specialist CI safety net).** Because typed payloads use `readonly record struct` (see "Typed Payload Pattern" below), CI static analysis must flag any LINQ expression or equality chain on a record-struct payload type inside a dispatch hot path method. `readonly record struct` is allocation-free when passed through `Fire<T>`, but misuse (e.g., `payloads.OrderBy(p => p.Key)`) allocates. This rule is listed as a Forbidden Pattern in `.claude/docs/technical-preferences.md`.

### Event Category Taxonomy

| Category | Channel Namespace | ID Schema (Data Registry) | Example |
|---|---|---|---|
| Taxonomy (Feedback) | `ChannelNamespace.Feedback` | `^(audio\|vfx)\.[a-z_]+\.[a-z_]+\.[a-z_]+$` | `audio.collect.coin.success` |
| Internal (Gameplay) | `ChannelNamespace.Gameplay` | `^gameplay\.[a-z_]+\.[a-z_]+$` | `gameplay.level.phase_changed` |

**Relationship**: a single gameplay occurrence may cause a publisher to fire on *both* a Gameplay channel (for state-machine subscribers: Level Runtime, Economy) AND a Feedback channel (for Audio Bus, VFX/Juice, HUD). These are two separate `Fire` calls with separate payloads. **No "fan-out by namespace" magic** — the publisher decides explicitly which channels to fire on. Keeps call sites auditable and keeps R4 effective per channel.

### Typed Payload Pattern

**Pattern**: `readonly record struct` (C# 10, Unity 6.3 LTS supports this). Zero-allocation (value type, stack-allocated through `Fire<T>`), immutable after construction (no subscriber can mutate a payload another subscriber will see), structural equality for free (trivial test assertions), `ToString` auto-generated. The `IEventPayload` marker interface is retained purely for tooling reflection (debug overlay, taxonomy validator) — no methods.

**Safety net (R7)**: misuse in LINQ/equality hot paths can still allocate. CI static analysis enforces no hot-path record-struct LINQ/equality.

```csharp
public interface IEventPayload { }

// Feedback channel — must match AudioEventTaxonomy key "audio.collect.coin.success"
public readonly record struct CollectionSuccessPayload(
    Vector2 WorldPosition,
    int CoinValue,
    bool IsCritical
) : IEventPayload;

// Fired on BOTH Feedback and Gameplay channels (two separate Fire calls)
public readonly record struct BombDetonatedPayload(
    Vector2 WorldPosition,
    int Radius,
    int CoinsDestroyed
) : IEventPayload;

// Gameplay channel only — no audio/visual entry in taxonomy
public readonly record struct PhaseChangedPayload(
    LevelPhase PreviousPhase,
    LevelPhase NextPhase,
    float TimeRemainingSeconds
) : IEventPayload;
```

### Subscription Lifecycle

**Subscribe API:**
```csharp
SubscriptionHandle Subscribe<T>(ChannelID channel, Action<T> handler)
    where T : struct, IEventPayload
```
Returns a `SubscriptionHandle` (struct wrapping an int token). Internally, each channel SO holds a pre-allocated `List<Action<T>>`.

**Auto-unsubscribe on MonoBehaviour destruction:**

Two paths, selected by subscriber type:

- **MonoBehaviour subscribers**: call `SubscribeAutoManaged<T>(channel, handler, this)`. The bus registers a thin `SubscriptionManager` component on the subscriber's GameObject (added automatically if absent). `SubscriptionManager.OnDestroy` iterates its token list and calls `Unsubscribe` for each. Zero allocations at unsubscribe time (pre-allocated token list).
- **Non-MonoBehaviour subscribers** (ScriptableObjects, pure C# services): manual `SubscriptionHandle.Dispose()` in a teardown method. In debug builds, the bus tracks live handles; any handle GC'd without `Dispose` logs an error.

**Leak prevention invariant**: in debug builds, the bus tracks live subscription count per channel. If count grows monotonically across two consecutive scene loads without any unsubscribes, an error is logged. Catches "forgot to Dispose a service-layer subscriber."

### States and Transitions

The bus is not a gameplay state machine, but has three operational states:

| State | Entry Condition | Exit Condition | Behavior |
|-------|----------------|----------------|----------|
| Uninitialized | Domain load start | Bus `Awake` completes | `Subscribe` throws immediately. `Fire` is captured into the R8 bootstrap buffer (capacity 16 events) and will drain on first `Update` after Ready. |
| Ready / Idle | Bus `Awake` completes | Dispatching starts OR domain unload | On entry: drain the bootstrap buffer via normal dispatch (FIFO). Then normal operation; `Subscribe` modifies channel tables; `Fire` dispatches synchronously |
| Dispatching | A `Fire` call is in progress | All subscriber handlers for that channel return | Any `Fire` called from a subscriber queues to the deferred ring buffer (R2). `Subscribe` is legal during dispatch — new subscriptions take effect next dispatch cycle, preventing "modified while iterating" crashes |
| Disposed | Scene unload or application quit | N/A — terminal | All subscriptions cleared; any `Fire` or `Subscribe` is a no-op with debug warning |

### Interactions with Other Systems

Latency tiers: **T1** (< 16ms, same frame) for input response, collection, slapstick hazard reads; **T2** (< 100ms, Feedback Floor budget) for state changes, economy updates, AI signals.

| Emitter | Event | Channel Namespace | Payload Type | Subscribers | Latency |
|---|---|---|---|---|---|
| Input System | `input.tap.main` | Feedback | `TapPayload(Vector2, GameObject?)` | VFX/Juice, Haptics, HUD | T1 |
| Input System | `input.drag.start` / `input.drag.end` | Feedback | `DragPayload(Vector2, Vector2)` | VFX/Juice, Tool Preview | T1 |
| Collection System | `audio.collect.coin.success` (+ gameplay channel) | Feedback + Gameplay | `CollectionSuccessPayload` | Audio Bus, VFX/Juice, HUD, Currency & Economy | T1 |
| Collection System | `audio.collect.miss` | Feedback | `CollectionMissPayload(Vector2)` | Audio Bus, VFX/Juice | T1 |
| Collection System | `gameplay.level.banked` | Gameplay | `BankedPayload(int TotalBanked)` | Currency & Economy, HUD | T2 |
| Tool System | `audio.tool.place.ramp` (+ gameplay) | Feedback + Gameplay | `ToolEventPayload(ToolType, Vector2)` | Audio Bus, VFX/Juice, Level Runtime | T2 |
| Tool System | `audio.tool.consumable.deploy` | Feedback | `ConsumableDeployedPayload(ToolType, Vector2)` | Audio Bus, VFX/Juice | T1 |
| Hazard System | `audio.hazard.bomb.detonate` (+ gameplay) | Feedback + Gameplay | `BombDetonatedPayload` | Audio Bus, VFX/Juice, Currency & Economy, HUD | T1 |
| Hazard System | `audio.hazard.cat.steal` / `audio.hazard.cat.caught` | Feedback + Gameplay | `CatEventPayload(Vector2, int AgentID)` | Audio Bus, VFX/Juice, Level Runtime | T1 |
| Critter AI | `audio.critter.dog.deploy` / `caught` / `return` | Feedback + Gameplay | `DogEventPayload(int AgentID, Vector2)` | Audio Bus, VFX/Juice, HUD | T2 |
| Gopher Spawn | `gameplay.gopher.launched` | Gameplay | `GopherLaunchedPayload(int GopherID, string LootTableId)` | Collection System, VFX/Juice | T2 |
| Level Runtime | `gameplay.level.phase_changed` | Gameplay | `PhaseChangedPayload` | HUD, Tool System, Currency & Economy | T2 |
| Level Runtime | `gameplay.level.complete` / `failed` | Gameplay + Feedback | `LevelEndPayload(bool Success, int FinalScore)` | Modal/Dialog + Level End, Audio Bus, Analytics (v1.0) | T2 |
| Level Runtime | `gameplay.level.quota_progress` | Gameplay | `QuotaProgressPayload(int Current, int Target)` | HUD, Currency & Economy | T2 |

**No event may span frames in the synchronous dispatch path** — R2's deferred queue handles re-entrant fires.

## Formulas

Event Bus has no gameplay balance formulas. It has operational constants derived from Unity 6.3 + Mali-G52 engineering budgets.

### D.1 Dispatch Latency Budget (by tier)

```
T1_latency_max = 16.67ms  (60fps frame budget)
T2_latency_max = 100ms    (Pillar 5 Feedback Floor)
```

| Variable | Value | Source |
|---|---|---|
| T1_latency_max | 16.67ms | 60fps mid-tier target (`.claude/docs/technical-preferences.md`) |
| T2_latency_max | 100ms | Pillar 5 "No Input Goes Silent" feedback floor |

### D.2 Deferred Queue Capacity

```
deferred_queue_capacity = 64 events (pre-allocated ring buffer; tuning knob)
```

Derivation: peak Rush Phase density = 40 active Rigidbody2D × avg 2 emits/body/frame = 80 events worst case. 64 is the capacity floor assuming most events subscribe via MonoBehaviour without firing inner events. Overflow triggers `Debug.LogError` with the overflowing event ID in debug; in release, the oldest queued event is dropped with a `[EventBus] Queue overflow` warning (advisory). Adjust upward if stress-scene profiling shows sustained overflow.

### D.3 Subscriber List Initial Capacity (per channel)

```
subscriber_list_initial_capacity = 8 per channel
```

Pre-allocated `List<Action<T>>` starting capacity per channel SO. 8 chosen because no MVP event in Section C has more than 5 documented subscribers. Underutilized channels waste a few pointers; over-subscribed channels reallocate once to 16 (Unity's `List<T>` growth strategy) — an acceptable one-time cost at scene load.

## Edge Cases

### Subscription / Lifecycle

- **If a MonoBehaviour subscribed via `SubscribeAutoManaged` has its GameObject destroyed mid-dispatch**: `SubscriptionManager.OnDestroy` fires and calls `Unsubscribe` — but the bus is in `Dispatching` state. Resolution: `Unsubscribe` during dispatch marks the entry as "pending removal" (nulls the delegate in the list); dispatch loop skips null entries; pending removals compact after dispatch completes. No exception, no stale-reference crash.
- **If a MonoBehaviour subscribes in `OnEnable` but is disabled mid-dispatch (not destroyed)**: subscription persists (disable ≠ destroy). Handler still fires. If disable-suspended behavior is desired, subscriber must unsubscribe in `OnDisable` manually — not via auto-managed path. Documented in the Subscribe API docstring.
- **If a non-MonoBehaviour subscriber's `SubscriptionHandle` is GC'd without `Dispose()`**: in debug builds, the bus logs `[EventBus] Leaked subscription for channel {id}` with the stack trace captured at Subscribe time (stored in a debug-only dictionary). In release builds, the subscription persists until scene unload — manageable memory waste, not a crash.
- **If `Subscribe` is called during `Dispatching` state for the same channel being dispatched**: new subscriber is registered but does NOT receive the current in-flight event. Takes effect next dispatch cycle. Prevents the race where a subscriber created by another subscriber's handler receives a partially-processed payload.

### Dispatch Safety

- **If a subscriber handler throws an unhandled exception**: dispatch loop catches, logs `[EventBus] Subscriber exception on channel {id}: {ex}` with subscriber type, and continues to the next subscriber. The bus never throws out of `Fire`. Prevents one buggy subscriber from breaking all subscribers on the same channel.
- **If an event is fired before the bus has initialized (Uninitialized state)**: `Fire` throws `EventBusNotReadyException` immediately (fail-fast per R5). Any system firing events before `Awake` completes is a system-ordering bug, not a bus bug.
- **If the deferred queue reaches capacity during re-entrant dispatch (64 events queued)**: in debug, `Debug.LogError` with queue contents; in release, drop oldest (FIFO), log `[EventBus] Queue overflow` advisory.
- **If two Feedback channels with the same ID string exist (duplicate SO assets)**: Data Registry's duplicate-ID check catches at editor time (AC-EC-A1 in Data Registry GDD). Bus assumes unique channel IDs and does not re-check at runtime.

### Channel / Namespace Safety

- **If a Gameplay-namespace channel ID is passed to a Feedback-expecting subscribe call (or vice versa)**: runtime assertion in debug builds throws `EventBusNamespaceMismatch`. In release, the subscribe silently succeeds but the channel never fires (taxonomy consumers don't fire gameplay events). Prevents gameplay-VFX cross-pollination silently corrupting feedback.
- **If a channel ID violates the taxonomy regex (e.g., uppercase chars)**: Data Registry's R7 validation catches at editor time. Bus assumes valid IDs.

### Re-entrancy

- **If a subscriber fires an event on the SAME channel it's currently handling (infinite recursion risk)**: R2 deferred queue handles this — the fire is queued, not recursed. However, if the handler unconditionally fires the same event every time, the deferred queue saturates in 64 frames. Safety net: debug build tracks dispatch-depth per channel; if the same channel fires > 10 times in consecutive frames via deferred queue, logs `[EventBus] Probable recursive fire on {channel}` as a warning.
- **If a subscriber fires a DIFFERENT event that transitively routes back to the same channel (indirect recursion)**: same deferred-queue protection applies — the nested fire queues for next frame. The recursive-fire warning threshold does NOT detect indirect recursion (would require graph analysis). Test coverage responsibility: subscriber authors must trace call chains mentally.

## Dependencies

### Upstream
**None.** Event Bus is Foundation with zero Money Miner system dependencies. Depends only on Unity 6.3 (MonoBehaviour lifecycle, ScriptableObject, `PlayerLoop` indirectly via `Update`).

### Downstream (hard — every Feedback/Gameplay cross-system communication routes through Event Bus)

| System | Priority | Nature |
|---|---|---|
| Data Registry | MVP | **Ownership contract** — Data Registry owns event ID schemas (`AudioEventTaxonomy` / `VisualEventTaxonomy`); Event Bus consumes those schemas as channel IDs via build-time codegen |
| Input System | MVP | Data dependency (fires `input.*` events) |
| Collection System | MVP | Data dependency (fires `collect.*` + `gameplay.level.banked`) |
| Tool System | MVP | Data dependency (fires `tool.*` events) |
| Critter AI | MVP | Data dependency (fires `critter.dog.*` events) |
| Hazard System | MVP | Data dependency (fires `hazard.*` events) |
| Gopher Spawn | MVP | Data dependency (fires `gopher.launched`) |
| Level Runtime | MVP | Data dependency (fires `level.phase_changed`, `quota_progress`, `complete`, `failed`) |
| Audio Bus | MVP | Subscriber (all `audio.*` channels) |
| HUD | MVP | Subscriber (quota, coin, tool events) |
| Modal / Dialog + Level End | MVP | Subscriber (`level.complete`, `level.failed`) |
| VFX / Juice | Vertical Slice | Subscriber (all `vfx.*` channels) |
| Analytics | Shippable v1.0 | Subscriber (all `gameplay.*` events for telemetry) |

### Cross-System Ownership Rule

**Data Registry owns schema** (event ID strings, payload class definitions).
**Event Bus owns delivery** (dispatch, subscription, lifecycle).
**Publishers own intent** (which events to fire when, which namespaces).
**Subscribers own response** (what to do on receive).

See Section C Interactions table for payload types and latency tiers per cross-system pair.

## Tuning Knobs

| Parameter | Current Value | Safe Range | Effect of Increase | Effect of Decrease |
|---|---|---|---|---|
| `deferred_queue_capacity` | 64 events | [32, 256] | More re-entrant fires survive; more preallocated memory (~512B per slot) | Smaller memory footprint; overflow risk during Rush Phase peak |
| `subscriber_list_initial_capacity` | 8 per channel | [4, 32] | Fewer reallocs on high-subscriber channels; more unused memory on low-subscriber channels | More reallocs (one-time cost at scene load); less overall memory |
| T1 latency budget | 16.67ms | [8ms, 33ms] | More slack for hot-path subscribers; risk of missing 60fps | Tighter frame budget; harder to hit with complex subscribers |
| T2 latency budget | 100ms | [50ms, 200ms] | More slack before Feedback Floor breach; perceptible lag at high end | Tighter budget; may force subscriber optimization |
| Recursive-fire warning threshold | 10 frames | [5, 30] | Quieter logs; real runaway fires take longer to flag | Noisier debug logs; catches runaway fires faster |
| Subscription leak check interval | 2 scene loads | [1, 5] | More forgiving of leaked subscriptions | Catches leaks faster; noisier on legitimate long-lived subs |

### Interaction Between Tuning Knobs

- `deferred_queue_capacity` and `recursive_fire_warning_threshold` interact: too-small capacity + too-high threshold = silent drops before detection; too-large capacity + too-low threshold = false-positive warnings. Adjust together.
- `T1 latency budget` and `T2 latency budget` must maintain T1 < T2 with a meaningful gap (at minimum 4× ratio) to distinguish frame-critical from feedback-floor events.

## Visual/Audio Requirements

**Not applicable.** Event Bus has no runtime visual or audio presence. It is the delivery layer for feedback events; the V/A requirements for the actual feedback emissions live in the emitting system GDDs (Collection System, Tool System, Hazard System, Critter AI) and the subscribing system GDDs (Audio Bus, VFX/Juice).

If a future author considers adding runtime visual/audio to Event Bus (e.g., a debug "event flow" visualizer rendered in-game), route that to a dedicated Debug Overlay system or the VFX/Juice GDD — not here.

## UI Requirements

**No player-facing UI.** Event Bus is entirely runtime infrastructure; players never see its surface.

**Editor tooling UX (not a GDD-level concern; noted for scope clarity):**
- Event Bus Editor Window — accumulates unhandled-event counts per channel across a session; sortable by count descending; one-click jump from row to the channel SO in the Project view. Mirrors the Data Registry Balance Audit window pattern.
- Channel Inspector extension — custom inspector on each `EventChannelSO<T>` showing current subscriber count + last-fired-timestamp in Play Mode.
- Codegen menu command — `/regenerate-event-ids` manually re-runs the `AudioEventIds` / `VisualEventIds` constant generation (normally auto-triggered on taxonomy SO changes via `AssetPostprocessor`).

Editor tooling is implemented alongside the Event Bus code and is scoped in Technical Setup / Production — not designed here.

## Cross-References

| This Document References | Target GDD | Specific Element | Nature |
|---|---|---|---|
| Event ID schemas come from AudioEventTaxonomy / VisualEventTaxonomy (Data Registry categories 10, 11) | `design/gdd/data-registry.md` | R7 format regex, Category 10 + 11 schemas, AC-R7 | Ownership handoff — Data Registry owns schema; Event Bus owns dispatch |
| R3 generated constants resolve Data Registry OQ-4 in favor of build-time codegen | `design/gdd/data-registry.md` | OQ-4 (event ID string constants source of truth) | Rule dependency — this GDD locks the Data-Registry open question |
| Feedback Floor contract (CD-SYSTEMS resolutions) | `design/gdd/systems-index.md` | CD-SYSTEMS Feedback Floor contract section | Rule dependency — Event Bus delivers the contract's audio + visual events |
| Collection System fires `audio.collect.coin.success` and `gameplay.level.banked` | `design/gdd/collection-system.md` (pending) | Bank/collection event definitions | Data dependency — consumer of Event Bus |
| Tool System fires `audio.tool.place.*` and `audio.tool.consumable.deploy` | `design/gdd/tool-system.md` (pending) | Tool placement + deploy events | Data dependency — consumer of Event Bus |
| Hazard System fires `audio.hazard.bomb.detonate` and `audio.hazard.cat.*` | `design/gdd/hazard-system.md` (pending) | Bomb + cat event definitions | Data dependency — consumer of Event Bus |
| Critter AI fires `audio.critter.dog.*` | `design/gdd/critter-ai.md` (pending) | Dog lifecycle event definitions | Data dependency — consumer of Event Bus |
| Level Runtime fires `gameplay.level.*` (phase_changed, complete, failed, quota_progress) | `design/gdd/level-runtime.md` (pending) | Level state event definitions | Data dependency — consumer of Event Bus |
| Audio Bus subscribes to all `audio.*` channels | `design/gdd/audio-bus.md` (pending) | Audio event subscription interface | Data dependency — subscriber to Event Bus |
| VFX / Juice subscribes to all `vfx.*` channels (VS) | `design/gdd/vfx-juice.md` (pending) | VFX event subscription interface | Data dependency — subscriber to Event Bus |
| Mali-G52 allocation budget (no heap allocation in dispatch) | `.claude/docs/technical-preferences.md` | Performance Budgets section | Rule dependency — R1 enforcement target |
| R7 record-struct LINQ misuse is a Forbidden Pattern | `.claude/docs/technical-preferences.md` | Forbidden Patterns section | Ownership handoff — this GDD adds the entry |

## Acceptance Criteria

28 criteria across 6 categories. Gate: **BLOCKING CI** (10) / **PLAYMODE** (14) / **ADVISORY** (3) / **MANUAL** (0 — all automatable given Unity's PlayMode runner).

### H.1 Rule Compliance (R1-R7)

- **AC-R1 — Zero Hot-Path Allocation** — GIVEN the Rush Phase stress scene (40 Rigidbody2D, 2 fires/frame each), WHEN the bus dispatches for 300 consecutive frames, THEN Memory Profiler reports 0 bytes/frame allocated by `EventBus.Fire<T>` and call chain. **BLOCKING CI**
- **AC-R2 — No Synchronous Cross-Frame Recursion** — GIVEN a subscriber that calls `Fire` during its handler, WHEN the inner `Fire` executes, THEN the event is placed in the deferred ring buffer; depth guard > 0; inner event dispatches next frame. **BLOCKING CI (PLAYMODE)**
- **AC-R3 — No String-Typed Channels at Call Sites** — GIVEN the full source tree, WHEN a CI static-analysis job scans for `EventBus.Subscribe` or `.Fire` calls with string literals, THEN zero matches; any match fails CI. **BLOCKING CI**
- **AC-R4 — No Silent Swallowing in Debug Builds** — GIVEN a `DEVELOPMENT_BUILD` or `UNITY_EDITOR` build, WHEN `Fire` is called on a channel with zero subscribers, THEN `Debug.LogWarning` fires with the channel SO as context within the frame; Editor Window unhandled-event count increments. **PLAYMODE**
- **AC-R5 — Namespaces Are Distinct** — GIVEN a Gameplay channel ID, WHEN `Subscribe` is called with a Feedback context (or vice versa) in debug, THEN `EventBusNamespaceMismatch` throws before registration; no subscription is created. **PLAYMODE**
- **AC-R6 — Subscriber Dispatch Order Is Undefined** — GIVEN three subscribers registered A, B, C, WHEN the channel dispatches 10 times across fresh scene loads, THEN no test may assert a specific order; all three receive the event regardless of registration sequence. **ADVISORY**
- **AC-R7 — CI Flags Record-Struct LINQ Misuse** — GIVEN any `.cs` file with LINQ (`OrderBy`, `Where`, `Select`, `ToList`, etc.) on an `IEventPayload` record-struct inside a dispatch hot path, WHEN the CI static-analysis job runs, THEN the job emits a rule-violation error and exits 1. **BLOCKING CI**

### H.2 Formula / Constant Correctness

- **AC-F1 — Zero-Alloc Dispatch Stress Test** — GIVEN the Rush Phase stress scene, WHEN it runs for 60 simulated seconds, THEN total GC allocations from EventBus code paths = 0 bytes; confirmed via headless profiler snapshot in CI artifacts. **BLOCKING CI**
- **AC-F2 — Deferred Queue Capacity = 64** — GIVEN a subscriber firing 65 re-entrant events in a frame, WHEN the 65th is submitted, THEN debug: `Debug.LogError` with queue contents; release: oldest dropped, `[EventBus] Queue overflow` logged; queue never exceeds 64 at any instant. **PLAYMODE**
- **AC-F3 — Subscriber List Initial Capacity = 8** — GIVEN a freshly-loaded scene with zero subscribers on a channel, WHEN the first subscriber registers, THEN the internal `List<Action<T>>` shows `Capacity == 8` before the 9th subscription. **PLAYMODE**
- **AC-F4 — T1 Latency Budget (≤ 16.67 ms)** — GIVEN all T1-tier pairs (Input→VFX/Juice, Input→HUD, Collection→Audio, Hazard→VFX), WHEN an event is fired with `Fire<T>` wrapped in a stopwatch, THEN measured round-trip ≤ 16.67 ms across 100 samples with no outliers. **PLAYMODE**
- **AC-F5 — T2 Latency Budget (≤ 100 ms)** — GIVEN all T2-tier pairs (Tool→Level Runtime, Critter AI→HUD, Gopher Spawn→Collection, Level Runtime→HUD), WHEN an event fires, THEN time-to-last-subscriber ≤ 100 ms across 100 samples. **PLAYMODE**

- **AC-F6 — End-to-End Feedback Floor (input → rendered frame)** — GIVEN a player tap on the playfield, WHEN `input.tap.main` fires through Event Bus to all subscribers (VFX/Juice, Audio Bus, HUD), THEN the visible feedback (coin splash particle, HUD numeral increment, audio cue onset) is presented on the screen within **100ms at P99** measured from input timestamp to rendered frame timestamp. This closes the CD-GDD-ALIGN Concern 1: dispatch-only latency (AC-F4/F5) proves the bus isn't the bottleneck, but AC-F6 proves the Pillar 5 contract as the player experiences it. Measured via an instrumented dev build that timestamps input events and the first subsequent presented frame containing the corresponding VFX. **PLAYMODE**

### H.3 Edge Case Handling

- **AC-EC1 — Mid-Dispatch Unsubscribe** — GIVEN subscriber B unsubscribes itself from within its handler during dispatch to A, B, C, WHEN the event dispatches, THEN no exception thrown; A + C receive the event; B marked pending-removal; list compacts after dispatch. **BLOCKING CI (PLAYMODE)**
- **AC-EC2 — Subscriber Exception Does Not Break Dispatch** — GIVEN subscriber A throws in its handler and subscriber B is registered after A, WHEN the channel fires, THEN B executes; `[EventBus] Subscriber exception` logged; `Fire` does not throw. **BLOCKING CI (PLAYMODE)**
- **AC-EC3 — Deferred Queue Overflow: Oldest Dropped** — GIVEN a subscriber fires 65 re-entrant events, WHEN overflow occurs, THEN index-0 event (oldest) is dropped; indices 1-64 dispatch FIFO next frame; confirmed by payload-sequence test. **PLAYMODE**
- **AC-EC4 — Namespace Mismatch Assertion** — GIVEN channel `gameplay.level.phase_changed`, WHEN debug code calls Subscribe using Feedback path, THEN `EventBusNamespaceMismatch` throws before handler registration; assertion message includes channel ID and conflicting namespace names. **PLAYMODE**
- **AC-EC5 — Recursive Fire Warning** — GIVEN a subscriber on channel X that unconditionally re-fires X every time, WHEN the deferred queue re-dispatches for 10 consecutive frames, THEN `[EventBus] Probable recursive fire on {channel}` logs ONCE per run (not per frame — no log flood). **PLAYMODE**
- **AC-EC6 — Subscribe During Dispatch: New Subscriber Excluded from In-Flight** — GIVEN subscriber A subscribes a new subscriber B during its handler, WHEN the event dispatches, THEN B does NOT receive the current event; B receives all subsequent dispatches; confirmed via capture list. **PLAYMODE**

### H.4 Performance

- **AC-P1 — Zero Bytes/Frame Steady State** — GIVEN a scene with all 13 MVP emitter/subscriber pairs active and firing normally, WHEN running for 10 minutes of continuous play, THEN Unity Memory Profiler shows 0 GC allocations/frame from EventBus code in steady state (post-scene-load). **BLOCKING CI**
- **AC-P2 — Memory Footprint per Channel** — GIVEN 30 channels loaded at scene start, WHEN profiled at steady state, THEN total memory attributed to channel SOs + subscriber lists ≤ 8 KB; individual channel backing array ≤ 256 bytes for ≤ 8 subscribers. **ADVISORY**
- **AC-P3 — T1/T2 Latency Under Peak Load** — GIVEN Rush Phase peak (40 Rigidbody2D, max emitter activity), WHEN T1 and T2 events fire concurrently, THEN T1 ≤ 16.67 ms, T2 ≤ 100 ms; T1 events are not delayed by T2 subscriber processing. **PLAYMODE**

### H.5 Consumer Integration

- **AC-INT1 — Input System** — subscriber on `input.tap.main` receives `TapPayload(Vector2(100,200), null)` within one frame. **PLAYMODE**
- **AC-INT2 — Collection System** — subscribers on both `audio.collect.coin.success` + corresponding Gameplay channel receive matching `CollectionSuccessPayload` for a single coin collection; both execute in same T1 frame. **PLAYMODE**
- **AC-INT3 — Tool System** — subscriber on `audio.tool.place.ramp` receives `ToolEventPayload(Ramp, Vector2(50,50))` within T2 budget. **PLAYMODE**
- **AC-INT4 — Hazard System** — subscribers on `audio.hazard.bomb.detonate` + Gameplay pair receive matching `BombDetonatedPayload`; dispatch ≤ T1. **PLAYMODE**
- **AC-INT5 — Critter AI** — subscriber on `audio.critter.dog.deploy` receives `DogEventPayload(AgentID:1, Vector2(0,0))` within T2; no Gameplay-namespace subscriber receives the Feedback event. **PLAYMODE**
- **AC-INT6 — Gopher Spawn** — subscriber on `gameplay.gopher.launched` receives `GopherLaunchedPayload(7, "standard")` with matching fields within T2. **PLAYMODE**
- **AC-INT7 — Level Runtime** — subscribers on `phase_changed`, `complete`, `quota_progress` receive payloads in correct order during phase transitions + win condition; `LevelEndPayload.Success == true` on win. **PLAYMODE**

### H.6 No Hardcoded Channel Strings

- **AC-STR1 — CI Grep: No Bare String Literals at Subscribe Call Sites** — `grep -rn "\.Subscribe\s*<.*>\s*(\s*\""` returns 0 matches; any match fails CI. **BLOCKING CI**
- **AC-STR2 — CI Grep: No Bare String Literals at Fire Call Sites** — `grep -rn "\.Fire\s*<.*>\s*(\s*\""` returns 0 matches; any match fails CI. **BLOCKING CI**
- **AC-STR3 — Generated Constants Compile After Taxonomy Change** — GIVEN a change to `AudioEventTaxonomy` or `VisualEventTaxonomy`, WHEN the codegen `AssetPostprocessor` runs on reimport, THEN `AudioEventIds.cs` + `VisualEventIds.cs` compile; previously-valid references remain valid OR produce compile-time error at the call site (not a silent runtime mismatch). **BLOCKING CI**

### Gate Summary

| Gate | Count |
|---|---|
| **BLOCKING CI** | 10 (AC-R1, AC-R3, AC-R7, AC-F1, AC-EC1, AC-EC2, AC-P1, AC-STR1, AC-STR2, AC-STR3) |
| **PLAYMODE** | 14 (AC-R2, AC-R4, AC-R5, AC-F2, AC-F3, AC-F4, AC-F5, AC-EC3-EC6, AC-P3, AC-INT1-INT7) |
| **ADVISORY** | 3 (AC-R6, AC-F3 note, AC-P2) |
| **MANUAL** | 0 — all criteria are automatable given Unity's PlayMode runner |

**Shift-left note**: the 10 BLOCKING CI criteria need stub EditMode/CI test scaffolding at story start, before bus implementation begins. PLAYMODE criteria require a test scene fixture with representative subscriber MonoBehaviours — scaffold alongside bus implementation, not deferred to QA hand-off.

## Open Questions

| # | Question | Owner | Target Resolution |
|---|---|---|---|
| OQ-1 | For non-MonoBehaviour subscribers (ScriptableObjects, pure C# services): is the manual `SubscriptionHandle.Dispose()` pattern sufficient, or should the bus track weak references with GC-time warnings in release builds (not just debug)? Current design: debug-only leak detection. Risk: service-layer subscribers leak quietly in production. | Lead Programmer | During first non-MonoBehaviour subscriber implementation |
| OQ-2 | `SubscriptionManager` component auto-attached vs. explicit attachment: when `SubscribeAutoManaged` is called, should the bus auto-attach a `SubscriptionManager` if the GameObject lacks one, or throw and require the author to add it explicitly? Auto-attach is more ergonomic; explicit requires discipline. | Lead Programmer + systems-designer | Architecture phase (`/architecture-decision`) |
| ~~OQ-3~~ **RESOLVED 2026-04-20 via CD-GDD-ALIGN Concern 3**: Analytics uses option (c) with codegen. An `AssetPostprocessor` generates `AnalyticsSubscribers.cs` from the `gameplay.*` taxonomy entries at build time. Matches R3 pattern; preserves R1 + R7; preserves Pillar 3 analytics tuning loop. See R9. |
| OQ-4 | Should the Event Bus expose a deterministic replay mode for testing (record all events + payloads + timestamps during a session, replay them in unit tests)? Would be invaluable for regression tests but adds complexity. Physics is non-deterministic per TD ADR, so full replay is impossible — partial event replay might still be useful. | QA Lead + Lead Programmer | Post-MVP (when first flaky test surfaces) |
| OQ-5 | Event ID format validation runs at editor time (Data Registry R7), but the bus performs a second runtime check in debug builds. Is this redundancy worth the cost, or should we remove the runtime check and trust Data Registry's CI gate? | Lead Programmer | During first profiling pass of debug-build overhead |
| ~~OQ-6~~ **RESOLVED 2026-04-20 via CD-GDD-ALIGN Concern 2**: Bus maintains a pre-allocated bootstrap buffer (capacity 16 events) during Uninitialized state. Events fired during `Awake` are queued; drain on first `Update` after Ready. Prevents Pillar 5 silent-drops during scene load. See R8. |
