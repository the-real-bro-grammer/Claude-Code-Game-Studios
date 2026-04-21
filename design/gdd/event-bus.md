# Event Bus

> **Status**: **APPROVED** 2026-04-21 — pass-3 revision accepted by user without further adversarial review, per CD pass-2 guidance ("Pass 3 should be gate-check-ready, not a third adversarial round"). 20 pass-2 blockers + 1 structural D.4 amendment resolved in fresh-session fix pass.
> **Author**: Robert Michael Watson + systems-designer / unity-specialist / creative-director / qa-lead / performance-analyst / game-designer
> **Last Updated**: 2026-04-21 (pass-3 revision)
> **Implements Pillar**: Foundation for Pillars 1 (Cute Chaos), 2 (Readable at a Glance), 5 (No Input Goes Silent). Indirect support for 3 and 4 via decoupling.
>
> **Creative Director Review (CD-GDD-ALIGN)**: CONCERNS → RESOLVED 2026-04-20
>
> **/design-review pass 1 (2026-04-21)**: NEEDS REVISION → RESOLVED (same-session). 14 blockers fixed: `readonly struct` fallback, per-channel bootstrap buffers, committed codegen + staleness CI, R4 tightened to UNITY_EDITOR-only, deferred queue 64→128, AC-F6 split F6a/b/c, Feedback Group concept, ProfilerRecorder API, grep fixes, hardware split. See review log pass-1.
>
> **/design-review pass 2 (2026-04-21, fresh session)**: NEEDS REVISION. 20 blockers + 1 structural D.4 Pillar 5 amendment (`intra_group_onset_spread` metric). See review log pass-2.
>
> **/design-review pass 3 (2026-04-21, fresh session fix pass per CD recommendation)**: NEEDS REVISION → RESOLVED. User-adjudicated 4 design decisions (queue 256; 2-hop chain cap; once-per-session DEV_BUILD overflow log; R2.b wall-clock framing). Structural D.4 amendment applied with onset-spread ≤ 22ms perceptual threshold + T3 tier introduction for Analytics. Tier-1 bundles: D.5 hard 2-hop cap (AC-F5 enforces), R4.a Dictionary pre-pop contract with ChannelIDComparer, AC-R1/P1 merged with cold-fire JIT sweep + thread-local GC delta + ProfilerRecorder secondary, Gate Summary recounted to 36 ACs (16 BLOCKING CI / 16 PLAYMODE / 2 ADVISORY / 2 MANUAL), D.4 onset-spread amendment. Tier-2 + surgical: D.2 128→256 with dual-namespace derivation, R2.b dual-framing (type + path), R8 OnEnable reset contract, R9 explicit ImportAsset + 512-slot fixed batch buffer, AC-P2 math rederived per-component, AC-F6a Stopwatch.GetTimestamp replaces float precision, AC-F6b/c P95@N=20 (CD pass-2 adjudication), AC-F3 test-only accessor replaces reflection, AC-R3 regex covers interpolated `$"..."`, AC-R7 filter script named + unit-tested, AC-R7a Python scanner replaces multi-line grep, AC-REL fixture paths pinned, AC-BOOT split A (bus-overhead BLOCKING) / B (subscriber-cost ADVISORY), AC-R2b deterministic depth-guard probe replaces tautological sleep, R8.a removed from tuning knobs (hardcoded invariant per CD), OQ-7 closed (C# 9 only confirmed), FIFO drop once-per-session DEV_BUILD log added. See review log pass-3 revision entry.

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

**R2. No synchronous cross-frame recursion.** Events fired from inside a subscriber handler are queued and dispatched at the start of the next frame, not inline. A depth guard (int, frame-local) enforces this: depth > 0 triggers the deferred queue. The queue is a pre-allocated ring buffer (**capacity: 256 events** — tuning knob; raised from 128 in pass-3 review per systems-designer pass-2 finding that dual-namespace worst case is 160 events/frame (40 RB2D × 2 emits × 2 namespaces), which exceeds the pass-1 128 capacity. 256 = 1.6× dual-namespace worst case, matches the safety margin strategy applied at pass-1 on the single-namespace case).

**R2.a. Depth-guard reset contract.** The frame-local depth int is reset **at the start of each `EventBus.Update()` tick, before any drain or dispatch occurs**. It is never reset mid-frame. This guarantees that a thermal-throttle stall (e.g., 50ms Update) cannot leave depth non-zero at the start of the next logical frame; the reset happens unconditionally at Update entry. Verified by AC-R2b (added in pass-1 review).

**R2.b. PlayerLoop injection point + synchronous dispatch model (pass-1; pass-3 rewrote to resolve unity-specialist pass-2 findings on PlayerLoop type/path conflation and physics T1 semantic).** The bus's **deferred-queue drain** (and ONLY the deferred-queue drain — see below) is injected into Unity's PlayerLoop.

**Injection specification — both type AND path (pass-3):**
- **Subsystem type**: `EventBusPlayerLoopSystem` — a `struct` conforming to `PlayerLoopSystem` (not a MonoBehaviour). Contains a `delegate*<void>` update function pointer.
- **Path**: inserted as a child of `UnityEngine.PlayerLoop.PreUpdate`, as the LAST child in PreUpdate's subSystemList (after all built-in PreUpdate subsystems).
- **Injection API**: `UnityEngine.LowLevel.PlayerLoop.GetCurrentPlayerLoop()` → clone → walk subSystemList to find `typeof(UnityEngine.PlayerLoop.PreUpdate)` → append new child struct → `PlayerLoop.SetPlayerLoop(updated)`.
- **Injection timing**: `[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.AfterAssembliesLoaded)]` static method on the bus; called once per domain load.

This distinguishes the TYPE (`EventBusPlayerLoopSystem` struct) from the PATH (`PreUpdate > EventBusPlayerLoopSystem`) — pass-2 text conflated them.

**Synchronous dispatch model — T1 latency is wall-clock, not frame-phase (pass-3 replaces prior "same rendered frame" claim per unity-specialist pass-2 finding that FixedUpdate fires 0–N times per rendered frame):**

- `Fire<T>(channel, payload)` is SYNCHRONOUS. Subscribers dispatch inline on the publisher's thread at the call site. This is the ONLY dispatch path when depth == 0.
- The PlayerLoop hook ONLY drains the deferred queue (re-entrant fires that hit depth > 0). Standard `Fire` calls do NOT go through the PlayerLoop hook.
- **Physics-originated events** (fired from `OnCollisionEnter2D`, `OnTriggerEnter2D`, or any FixedUpdate-time callback): dispatch synchronously at the Fire call site, INSIDE the FixedUpdate execution window. Subscribers see the event at Fire time regardless of whether 0, 1, or N FixedUpdate steps occur per rendered frame. No queuing occurs for standard physics events.
- **T1 latency budget is wall-clock** (≤ 16.67ms from `Fire` call to last subscriber return — measured via `Stopwatch.GetTimestamp()` delta per AC-F6a), independent of FixedUpdate/Update ordering. "T1 same rendered frame" is a consequence of the wall-clock budget when subscribers are fast, not a semantic guarantee tied to frame phase.
- Re-entrant fires (a subscriber calling `Fire` during its handler) are the ONLY case where PlayerLoop ordering matters: the inner event queues, drains on next PlayerLoop `PreUpdate > EventBusPlayerLoopSystem` tick. A physics-originated handler that re-fires still queues; the re-fired event drains at the next PreUpdate phase of whichever rendered frame comes next.

Previous "queued during FixedUpdate and drain within the same rendered frame, preserving T1 tier semantics" language is retired — it implied a queue-and-drain model that is not how synchronous physics events actually work.

**R3. No string-typed channels at call sites.** All channel IDs are typed constants generated from the Data Registry taxonomies at editor/build time (an `AssetPostprocessor`-triggered codegen script writes `AudioEventIds.ToolPlaceRamp` / `VisualEventIds.CollectCoinSplash` const strings). Publishers reference these constants; passing a bare string literal to `Subscribe` or `Fire` is a CI static-analysis failure. This resolves Data Registry OQ-4 in favor of generated constants.

**R4. No silent swallowing in debug builds.** If an event is fired on a channel with zero active subscribers, a `Debug.LogWarning` fires with the channel SO as context (clickable in Unity Console). In release + development builds this check is stripped via `#if UNITY_EDITOR` (tightened from `UNITY_EDITOR || DEVELOPMENT_BUILD` in pass-1 review per performance-analyst finding: `DEVELOPMENT_BUILD` inclusion would break AC-R1 zero-alloc measurement, because the profiler stress scene runs as a dev build and string-interpolated warning logs allocate). A custom Editor Window also accumulates unhandled-event counts per channel for post-session audit (mirrors Data Registry's Balance Audit window pattern).

**R4.a. Release-build telemetry counter (pass-1; pre-population contract pass-3 per unity-specialist + performance-analyst pass-2 convergent finding).** Separate from the Editor-only `Debug.LogWarning` path, the bus maintains a per-channel **zero-subscriber fire counter** incremented on any `Fire` call that finds no subscribers.

**Pre-population contract (pass-3):** The counter Dictionary is declared `Dictionary<ChannelID, int>` with **explicit pre-population of ALL N channel keys** in `EventBus.Awake`, immediately AFTER `AudioEventIds.RegisterAll()` and `VisualEventIds.RegisterAll()` complete (channel IDs known) but BEFORE the bus transitions to Ready. The pre-pop loop iterates the full `ChannelID` enumeration (from the codegen registries) and assigns `_zeroSubscriberFireCounts[id] = 0` for each. The backing Dictionary MUST be constructed as `new Dictionary<ChannelID, int>(capacity: N_channels, comparer: ChannelIDComparer.Default)` — both the capacity pre-allocation (avoids rehash-grow allocations) AND the custom `IEqualityComparer<ChannelID>` (avoids boxing on `GetHashCode`/`Equals` via the default comparer's runtime-type check for `IEquatable<T>`, which would box in certain Mono runtime paths). Increment at Fire site uses `_zeroSubscriberFireCounts[channel]++` — a read-write on a pre-existing key, zero allocation, no branch for key-missing.

**Rationale for pre-pop contract:** lazy key insert (`if (!dict.ContainsKey) dict[key] = 0; dict[key]++`) allocates the first time each channel fires without subscribers — violating R1 zero-alloc steady-state measurement. Without explicit pre-pop, AC-R1 would measure allocations on rarely-fired channels only after they first fire in the test window (a partial miss per performance-analyst finding #6). Pre-pop eliminates the first-fire allocation entirely.

**Downstream usage:** at v1.0, Analytics reads the counter via `EventBus.DrainTelemetryCounters()` (returns a pre-allocated struct; called once per session-end) and forwards `eventbus.zero_subscriber_rate` to the analytics batch buffer (R9). Provides production visibility into silent Pillar 5 failures (a taxonomy event fired with no subscribers = the player tap got no feedback). Resolves game-designer pass-1 finding #4 + pass-2 pre-pop contract blocker.

**R5. Taxonomy events and internal events are distinct channel namespaces.** Taxonomy events (whose IDs appear in `AudioEventTaxonomy` or `VisualEventTaxonomy` and match Data Registry R7 regex `^(audio|vfx)\.[a-z_]+\.[a-z_]+\.[a-z_]+$`) live in `ChannelNamespace.Feedback`. Internal gameplay events (e.g., `gameplay.level.phase_changed`) live in `ChannelNamespace.Gameplay` (format `^gameplay\.[a-z_]+\.[a-z_]+$`). A channel from one namespace cannot be used on the other — a runtime assertion enforces this in debug builds. Prevents gameplay subscribers from accidentally registering on audio-routed channels.

**R6. Subscriber dispatch order is undefined.** The bus makes no guarantee about which subscriber is invoked first. Any system requiring ordering (e.g., HUD must update before a post-frame screenshot) uses a two-phase pattern: subscribe to the event, then post a local update to a sorted execution queue it owns. Ordering dependencies between subscribers are a smell — the subscribing systems own resolution, not bus insertion order.

**R6.a. Feedback Group concept (pass-1; amended pass-3 with onset-spread metric per game-designer pass-2 finding #1).** While R6 leaves per-subscriber order undefined, a **Feedback Group** is a named set of subscribers for the SAME event (e.g., `CollectionSuccess` Feedback Group = {Audio Bus, VFX/Juice, HUD, Haptics}) whose Pillar 5 guarantee is measured by TWO independent metrics that BOTH must pass: (1) `group_latency` — aggregate completion within the tier budget (T1 ≤ 16.67ms, T2 ≤ 100ms), and (2) `intra_group_onset_spread` — the first and last subscriber in the group START their handlers within 22ms of each other (perceptual synchrony threshold). See D.4 for formal definitions. The bus exposes `RegisterFeedbackGroup(ChannelID channel, string groupName)` for diagnostic purposes; AC-F4b measures BOTH metrics for each registered group. The two-metric design closes the gap that a 100ms-aggregate pass could hide a perceptible 80ms audio-visual desync. Per-subscriber order remains undefined (R6); the onset-spread budget constrains only the window between first and last onset, not the order. Analytics subscribers are T3-tier (R9 codegen path, async-flush in PostLateUpdate) and are NEVER registered in a Feedback Group. Playtest sign-off remains the gate for residual perceived jitter within the 22ms envelope.

**R7 (from Unity-specialist CI safety net — renumbered into sequence in pass-1).** Because typed payloads use `readonly struct` (see "Typed Payload Pattern" below), CI static analysis must flag any LINQ expression or equality chain on an `IEventPayload`-implementing struct inside a dispatch hot path method. `readonly struct` is allocation-free when passed through `Fire<T>`, but misuse (e.g., `payloads.OrderBy(p => p.Key)` or `payloads.Where(...)`) allocates via enumerator boxing and intermediate collection creation. This rule is listed as a Forbidden Pattern in `.claude/docs/technical-preferences.md`. Pass-1 note: the Roslyn-analyzer implementation path for this rule is scoped to a simpler two-tier check — (1) grep-level flag on any LINQ method on an `IEventPayload` implementor regardless of context (noisy but zero false-negatives), (2) manual code review as secondary safety net. A full context-aware Roslyn analyzer is scoped as an optional follow-up engineering task, not a prerequisite for R7 enforcement.

**R7.a. String-typed payload fields flagged (resolves performance-analyst finding #6).** CI static analysis ALSO flags any `string` field on an `IEventPayload`-implementing struct. Strings are reference types stored by-reference in the ring buffer, increasing GC scan cost and inviting publishers to inline-construct strings (which allocates outside the bus's measured scope). Preferred alternatives: enum-backed ID (`int` or `LootTableId` enum), `ChannelID` struct reference, or a pre-interned-string reference via `StringPool`. Existing `GopherLaunchedPayload.LootTableId: string` must be migrated to `LootTableId: int` (enum-backed lookup into Data Registry's loot table catalog).

**R8 (locked from CD-GDD-ALIGN Concern 2; revised pass-1 to use per-channel buffers; pass-3 added OnEnable reset contract per unity-specialist pass-2 finding on cross-scene contamination).** Each `EventChannelSO<T>` owns its own **typed bootstrap ring buffer of capacity 4**. Any `Fire<T>` call during the `Uninitialized` state (after the bus awakens but before all subscribers have completed registration — typically the first `Awake → Start` window) is written to that specific channel's typed buffer as a value-type `T` (no boxing, no `IEventPayload` interface storage). On the bus's **first `EventBusPlayerLoopSystem.Update` tick after Ready**, each channel drains its own buffer in FIFO order via normal dispatch, subject to the per-tick drain cap in R8.a. Per-channel bootstrap overflow (5th event on a channel) drops the newest event and logs `[EventBus] Bootstrap overflow on {channel}` (Editor only per R4). Scene-wide bootstrap capacity is effectively `channel_count × 4` (e.g., ~30 channels × 4 = 120 slots) with zero boxing because each buffer is typed.

**Bootstrap buffer reset contract (pass-3 — unity-specialist pass-2 finding on cross-scene contamination):**

Each `EventChannelSO<T>` implements `OnEnable()` with the following reset sequence:
```
_bootstrapBuffer.ReadIndex = 0;
_bootstrapBuffer.WriteIndex = 0;
_bootstrapBuffer.Count = 0;
// underlying fixed-size T[4] array is zeroed via Array.Clear(_bootstrapBuffer.Slots, 0, 4)
```
This runs on every scene load where the SO is referenced — ScriptableObjects are cached across scene loads in Unity; without an explicit reset, bootstrap events buffered in Scene A but never drained (because the scene unloaded mid-drain per AC-EC7) would leak into Scene B's bootstrap drain, delivering stale payloads to new subscribers.

`OnEnable` reset does NOT interact with an already-Ready bus because channel SOs load before the bus's PlayerLoop hook runs the first drain; by the time `EventBusPlayerLoopSystem.Update` executes its first-tick-after-Ready drain, any `Fire<T>` during Uninitialized has written to the freshly-reset buffer. If a scene reload occurs mid-session, the SO `OnEnable` runs AFTER the previous bus's Disposed state, before the new bus's Uninitialized → Ready transition completes. Cross-scene contamination path is closed.

Pass-1 rationale retained (resolves unity-specialist finding #2): a single heterogeneous 16-slot buffer would have required storing payloads as `IEventPayload` (interface box → heap allocation per queued boot event) or a discriminated-union struct (high implementation cost). Per-channel typed buffers eliminate boxing at the cost of slightly higher memory overhead (negligible — each buffer is 4 × sizeof(payload) ≈ 64 bytes). This replaces the original "throw on Uninitialized" behavior for the narrow scene-load window only; mid-session `Fire` on a Disposed bus still errors.

**R8.a. Bootstrap drain budget — hardcoded invariant (pass-1; CD pass-2 adjudicated hardcoded-over-tuning-knob).** Frame 0 after Ready is **NOT subject to the T1 16.67ms budget** — it is explicitly carved out. To prevent first-frame spikes, the drain is time-sliced: **at most 4 events per channel per `Update` tick** — a **hardcoded invariant**, NOT a tuning knob. Per-channel FIFO order preserved across ticks. Channels whose buffers are not fully drained continue draining on subsequent ticks. AC-BOOT-A (split in pass-3 from prior AC-BOOT) asserts first-frame bus-dispatch-overhead drain completes in ≤ 8ms aggregate on CI hardware even with all 30 channels loaded.

**Why hardcoded, not a tuning knob (CD adjudication of performance-analyst pass-2 finding):** the drain cap exists specifically to prevent Awake-time event storms from overwhelming the first frame. Making it tunable lets someone raise it to `int.MaxValue` and defeat the spike-prevention contract entirely. The value 4 matches `bootstrap_buffer_capacity_per_channel = 4` — they are LOCKED to the same number by design. If a future profile surfaces a real need to change one, both must move together in a coordinated ADR, not a casual knob adjustment. Removed from the Tuning Knobs table in pass-3.

In practice, well-behaved boot profiles drain in a single tick; the time-slice is a safety net for future growth.

**R8.b. Telemetry for overflow events.** The bus maintains a release-build counter for `deferred_queue_overflow_count` and `bootstrap_overflow_count` (incremented on each drop). These are forwarded to Analytics at v1.0 end-of-session (reusing the R4.a telemetry bucket). Resolves creative-director adjudication of drop-policy disagreement: FIFO drop retained, but telemetry provides the signal to revisit the policy with real data.

**R9 (locked from CD-GDD-ALIGN Concern 3; revised pass-1 for codegen CI integration; pass-3 added explicit ImportAsset + batch-buffer sizing per unity-specialist + performance-analyst pass-2 findings).** A companion `AssetPostprocessor` generates `AnalyticsSubscribers.cs` from the `gameplay.*` entries of every taxonomy SO. Generated file is **committed to version control** (not regenerated at build time) with a staleness CI check.

**Staleness CI workflow (pass-3 explicit ImportAsset per unity-specialist pass-2 finding on silent false-pass):**

The `.github/workflows/eventbus-codegen-staleness.yml` job executes this sequence in order:
1. Launch Unity headless: `unity -batchmode -quit -executeMethod EventBusCodegen.RunStalenessCheck -logFile -`.
2. `EventBusCodegen.RunStalenessCheck` performs the following inside Unity:
   - **Explicit reimport of taxonomy SOs** (required — batch mode does NOT auto-run AssetPostprocessor for assets unchanged on disk since last import):
     ```
     AssetDatabase.ImportAsset("Assets/Data/AudioEventTaxonomy.asset", ImportAssetOptions.ForceUpdate);
     AssetDatabase.ImportAsset("Assets/Data/VisualEventTaxonomy.asset", ImportAssetOptions.ForceUpdate);
     ```
   - Wait for import completion via `AssetDatabase.Refresh()` + `AssetDatabase.SaveAssets()`.
   - Trigger codegen method that writes the three generated files (`AudioEventIds.cs`, `VisualEventIds.cs`, `AnalyticsSubscribers.cs`) to their canonical locations.
3. After Unity exits, CI runs `git diff --exit-code Generated/AudioEventIds.cs Generated/VisualEventIds.cs Generated/AnalyticsSubscribers.cs`. Any drift fails CI.

Without the explicit `ImportAsset(ForceUpdate)`, the AssetPostprocessor never fires in headless mode and stale generated files pass the `git diff` check silently — a Pillar 3 analytics subscriber could go missing with no CI signal.

**Analytics batch buffer specification (pass-3 — performance-analyst pass-2 finding on unspecified resize policy):**

Each generated Analytics subscriber forwards its typed payload to a pre-allocated fixed-schema batch buffer:
- Buffer type: `AnalyticsBatchBuffer` — a bus-owned struct containing a fixed-capacity `AnalyticsEventSlot[512]` array plus head/count indices.
- Slot schema: each `AnalyticsEventSlot` is a tagged-union struct (event ID int + 64-byte payload scratch area for typical payload sizes + variant tag). Fixed 80 bytes per slot; total buffer = 40 KB pre-allocated once at bus Awake.
- Capacity rationale: 512 events per-frame flush = 8.5× worst-case dual-namespace Rush Phase (160 events/frame × 60fps flush cadence; Analytics flushes once per rendered frame, so per-frame capacity is 60× less than per-frame fire rate would suggest) — plenty of headroom.
- **Resize policy: NO RESIZE**. Overflow drops the new event, increments a release-build `analytics_batch_overflow_count` counter (added to R8.b set), and in UNITY_EDITOR/DEVELOPMENT_BUILD fires a once-per-session `Debug.LogError`. Dynamic resize would allocate mid-frame (breaks R1) and would hide the real issue (Analytics taxonomy events exceeding budget = designer problem, not bus problem).
- Flush cadence: once per rendered frame in `PostLateUpdate`, injected into PlayerLoop alongside the R2.b deferred-drain hook but at `PostLateUpdate > AnalyticsBatchFlushSystem` (LATE in PostLateUpdate's child list, after all subscriber work completes).

Analytics is **T3 tier** (see §Interactions latency-tier note): async-flush, NOT in Feedback Groups, NOT subject to T1/T2 ACs.

Pass-1 rationale retained (resolves unity-specialist finding #5): committing the generated files sidesteps AssetPostprocessor batch-mode fragility AND makes generated Analytics subscribers visible in PR reviews. Matches the R3 codegen pattern; preserves R1 zero-allocation (each subscription is typed, batch buffer is pre-allocated, no reflection); preserves R7 (no LINQ on payloads). Analytics consumers never manually subscribe to individual gameplay channels — the generated file is the single source of truth. Resolves OQ-3; preserves Pillar 3's analytics tuning loop.

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

Latency tiers: **T1** (< 16.67ms wall-clock, synchronous dispatch) for input response, collection, slapstick hazard reads; **T2** (< 100ms, Feedback Floor budget) for state changes, economy updates, AI signals; **T3** (< 500ms, async-flush in PostLateUpdate) for Analytics batch-and-flush and other non-feedback subscribers that queue-then-flush outside the synchronous dispatch envelope. T3 subscribers are NEVER registered in Feedback Groups and are NOT subject to T1/T2 ACs (AC-F4, AC-F4b, AC-F5, AC-P3). Pass-3 added T3 to close systems-designer pass-2 finding on Analytics semantic gap.

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
deferred_queue_capacity = 256 events (pre-allocated ring buffer; tuning knob)
```

**Pass-3 revision (raised 128 → 256 per systems-designer pass-2 finding on dual-namespace worst case):**

Derivation corrected:
- Peak Rush Phase density = 40 active Rigidbody2D × avg 2 emits/body/frame = 80 events worst case per namespace
- Single gameplay occurrences may fire on BOTH Feedback AND Gameplay namespaces (per R5 + the Interactions table — e.g., `audio.collect.coin.success` + `gameplay.level.banked` from a single coin)
- Dual-namespace worst case: 80 × 2 = 160 events/frame
- Safe ceiling: 256 events = 1.6× dual-namespace worst case, matching the pass-1 safety margin strategy (which used 1.6× for the single-namespace case of 80 but did not account for dual-namespace multiplication)
- Memory cost: 256 slots × ~16 bytes/slot (ChannelID + payload pointer + metadata) ≈ 4 KB — negligible on the 3 GB mobile budget

Pass-1 revision raised from 64 → 128 (single-namespace analysis); pass-3 corrects the derivation to account for dual-namespace fan-out and raises to 256. Overflow triggers `Debug.LogError` with the overflowing event ID in UNITY_EDITOR (per R4); in DEVELOPMENT_BUILD, a once-per-session `Debug.LogError` fires on the FIRST overflow only (per pass-3 FIFO-drop dev-visibility amendment — see Edge Cases) AND the release-build counter `deferred_queue_overflow_count` increments (per R8.b) for both DEV_BUILD and release; in release, the oldest queued event is dropped (FIFO-drop-oldest per CD adjudication of drop-policy disagreement pending telemetry data at v1.0).

### D.3 Subscriber List Initial Capacity (per channel)

```
subscriber_list_initial_capacity = 16 per channel
```

Pass-1 revision: raised from 8 → 16 per systems-designer finding #6. Realistic v1.0 per-channel subscriber counts: `gameplay.level.complete` = {Modal/Dialog, Audio Bus, Analytics, Save/Load write-trigger, debug-overlay} = 5; high-density Feedback channels = {Audio Bus, VFX/Juice, HUD, Haptics, Analytics (via gameplay pair), debug-overlay} = 6; future Tutorial overlay + Accessibility Service subscribers push maximum to 8 on a few channels. Initial capacity 16 eliminates the one-time realloc-and-copy at scene load (which is an allocation spike that would break AC-P1 steady-state measurement).

Implementation contract (resolves systems-designer finding #4 + qa-lead finding #5): each channel SO MUST construct its subscriber list as `new List<Action<T>>(16)` in the SO's `OnEnable` (not on first Subscribe call). AC-F3 asserts `Capacity == 16` **before** any subscriber registers (constructor-time pre-allocation, not post-add growth).

### D.4 Feedback Group Aggregate Latency + Onset-Spread (amended pass-3)

```
group_latency              = max(subscriber.completion_timestamp) - fire_timestamp
group_latency_max_T1       = 16.67ms   (all T1-group subscribers complete within same frame)
group_latency_max_T2       = 100ms     (all T2-group subscribers complete within Feedback Floor)

intra_group_onset_spread   = max(subscriber.handler_entry_timestamp) - min(subscriber.handler_entry_timestamp)
intra_group_onset_spread_max = 22ms    (perceptual synchrony threshold — Pillar 5 structural gate)
```

A **Feedback Group** is a named set of subscribers for one event whose Pillar 5 guarantee is measured by TWO independent metrics that BOTH must pass:

1. **group_latency** — aggregate completion budget. The last subscriber in the group finishes before the tier budget elapses (T1 / T2). Answers: "did the whole group respond inside the Feedback Floor?"
2. **intra_group_onset_spread** — perceptual synchrony budget. The first and last subscriber in the group START their handlers within 22ms of each other. Answers: "did the voices speak together, or did one lag the others perceptibly?"

**Rationale for onset-spread (pass-3 structural amendment — game-designer pass-2 finding #1).** `group_latency` alone can pass an 80ms audio-vs-visual desync that a human player perceives as broken — the group completes within 100ms aggregate, but a coin-splash VFX arrives 80ms before its audio chirp. Perceptual synchrony research places the audio-visual binding threshold at ~22ms for simultaneous events. `intra_group_onset_spread` measures the window between the first subscriber starting and the last subscriber starting — the perceptible jitter a player can feel. Both budgets must pass for Pillar 5 to hold.

Per-subscriber order remains undefined (R6). The onset-spread budget constrains the MAX delta between first and last subscriber onset, not the order in which they fire. Registered via `RegisterFeedbackGroup(ChannelID, string groupName)` for diagnostic purposes. Measured by AC-F4b across all documented groups (Input tap group, Collection success group, Bomb detonate group, Tool place group).

**Synchronous-dispatch note.** Under the current synchronous model, subscribers dispatch inline on the publisher's thread; onset-spread equals the sum of preceding subscribers' execution time. A 22ms onset-spread budget therefore implies each of N group subscribers completes in ≤ `22 / (N-1)` ms (e.g., 4-subscriber group = ≤ 7.3ms per subscriber). Enforced per consumer-system GDD via each subscriber's own latency budget.

**Analytics is NOT in Feedback Groups.** Analytics subscribers (R9 codegen path) are T3-tier — async-flush in PostLateUpdate, outside the synchronous dispatch envelope. They are NOT measured by `group_latency` or `intra_group_onset_spread` and are NEVER registered in a Feedback Group.

### D.5 Multi-hop Chain Latency with Hard 2-Hop Cap (pass-1; amended pass-3 with hop cap)

```
chain_latency_max = (hops_requiring_defer × frame_budget_ms) + Σ(subscriber_execution_ms)
hops_requiring_defer = count of re-entrant Fires that hit R2's deferred queue
frame_budget_ms = 16.67 (60fps) or 33.33 (30fps floor)

MAX_CHAIN_DEPTH = 2 (hard rule — enforced by AC-F5)
```

**Pass-3 structural constraint (systems-designer + game-designer + performance-analyst 3-specialist convergence on chain latency at 30fps):**

Any chain with depth ≥ 3 hops breaches 100ms at the 30fps floor with zero headroom (3 × 33.33ms = 99.99ms, before any subscriber execution time). `MAX_CHAIN_DEPTH = 2` is a **hard architectural rule**: if a prospective gameplay flow requires 3+ hops, it MUST be restructured before the first story is authored.

**Restructuring pattern — convert chain to fan-out:** instead of `A fires → B subscribes + fires → C subscribes` (2 hops), make both B and C subscribe to A directly. Sequential consequences that cannot fan out (B must complete before C has valid data) indicate a design smell — the coupling belongs inside a single subscriber, not across bus hops.

**Documented multi-hop chains (all MVP/VS chains enumerated here + validated in AC-F5 CI):**

| Chain | Depth | 30fps worst case | Within budget? |
|---|---|---|---|
| Collection → Currency & Economy → HUD | 2 hops | 2 × 33.33 + ~5ms subscriber = ~72ms | ✓ |
| Hazard → Currency & Economy → HUD | 2 hops | 2 × 33.33 + ~5ms subscriber = ~72ms | ✓ |

Any new chain proposal MUST be added to this table by the consumer system's GDD before AC-F5 passes. AC-F5 enumerates the declared chains from this table and rejects any that exceed 2 hops.

**3+ hop chains are a Pillar 5 blocker — not a playtest concern**, not a "known risk" with sign-off, not deferrable to VS. Restructure or block.

AC-F5 expanded in pass-1 + amended pass-3 to include the 2-hop cap enforcement. See §Acceptance Criteria.

## Edge Cases

### Subscription / Lifecycle

- **If a MonoBehaviour subscribed via `SubscribeAutoManaged` has its GameObject destroyed mid-dispatch**: `SubscriptionManager.OnDestroy` fires and calls `Unsubscribe` — but the bus is in `Dispatching` state. Resolution: `Unsubscribe` during dispatch marks the entry as "pending removal" (nulls the delegate in the list); dispatch loop skips null entries; pending removals compact after dispatch completes. No exception, no stale-reference crash.
- **If a MonoBehaviour subscribes in `OnEnable` but is disabled mid-dispatch (not destroyed)**: subscription persists (disable ≠ destroy). Handler still fires. If disable-suspended behavior is desired, subscriber must unsubscribe in `OnDisable` manually — not via auto-managed path. Documented in the Subscribe API docstring.
- **If a non-MonoBehaviour subscriber's `SubscriptionHandle` is GC'd without `Dispose()`**: in debug builds, the bus logs `[EventBus] Leaked subscription for channel {id}` with the stack trace captured at Subscribe time (stored in a debug-only dictionary). In release builds, the subscription persists until scene unload — manageable memory waste, not a crash.
- **If `Subscribe` is called during `Dispatching` state for the same channel being dispatched**: new subscriber is registered but does NOT receive the current in-flight event. Takes effect next dispatch cycle. Prevents the race where a subscriber created by another subscriber's handler receives a partially-processed payload.

### Dispatch Safety

- **If a subscriber handler throws an unhandled exception**: dispatch loop catches, logs `[EventBus] Subscriber exception on channel {id}: {ex}` with subscriber type (**Editor-only per R4** — stripped in Development and Release builds to preserve R1 zero-alloc), and continues to the next subscriber. The bus never throws out of `Fire`. Prevents one buggy subscriber from breaking all subscribers on the same channel. In non-Editor builds, subscriber exceptions are silently swallowed at the bus boundary; subscribers are responsible for their own try/catch if they need non-Editor error handling.
- **If an event is fired before the bus has initialized (Uninitialized state)**: `Fire<T>` writes the payload to that channel's **per-channel bootstrap ring buffer** (capacity 4 per channel per R8). Pass-1 correction: prior revision said "throws `EventBusNotReadyException` immediately (fail-fast per R5)" — this was a stale reference that predated the CD-GDD-ALIGN R8 resolution, AND the "per R5" cross-reference was incorrect (R5 governs namespace distinctness, not init state). Fire during Uninitialized is now safe and buffered; throw semantics apply only to `Fire` on a **Disposed** bus (post-scene-unload).
- **If the deferred queue reaches capacity during re-entrant dispatch (256 events queued per D.2)**: in UNITY_EDITOR, `Debug.LogError` with queue contents (every overflow). In **DEVELOPMENT_BUILD** (pass-3 addition per game-designer pass-2 finding on playtest visibility), a **once-per-session** `Debug.LogError` fires on the FIRST overflow of the session only — subsequent overflows increment the counter silently. The "once-per-session" gate is a single `private static bool _overflowLoggedThisSession` flag checked before the log call; the flag is never reset mid-session; resets to `false` only on domain reload / fresh session. This preserves R1 zero-allocation steady-state (only the first overflow allocates a log string; every subsequent overflow is counter-only). In release builds, no log fires — R8.b counter increments only. Playtesters with DEVELOPMENT_BUILD builds therefore see a single actionable error on the first queue overflow, surfacing Pillar 5 silent violations without flooding logs.
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
| `deferred_queue_capacity` | **256 events (raised pass-3 for dual-namespace worst case)** | [128, 512] | More re-entrant fires survive; more preallocated memory (~16B per slot × 256 = 4KB) | Smaller memory footprint; overflow risk during Rush Phase peak dual-namespace fan-out; floor 128 = 80% of derived dual-namespace worst case 160 (margin only for subscriber misbehavior) |
| `subscriber_list_initial_capacity` | 16 per channel (raised pass-1) | [8, 32] | Fewer reallocs on high-subscriber channels; more unused memory on low-subscriber channels | More reallocs (one-time cost at scene load); breaks AC-R1 steady-state if alloc occurs post-scene-load |
| `bootstrap_buffer_capacity_per_channel` | 4 events | [2, 16] | More resilient to Awake-time event floods | Scene-load bootstrap overflow risk. **LOCKED to R8.a per-tick drain cap (both = 4)** — if this knob changes, R8.a hardcoded invariant must move in lockstep via coordinated ADR. |
| T1 latency budget | 16.67ms (fixed wall-clock) | [8ms, 33ms] | More slack for hot-path subscribers; risk of missing 60fps | Tighter frame budget; harder to hit with complex subscribers |
| T2 latency budget | 100ms | [50ms, 200ms] | More slack before Feedback Floor breach; perceptible lag at high end | Tighter budget; may force subscriber optimization |
| `intra_group_onset_spread_max` (**new pass-3**) | 22ms | [16ms, 33ms] | More desync tolerated within Feedback Groups; risk of perceptible AV jitter | Tighter synchrony; may force faster subscribers; 16ms = 1 frame at 60fps (aggressive) |
| Recursive-fire warning threshold | 10 frames | [5, 30] | Quieter logs; real runaway fires take longer to flag | Noisier Editor logs; catches runaway fires faster |
| Subscription leak check interval | 2 scene loads | [1, 5] | More forgiving of leaked subscriptions | Catches leaks faster; noisier on legitimate long-lived subs |

**Removed from tuning knobs in pass-3 (per CD pass-2 adjudication — hardcoded invariants, not knobs):**
- **R8.a per-tick drain cap** (value = 4): removed because it is the spike-prevention contract itself; making it a knob lets callers set it to `int.MaxValue` and defeat the entire R8.a purpose.

### Interaction Between Tuning Knobs

- `deferred_queue_capacity` and `recursive_fire_warning_threshold` interact: too-small capacity + too-high threshold = silent drops before detection; too-large capacity + too-low threshold = false-positive warnings. Adjust together.
- `T1 latency budget` and `T2 latency budget` must maintain T1 < T2 with a meaningful gap (at minimum 4× ratio) to distinguish frame-critical from feedback-floor events.
- `bootstrap_buffer_capacity_per_channel` (knob) and R8.a per-tick drain cap (hardcoded invariant, value 4) are locked together. Changing the knob alone without a matching R8.a ADR defeats the spike-prevention contract. The knob's safe range [2, 16] assumes R8.a remains proportional.
- `intra_group_onset_spread_max` (D.4) and T1/T2 latency budgets interact: the onset-spread is a WITHIN-group metric, while T1/T2 are end-to-end metrics. Tightening onset-spread below 22ms forces faster per-subscriber budgets via `onset_spread / (N-1)` math — a Feedback Group of 4 subscribers with 16ms spread caps each subscriber at ~5.3ms execution.

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

**Pass-3 recount**: **36 criteria across 6 categories** (up from pass-1's 32 claim → actual body was 30 after mergers; pass-3 net +6 from: AC-BOOT-A/AC-BOOT-B split (+1), AC-F4b elevated to BLOCKING, AC-F5 elevated to BLOCKING, AC-REL elevated to BLOCKING, AC-P1 merged into AC-R1 (−1 offset by the split above); AC-F1/AC-EC3/AC-STR1/AC-STR2 remain merged from pass-1). Gate totals: **BLOCKING CI** (16) / **PLAYMODE non-CI-blocking** (16) / **ADVISORY** (2) / **MANUAL** (2 — AC-F6b and AC-F6c require Mali-G52 device capture). All counts verified by individual AC enumeration in the Gate Summary table below; the pass-2 body-vs-claim-count mismatch is resolved.

### H.1 Rule Compliance (R1-R7)

- **AC-R1 — Zero Hot-Path Allocation (merged from pass-1 AC-R1 + AC-P1 per CD pass-2 adjudication)** — GIVEN the Rush Phase stress scene (40 Rigidbody2D, 2 fires/frame each, all 13 MVP emitter/subscriber pairs wired), WHEN a PlayMode test runs the combined allocation protocol below, THEN every frame in the steady-state window reports 0 bytes allocated by the bus hot path; the test logs a per-frame `eventbus-alloc-report.json` artifact and exits 1 if any frame shows > 0 bytes.

  **Test protocol (pass-3 rewritten per qa-lead + performance-analyst pass-2 convergent findings):**

  1. **Scene-load phase (0–3 s):** `SceneManager.LoadScene` completes; subscribers complete registration; bus transitions to Ready. Allocations in this phase are out-of-scope.
  2. **Cold-fire JIT-warmup sweep (3–5 s):** test fires each of the N MVP channels EXACTLY ONCE in a scripted sequence to force JIT compilation of every Fire→dispatch code path (addresses performance-analyst pass-2 finding: rarely-fired channels' JIT compilation happens on first fire and allocates). `GC.Collect(2, GCCollectionMode.Forced, blocking: true)` executes once at the end of this phase.
  3. **Steady-state window (5–20 s):** test fires all 13 pairs at documented rates (900 frames at 60fps). Allocation sampling uses TWO independent signals, both must pass:
     - **Primary: per-Fire delta via `GC.GetAllocatedBytesForCurrentThread()`** — test wraps each `EventBus.Fire<T>(...)` call in a harness that reads the thread-local allocation counter before and after; asserts delta == 0. This is call-stack-scoped in the only sense that matters (bus fires on main thread only; no other test code allocates during the call because the test harness is isolated). Resolves qa-lead pass-2 finding: "ProfilerRecorder cannot scope GC.Alloc to EventBus.Fire<T> call-stack" — correct, so we use a different API that IS thread-scoped.
     - **Secondary: `ProfilerRecorder` sampling `"GC.Alloc"` global marker at frame-granularity** — asserts per-frame `sample.Value == 0` across 900 frames. This catches allocation in any code path reached via bus dispatch (subscriber-code allocations that Fire triggers) which the per-call delta would miss if the allocation happens in a subscriber called by Fire.
  4. **Test fails immediately** on any frame (secondary) or any Fire call (primary) showing > 0 bytes; both signals are reported in the artifact.

  Pass-3 notes: (a) replaces prior "ProfilerRecorder scoped to EventBus.Fire<T> call-stack" which is not an API that exists; (b) merged AC-P1 into AC-R1 per CD pass-2 adjudication — they tested the same invariant; (c) 15-second steady-state window (up from 10s) catches rare-channel cold-fires; (d) cold-fire JIT sweep prevents false passes on channels that fire once in the window and allocate on first JIT. **BLOCKING CI (PLAYMODE)**
- **AC-R2 — No Synchronous Cross-Frame Recursion** — GIVEN a subscriber that calls `Fire` during its handler, WHEN the inner `Fire` executes, THEN the event is placed in the deferred ring buffer; depth guard > 0; inner event dispatches on next `EventBusPlayerLoopSystem.Update` tick. **BLOCKING CI (PLAYMODE)**
- **AC-R2b — Depth-Guard Reset Across Frames (deterministic probe, not sleep-based)** (pass-1; pass-3 replaced tautological `Thread.Sleep(50)` per qa-lead pass-2 finding) — GIVEN the bus has dispatched re-entrant events in frame N (depth-guard counter > 0 at some point during frame N), WHEN frame N+1 begins (next `EventBusPlayerLoopSystem.Update` tick), THEN the very first `Fire` call in frame N+1 dispatches synchronously (`DepthGuard_TestOnly` reads 0 at entry, not deferred); verified by asserting the new event's subscriber executes inline before `Fire` returns.

  **Deterministic reset probe (pass-3):** the test uses a new `internal int DepthGuard_TestOnly => _depthGuard;` accessor on the bus to READ the field at frame N+1 entry and assert it equals 0. This is a direct semantic check that does not depend on wall-clock time passing. Separately, a second unit test simulates a long coroutine yield (`yield return new WaitForSeconds(0.5f)`) between frames N and N+1 to assert the reset is coroutine-tolerant — this is a real behavioral test (does coroutine suspend affect the depth field?), not a sleep-for-sleeping's sake. The prior `Thread.Sleep(50)` probe was tautological because sleeping does not touch the depth-guard field; it tested nothing. **BLOCKING CI (PLAYMODE)**
- **AC-R3 — No String-Typed Channels at Call Sites (regex + interpolated-string coverage)** (pass-1; amended pass-3 per qa-lead pass-2 finding that `$"..."` interpolation was uncovered) — GIVEN the full `src/` tree, WHEN the CI step runs `grep -rPn '\.(?:Subscribe|Fire)\s*<[^>]+>\s*\(\s*\$?"' src/` (the `\$?` match class covers both `"..."` literal strings AND `$"..."` interpolated strings; `-P` flag provides PCRE + CRLF handling), THEN exit code is 0 (no matches). Any match fails CI with exit 1 and logs the matching file+line with the detected kind (literal vs interpolated). The grep command is committed to `.github/workflows/eventbus-lint.yml` as the authoritative check; a matching test fixture at `tests/editmode/event-bus/fixtures/string-literal-violations/` contains one literal-string violation + one interpolated-string violation that the CI job MUST flag to self-verify the regex. Pass-1 fixed regex defects (greedy `.*`, unclosed capture group); pass-3 added `\$?` for interpolated coverage + self-verifying fixture. **BLOCKING CI**
- **AC-R4 — No Silent Swallowing in Editor Builds** — GIVEN a `UNITY_EDITOR` build (not Development or Release per pass-1 R4 tightening), WHEN `Fire` is called on a channel with zero subscribers, THEN `Debug.LogWarning` fires with the channel SO as context within the frame; Editor Window unhandled-event count increments. Separately, the release-build counter (R4.a) increments in ALL build configurations. **PLAYMODE**
- **AC-R5 — Namespaces Are Distinct** — GIVEN a Gameplay channel ID, WHEN `Subscribe` is called with a Feedback context (or vice versa) in Editor builds, THEN `EventBusNamespaceMismatch` throws before registration; no subscription is created. **PLAYMODE**
- **AC-R6 — All Subscribers Receive Event Regardless of Registration Order** — (split pass-1 per qa-lead finding #3) GIVEN three subscribers A, B, C registered in that order, WHEN the channel dispatches 100 times across 10 fresh scene loads, THEN each subscriber receives all 100 events; observed per-dispatch order is logged to a CI artifact. The "no test may assert a specific order" clause is moved out of AC scope into `tests/helpers/event-bus-test-guidelines.md` (test-author guideline, not a bus AC). **PLAYMODE** (was ADVISORY)
- **AC-R7 — CI Flags IEventPayload LINQ Misuse (named filter script)** (pass-1; pass-3 named the committed script per qa-lead pass-2 finding) — GIVEN any `.cs` file with LINQ (`OrderBy`, `Where`, `Select`, `ToList`, `ToArray`, `GroupBy`) on an `IEventPayload`-implementing struct, WHEN the CI static-analysis job runs the two-step pipeline (`grep -rPn '\.(?:OrderBy|Where|Select|ToList|ToArray|GroupBy)\s*\(' src/` piped into `python3 tools/ci/eventbus-linq-on-payload-filter.py`), THEN the job emits a rule-violation error on matches and exits 1.

  **Committed filter script spec (pass-3):** `tools/ci/eventbus-linq-on-payload-filter.py` reads grep output on stdin; for each matched line, resolves the variable on the left of the LINQ call via a best-effort type inference using a lightweight Roslyn-lite AST walk (Python scanner over `src/` to build a symbol table of variables declared as `IEventPayload`-implementing types); emits a diagnostic for matches where the LHS is an `IEventPayload` struct or collection thereof. The script is committed to the repo with its own EditMode unit tests at `tests/editmode/ci-scripts/test-eventbus-linq-filter.py` covering: (a) true-positive LINQ on an IEventPayload struct, (b) true-negative LINQ on an unrelated struct, (c) edge case LINQ inside a subscriber lambda closure. False positives are expected (scope-narrowed design per R7); manual code review is the secondary safety net. **BLOCKING CI**
- **AC-R7a — CI Flags String Fields on IEventPayload (multi-line scanner)** (pass-1; pass-3 specified the multi-line scanning mechanism per qa-lead pass-2 finding) — GIVEN any `IEventPayload`-implementing struct, WHEN the CI static-analysis job runs `python3 tools/ci/eventbus-string-field-scanner.py src/` (a committed Python script that parses each `.cs` file with a lightweight struct/class body scanner; identifies `readonly struct X : [...] IEventPayload [...]` declarations spanning multiple lines; reports any `public readonly string <name>` or `public string <name>` field inside such a struct), THEN zero matches; any match fails CI with exit 1. Multi-line grep is intentionally NOT used — grep's line-oriented design cannot reliably match a struct declaration spanning `public readonly struct Foo :\n    IEventPayload,\n    IEquatable<Foo>\n{\n    public readonly string Bar;\n}` across 5 lines without false positives. The committed Python scanner is authoritative. Scanner has unit tests at `tests/editmode/ci-scripts/test-eventbus-string-field-scanner.py` with fixtures covering single-line, multi-line-decl, and generic-constraint cases. Prevents regression of the R7.a rule. **BLOCKING CI**

### H.2 Formula / Constant Correctness

- **AC-F1 — MERGED INTO AC-R1** — pass-1 revision merged AC-F1 into AC-R1 per qa-lead finding #4. The prior AC-F1 "60 seconds at 60fps" and AC-R1 "300 frames" tested the same invariant. Merged spec uses the 300-frame + 5-sec-steady-state window from AC-R1.
- **AC-F2 — Deferred Queue Capacity = 128; Overflow Drops Oldest (FIFO)** — GIVEN a test fixture where each event payload carries a monotonic `SequenceId` (0–128), AND a subscriber fires 129 re-entrant events in one frame (SequenceId 0 through 128), WHEN the next `EventBusPlayerLoopSystem.Update` tick drains the queue, THEN the received-event log contains exactly SequenceIds 1–128 in FIFO order; SequenceId 0 is absent; queue length never exceeded 128 at any drain point; R8.b release-build counter `deferred_queue_overflow_count` incremented exactly once. Pass-1 revision: capacity 64→128, reconciled with AC-EC3 (both now describe same contract), added SequenceId fixture for unambiguous verification. **BLOCKING CI (PLAYMODE)** (promoted from PLAYMODE-only per qa-lead finding #6)
- **AC-F3 — Subscriber List Initial Capacity = 16 (Constructor-Time, test-only accessor)** (pass-1; pass-3 removed reflection path per qa-lead pass-2 finding on BCL fragility) — GIVEN a channel SO loaded in a fresh scene, WHEN no subscribers have registered yet, THEN `channel.SubscriberCapacity_TestOnly == 16`; AND after 16 subscriptions, `channel.SubscriberBackingArrayRef_TestOnly` is reference-equal to its pre-add value (no reallocation); AND after the 17th subscription `channel.SubscriberCapacity_TestOnly >= 32` (one reallocation occurred; the backing-array ref changed).

  **Test-only accessor contract (pass-3):** `EventChannelSO<T>` exposes two `internal` members used only by tests: (a) `internal int SubscriberCapacity_TestOnly => _subscribers.Capacity;` and (b) `internal object SubscriberBackingArrayRef_TestOnly => <fieldref of List's backing array via a stable interface, not reflection>`. To avoid reflection into `List<T>._items` (a private BCL field that has been renamed across .NET versions and is fragile per qa-lead finding), the channel SO wraps its subscriber collection in a custom internal type `ChannelSubscriberList<T>` that exposes `BackingArrayRef` as a public-internal property. The test assembly has `[InternalsVisibleTo("EventBus.Tests")]` declared on the Event Bus assembly. Pass-1 revision: capacity 8→16 + reframed from post-add to constructor-time pre-allocation; pass-3: removed reflection fragility via explicit test-only accessors on an owned wrapper type. **BLOCKING CI (PLAYMODE)**
- **AC-F4 — T1 Latency Budget (P95 ≤ 16.67ms, P99 ≤ 33ms on CI hardware)** — GIVEN all T1-tier pairs (Input→VFX/Juice, Input→HUD, Collection→Audio, Hazard→VFX) in a PlayMode test on the CI runner (specify: GitHub Actions `ubuntu-latest` or `macos-latest` with Unity 6.3 LTS headless), WHEN each pair fires 100 times with `System.Diagnostics.Stopwatch` wrapping `Fire<T>` call through all subscriber returns, THEN P95 round-trip ≤ 16.67ms AND P99 ≤ 33ms (one-frame tolerance at 30fps floor); any sample exceeding 33ms is logged as a warning; test fails only if P95 is breached. Mali-G52 device validation deferred to AC-F6b/c (MANUAL at VS milestone). Pass-1 revision: replaced "no outliers" (unverifiable) with P95/P99 definitions + hardware specification per qa-lead finding #7. **PLAYMODE**
- **AC-F4b — Feedback Group Latency + Onset-Spread (both must pass)** (pass-1; amended pass-3) — GIVEN a registered Feedback Group for `input.tap.main` = {VFX/Juice, Audio Bus, HUD, Haptics}, WHEN the event fires 100 times in a PlayMode test, THEN BOTH of the following hold across all samples (P95):
  - **(1) group_latency**: `max(subscriber.completion_timestamp) - fire_timestamp` ≤ 16.67ms for T1 groups / ≤ 100ms for T2 groups
  - **(2) intra_group_onset_spread**: `max(subscriber.handler_entry_timestamp) - min(subscriber.handler_entry_timestamp)` ≤ **22ms** for ALL tier groups (T1 and T2 share the same perceptual synchrony budget)

  Measurement uses per-subscriber timestamps captured via `System.Diagnostics.Stopwatch.GetTimestamp()` (NOT `Time.realtimeSinceStartup` — see AC-F6a rationale for precision) logged inside the bus's dispatch loop; timestamps are read at handler entry (onset) and handler return (completion). Per-subscriber order remains undefined (R6); onset-spread constrains the window between first and last onset only.

  The AC fails if EITHER metric breaches. A passing `group_latency` + breaching `intra_group_onset_spread` means "voices complete within budget but audibly desync" — a Pillar 5 violation the pass-1 metric could not catch. Pass-3 amendment resolves game-designer pass-2 finding #1 (AV desync can hide inside aggregate-completion budget). **BLOCKING CI (PLAYMODE)**
- **AC-F5 — T2 Latency Budget + 2-Hop Chain Cap** (pass-1; amended pass-3 with hard hop cap) — GIVEN all T2-tier pairs (Tool→Level Runtime, Critter AI→HUD, Gopher Spawn→Collection, Level Runtime→HUD) AND the documented multi-hop chains from D.5's chain table (Collection→Currency→HUD, Hazard→Currency→HUD), WHEN events fire in a PlayMode test on CI hardware with the frame budget simulated at 30fps (33.33ms/frame via `Time.captureFramerate = 30`), THEN:
  - **(1) single-hop T2 round-trip P95 ≤ 100ms**
  - **(2) multi-hop chain_latency (per D.5 formula) ≤ 100ms at 30fps** for every chain in D.5's enumeration
  - **(3) chain depth enforcement (BLOCKING CI)**: a separate EditMode static analysis pass scans `src/` for `Fire<...>(` calls made from inside `Subscribe<...>` handler bodies. Each detected re-entrant-fire path is compared against the D.5 chain table; any chain exceeding `MAX_CHAIN_DEPTH = 2` fails CI with exit 1 regardless of latency numbers.

  Any chain breaching (1) or (2) at 30fps fails; the response is restructure to fan-out (see D.5). **"Pillar 5 known risk with playtest sign-off" is no longer an accepted outcome** — pass-3 removed that escape hatch because the 30fps 3-hop worst case has zero headroom. Pass-1 added multi-hop clause; pass-3 added hard depth cap per D.5 amendment. **BLOCKING CI (PLAYMODE)**

- **AC-F6a — Automated Bus-Internal Latency (Stopwatch high-res ticks)** (pass-1 split from prior AC-F6; pass-3 replaced `Time.realtimeSinceStartup` per unity-specialist pass-2 finding on float LSB precision 0.12–0.61ms at large values) — GIVEN `input.tap.main` fires with a `System.Diagnostics.Stopwatch.GetTimestamp()` long tick value injected into `TapPayload.TimestampTicks`, WHEN all registered subscribers complete their handlers, THEN `(Stopwatch.GetTimestamp() - payload.TimestampTicks) * 1000.0 / Stopwatch.Frequency` (converted to milliseconds; the `long` ticks and floating-point multiply have sub-microsecond precision throughout) is ≤ 2ms on CI hardware across 100 samples (P95); proves the bus itself is not the bottleneck. `Time.realtimeSinceStartup` is explicitly REJECTED for this AC — a float storing seconds-since-domain-load exhibits 0.12ms LSB precision at 1 hour of Editor runtime and 0.61ms at 5 hours, either of which would falsely pass or fail a 2ms AC. Stopwatch.GetTimestamp returns Int64 platform ticks (typically 100ns precision on Windows, higher on modern Linux). **BLOCKING CI (PLAYMODE)**
- **AC-F6b — Manual Device Visual Feedback Floor (Mali-G52, P95@N=20)** (pass-1; pass-3 corrected percentile per CD pass-2 adjudication — P99 on N=20 is statistically P100/max-only and invalid) — GIVEN a dev build on a minimum-spec Android device with Mali-G52 GPU, a human tester records gameplay at 240fps (via HDMI capture card or high-speed camera), WHEN the tester marks the frame of tap-on-playfield and the first frame showing the coin-splash VFX, THEN `(frame_delta × 1000/240)` ≤ 100ms at **P95 across N=20 test taps** (pass-3 corrected from P99; the 19th-of-20 sample IS P95, which is the highest statistically meaningful percentile at this sample size; P99 at N=20 is mathematically undefined by interpolation and defaults to the max sample, making the AC a max-sample gate, not a percentile gate). Raw sample set also reported for transparency. Evidence filed in `production/qa/evidence/pillar5-visual-feedback-floor-[date].md` with lead sign-off. **MANUAL (VS milestone)**
- **AC-F6c — Manual Device Audio Feedback Floor (Mali-G52, P95@N=20)** (pass-1; pass-3 corrected percentile per CD pass-2 adjudication) — GIVEN the same dev build on Mali-G52 Android device with speaker output captured via external microphone or line-in, WHEN the tester marks the tap timestamp and the audio cue onset peak at the speaker (waveform analysis), THEN delta ≤ 100ms at **P95 across N=20 test taps** (pass-3 corrected from P99 — see AC-F6b rationale). Captures Android audio-pipeline latency (AudioTrack buffering + OS scheduler) that dispatch-only measurements cannot see. Evidence filed in `production/qa/evidence/pillar5-audio-feedback-floor-[date].md` with audio-director + lead sign-off. **MANUAL (VS milestone)**

- **AC-BOOT — Bootstrap First-Frame Drain Budget (split: bus overhead vs total)** (pass-1; pass-3 split into AC-BOOT-A / AC-BOOT-B per performance-analyst pass-2 finding that 8ms budget excluded realistic subscriber cost 120 × 6 × 50μs = 36ms):

  - **AC-BOOT-A (bus overhead only, BLOCKING CI)** — GIVEN the scene-load fixture where all ~30 MVP channels each receive 4 buffered events during Uninitialized state (120 events total), AND all subscribers in this fixture are near-zero-cost stubs (each handler does `counter++` and returns), WHEN the first `EventBusPlayerLoopSystem.Update` after Ready executes through to full drain completion, THEN the aggregate drain duration measured from first-tick-entry to last-tick-complete is ≤ **8ms on CI hardware** (measures bus dispatch overhead alone: ring-buffer reads, subscriber list iteration, stub handler invocation, per-tick slice cap enforcement). AND no single `Update` tick exceeds 4 events per channel (R8.a cap respected). AND no `[EventBus] Bootstrap overflow` appears in a clean scene load. **BLOCKING CI (PLAYMODE)**

  - **AC-BOOT-B (total-with-realistic-subscribers, ADVISORY)** — GIVEN the same scene-load fixture but with realistic MVP subscribers wired in (6 subscribers/channel avg per D.3 typical, each 50μs average per performance-analyst finding #4), WHEN the drain completes, THEN aggregate drain duration ≤ **40ms on CI hardware** (8ms bus overhead + 120 × 6 × ~50μs subscriber cost = ~44ms worst case; budget 40ms; allows 10% headroom-cut from worst). Reported as a smoke-test timing in `production/qa/smoke-[date].md`; does not block CI because subscriber cost is owned by consumer system GDDs, not the bus. **ADVISORY**

  Pass-3 rationale: AC-BOOT as a single AC conflated bus architectural budget (BLOCKING — bus is the owner) with integration budget (ADVISORY — subscribers are owned elsewhere). Splitting makes each AC independently testable and correctly attributes the gate to the owning system.

### H.3 Edge Case Handling

- **AC-EC1 — Mid-Dispatch Unsubscribe** — GIVEN subscriber B unsubscribes itself from within its handler during dispatch to A, B, C, WHEN the event dispatches, THEN no exception thrown; A + C receive the event; B marked pending-removal; list compacts after dispatch. **BLOCKING CI (PLAYMODE)**
- **AC-EC2 — Subscriber Exception Does Not Break Dispatch** — GIVEN subscriber A throws in its handler and subscriber B is registered after A, WHEN the channel fires in a UNITY_EDITOR build, THEN B executes; `[EventBus] Subscriber exception` logged (Editor only per R4); `Fire` does not throw. Separately in Release/Development builds, B still executes and Fire still does not throw, but no log emits (per R4 tightening). **BLOCKING CI (PLAYMODE)**
- **AC-EC3 — MERGED INTO AC-F2 (pass-1)** — the prior AC-EC3 described the same invariant as AC-F2 (queue overflow drops oldest). Unified into AC-F2 with explicit SequenceId fixture per qa-lead finding #6.
- **AC-EC4 — Namespace Mismatch Assertion** — GIVEN channel `gameplay.level.phase_changed`, WHEN Editor-build code calls Subscribe using Feedback path, THEN `EventBusNamespaceMismatch` throws before handler registration; assertion message includes channel ID and conflicting namespace names. **PLAYMODE**
- **AC-EC5 — Recursive Fire Warning (Editor only)** — GIVEN a subscriber on channel X that unconditionally re-fires X every time, WHEN the deferred queue re-dispatches for 10 consecutive frames in a UNITY_EDITOR build, THEN `[EventBus] Probable recursive fire on {channel}` logs ONCE per run (not per frame — no log flood). Release-build counter R8.b increments `recursive_fire_count` instead of logging. **PLAYMODE**
- **AC-EC6 — Subscribe During Dispatch: New Subscriber Excluded from In-Flight** — GIVEN subscriber A subscribes a new subscriber B during its handler, WHEN the event dispatches, THEN B does NOT receive the current event; B receives all subsequent dispatches; confirmed via capture list. **PLAYMODE**
- **AC-EC7 — Mid-Dispatch Scene Unload (Dispatching → Disposed)** (NEW pass-1) — GIVEN the bus is in `Dispatching` state with queued deferred events AND `SceneManager.LoadScene` is triggered, WHEN the currently-dispatching `Fire` completes, THEN the bus transitions to Disposed only after the in-flight chain returns (no preemption mid-subscriber); queued deferred events are discarded; R8.b `scene_unload_discard_count` increments by the discarded count; no exception thrown. **PLAYMODE**

### H.4 Performance

- **AC-P1 — MERGED INTO AC-R1 (pass-3)** — the prior AC-P1 and AC-R1 tested the same zero-allocation invariant; CD pass-2 adjudicated merge-over-annotate. See AC-R1 for the unified protocol (cold-fire JIT sweep + per-Fire `GC.GetAllocatedBytesForCurrentThread()` delta + `ProfilerRecorder` frame-granularity secondary).
- **AC-P2 — Memory Footprint per Channel (Order-of-Magnitude, math clarified)** (pass-1 revised 8→24 KB; pass-3 rederived and clarified slot-vs-delegate math per performance-analyst pass-2 finding on delegate-size-vs-slot-size confusion) — GIVEN 30 `EventChannelSO<T>` assets loaded at scene start, WHEN profiled via `UnityEngine.Profiling.Profiler.GetRuntimeMemorySizeLong(channelSO)` summed across all 30 instances in a PlayMode test, THEN total ≤ **30 KB** (revised ceiling per pass-3 rederivation below; prior 24 KB was close but the math was misstated).

**Pass-3 corrected derivation** (replaces the ambiguous "16-slot delegate list at 40 bytes/delegate" phrase):

Per channel SO, at typical MVP subscriber load:
| Component | Size | Notes |
|---|---|---|
| `List<Action<T>>` backing array | 16 slots × 8 bytes = **128 bytes** | Each slot is a managed reference (8 bytes on 64-bit IL2CPP); Capacity = 16 per D.3 |
| Subscribed `Action<T>` delegate instances | 6 avg × ~64 bytes = **384 bytes** | Each delegate is a separate heap allocation containing target-object reference, method pointer, and MulticastDelegate fields; 6 is the typical MVP subscriber count per channel |
| Bootstrap ring buffer (typed `T[4]`) | 4 × sizeof(T) ≈ **48–128 bytes** | Depends on payload size; worst case is `BombDetonatedPayload` at ~32 bytes → 128 bytes |
| SO overhead + `ChannelSubscriberList<T>` wrapper | **~200 bytes** | Unity object header + SO fields + wrapper type fields |
| **Per-channel total** | **~800–850 bytes** | |
| **30 channels** | **~24–25.5 KB** | |

AC ceiling: **30 KB** (7.5× margin over per-channel midpoint; absorbs payload-size variation across channel types, alignment padding, and future subscriber growth up to the 16-slot capacity). Reclassified as order-of-magnitude check — the AC exists to catch runaway memory (a channel accidentally allocating a 1MB buffer), not to gate nanosecond-level profiling. Result logged to `production/qa/smoke-[date].md`. **ADVISORY**

Prior "16-slot delegate list at 40 bytes/delegate" phrasing conflated two distinct allocations: (a) the 16-slot backing array (8 bytes per slot = 128 bytes total) and (b) the delegate objects themselves (~64 bytes each, separately heap-allocated per subscription, not sized into the list). Corrected table above separates these.
- **AC-P3 — T1 Latency Under Peak Load (Subscriber-Discipline Acknowledged)** (REVISED pass-1 per performance-analyst finding #3) — GIVEN Rush Phase peak (40 Rigidbody2D, max emitter activity), WHEN T1 and T2 events fire concurrently in the same frame, THEN T1 round-trip P95 ≤ 16.67 ms AND T2 round-trip P95 ≤ 100 ms, **provided individual T2 subscriber handlers each complete in ≤ 5ms** (documented dependency — synchronous dispatch cannot enforce T1 priority over a slow T2 subscriber; this AC now explicitly names the dependency). Subscriber-level latency is a separate budget enforced per consumer system GDD. **PLAYMODE**
- **AC-REL — Release-Build Telemetry Counters (with named injection fixtures)** (pass-1; pass-3 specified fixture paths + contents per qa-lead pass-2 finding that overflow/exception injection was undefined) — GIVEN a release build with Analytics v1.0 enabled, WHEN a session runs for 10 minutes executing the scripted telemetry-injection fixture at `tests/playmode/event-bus/fixtures/release-telemetry/`, THEN the bus's release counters (`zero_subscriber_fire_count`, `deferred_queue_overflow_count`, `bootstrap_overflow_count`, `scene_unload_discard_count`, `recursive_fire_count`) increment monotonically and flush to Analytics at session-end via the R9 codegen path. Verified by reading the Analytics service's received event log.

  **Fixture contents (pass-3 pinned):** three required fixture C# files, each containing a named injection method callable from the test harness:
  - `ZeroSubscriberFireInjector.cs` — `InjectZeroSubscriberFire(ChannelID channel, int count)` fires `count` events on a channel with no registered subscribers; asserts `zero_subscriber_fire_count` delta == count.
  - `QueueOverflowInjector.cs` — `InjectQueueOverflow(int depth)` registers a subscriber whose handler re-fires on its own channel `depth` times in one frame; asserts `deferred_queue_overflow_count` delta == max(0, depth − deferred_queue_capacity).
  - `RecursiveFireInjector.cs` — `InjectRecursiveFire(int frames)` registers a subscriber that unconditionally re-fires on its own channel for `frames` consecutive frames; asserts `recursive_fire_count` delta == 1 (single-warn-per-run contract per AC-EC5).

  Analytics sink: the fixture uses a stubbed `IAnalyticsService` (`tests/playmode/event-bus/fixtures/release-telemetry/MockAnalyticsService.cs`) that records event batches to an in-memory list readable from the test assertion. Fixture must compile and pass under UNITY_EDITOR, DEVELOPMENT_BUILD, and release-IL2CPP pipelines.

  **PLAYMODE (BLOCKING CI at VS milestone when Analytics stub available)** — pass-3 elevated from PLAYMODE/non-blocking to BLOCKING once the fixture exists; telemetry counters are the only observability into silent Pillar 5 failures in release.

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

### Gate Summary (recounted pass-3 — body-verified)

| Gate | Count | ACs |
|---|---|---|
| **BLOCKING CI** | 16 | AC-R1 (merged from R1+P1), AC-R2, AC-R2b, AC-R3, AC-R7, AC-R7a, AC-F2, AC-F3, AC-F4b (elevated pass-3), AC-F5 (elevated pass-3), AC-F6a, AC-BOOT-A (split pass-3), AC-EC1, AC-EC2, AC-REL (elevated pass-3), AC-STR3 |
| **PLAYMODE (non-CI-blocking)** | 16 | AC-R4, AC-R5, AC-R6, AC-F4, AC-EC4, AC-EC5, AC-EC6, AC-EC7, AC-P3, AC-INT1, AC-INT2, AC-INT3, AC-INT4, AC-INT5, AC-INT6, AC-INT7 |
| **ADVISORY** | 2 | AC-BOOT-B (split pass-3), AC-P2 |
| **MANUAL (device capture at VS)** | 2 | AC-F6b (visual at Mali-G52, P95@N=20), AC-F6c (audio at Mali-G52, P95@N=20) |

**Total: 36 AC identifiers** (16 BLOCKING CI + 16 PLAYMODE + 2 ADVISORY + 2 MANUAL = 36). Merged-but-retained-as-marker IDs (AC-F1, AC-P1, AC-EC3, AC-STR1, AC-STR2) are NOT counted — they remain in the document as pointers only.

**Pass-3 body-consistency check:** each AC in the list above is individually cross-referenced to its definition in §H.1–§H.6. The pass-2 finding that the body count (20) did not match the claim (14) is resolved — pass-3 verifies by enumeration, not by section-level summation.

**Shift-left note**: the 16 BLOCKING CI criteria need stub EditMode/PlayMode test scaffolding at story start, before bus implementation begins. The 2 MANUAL criteria (AC-F6b/c) are deferred to Vertical Slice milestone when Mali-G52 device access + HDMI/microphone capture setup are available. PLAYMODE criteria require a test scene fixture with representative subscriber MonoBehaviours — scaffold alongside bus implementation, not deferred to QA hand-off.

**Scaffolding artifact count (pass-3):** 25 test scaffolding units total — 16 BLOCKING CI ACs + 5 fixtures (AC-REL 3 injectors + AC-R3 string-literal violations fixture + AC-R7 filter-script unit tests) + 2 Python CI scripts (eventbus-linq-on-payload-filter.py + eventbus-string-field-scanner.py) + 2 CI workflows (eventbus-lint.yml + eventbus-codegen-staleness.yml).

## Open Questions

| # | Question | Owner | Target Resolution |
|---|---|---|---|
| OQ-1 | For non-MonoBehaviour subscribers (ScriptableObjects, pure C# services): is the manual `SubscriptionHandle.Unsubscribe()` pattern sufficient, or should the bus track weak references with GC-time warnings in release builds (not just Editor)? Current design: Editor-only leak detection. Risk: service-layer subscribers leak quietly in production. | Lead Programmer | During first non-MonoBehaviour subscriber implementation |
| OQ-2 | `SubscriptionManager` component auto-attached vs. explicit attachment: when `SubscribeAutoManaged` is called, should the bus auto-attach a `SubscriptionManager` if the GameObject lacks one, or throw and require the author to add it explicitly? Auto-attach is more ergonomic; explicit requires discipline. | Lead Programmer + systems-designer | Architecture phase (`/architecture-decision`) |
| ~~OQ-3~~ **RESOLVED 2026-04-20 via CD-GDD-ALIGN Concern 3**: Analytics uses option (c) with codegen. An `AssetPostprocessor` generates `AnalyticsSubscribers.cs` from the `gameplay.*` taxonomy entries at build time. Matches R3 pattern; preserves R1 + R7; preserves Pillar 3 analytics tuning loop. See R9. |
| OQ-4 | Should the Event Bus expose a deterministic replay mode for testing (record all events + payloads + timestamps during a session, replay them in unit tests)? Would be invaluable for regression tests but adds complexity. Physics is non-deterministic per TD ADR, so full replay is impossible — partial event replay might still be useful. | QA Lead + Lead Programmer | Post-MVP (when first flaky test surfaces) |
| OQ-5 | Event ID format validation runs at editor time (Data Registry R7), but the bus performs a second runtime check in debug builds. Is this redundancy worth the cost, or should we remove the runtime check and trust Data Registry's CI gate? | Lead Programmer | During first profiling pass of debug-build overhead |
| ~~OQ-6~~ **RESOLVED 2026-04-20 via CD-GDD-ALIGN Concern 2; REVISED 2026-04-21 pass-1**: Bus maintains per-channel typed bootstrap ring buffers (capacity 4 per channel) during Uninitialized state — replaces original single 16-slot buffer to eliminate boxing risk. Events fired during `Awake` are queued per-channel; drain time-sliced (≤4 events per channel per Update tick) on first tick after Ready. See R8 + R8.a. |
| ~~OQ-7~~ **RESOLVED 2026-04-21 pass-3 per unity-specialist pass-2 verification**: Unity 6.3 LTS ships **C# 9 only** (confirmed against `docs/engine-reference/unity/VERSION.md` and Unity 6.3 LTS official release notes). `readonly struct` is the final payload pattern; `readonly record struct` (C# 10) is NOT available. No retrofit needed. |
| OQ-8 | Drop policy on overflow: pass-1 retained FIFO (drop oldest) per CD adjudication, with release-build telemetry counter (R8.b) to surface real-world overflow rate. Revisit policy at v1.0 + 3 months live-ops with actual telemetry. Possible alternatives: drop-newest (prefer preserving chain coherence), priority-tag-based drop (require per-event priority metadata). | Lead Programmer + game-designer | Post-launch with overflow-rate telemetry data |
