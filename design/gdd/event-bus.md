# Event Bus

> **Status**: Revised pass 1 — 14 blocking items resolved via `/design-review` (2026-04-21); pending re-review in a fresh session
> **Author**: Robert Michael Watson + systems-designer / unity-specialist / creative-director / qa-lead / performance-analyst / game-designer
> **Last Updated**: 2026-04-21
> **Implements Pillar**: Foundation for Pillars 1 (Cute Chaos), 2 (Readable at a Glance), 5 (No Input Goes Silent). Indirect support for 3 and 4 via decoupling.
>
> **Creative Director Review (CD-GDD-ALIGN)**: CONCERNS → RESOLVED 2026-04-20 (3 fixes applied: AC-F6 end-to-end Feedback Floor added; R8 bootstrap buffer locked; R9 Analytics codegen locked)
>
> **/design-review pass 1 (2026-04-21)**: NEEDS REVISION → RESOLVED. 5 specialists (game-designer, systems-designer, qa-lead, unity-specialist, performance-analyst) + creative-director senior synthesis. 14 blocking + 16 recommended items addressed via: `readonly struct` fallback (C#9-safe), per-channel bootstrap buffers (no boxing), committed codegen files + staleness CI, `#if UNITY_EDITOR`-only log paths, deferred queue capacity raised 64→128, AC-F6 split into F6a/b/c, Feedback Group concept introduced, ProfilerRecorder API, grep patterns fixed, hardware split (CI PlayMode + Mali-G52 manual VS), R8/Edge-Case contradiction reconciled, R7 renumbered in sequence.

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

**R2. No synchronous cross-frame recursion.** Events fired from inside a subscriber handler are queued and dispatched at the start of the next frame, not inline. A depth guard (int, frame-local) enforces this: depth > 0 triggers the deferred queue. The queue is a pre-allocated ring buffer (capacity: 128 events — tuning knob; raised from 64 in pass-1 review per convergent systems-designer + performance-analyst findings that worst-case peak load is 80 events/frame, making 64 pre-committed to overflow).

**R2.a. Depth-guard reset contract.** The frame-local depth int is reset **at the start of each `EventBus.Update()` tick, before any drain or dispatch occurs**. It is never reset mid-frame. This guarantees that a thermal-throttle stall (e.g., 50ms Update) cannot leave depth non-zero at the start of the next logical frame; the reset happens unconditionally at Update entry. Verified by AC-R2b (added in pass-1 review).

**R2.b. PlayerLoop injection point.** The bus's drain and dispatch hook are injected into Unity's PlayerLoop at `PreUpdate.EventBusPlayerLoopSystem` via the `UnityEngine.LowLevel.PlayerLoop` API (not via a standard MonoBehaviour `Update`). This placement runs BEFORE all MonoBehaviour `Update` calls but AFTER `FixedUpdate`/physics callbacks of the same frame. Physics-originating events (e.g., `OnCollisionEnter2D` firing a hazard event) are therefore queued during FixedUpdate and drain within the same rendered frame, preserving T1 tier semantics.

**R3. No string-typed channels at call sites.** All channel IDs are typed constants generated from the Data Registry taxonomies at editor/build time (an `AssetPostprocessor`-triggered codegen script writes `AudioEventIds.ToolPlaceRamp` / `VisualEventIds.CollectCoinSplash` const strings). Publishers reference these constants; passing a bare string literal to `Subscribe` or `Fire` is a CI static-analysis failure. This resolves Data Registry OQ-4 in favor of generated constants.

**R4. No silent swallowing in debug builds.** If an event is fired on a channel with zero active subscribers, a `Debug.LogWarning` fires with the channel SO as context (clickable in Unity Console). In release + development builds this check is stripped via `#if UNITY_EDITOR` (tightened from `UNITY_EDITOR || DEVELOPMENT_BUILD` in pass-1 review per performance-analyst finding: `DEVELOPMENT_BUILD` inclusion would break AC-R1 zero-alloc measurement, because the profiler stress scene runs as a dev build and string-interpolated warning logs allocate). A custom Editor Window also accumulates unhandled-event counts per channel for post-session audit (mirrors Data Registry's Balance Audit window pattern).

**R4.a. Release-build telemetry counter.** Separate from the Editor-only `Debug.LogWarning` path, the bus maintains a per-channel **zero-subscriber fire counter** incremented on any `Fire` call that finds no subscribers. The counter is an `int` in a pre-allocated `Dictionary<ChannelID, int>` (no allocation at increment). At v1.0, Analytics reads this counter once per session-end and forwards `eventbus.zero_subscriber_rate` as a telemetry event. Provides production visibility into silent Pillar 5 failures (a taxonomy event fired with no subscribers = the player tap got no feedback). Resolves game-designer pass-1 finding #4.

**R5. Taxonomy events and internal events are distinct channel namespaces.** Taxonomy events (whose IDs appear in `AudioEventTaxonomy` or `VisualEventTaxonomy` and match Data Registry R7 regex `^(audio|vfx)\.[a-z_]+\.[a-z_]+\.[a-z_]+$`) live in `ChannelNamespace.Feedback`. Internal gameplay events (e.g., `gameplay.level.phase_changed`) live in `ChannelNamespace.Gameplay` (format `^gameplay\.[a-z_]+\.[a-z_]+$`). A channel from one namespace cannot be used on the other — a runtime assertion enforces this in debug builds. Prevents gameplay subscribers from accidentally registering on audio-routed channels.

**R6. Subscriber dispatch order is undefined.** The bus makes no guarantee about which subscriber is invoked first. Any system requiring ordering (e.g., HUD must update before a post-frame screenshot) uses a two-phase pattern: subscribe to the event, then post a local update to a sorted execution queue it owns. Ordering dependencies between subscribers are a smell — the subscribing systems own resolution, not bus insertion order.

**R6.a. Feedback Group concept (new in pass-1, resolves game-designer + CD adjudication).** While R6 leaves per-subscriber order undefined, a **Feedback Group** is a named set of subscribers for the SAME event (e.g., `CollectionSuccess` Feedback Group = {Audio Bus, VFX/Juice, HUD, Haptics}) whose AGGREGATE dispatch latency is measured as a single budget against Pillar 5's 100ms floor. The bus exposes `RegisterFeedbackGroup(ChannelID channel, string groupName)` for diagnostic purposes; AC-F4 measures `group_latency = max(subscriber.completion_timestamp) - fire_timestamp` for each registered group. This gives game design a measurable Pillar 5 guarantee ("all three voices complete within 100ms") without requiring dispatch-order preservation that would break R6 and force a priority queue redesign (structurally infeasible under synchronous dispatch per unity-specialist + performance-analyst findings). Audio/visual desync between subscribers remains possible within the group's aggregate budget; playtest sign-off is the gate for whether the perceived jitter is acceptable.

**R7 (from Unity-specialist CI safety net — renumbered into sequence in pass-1).** Because typed payloads use `readonly struct` (see "Typed Payload Pattern" below), CI static analysis must flag any LINQ expression or equality chain on an `IEventPayload`-implementing struct inside a dispatch hot path method. `readonly struct` is allocation-free when passed through `Fire<T>`, but misuse (e.g., `payloads.OrderBy(p => p.Key)` or `payloads.Where(...)`) allocates via enumerator boxing and intermediate collection creation. This rule is listed as a Forbidden Pattern in `.claude/docs/technical-preferences.md`. Pass-1 note: the Roslyn-analyzer implementation path for this rule is scoped to a simpler two-tier check — (1) grep-level flag on any LINQ method on an `IEventPayload` implementor regardless of context (noisy but zero false-negatives), (2) manual code review as secondary safety net. A full context-aware Roslyn analyzer is scoped as an optional follow-up engineering task, not a prerequisite for R7 enforcement.

**R7.a. String-typed payload fields flagged (resolves performance-analyst finding #6).** CI static analysis ALSO flags any `string` field on an `IEventPayload`-implementing struct. Strings are reference types stored by-reference in the ring buffer, increasing GC scan cost and inviting publishers to inline-construct strings (which allocates outside the bus's measured scope). Preferred alternatives: enum-backed ID (`int` or `LootTableId` enum), `ChannelID` struct reference, or a pre-interned-string reference via `StringPool`. Existing `GopherLaunchedPayload.LootTableId: string` must be migrated to `LootTableId: int` (enum-backed lookup into Data Registry's loot table catalog).

**R8 (locked from CD-GDD-ALIGN Concern 2; revised pass-1 to use per-channel buffers).** Each `EventChannelSO<T>` owns its own **typed bootstrap ring buffer of capacity 4**. Any `Fire<T>` call during the `Uninitialized` state (after the bus awakens but before all subscribers have completed registration — typically the first `Awake → Start` window) is written to that specific channel's typed buffer as a value-type `T` (no boxing, no `IEventPayload` interface storage). On the bus's **first `EventBusPlayerLoopSystem.Update` tick after Ready**, each channel drains its own buffer in FIFO order via normal dispatch, subject to the per-tick drain cap in R8.a. Per-channel bootstrap overflow (5th event on a channel) drops the newest event and logs `[EventBus] Bootstrap overflow on {channel}` (Editor only per R4). Scene-wide bootstrap capacity is effectively `channel_count × 4` (e.g., ~30 channels × 4 = 120 slots) with zero boxing because each buffer is typed.

Pass-1 review rationale (resolves unity-specialist finding #2): a single heterogeneous 16-slot buffer would have required storing payloads as `IEventPayload` (interface box → heap allocation per queued boot event) or a discriminated-union struct (high implementation cost). Per-channel typed buffers eliminate boxing at the cost of slightly higher memory overhead (negligible — each buffer is 4 × sizeof(payload) ≈ 64 bytes). This replaces the original "throw on Uninitialized" behavior for the narrow scene-load window only; mid-session `Fire` on a Disposed bus still errors.

**R8.a. Bootstrap drain budget (resolves systems-designer + performance-analyst finding #3).** Frame 0 after Ready is **NOT subject to the T1 16.67ms budget** — it is explicitly carved out. To prevent first-frame spikes, the drain is time-sliced: **at most 4 events per channel per `Update` tick**, with per-channel FIFO order preserved across ticks. Channels whose buffers are not fully drained continue draining on subsequent ticks. AC-BOOT (new in pass-1) asserts first-frame drain completes in ≤ 8ms aggregate on CI hardware even with all 30 channels loaded. In practice, well-behaved boot profiles drain in a single tick; the time-slice is a safety net for future growth.

**R8.b. Telemetry for overflow events.** The bus maintains a release-build counter for `deferred_queue_overflow_count` and `bootstrap_overflow_count` (incremented on each drop). These are forwarded to Analytics at v1.0 end-of-session (reusing the R4.a telemetry bucket). Resolves creative-director adjudication of drop-policy disagreement: FIFO drop retained, but telemetry provides the signal to revisit the policy with real data.

**R9 (locked from CD-GDD-ALIGN Concern 3; revised pass-1 for codegen CI integration).** A companion `AssetPostprocessor` generates `AnalyticsSubscribers.cs` from the `gameplay.*` entries of every taxonomy SO. Generated file is **committed to version control** (not regenerated at build time) with a staleness CI check: a `.github/workflows/eventbus-codegen-staleness.yml` job runs the codegen in a headless Unity step and `git diff --exit-code`s the three generated files (`AudioEventIds.cs`, `VisualEventIds.cs`, `AnalyticsSubscribers.cs`); any drift fails CI. Pass-1 rationale (resolves unity-specialist finding #5): `AssetPostprocessor` does NOT run reliably in `-batchmode` headless CI without explicit `AssetDatabase.ImportAsset`; committing the generated files sidesteps this fragility AND makes generated Analytics subscribers visible in PR reviews (critical: a reviewer must be able to see what new Analytics subscriber was added when a taxonomy SO changes). Each generated entry is a per-channel explicit subscription that forwards the typed payload to the Analytics service **via a pre-allocated fixed-schema batch buffer** (resolves performance-analyst finding #7); the Analytics service call is NOT in the bus dispatch hot path — it queues into a per-frame flush that runs once per rendered frame in `PostLateUpdate`, outside the T1/T2 dispatch envelope. Matches the R3 codegen pattern; preserves R1 zero-allocation (each subscription is typed, batch buffer is pre-allocated, no reflection); preserves R7 (no LINQ on payloads). Analytics consumers never manually subscribe to individual gameplay channels — the generated file is the single source of truth. Resolves OQ-3; preserves Pillar 3's analytics tuning loop.

### Event Category Taxonomy

| Category | Channel Namespace | ID Schema (Data Registry) | Example |
|---|---|---|---|
| Taxonomy (Feedback) | `ChannelNamespace.Feedback` | `^(audio\|vfx)\.[a-z_]+\.[a-z_]+\.[a-z_]+$` | `audio.collect.coin.success` |
| Internal (Gameplay) | `ChannelNamespace.Gameplay` | `^gameplay\.[a-z_]+\.[a-z_]+$` | `gameplay.level.phase_changed` |

**Relationship**: a single gameplay occurrence may cause a publisher to fire on *both* a Gameplay channel (for state-machine subscribers: Level Runtime, Economy) AND a Feedback channel (for Audio Bus, VFX/Juice, HUD). These are two separate `Fire` calls with separate payloads. **No "fan-out by namespace" magic** — the publisher decides explicitly which channels to fire on. Keeps call sites auditable and keeps R4 effective per channel.

### Typed Payload Pattern

**Pattern**: `readonly struct` (C# 7.2+, confirmed available on Unity 6.3 LTS's C# 9 scripting backend). Zero-allocation (value type, stack-allocated through `Fire<T>`), immutable after construction (no subscriber can mutate a payload another subscriber will see). Pass-1 review revised from `readonly record struct` (C# 10) to `readonly struct` (C# 9-safe) per unity-specialist finding #1: the engine-reference docs confirm Unity 6 ships C# 9, and `readonly record struct` was introduced in C# 10. Conservative fallback locks the payload pattern to a language version that is confirmed to exist on the engine.

**Consequences of `readonly struct` vs `readonly record struct`:**
- **No auto-generated structural `Equals`** — tests that compare payload values must implement `IEquatable<T>` explicitly on each payload struct (documented pattern below).
- **No auto-generated `ToString`** — debug overlay and test failure messages must read field-by-field via a manual `ToString()` override or reflection (acceptable: debug path, not hot path).
- **No `with` expressions** — payloads are constructed once at fire site; mutation patterns are not part of the design.

**Safety net (R7)**: misuse in LINQ/equality hot paths still allocates. CI static analysis enforces no hot-path `IEventPayload`-struct LINQ/equality chains.

**Field typing constraints (R7.a)**: payload fields MUST be value types (struct, primitive, enum). `string` fields are forbidden by CI static analysis; use enum-backed IDs or `ChannelID` references. The Typed Payload examples below have been revised to enum-backed IDs where applicable.

**Dispatch field typing**: the internal subscriber-list field per channel MUST be typed as the concrete `List<Action<T>>` (never `IEnumerable<Action<T>>` or `IList<Action<T>>`). Interface typing boxes the List's struct enumerator and allocates on `foreach`. This is a one-line implementation constraint with R1 significance; named explicitly here per unity-specialist finding #3.

```csharp
public interface IEventPayload { }

// Feedback channel — must match AudioEventTaxonomy key "audio.collect.coin.success"
public readonly struct CollectionSuccessPayload : IEventPayload, IEquatable<CollectionSuccessPayload>
{
    public readonly Vector2 WorldPosition;
    public readonly int CoinValue;
    public readonly bool IsCritical;

    public CollectionSuccessPayload(Vector2 pos, int coinValue, bool isCritical)
    {
        WorldPosition = pos; CoinValue = coinValue; IsCritical = isCritical;
    }

    public bool Equals(CollectionSuccessPayload other) =>
        WorldPosition == other.WorldPosition && CoinValue == other.CoinValue && IsCritical == other.IsCritical;

    public override int GetHashCode() => HashCode.Combine(WorldPosition, CoinValue, IsCritical);
}

// Fired on BOTH Feedback and Gameplay channels (two separate Fire calls)
public readonly struct BombDetonatedPayload : IEventPayload, IEquatable<BombDetonatedPayload>
{
    public readonly Vector2 WorldPosition;
    public readonly int Radius;
    public readonly int CoinsDestroyed;
    // ... IEquatable<> impl as above
}

// Gameplay channel only — no audio/visual entry in taxonomy
public readonly struct PhaseChangedPayload : IEventPayload, IEquatable<PhaseChangedPayload>
{
    public readonly LevelPhase PreviousPhase;
    public readonly LevelPhase NextPhase;
    public readonly float TimeRemainingSeconds;
    // ... IEquatable<> impl as above
}

// Revised per R7.a: LootTableId is now an int-backed enum (was string in prior revision)
public readonly struct GopherLaunchedPayload : IEventPayload, IEquatable<GopherLaunchedPayload>
{
    public readonly int GopherID;
    public readonly LootTableId LootTableId; // enum-backed, looked up in Data Registry
    // ... IEquatable<> impl as above
}
```

### ChannelID Type Specification (new in pass-1, resolves unity-specialist finding #4)

`ChannelID` is a `readonly struct` wrapping a short `int` hash of the taxonomy-defined string plus a `ChannelNamespace` tag:

```csharp
public readonly struct ChannelID : IEquatable<ChannelID>
{
    public readonly int Hash;
    public readonly ChannelNamespace Namespace;
    // debug-only string retained in a separate editor-only lookup table, not in the struct

    public bool Equals(ChannelID other) => Hash == other.Hash && Namespace == other.Namespace;
    public override int GetHashCode() => HashCode.Combine(Hash, Namespace);
}
```

The codegen-generated `AudioEventIds` and `VisualEventIds` files expose channels as `public static readonly ChannelID ToolPlaceRamp = new(hash: 0x4A2B..., namespace: ChannelNamespace.Feedback);` (NOT `const string`). This preserves the compile-time constant semantics for R3 (no string-literal access at call sites) while giving dictionary lookups a fast, box-free `IEquatable<ChannelID>` path. The `EventBus` internal dispatch table uses `Dictionary<ChannelID, ChannelSOBase>` with `EqualityComparer<ChannelID>.Default` — which devirtualizes correctly through the `IEquatable<ChannelID>` implementation without boxing.

### Subscription Lifecycle

**Subscribe API:**
```csharp
SubscriptionHandle Subscribe<T>(ChannelID channel, Action<T> handler)
    where T : struct, IEventPayload
```
Returns a `SubscriptionHandle` (readonly struct wrapping an int token). Internally, each channel SO holds a pre-allocated `List<Action<T>>` (typed as the concrete `List<T>`, never `IEnumerable` or `IList` — see R1 discussion in Typed Payload Pattern).

**`SubscriptionHandle` is a `readonly struct` and does NOT implement `IDisposable`** (pass-1 revision per unity-specialist finding #11). Rationale: `IDisposable` interface dispatch on a struct boxes to the heap on every `using` block or `IDisposable`-typed variable. Instead, `SubscriptionHandle` exposes `Unsubscribe()` as a direct struct method; callers call `handle.Unsubscribe()` explicitly in their teardown code. This avoids the `using` pattern (which forces `IDisposable` boxing) while preserving the same semantic (single-shot teardown).

**Auto-unsubscribe on MonoBehaviour destruction:**

Two paths, selected by subscriber type:

- **MonoBehaviour subscribers**: call `SubscribeAutoManaged<T>(channel, handler, this)`. The bus registers a thin `SubscriptionManager` component on the subscriber's GameObject (added automatically if absent). `SubscriptionManager.OnDestroy` iterates its token list and calls `Unsubscribe` for each. Zero allocations at unsubscribe time (pre-allocated token list).

  **Allocation note for pooled objects (pass-1, resolves unity-specialist finding #6)**: `AddComponent<SubscriptionManager>()` allocates the component on the managed heap at subscribe time. Lambda-capture handlers also allocate a closure object at subscribe time. These are **Subscribe-time allocations**, not dispatch-time — R1 is not violated. HOWEVER, for pooled objects that subscribe/unsubscribe on pool activate/release (e.g., gopher pool, coin pool), the allocation recurs on every reuse. **Convention**: pooled objects subscribe in `Awake` (once per pool instance lifetime) and use flag-based enable/disable checks in their handlers, NOT `SubscribeAutoManaged` on every `OnEnable`. Documented in Subscribe API docstring; enforced by project-level code review guideline (not a CI rule).

- **Non-MonoBehaviour subscribers** (ScriptableObjects, pure C# services): manual `handle.Unsubscribe()` in a teardown method (NOT `Dispose()` — see note above on IDisposable boxing). In debug builds, the bus tracks live handles; any handle whose originating SubscribeX call is never matched by Unsubscribe logs an error at scene unload.

**Leak prevention invariant**: in debug builds, the bus tracks live subscription count per channel. If count grows monotonically across two consecutive scene loads without any unsubscribes, an error is logged. Catches "forgot to Unsubscribe a service-layer subscriber."

### States and Transitions

The bus is not a gameplay state machine, but has three operational states:

| State | Entry Condition | Exit Condition | Behavior |
|-------|----------------|----------------|----------|
| Uninitialized | Domain load start | Bus `Awake` completes | `Subscribe` throws immediately. `Fire<T>` is captured into the target channel's per-channel bootstrap ring buffer (capacity 4 events per channel per R8) and will drain on first `EventBusPlayerLoopSystem.Update` tick after Ready (time-sliced per R8.a). |
| Ready / Idle | Bus `Awake` completes | Dispatching starts OR domain unload | On entry: drain per-channel bootstrap buffers via normal dispatch, time-sliced at ≤4 events per channel per tick (R8.a). Normal operation: `Subscribe` modifies channel tables; `Fire` dispatches synchronously |
| Dispatching | A `Fire` call is in progress | All subscriber handlers for that channel return | Any `Fire` called from a subscriber queues to the deferred ring buffer (R2). `Subscribe` is legal during dispatch — new subscriptions take effect next dispatch cycle, preventing "modified while iterating" crashes |
| Disposed | Scene unload or application quit | N/A — terminal | All subscriptions cleared; any `Fire` or `Subscribe` is a no-op with Editor-only warning. Unfinished deferred-queue events are discarded; release-build counter `scene_unload_discard_count` tracks the discards (R8.b). |

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
| Gopher Spawn | `gameplay.gopher.launched` | Gameplay | `GopherLaunchedPayload(int GopherID, LootTableId LootTableId)` (LootTableId is int-backed enum per R7.a) | Collection System, VFX/Juice | T2 |
| Level Runtime | `gameplay.level.phase_changed` | Gameplay | `PhaseChangedPayload` | HUD, Tool System, Currency & Economy | T2 |
| Level Runtime | `gameplay.level.complete` / `failed` | Gameplay + Feedback | `LevelEndPayload(bool Success, int FinalScore)` | Modal/Dialog + Level End, Audio Bus, Analytics (v1.0) | T2 |
| Level Runtime | `gameplay.level.quota_progress` | Gameplay | `QuotaProgressPayload(int Current, int Target)` | HUD, Currency & Economy | T2 |

**No event may span frames in the synchronous dispatch path** — R2's deferred queue handles re-entrant fires.

## Formulas

Event Bus has no gameplay balance formulas. It has operational constants derived from Unity 6.3 + Mali-G52 engineering budgets.

### D.1 Dispatch Latency Budget (by tier)

```
T1_latency_max = 16.67ms  (fixed wall-clock; NOT a per-frame fraction)
T2_latency_max = 100ms    (Pillar 5 Feedback Floor)
```

| Variable | Value | Source |
|---|---|---|
| T1_latency_max | 16.67ms | 60fps mid-tier target (`.claude/docs/technical-preferences.md`). **Fixed wall-clock**: at the 30fps floor on Mali-G52 (33.33ms frame budget), T1 remains 16.67ms. "Same frame" is a consequence of the wall-clock budget, not its definition. AC-F4 must include a 30fps test pass. |
| T2_latency_max | 100ms | Pillar 5 "No Input Goes Silent" feedback floor |

Clarification added pass-1 per systems-designer finding #9.

### D.2 Deferred Queue Capacity

```
deferred_queue_capacity = 128 events (pre-allocated ring buffer; tuning knob)
```

Pass-1 revision: raised from 64 → 128 per convergent systems-designer + performance-analyst findings. Derivation: peak Rush Phase density = 40 active Rigidbody2D × avg 2 emits/body/frame = 80 events worst case; single gameplay occurrences may fire on BOTH Feedback and Gameplay namespaces (2× multiplier on certain events); safe ceiling is 128 (1.6× worst case with margin for v1.0 subscriber growth). Overflow triggers `Debug.LogError` with the overflowing event ID in Editor (per R4); in release, the oldest queued event is dropped with a `[EventBus] Queue overflow` release-build counter increment (per R8.b). Policy retained as FIFO-drop-oldest pending telemetry data at v1.0 (per CD adjudication of drop-policy disagreement).

### D.3 Subscriber List Initial Capacity (per channel)

```
subscriber_list_initial_capacity = 16 per channel
```

Pass-1 revision: raised from 8 → 16 per systems-designer finding #6. Realistic v1.0 per-channel subscriber counts: `gameplay.level.complete` = {Modal/Dialog, Audio Bus, Analytics, Save/Load write-trigger, debug-overlay} = 5; high-density Feedback channels = {Audio Bus, VFX/Juice, HUD, Haptics, Analytics (via gameplay pair), debug-overlay} = 6; future Tutorial overlay + Accessibility Service subscribers push maximum to 8 on a few channels. Initial capacity 16 eliminates the one-time realloc-and-copy at scene load (which is an allocation spike that would break AC-P1 steady-state measurement).

Implementation contract (resolves systems-designer finding #4 + qa-lead finding #5): each channel SO MUST construct its subscriber list as `new List<Action<T>>(16)` in the SO's `OnEnable` (not on first Subscribe call). AC-F3 asserts `Capacity == 16` **before** any subscriber registers (constructor-time pre-allocation, not post-add growth).

### D.4 Feedback Group Aggregate Latency (new in pass-1)

```
group_latency = max(subscriber.completion_timestamp) - fire_timestamp
group_latency_max_T1 = 16.67ms   (all T1-group subscribers complete within same frame)
group_latency_max_T2 = 100ms     (all T2-group subscribers complete within Feedback Floor)
```

A **Feedback Group** is a named set of subscribers for one event whose AGGREGATE completion latency is the measurable Pillar 5 guarantee. Registered via `RegisterFeedbackGroup(ChannelID, string groupName)` for diagnostic purposes. The budget covers all subscribers collectively — per-subscriber order remains undefined (R6). Measured by AC-F4 across all documented groups (Input tap group, Collection success group, Bomb detonate group, Tool place group).

### D.5 Multi-hop Chain Latency (new in pass-1, resolves game-designer finding #3)

```
chain_latency_max = (hops_requiring_defer × frame_budget_ms) + Σ(subscriber_execution_ms)
hops_requiring_defer = count of re-entrant Fires that hit R2's deferred queue
frame_budget_ms = 16.67 (60fps) or 33.33 (30fps floor)
```

For every multi-hop interaction in Section §Interactions (e.g., Collection → Currency & Economy → HUD where each hop fires a new event), compute `chain_latency_max` at the 30fps floor. If any chain breaches 100ms T2 budget at 30fps, the chain must either:
- Be restructured so dependent subscribers subscribe to the SAME event (single-hop fan-out, no re-entrant fire), OR
- Be flagged as a Pillar 5 known risk with playtest sign-off.

AC-F5 is expanded in pass-1 to include chain latency validation for the documented multi-hop interactions. See §Acceptance Criteria.

## Edge Cases

### Subscription / Lifecycle

- **If a MonoBehaviour subscribed via `SubscribeAutoManaged` has its GameObject destroyed mid-dispatch**: `SubscriptionManager.OnDestroy` fires and calls `Unsubscribe` — but the bus is in `Dispatching` state. Resolution: `Unsubscribe` during dispatch marks the entry as "pending removal" (nulls the delegate in the list); dispatch loop skips null entries; pending removals compact after dispatch completes. No exception, no stale-reference crash.
- **If a MonoBehaviour subscribes in `OnEnable` but is disabled mid-dispatch (not destroyed)**: subscription persists (disable ≠ destroy). Handler still fires. If disable-suspended behavior is desired, subscriber must unsubscribe in `OnDisable` manually — not via auto-managed path. Documented in the Subscribe API docstring.
- **If a non-MonoBehaviour subscriber's `SubscriptionHandle` is GC'd without `Dispose()`**: in debug builds, the bus logs `[EventBus] Leaked subscription for channel {id}` with the stack trace captured at Subscribe time (stored in a debug-only dictionary). In release builds, the subscription persists until scene unload — manageable memory waste, not a crash.
- **If `Subscribe` is called during `Dispatching` state for the same channel being dispatched**: new subscriber is registered but does NOT receive the current in-flight event. Takes effect next dispatch cycle. Prevents the race where a subscriber created by another subscriber's handler receives a partially-processed payload.

### Dispatch Safety

- **If a subscriber handler throws an unhandled exception**: dispatch loop catches, logs `[EventBus] Subscriber exception on channel {id}: {ex}` with subscriber type (**Editor-only per R4** — stripped in Development and Release builds to preserve R1 zero-alloc), and continues to the next subscriber. The bus never throws out of `Fire`. Prevents one buggy subscriber from breaking all subscribers on the same channel. In non-Editor builds, subscriber exceptions are silently swallowed at the bus boundary; subscribers are responsible for their own try/catch if they need non-Editor error handling.
- **If an event is fired before the bus has initialized (Uninitialized state)**: `Fire<T>` writes the payload to that channel's **per-channel bootstrap ring buffer** (capacity 4 per channel per R8). Pass-1 correction: prior revision said "throws `EventBusNotReadyException` immediately (fail-fast per R5)" — this was a stale reference that predated the CD-GDD-ALIGN R8 resolution, AND the "per R5" cross-reference was incorrect (R5 governs namespace distinctness, not init state). Fire during Uninitialized is now safe and buffered; throw semantics apply only to `Fire` on a **Disposed** bus (post-scene-unload).
- **If the deferred queue reaches capacity during re-entrant dispatch (128 events queued per D.2)**: in Editor, `Debug.LogError` with queue contents; in release + development, drop oldest (FIFO), increment release-build counter (R8.b) which Analytics reads at v1.0 session-end.
- **If two Feedback channels with the same ID string exist (duplicate SO assets)**: Data Registry's duplicate-ID check catches at editor time (AC-EC-A1 in Data Registry GDD). Bus assumes unique channel IDs and does not re-check at runtime.
- **If a scene unload fires mid-dispatch** (bus in `Dispatching` state, then `OnDisable`/`OnDestroy` triggers scene tear-down): the bus transitions `Dispatching → Disposed` after the current `Fire` call's subscriber chain completes (NOT preemptively). Any deferred-queued events that have not yet drained are discarded silently; the release-build counter increments `scene_unload_discard_count` per discarded event. Subscribers mid-chain are NOT interrupted — a subscriber holding a handle to a destroyed object is a subscriber bug, not a bus bug. Pass-1 added per nice-to-have finding #33.

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
| `deferred_queue_capacity` | 128 events (raised pass-1) | [64, 256] (floor raised pass-1 from 32) | More re-entrant fires survive; more preallocated memory (~512B per slot) | Smaller memory footprint; overflow risk during Rush Phase peak; floor 64 = 80% of derived worst case (margin only for subscriber misbehavior) |
| `subscriber_list_initial_capacity` | 16 per channel (raised pass-1) | [8, 32] | Fewer reallocs on high-subscriber channels; more unused memory on low-subscriber channels | More reallocs (one-time cost at scene load); breaks AC-P1 steady-state if alloc occurs post-scene-load |
| `bootstrap_buffer_capacity_per_channel` (new pass-1) | 4 events | [2, 16] | More resilient to Awake-time event floods | Scene-load bootstrap overflow risk; 4 matches R8.a per-tick drain cap |
| T1 latency budget | 16.67ms (fixed wall-clock) | [8ms, 33ms] | More slack for hot-path subscribers; risk of missing 60fps | Tighter frame budget; harder to hit with complex subscribers |
| T2 latency budget | 100ms | [50ms, 200ms] | More slack before Feedback Floor breach; perceptible lag at high end | Tighter budget; may force subscriber optimization |
| Recursive-fire warning threshold | 10 frames | [5, 30] | Quieter logs; real runaway fires take longer to flag | Noisier Editor logs; catches runaway fires faster |
| Subscription leak check interval | 2 scene loads | [1, 5] | More forgiving of leaked subscriptions | Catches leaks faster; noisier on legitimate long-lived subs |

### Interaction Between Tuning Knobs

- `deferred_queue_capacity` and `recursive_fire_warning_threshold` interact: too-small capacity + too-high threshold = silent drops before detection; too-large capacity + too-low threshold = false-positive warnings. Adjust together.
- `T1 latency budget` and `T2 latency budget` must maintain T1 < T2 with a meaningful gap (at minimum 4× ratio) to distinguish frame-critical from feedback-floor events.
- `bootstrap_buffer_capacity_per_channel` and R8.a per-tick drain cap are locked together (both default 4). Raising one without the other defeats the spike-prevention contract.

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
| R7 `IEventPayload`-struct LINQ misuse + R7.a string-field ban are Forbidden Patterns (pass-1 revised from record-struct to `readonly struct`) | `.claude/docs/technical-preferences.md` | Forbidden Patterns section | Ownership handoff — this GDD adds both entries |

## Acceptance Criteria

**Pass-1 recount**: 32 criteria across 6 categories (up from 28 — 4 new ACs: AC-R2b depth-guard reset, AC-BOOT first-frame drain, AC-REL release-telemetry, AC-F4b Feedback Group aggregate; 1 renumbered: AC-F6 split into F6a/F6b/F6c). Gate: **BLOCKING CI** (12) / **PLAYMODE** (16) / **ADVISORY** (2) / **MANUAL** (2 — AC-F6b and AC-F6c require Mali-G52 device capture; MANUAL=0 prior claim was incorrect per qa-lead finding #5).

### H.1 Rule Compliance (R1-R7)

- **AC-R1 — Zero Hot-Path Allocation (ProfilerRecorder)** — GIVEN the Rush Phase stress scene (40 Rigidbody2D, 2 fires/frame each), WHEN a PlayMode test runs a `ProfilerRecorder` subscribed to the `"GC.Alloc"` marker scoped to `EventBus.Fire<T>` call-stack frames for 300 consecutive frames measured **starting 5 seconds after `SceneManager.LoadScene` completes** (steady-state window), THEN every sampled frame reports 0 bytes allocated; the test logs a per-frame `eventbus-alloc-report.json` artifact and exits 1 if any frame shows > 0 bytes. Pass-1 revision replaces "Memory Profiler" (snapshot-only tool) with `ProfilerRecorder` (per-frame delta) per qa-lead + unity-specialist convergent finding. **BLOCKING CI**
- **AC-R2 — No Synchronous Cross-Frame Recursion** — GIVEN a subscriber that calls `Fire` during its handler, WHEN the inner `Fire` executes, THEN the event is placed in the deferred ring buffer; depth guard > 0; inner event dispatches on next `EventBusPlayerLoopSystem.Update` tick. **BLOCKING CI (PLAYMODE)**
- **AC-R2b — Depth-Guard Reset Across Frames** (NEW pass-1) — GIVEN the bus has dispatched re-entrant events in frame N (depth > 0 at some point), WHEN frame N+1 begins, THEN the very first `Fire` call in frame N+1 dispatches synchronously (depth == 0 at entry, not deferred); verified by asserting the new event's subscriber executes inline before `Fire` returns. Additionally: simulate a 50ms frame stall between N and N+1 (thermal throttle sim) and verify depth-guard still resets. **BLOCKING CI (PLAYMODE)**
- **AC-R3 — No String-Typed Channels at Call Sites (Roslyn + Grep)** — GIVEN the full `src/` tree, WHEN the CI step runs `grep -rPn '\.(?:Subscribe|Fire)\s*<[^>]+>\s*\(\s*"' src/` (POSIX-compatible, handles CRLF via `-P` flag), THEN exit code is 0 (no matches). Any match fails CI with exit 1 and logs the matching file+line. The grep command is committed to `.github/workflows/eventbus-lint.yml` as the authoritative check. Pass-1 revision fixes regex defects (greedy `.*`, unclosed capture group) per qa-lead finding #2. **BLOCKING CI**
- **AC-R4 — No Silent Swallowing in Editor Builds** — GIVEN a `UNITY_EDITOR` build (not Development or Release per pass-1 R4 tightening), WHEN `Fire` is called on a channel with zero subscribers, THEN `Debug.LogWarning` fires with the channel SO as context within the frame; Editor Window unhandled-event count increments. Separately, the release-build counter (R4.a) increments in ALL build configurations. **PLAYMODE**
- **AC-R5 — Namespaces Are Distinct** — GIVEN a Gameplay channel ID, WHEN `Subscribe` is called with a Feedback context (or vice versa) in Editor builds, THEN `EventBusNamespaceMismatch` throws before registration; no subscription is created. **PLAYMODE**
- **AC-R6 — All Subscribers Receive Event Regardless of Registration Order** — (split pass-1 per qa-lead finding #3) GIVEN three subscribers A, B, C registered in that order, WHEN the channel dispatches 100 times across 10 fresh scene loads, THEN each subscriber receives all 100 events; observed per-dispatch order is logged to a CI artifact. The "no test may assert a specific order" clause is moved out of AC scope into `tests/helpers/event-bus-test-guidelines.md` (test-author guideline, not a bus AC). **PLAYMODE** (was ADVISORY)
- **AC-R7 — CI Flags IEventPayload LINQ Misuse** — GIVEN any `.cs` file with LINQ (`OrderBy`, `Where`, `Select`, `ToList`, `ToArray`, `GroupBy`) on an `IEventPayload`-implementing struct, WHEN the CI static-analysis job runs (`grep -rPn '\.(?:OrderBy|Where|Select|ToList|ToArray|GroupBy)\s*\(' src/ | xargs -I{} <IEventPayload-context-filter>` with a committed Python filter script), THEN the job emits a rule-violation error and exits 1. Pass-1 scope-narrowed from full Roslyn analyzer to two-tier grep + code review per R7 revision. **BLOCKING CI**
- **AC-R7a — CI Flags String Fields on IEventPayload** (NEW pass-1) — GIVEN any `IEventPayload`-implementing struct, WHEN the CI static-analysis job greps for `string` field declarations in structs that inherit `IEventPayload`, THEN zero matches; any match fails CI with exit 1. Prevents regression of the R7.a rule. **BLOCKING CI**

### H.2 Formula / Constant Correctness

- **AC-F1 — MERGED INTO AC-R1** — pass-1 revision merged AC-F1 into AC-R1 per qa-lead finding #4. The prior AC-F1 "60 seconds at 60fps" and AC-R1 "300 frames" tested the same invariant. Merged spec uses the 300-frame + 5-sec-steady-state window from AC-R1.
- **AC-F2 — Deferred Queue Capacity = 128; Overflow Drops Oldest (FIFO)** — GIVEN a test fixture where each event payload carries a monotonic `SequenceId` (0–128), AND a subscriber fires 129 re-entrant events in one frame (SequenceId 0 through 128), WHEN the next `EventBusPlayerLoopSystem.Update` tick drains the queue, THEN the received-event log contains exactly SequenceIds 1–128 in FIFO order; SequenceId 0 is absent; queue length never exceeded 128 at any drain point; R8.b release-build counter `deferred_queue_overflow_count` incremented exactly once. Pass-1 revision: capacity 64→128, reconciled with AC-EC3 (both now describe same contract), added SequenceId fixture for unambiguous verification. **BLOCKING CI (PLAYMODE)** (promoted from PLAYMODE-only per qa-lead finding #6)
- **AC-F3 — Subscriber List Initial Capacity = 16 (Constructor-Time)** — GIVEN a channel SO loaded in a fresh scene, WHEN no subscribers have registered yet, THEN `channel.SubscriberList.Capacity == 16` (verified via reflection or test-only accessor); AND after 16 subscriptions, `Capacity` remains 16 with no reallocation (verified by comparing the list's internal array reference before and after subscriptions 1–16); AND after the 17th subscription `Capacity >= 32` (one reallocation occurred). Pass-1 revision: capacity 8→16; reframed from post-add assertion (which fails against .NET List<T> growth semantics) to constructor-time pre-allocation (which matches the D.3 spec) per systems-designer + qa-lead findings. **BLOCKING CI (PLAYMODE)**
- **AC-F4 — T1 Latency Budget (P95 ≤ 16.67ms, P99 ≤ 33ms on CI hardware)** — GIVEN all T1-tier pairs (Input→VFX/Juice, Input→HUD, Collection→Audio, Hazard→VFX) in a PlayMode test on the CI runner (specify: GitHub Actions `ubuntu-latest` or `macos-latest` with Unity 6.3 LTS headless), WHEN each pair fires 100 times with `System.Diagnostics.Stopwatch` wrapping `Fire<T>` call through all subscriber returns, THEN P95 round-trip ≤ 16.67ms AND P99 ≤ 33ms (one-frame tolerance at 30fps floor); any sample exceeding 33ms is logged as a warning; test fails only if P95 is breached. Mali-G52 device validation deferred to AC-F6b/c (MANUAL at VS milestone). Pass-1 revision: replaced "no outliers" (unverifiable) with P95/P99 definitions + hardware specification per qa-lead finding #7. **PLAYMODE**
- **AC-F4b — Feedback Group Aggregate Latency** (NEW pass-1) — GIVEN a registered Feedback Group for `input.tap.main` = {VFX/Juice, Audio Bus, HUD, Haptics}, WHEN the event fires, THEN `group_latency = max(subscriber.completion_timestamp) - fire_timestamp` ≤ 16.67ms for T1 groups / ≤ 100ms for T2 groups across 100 samples; measured via per-subscriber timestamps logged inside the bus's dispatch loop. Per-subscriber order remains undefined (R6). Pass-1 resolves game-designer finding #5 + CD adjudication of R6 mitigation. **PLAYMODE**
- **AC-F5 — T2 Latency Budget + Multi-hop Chain Latency** — GIVEN all T2-tier pairs (Tool→Level Runtime, Critter AI→HUD, Gopher Spawn→Collection, Level Runtime→HUD) AND the documented multi-hop chains (Collection→Currency→HUD, Hazard→Currency→HUD), WHEN events fire in a PlayMode test on CI hardware, THEN single-hop T2 round-trip P95 ≤ 100ms AND multi-hop chain_latency (per D.5 formula) ≤ 100ms at 30fps simulated frame budget (33.33ms × hop count + measured subscriber execution). Any chain breaching 100ms at 30fps must either be restructured or flagged as Pillar 5 known risk. Pass-1 revision adds chain latency clause per game-designer finding #3 + D.5 formula. **PLAYMODE**

- **AC-F6a — Automated Bus-Internal Latency (PlayMode)** — (split from prior AC-F6 per CD adjudication) GIVEN `input.tap.main` fires with `Time.realtimeSinceStartup` timestamp injected into `TapPayload`, WHEN all registered subscribers complete their handlers, THEN delta from `TapPayload.InputTimestamp` to the last subscriber's handler-entry timestamp is ≤ 2ms on CI hardware; proves the bus itself is not the bottleneck. **PLAYMODE (BLOCKING CI)**
- **AC-F6b — Manual Device Visual Feedback Floor (Mali-G52)** (NEW pass-1) — GIVEN a dev build on a minimum-spec Android device with Mali-G52 GPU, a human tester records gameplay at 240fps (via HDMI capture card or high-speed camera), WHEN the tester marks the frame of tap-on-playfield and the first frame showing the coin-splash VFX, THEN `(frame_delta × 1000/240)` ≤ 100ms at P99 across 20 test taps; evidence filed in `production/qa/evidence/pillar5-visual-feedback-floor-[date].md` with lead sign-off. **MANUAL (VS milestone)**
- **AC-F6c — Manual Device Audio Feedback Floor (Mali-G52)** (NEW pass-1) — GIVEN the same dev build on Mali-G52 Android device with speaker output captured via external microphone or line-in, WHEN the tester marks the tap timestamp and the audio cue onset peak at the speaker (waveform analysis), THEN delta ≤ 100ms at P99 across 20 test taps; captures Android audio-pipeline latency (AudioTrack buffering + OS scheduler) that dispatch-only measurements cannot see. Evidence filed in `production/qa/evidence/pillar5-audio-feedback-floor-[date].md` with audio-director + lead sign-off. **MANUAL (VS milestone)**

- **AC-BOOT — Bootstrap First-Frame Drain Budget** (NEW pass-1) — GIVEN a scene-load fixture where all ~30 MVP channels each receive 4 buffered events during Uninitialized state (120 events total), WHEN the first `EventBusPlayerLoopSystem.Update` after Ready executes, THEN aggregate drain duration (including all subscriber invocations across all ticks needed to complete drain) ≤ 8ms on CI hardware; AND no single `Update` tick exceeds 4 events per channel (R8.a cap respected); AND no `[EventBus] Bootstrap overflow` appears in a clean scene load. **BLOCKING CI (PLAYMODE)**

### H.3 Edge Case Handling

- **AC-EC1 — Mid-Dispatch Unsubscribe** — GIVEN subscriber B unsubscribes itself from within its handler during dispatch to A, B, C, WHEN the event dispatches, THEN no exception thrown; A + C receive the event; B marked pending-removal; list compacts after dispatch. **BLOCKING CI (PLAYMODE)**
- **AC-EC2 — Subscriber Exception Does Not Break Dispatch** — GIVEN subscriber A throws in its handler and subscriber B is registered after A, WHEN the channel fires in a UNITY_EDITOR build, THEN B executes; `[EventBus] Subscriber exception` logged (Editor only per R4); `Fire` does not throw. Separately in Release/Development builds, B still executes and Fire still does not throw, but no log emits (per R4 tightening). **BLOCKING CI (PLAYMODE)**
- **AC-EC3 — MERGED INTO AC-F2 (pass-1)** — the prior AC-EC3 described the same invariant as AC-F2 (queue overflow drops oldest). Unified into AC-F2 with explicit SequenceId fixture per qa-lead finding #6.
- **AC-EC4 — Namespace Mismatch Assertion** — GIVEN channel `gameplay.level.phase_changed`, WHEN Editor-build code calls Subscribe using Feedback path, THEN `EventBusNamespaceMismatch` throws before handler registration; assertion message includes channel ID and conflicting namespace names. **PLAYMODE**
- **AC-EC5 — Recursive Fire Warning (Editor only)** — GIVEN a subscriber on channel X that unconditionally re-fires X every time, WHEN the deferred queue re-dispatches for 10 consecutive frames in a UNITY_EDITOR build, THEN `[EventBus] Probable recursive fire on {channel}` logs ONCE per run (not per frame — no log flood). Release-build counter R8.b increments `recursive_fire_count` instead of logging. **PLAYMODE**
- **AC-EC6 — Subscribe During Dispatch: New Subscriber Excluded from In-Flight** — GIVEN subscriber A subscribes a new subscriber B during its handler, WHEN the event dispatches, THEN B does NOT receive the current event; B receives all subsequent dispatches; confirmed via capture list. **PLAYMODE**
- **AC-EC7 — Mid-Dispatch Scene Unload (Dispatching → Disposed)** (NEW pass-1) — GIVEN the bus is in `Dispatching` state with queued deferred events AND `SceneManager.LoadScene` is triggered, WHEN the currently-dispatching `Fire` completes, THEN the bus transitions to Disposed only after the in-flight chain returns (no preemption mid-subscriber); queued deferred events are discarded; R8.b `scene_unload_discard_count` increments by the discarded count; no exception thrown. **PLAYMODE**

### H.4 Performance

- **AC-P1 — Zero GC Allocations/Frame Steady State (ProfilerRecorder)** — GIVEN a PlayMode test scene firing all 13 MVP emitter/subscriber pairs at documented rates using a scripted event emitter, WHEN the scene has been running for **at least 5 seconds after `SceneManager.LoadScene` completes** (steady-state window start), THEN `ProfilerRecorder` sampling the `"GC.Alloc"` marker scoped to EventBus call-stack methods over the 600 subsequent frames (10 seconds at 60fps) reports 0 bytes total allocation; test fails immediately if any frame shows > 0 bytes. Pass-1 revision: named ProfilerRecorder API (not Memory Profiler package), defined steady-state window, specified 600-frame sample duration per qa-lead findings #1 and #8. **BLOCKING CI**
- **AC-P2 — Memory Footprint per Channel (Order-of-Magnitude)** (REVISED pass-1 per performance-analyst finding #4) — GIVEN 30 `EventChannelSO<T>` assets loaded at scene start with 16-slot subscriber list each plus per-channel bootstrap ring buffers, WHEN profiled via `UnityEngine.Profiling.Profiler.GetRuntimeMemorySizeLong(channelSO)` summed across all 30 instances in a PlayMode test, THEN total ≤ **24 KB** (revised from 8 KB: 30 channels × ~800 bytes per channel including 16-slot delegate list at 40 bytes/delegate = 19.2 KB delegates + SO overhead). Reclassified as order-of-magnitude check. Result logged to `production/qa/smoke-[date].md`. **ADVISORY**
- **AC-P3 — T1 Latency Under Peak Load (Subscriber-Discipline Acknowledged)** (REVISED pass-1 per performance-analyst finding #3) — GIVEN Rush Phase peak (40 Rigidbody2D, max emitter activity), WHEN T1 and T2 events fire concurrently in the same frame, THEN T1 round-trip P95 ≤ 16.67 ms AND T2 round-trip P95 ≤ 100 ms, **provided individual T2 subscriber handlers each complete in ≤ 5ms** (documented dependency — synchronous dispatch cannot enforce T1 priority over a slow T2 subscriber; this AC now explicitly names the dependency). Subscriber-level latency is a separate budget enforced per consumer system GDD. **PLAYMODE**
- **AC-REL — Release-Build Telemetry Counters** (NEW pass-1, resolves game-designer finding #4 + R4.a + R8.b) — GIVEN a release build with Analytics v1.0 enabled, WHEN a session runs for 10 minutes with mixed normal + overflow + exception scenarios injected, THEN the bus's release counters (`zero_subscriber_fire_count`, `deferred_queue_overflow_count`, `bootstrap_overflow_count`, `scene_unload_discard_count`, `recursive_fire_count`) increment monotonically and flush to Analytics at session-end via the R9 codegen path. Verified by reading the Analytics service's received event log. **PLAYMODE (at VS milestone when Analytics stub available)**

### H.5 Consumer Integration

Pass-1 note: the AC-INT series below references "T1/T2 budget" — this is the CI-hardware PlayMode budget defined in AC-F4/AC-F5 (P95 ≤ 16.67ms / ≤ 100ms). Mali-G52 device validation is deferred to AC-F6b/c (MANUAL at VS milestone).

- **AC-INT1 — Input System** — subscriber on `input.tap.main` receives `TapPayload(Vector2(100,200), null)` within one frame. **PLAYMODE**
- **AC-INT2 — Collection System** — subscribers on both `audio.collect.coin.success` + corresponding Gameplay channel receive matching `CollectionSuccessPayload` for a single coin collection; both execute in same T1 frame. **PLAYMODE**
- **AC-INT3 — Tool System** — subscriber on `audio.tool.place.ramp` receives `ToolEventPayload(Ramp, Vector2(50,50))` within T2 budget. **PLAYMODE**
- **AC-INT4 — Hazard System** — subscribers on `audio.hazard.bomb.detonate` + Gameplay pair receive matching `BombDetonatedPayload`; dispatch ≤ T1. **PLAYMODE**
- **AC-INT5 — Critter AI** — subscriber on `audio.critter.dog.deploy` receives `DogEventPayload(AgentID:1, Vector2(0,0))` within T2; no Gameplay-namespace subscriber receives the Feedback event. **PLAYMODE**
- **AC-INT6 — Gopher Spawn** — subscriber on `gameplay.gopher.launched` receives `GopherLaunchedPayload(7, "standard")` with matching fields within T2. **PLAYMODE**
- **AC-INT7 — Level Runtime** — subscribers on `phase_changed`, `complete`, `quota_progress` receive payloads in correct order during phase transitions + win condition; `LevelEndPayload.Success == true` on win. **PLAYMODE**

### H.6 No Hardcoded Channel Strings (pass-1 merged with AC-R3)

- **AC-STR1 / AC-STR2 — MERGED INTO AC-R3 (pass-1)** — prior AC-STR1 (Subscribe) and AC-STR2 (Fire) used defective regex patterns (greedy `.*`, unclosed capture group, CRLF handling). Unified into the revised AC-R3 which runs the fixed POSIX-compatible pattern across BOTH Subscribe and Fire call sites via a single `grep -rPn '\.(?:Subscribe|Fire)\s*<[^>]+>\s*\(\s*"'` check.
- **AC-STR3 — Generated Constants Fresh + Compile After Taxonomy Change** — GIVEN a change to `AudioEventTaxonomy` or `VisualEventTaxonomy` SO, WHEN the PR is opened, THEN the `.github/workflows/eventbus-codegen-staleness.yml` job runs codegen in headless Unity and `git diff --exit-code`s the three generated files (`AudioEventIds.cs`, `VisualEventIds.cs`, `AnalyticsSubscribers.cs`); any drift fails CI. AND: the full project compiles with the regenerated files; any call-site reference to a removed channel produces a compile-time error (not a silent runtime mismatch). Pass-1 revision: switched mechanism from AssetPostprocessor-at-build-time (unreliable in headless CI per unity-specialist finding #5) to committed-generated-files + staleness-CI-check. **BLOCKING CI**

### Gate Summary (recounted pass-1)

| Gate | Count | ACs |
|---|---|---|
| **BLOCKING CI** | 12 | AC-R1, AC-R2, AC-R2b, AC-R3, AC-R7, AC-R7a, AC-F2, AC-F3, AC-F6a, AC-BOOT, AC-EC1, AC-EC2, AC-P1, AC-STR3 |
| **PLAYMODE (non-CI-blocking)** | 13 | AC-R4, AC-R5, AC-R6, AC-F4, AC-F4b, AC-F5, AC-EC4, AC-EC5, AC-EC6, AC-EC7, AC-P3, AC-REL, AC-INT1-INT7 (INT series collectively) |
| **ADVISORY** | 1 | AC-P2 |
| **MANUAL (device capture)** | 2 | AC-F6b (visual at Mali-G52), AC-F6c (audio at Mali-G52) |

Pass-1 recount note: the prior Gate Summary claimed BLOCKING CI = 10 / PLAYMODE = 14 / ADVISORY = 3 / MANUAL = 0. Actual body count of PLAYMODE labels was 20 (not 14) and MANUAL = 0 was false due to AC-F6's off-device measurement requirement. Table rebuilt after AC rewrites. Totals: 14+13+1+2 = 30 ACs in H.1–H.6 (AC-F1 and AC-STR1/STR2 merged; AC-R2b/AC-F4b/AC-F6a/b/c/AC-BOOT/AC-EC7/AC-REL/AC-R7a added).

**Shift-left note**: the 14 BLOCKING CI criteria need stub EditMode/CI test scaffolding at story start, before bus implementation begins. The 2 MANUAL criteria (AC-F6b/c) are deferred to Vertical Slice milestone when Mali-G52 device access + HDMI/microphone capture setup are available. PLAYMODE criteria require a test scene fixture with representative subscriber MonoBehaviours — scaffold alongside bus implementation, not deferred to QA hand-off.

## Open Questions

| # | Question | Owner | Target Resolution |
|---|---|---|---|
| OQ-1 | For non-MonoBehaviour subscribers (ScriptableObjects, pure C# services): is the manual `SubscriptionHandle.Unsubscribe()` pattern sufficient, or should the bus track weak references with GC-time warnings in release builds (not just Editor)? Current design: Editor-only leak detection. Risk: service-layer subscribers leak quietly in production. | Lead Programmer | During first non-MonoBehaviour subscriber implementation |
| OQ-2 | `SubscriptionManager` component auto-attached vs. explicit attachment: when `SubscribeAutoManaged` is called, should the bus auto-attach a `SubscriptionManager` if the GameObject lacks one, or throw and require the author to add it explicitly? Auto-attach is more ergonomic; explicit requires discipline. | Lead Programmer + systems-designer | Architecture phase (`/architecture-decision`) |
| ~~OQ-3~~ **RESOLVED 2026-04-20 via CD-GDD-ALIGN Concern 3**: Analytics uses option (c) with codegen. An `AssetPostprocessor` generates `AnalyticsSubscribers.cs` from the `gameplay.*` taxonomy entries at build time. Matches R3 pattern; preserves R1 + R7; preserves Pillar 3 analytics tuning loop. See R9. |
| OQ-4 | Should the Event Bus expose a deterministic replay mode for testing (record all events + payloads + timestamps during a session, replay them in unit tests)? Would be invaluable for regression tests but adds complexity. Physics is non-deterministic per TD ADR, so full replay is impossible — partial event replay might still be useful. | QA Lead + Lead Programmer | Post-MVP (when first flaky test surfaces) |
| OQ-5 | Event ID format validation runs at editor time (Data Registry R7), but the bus performs a second runtime check in debug builds. Is this redundancy worth the cost, or should we remove the runtime check and trust Data Registry's CI gate? | Lead Programmer | During first profiling pass of debug-build overhead |
| ~~OQ-6~~ **RESOLVED 2026-04-20 via CD-GDD-ALIGN Concern 2; REVISED 2026-04-21 pass-1**: Bus maintains per-channel typed bootstrap ring buffers (capacity 4 per channel) during Uninitialized state — replaces original single 16-slot buffer to eliminate boxing risk. Events fired during `Awake` are queued per-channel; drain time-sliced (≤4 events per channel per Update tick) on first tick after Ready. See R8 + R8.a. |
| OQ-7 | C# version: pass-1 adopted conservative `readonly struct` (C# 9) fallback rather than `readonly record struct` (C# 10). If Unity 6.3 LTS confirms C# 10 support in official release notes, consider reverting to `readonly record struct` for ergonomic gains (auto `Equals`, `ToString`, `with` expressions). Impact: retrofit payload definitions across all 12 downstream systems. | Lead Programmer + unity-specialist | During story start, verify Unity 6.3 LTS C# language version via official release notes |
| OQ-8 | Drop policy on overflow: pass-1 retained FIFO (drop oldest) per CD adjudication, with release-build telemetry counter (R8.b) to surface real-world overflow rate. Revisit policy at v1.0 + 3 months live-ops with actual telemetry. Possible alternatives: drop-newest (prefer preserving chain coherence), priority-tag-based drop (require per-event priority metadata). | Lead Programmer + game-designer | Post-launch with overflow-rate telemetry data |
