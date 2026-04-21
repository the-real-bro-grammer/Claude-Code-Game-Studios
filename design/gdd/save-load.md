# Save / Load

> **Status**: Approved (pending independent `/design-review` in a fresh session)
> **Author**: Robert Michael Watson + game-designer / systems-designer / unity-specialist / gameplay-programmer / qa-lead / creative-director
> **Last Updated**: 2026-04-20
> **Implements Pillar**: Foundation for Pillar 3 (Mastery Through Economy — progression must persist) and Pillar 4 (Prep + React — level retry state must be durable). No direct player fantasy; enables every persistent system.
>
> **Creative Director Review (CD-GDD-ALIGN)**: APPROVED 2026-04-20 — no concerns. "One more level without the asterisk" framing is the strongest Pillar 3 articulation in the repo; R9 (state-vs-rules boundary) protects no-pay-to-win at MVP; Modal punt correctly scoped to Modal/Dialog GDD; T1/T2 latency exclusion architecturally correct. OQ-3 (save tamper-proofing) flagged for re-evaluation before any competitive/leaderboard feature ships — future gate, not current block.

## Summary

Save / Load is Money Miner's durable-state layer — the system that persists player progress, currency balances, tool and cosmetic ownership, progression unlocks, and settings across app sessions. It uses Unity's `JsonUtility` against local files under `Application.persistentDataPath`, writes atomically via a temp-then-rename pattern, and saves on every consequential gameplay event (not on quit) so mobile process kill cannot destroy progress. Every persistable type carries a schema version; a deterministic migration pipeline upgrades older saves to the current schema on load. Cloud Save is a post-launch adapter over this same local contract — this system owns local persistence only.

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: `Data Registry` (reads GUID-stable IDs), `Event Bus` (subscribes for write triggers)

## Overview

Mobile process death is the norm, not the exception. iOS and Android can kill a backgrounded game at any moment without warning, and players retry levels constantly during the Rush Phase. Save / Load is the one system standing between the player's banked treasure, unlocked biomes, owned upgrades — and those being silently erased the next time the OS wakes up the hardware. If it misbehaves, every other persistent system in Money Miner fails with it, regardless of how polished those systems look.

The system provides four capabilities: (1) **typed persistable state** — each consuming system defines its own POCO payload (`LevelProgressState`, `CurrencyState`, `ToolOwnershipState`, etc.) with no polymorphism, no `Dictionary` fields, and no `DateTime` without a wrapped representation; (2) **save-on-event** — Save / Load subscribes to durable-consequence events on the Event Bus (`gameplay.level.complete`, `gameplay.level.banked`, currency writes, purchases) and flushes state asynchronously off the main thread; (3) **atomic write with verify** — every save goes to `save.tmp`, fsyncs, then replaces `save.json` via `File.Replace`, keeping `save.bak` as the previous good copy; (4) **schema migration** — every persistable carries `_schemaVersion`, and a registered migrator chain upgrades older saves forward on load before any consumer sees them.

The system enforces five rules: **no blocking writes on the gameplay thread** (flushes are `Task.Run`-dispatched with a per-key coalescing queue so rapid writes collapse to one disk touch); **no raw asset paths** (all cross-references to Data Registry assets use the GUID `_id` field, not `AssetDatabase` paths, which don't exist at runtime); **no silent migration loss** (a failed migration writes a corrupted-save sentinel, preserves the original file as `save.corrupt-[timestamp].json`, and hands the consuming system a fresh default state with a `SaveRecoveryReason` flag so UI can surface it to the player); **no Save / Load owns gameplay rules** (quota values, unlock thresholds, prices — all live in Data Registry; Save / Load persists *which* nodes are unlocked, not what the threshold was); **no cloud logic in MVP** (the post-launch Cloud Save adapter subscribes to the same write events and reads the same local files — Save / Load does not know cloud exists).

This GDD defines the design contract — behavior, schema, migration rules, acceptance criteria. The implementation-level format specification, migration strategy semantics, and storage layout are scoped for **ADR-003: Save Schema Versioning** (pre-production action item from TD-FEASIBILITY). Consumers designing against this system may rely on the contract described here; ADR-003 will lock the wire format before implementation begins.

## Player Fantasy

Save / Load has no player-facing fantasy — and that's the point. No player will ever say "I love this game's persistence layer." They will say "I'll play one more level," and this is the system that makes "one more" a decision they don't have to think twice about.

What Save / Load actually delivers is **the absence of hesitation before the next tap**. It is 11:47 at night and the player has already said "one more level" for the fourth time. They commit three tools, a shop reroll, and a risky routing choice into a four-minute level. The phone drops to 3% battery. iOS force-quits the app. They plug in, they relaunch — and the run resumes cleanly. No "wait, did I lose that" pause. No scramble to remember what they'd bought. The coin count is right, the biome they just unlocked is still unlocked, the loadout they were tuning is exactly how they left it. The heist crew didn't stop working just because the phone did — they're a professional operation.

The fantasy is that the player can lean in. Money Miner's core loop ("I'm too clever for them") only works when the smug little heist gets to accumulate — three stars here, a new tool purchased there, a biome unlocked, a shop balance slowly climbing. Every one of those moments is a tiny ownership contract between the player and the game, and Save / Load is the bookkeeper that makes each contract real instead of performative. Without it, the failure mode is specific and corrosive: thirty minutes of progress evaporates to an OS memory kill, and now every future "one more level" decision is shadowed by a quiet *"…but what if."* The smugness curdles into caution — and caution is the death of mobile casual engagement.

Players won't articulate it. They'll just keep tapping the icon, trusting without thinking, committing without flinching. Save / Load succeeds when "one more" never comes with an asterisk.

## Detailed Design

### Core Rules

**R1. No main-thread file I/O.** All `File.*`, stream, and disk operations execute on `Task.Run` threads. Main-thread code calls `SaveManager.RequestFlush(key, payload)` and `SaveManager.GetState<T>()` only — never direct file access. Enforced by CI static-analysis grep banning `File.Write*` / `File.Read*` outside `SaveManager.cs` and `MigrationPipeline.cs`.

**R2. Immutable loaded state.** On load, each persistable domain is deserialized once into `SaveManager`'s internal authoritative cache. `GetState<T>()` returns a copy (deep-clone or re-deserialization), never the authoritative reference. Consumers may mutate their copy and pass it back via `RequestFlush`; they may not mutate what they received. Enforced by unit test that mutates a returned state and asserts the cached version is unchanged.

**R3. Atomic write with 1-deep backup.** Every flush sequence is: serialize → write to `save.tmp` → `FileStream.Flush(flushToDisk: true)` (maps to `fsync` on iOS APFS + Android ext4/F2FS) → close → `File.Replace(save.tmp, save.json, save.bak)`. `save.bak` holds the previous good copy, rotated on every successful replace. Any exception before `File.Replace` returns leaves `save.json` untouched. Enforced by integration test that aborts the write thread mid-flush and verifies `save.json` remains the prior valid version.

**R4. Two-level schema versioning.**
- **R4.a Per-type schema version.** Every persistable POCO declares `public int _schemaVersion` as its first field. Serializer asserts the field is present and non-zero at both write and load time; `_schemaVersion = 0` is treated as corrupt.
- **R4.b Root-level file version.** The top-level `SaveFileRoot` container declares `public int _saveFileVersion`, incremented by one whenever any per-type schema is revised. A migration run that fails mid-way does not partially commit: either all per-type migrations succeed and `_saveFileVersion` is advanced atomically on write, or the original `save.json` is preserved and the run aborts to `CorruptedRecovery`. This prevents the mixed-schema save file that no migrator can handle.

**R5. Migration determinism.** Each migrator `(fromVersion: int, json: string) → (toVersion: int, json: string)` is a pure function: no `UnityEngine.Random`, no `DateTime.Now`, no `UnityEngine.Object` access, no I/O, no `PlayerPrefs`. Enforced by unit test that runs each migrator 1000× on identical input and asserts byte-identical output.

**R6. ID references, never asset paths.** Every cross-reference to a Data Registry asset uses the GUID `_id` string (e.g., `"a1b2c3d4e5f6..."`). `AssetDatabase.GetAssetPath` does not exist at runtime and is banned in save-domain files by CI grep. Runtime assertion in `SaveManager.Load()` validates every loaded ID string against the GUID regex `^[0-9a-f]{32}$`. An ID that does not resolve to a loaded Data Registry asset is logged as a *stale reference*, not an error — the save is valid, but the referenced entity no longer exists (e.g., a tool renamed after release).

**R7. Snapshot-based coalescing — never deltas.** Write payloads must represent the FULL current state of their domain, not a delta against the prior state. Consumer systems build a complete `ToolOwnershipState` / `CurrencyState` / etc. and pass it to `RequestFlush`; the save manager stores it as the in-flight payload for that coalescing key and discards any earlier pending payload without logging. The discarded payload is superseded, not lost — but only because the newer payload is a complete snapshot. Delta-shaped payloads passed to `RequestFlush` are a contract violation caught by a compile-time type constraint (`IPersistableSnapshot` marker interface) and by a CI static-analysis rule.

**R8. Corrupted-save recovery is non-destructive and surfaceable.** The load pipeline distinguishes five distinct outcomes, each represented in `SaveLoadResult`:
- **R8.a `FirstRun`** — no `save.json`, no `save.tmp`, no `save.bak`. Fresh install or post-delete-data. Consumers receive default states; UI may show onboarding/welcome flow.
- **R8.b `RecoveredFromTmp`** — `save.tmp` exists at cold boot (proof of a process kill during the rename window) AND parses as valid, well-formed JSON with a valid `_saveFileVersion`. `save.tmp` is promoted to `save.json` before load continues. Recovers a complete write that would otherwise be silently discarded.
- **R8.c `LoadedClean`** — normal path. `save.json` loads, migrates, validates.
- **R8.d `RecoveredFromBackup`** — `save.json` fails schema validation or JSON parse. Original is renamed to `save.corrupt-[unix-timestamp].json`; `save.bak` is promoted to `save.json`; load retries. UI may surface a mild "some progress recovered" message.
- **R8.e `ResetToDefault`** — both `save.json` and `save.bak` fail to load. Defaults returned for all domains. UI MUST surface a clear "progress was reset, corrupted save preserved for support at `save.corrupt-*.json`" message. Save / Load never silently discards progress.

Enforced by integration tests for each outcome branch.

**R9. Save / Load owns state, not rules.** Save / Load stores *which* progression nodes are unlocked, *which* tools are owned, *what* currency balance is — it never stores unlock thresholds, prices, or quota values. Those live in Data Registry. If a Data Registry asset changes its unlock cost after a save was written, the player's already-unlocked state is unaffected; the new cost applies only to future purchases. Enforced by design-review grep: no Data Registry tuning values (costs, thresholds, quotas) may appear in any persistable POCO type.

**R10. Serialization on main thread; I/O off-main.** `JsonUtility.ToJson` / `FromJson` require Unity's main-thread context and are NOT safe to call from inside a `Task.Run` lambda. The handler's main-thread portion performs: (a) snapshot build from runtime objects, (b) `JsonUtility.ToJson(snapshot)` to produce a string. The resulting string (plus the target filename) is what gets passed to `Task.Run`, which performs only file I/O. Similarly, `await` continuations that need to touch Unity APIs must return to the main thread; internal continuations that only mutate `SaveManager`'s own fields use `ConfigureAwait(false)` to avoid synchronization-context overhead.

**R11. Pause/focus triggers synchronous flush under a single-writer lock.** `OnApplicationPause(true)` and `OnApplicationFocus(false)` drain the coalescing queue **synchronously** on the main thread. The OS gives approximately 1-5 seconds of CPU time on mobile before kill; async is unsafe. The pause handler acquires a `SemaphoreSlim(1,1)` write lock (held by any in-flight async flush), waits up to `pause_flush_timeout_ms` (500ms default — tuning knob), then writes each pending snapshot synchronously. If a Task.Run flush is already mid-write when pause fires, the pause handler awaits its completion via `task.Wait(timeout)` before starting its own writes. The single-writer lock prevents two threads from racing on `save.tmp`.

**R12. Persistable schema format constraints.** Every field of every persistable type must be directly serializable by Unity `JsonUtility`. Banned and their required replacements:
- `Dictionary<K,V>` → `List<KVPair<K,V>>` with `[Serializable] struct KVPair { K key; V value; }`
- Polymorphic base-class fields → flat discriminated-union structs with an `int _kind` tag
- `DateTime` / `TimeSpan` → `long` ticks
- `Guid` → `string` (32-char lowercase hex)
- Nullable value types (`int?`, `long?`) → sentinel values (`-1`, `long.MinValue`) documented per-field
- `float.NaN` / `float.PositiveInfinity` / `float.NegativeInfinity` — must be clamped to valid finite values before serialize; NaN in JSON is not portable

Enforced by a schema-validation EditMode test that round-trips each persistable type through `JsonUtility.ToJson` / `FromJson` and asserts field-count parity with the live object. Silent field-drop (the most common JsonUtility bug) fails this test.

**R13. Settings bypass Event Bus.** Settings (audio volumes, reduce-motion at VS, locale at v1.0) are written via a dedicated `SettingsManager` that reuses `SaveManager`'s file-I/O utilities but does not subscribe to Event Bus. Rationale: settings are UI-driven, low-stakes, and writing them does not benefit from the Gameplay event namespace. `settings.json` lives in the same `persistentDataPath` directory, uses the same atomic-write + schema-versioning rules, but has no coalescing queue (settings changes are human-speed) and no 1-deep backup (a bad volume value is not a progress-loss event).

### States and Transitions

| State | Entry Condition | Exit Condition | Behavior |
|-------|----------------|----------------|----------|
| **Uninitialized** | `SaveManager.Awake()` | `Load()` called | All `GetState<T>()` and `RequestFlush()` calls throw `InvalidOperationException`. |
| **LoadingFromDisk** | `Load()` called | Load + migration completes (success or recovery) | Attempts, in order: (1) check for `save.tmp` and promote if valid (R8.b); (2) load `save.json` → migrate → validate (R8.c); (3) on failure, promote `save.bak` (R8.d); (4) on double failure, default state (R8.e); (5) no files at all → `FirstRun` (R8.a). Main-thread scene advancement blocks on this completing. |
| **Ready** | `LoadingFromDisk` exits | `Dispose()` called | Default operating state. `GetState<T>()` returns cached copies. `RequestFlush(key, snapshot)` enqueues to coalescing queue. Event Bus subscriptions active. |
| **WritingToDisk** | Coalescing queue drains a key; flush begins | Write succeeds OR transitions to `CorruptedRecovery` on `File.Replace` failure | Sub-state of `Ready` — other operations continue normally. Tracked for metrics (e.g., writes per minute). |
| **FlushingSynchronous** | `OnApplicationPause(true)` or `OnApplicationFocus(false)` fires | Pause handler returns, or timeout `pause_flush_timeout_ms` elapsed | Main thread drains queue synchronously. Acquires single-writer lock; awaits any in-flight async write; writes remaining pending snapshots inline. Blocks pause callback until done (or timeout). |
| **CorruptedRecovery** | `File.Replace` fails mid-flush, OR both `save.json` and `save.bak` fail to load | UI acknowledges (or on-pause flush completes synchronously without recovery prompt) | Suspends further writes until acknowledged. Copies the corrupt file to `save.corrupt-[timestamp].json`. Loads default states. `SaveLoadResult.ResetToDefault` flag raised for UI. On acknowledge, transitions to `Ready` with fresh state. |
| **Disposed** | `SaveManager.OnDestroy()` | Terminal | Drains coalescing queue synchronously (`FlushingSynchronous`-equivalent), unsubscribes all Event Bus handlers, clears caches. Further calls throw. |

### Coalescing Queue Model

Two data structures:

1. **Coalescing map** — `Dictionary<CoalescingKey, PendingWrite>`, where `PendingWrite` holds the serialized JSON string (produced on the main thread before enqueue), an `InFlight` bool, and an optional `NextPendingPayload` string (held when a new snapshot arrives during an in-flight write).
2. **Flush queue** — `Queue<CoalescingKey>` of distinct keys with pending payloads; ordered FIFO per key insertion.

**Enqueue rule (main thread):**

```
On RequestFlush(key, snapshot):
  serialized = JsonUtility.ToJson(snapshot)     // main-thread-only; R10
  if coalescing_map[key].InFlight:
    coalescing_map[key].NextPendingPayload = serialized
  else:
    coalescing_map[key].Payload = serialized     // overwrite, not append
  if key not already in flush_queue:
    flush_queue.Enqueue(key)
```

**Dispatch rule (background worker):**

```
Loop:
  key = flush_queue.Dequeue()        (blocks if empty)
  entry = coalescing_map[key]
  entry.InFlight = true
  try: AtomicWrite(key.FilePath, entry.Payload)  // tmp → fsync → Replace
  finally: entry.InFlight = false
  if entry.NextPendingPayload != null:
    entry.Payload = entry.NextPendingPayload
    entry.NextPendingPayload = null
    flush_queue.Enqueue(key)         // re-enqueue for next cycle
```

**Latency budget:** event-to-disk target ≤ 2000 ms under normal I/O (tuning knob: `flush_latency_target_ms`). This is an advisory, not a real-time guarantee — the hard guarantee ("save completes before OS kill") is served by **R11's synchronous pause-flush**. For T1/T2 Event Bus consumers, Save / Load never participates in the T1 (16.67 ms) or T2 (100 ms) budget; Save / Load's handlers dispatch to Task.Run within the first frame after the event fires, freeing the dispatch thread.

### Interactions with Other Systems

**Upstream (Save / Load reads from):**

| System | Reads | Interface | Ownership |
|---|---|---|---|
| **Data Registry** | GUID `_id` strings (for cross-referencing owned tools, unlocked nodes, referenced biomes) | Reads catalog lookups to validate loaded ID references at startup; stale references logged but not errored | Data Registry owns schema; Save / Load owns persistence of which IDs are player-owned |
| **Event Bus** | Subscription channel for write triggers (see table below) | `SubscribeAutoManaged<T>(channel, OnEvent, this)` lifecycle | Event Bus owns dispatch; Save / Load owns write decisions |

**Event Bus subscription catalog (write triggers):**

| Event ID (channel) | Persistable Domain Written | Coalescing Key | Existing or Proposed |
|---|---|---|---|
| `gameplay.level.complete` | `LevelProgressState`, `CurrencyState`, `BiomeUnlockState` (if unlock triggered) | `level_progress`, `currency`, `biome_unlock` | Exists in Event Bus GDD |
| `gameplay.level.failed` | `LevelProgressState` (retry count, last attempt outcome) | `level_progress` | Exists in Event Bus GDD |
| `gameplay.level.banked` | `CurrencyState` (running banked for this level), `LevelProgressState` | `currency`, `level_progress` | Exists in Event Bus GDD |
| `gameplay.currency.changed` | `CurrencyState` | `currency` | **PROPOSED addition to Event Bus** |
| `gameplay.tool.purchased` | `ToolOwnershipState`, `CurrencyState` | `tool_ownership`, `currency` | **PROPOSED addition** |
| `gameplay.tool.upgraded` | `ToolOwnershipState`, `CurrencyState` | `tool_ownership`, `currency` | **PROPOSED addition** |
| `gameplay.progression.node_unlocked` | `ProgressionNodeState` | `progression` | **PROPOSED addition** |
| `gameplay.biome.unlocked` | `BiomeUnlockState` | `biome_unlock` | **PROPOSED addition** |
| `gameplay.session.level_selected` | `SessionState` (last-played level ID) | `session` | **PROPOSED addition** |
| `gameplay.cosmetic.purchased` (VS) | `CosmeticOwnershipState`, `CurrencyState` | `cosmetic_ownership`, `currency` | **PROPOSED addition** (inert at MVP) |

**Not routed through Event Bus:**
- Settings writes — `SettingsManager.SetAudioMaster(float)` etc. call directly into Save / Load's file-I/O for `settings.json` (R13).
- Playtest logger — outside Save / Load; owns its own `playtest-log-[date].jsonl` per CD-SYSTEMS; `DEVELOPMENT_BUILD` only.

**Downstream (systems that read persisted state on load):**

| System | Reads From Save/Load | On Load | On First Run (R8.a) |
|---|---|---|---|
| **Level Runtime** | last-played level ID (for "continue" flow) | `GetState<SessionState>()` | Points to level 1 of first biome |
| **Level Scoring** | best star rating per level | `GetState<LevelProgressState>()` | All levels 0 stars |
| **Currency & Economy** | coin + gem balances | `GetState<CurrencyState>()` | Starting-balance defaults from `CurrencyDefinition` (Data Registry) |
| **Progression** | unlocked node IDs | `GetState<ProgressionNodeState>()` | Initial unlocked set from `ProgressionNode` root-level flag (Data Registry) |
| **Biome Map** | unlocked biomes + last-played | `GetState<BiomeUnlockState>` + `SessionState` | First biome only |
| **Tool System** | owned tool IDs + tier index | `GetState<ToolOwnershipState>()` | Tutorial-gated starter set |
| **Main Menu / Settings** | audio volumes, reduce-motion (VS) | `SettingsManager.GetCurrent()` | Defaults from `SettingsDefaults` constants |

**Open question flagged to Level Runtime / Currency GDDs (not to be resolved here):** Does level-fail forfeit the banked coins from the current Rush Phase? Save / Load persists whatever Economy commits; the gameplay rule is Level Runtime's + Economy's to define. Noted in Open Questions (OQ-1).

**Resume-from-interruption policy:** Process kill during Rush Phase → on relaunch, player lands on the **Biome Map** with the last banked-coin state intact. No in-flight Rush Phase state is preserved (physics non-determinism per pre-prod ADR-1 makes this impossible regardless). UI surfaces a neutral "picked up where you left off" message if `SessionState.lastPlayedAt` is within the last session. No data-loss dialog — the banked state IS the saved state.

## Formulas

Save / Load's formulas are operational budgets and capacities, not gameplay balance formulas. Gameplay balance lives in consuming systems (Economy, Progression, Level Scoring).

### D.1 FlushLatencyTarget

Advisory event-to-disk latency ceiling for a coalesced async save write. Not a real-time guarantee; not tied to Event Bus T1/T2 tiers.

`FlushLatencyTarget = pause_flush_timeout_ms * safety_margin_factor`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Pause flush timeout | `pause_flush_timeout_ms` | int (ms) | 200–1000 | Hard synchronous budget from D.2 |
| Safety margin factor | `safety_margin_factor` | float | 0.1–4.0 | Multiplier so the advisory target sits well outside the hard sync budget |
| Flush latency target | `FlushLatencyTarget` | int (ms) | 50–2000 | Advisory ceiling; async flushes should complete within this under normal I/O |

**Output Range:** 50–2000 ms. Default: 2000 ms (= 500 × 4.0). Not clamped at runtime; a sustained write exceeding the target logs `[SaveManager] FlushLatency exceeded target` in debug only — advisory, not a fault.

**Example:** `pause_flush_timeout_ms = 500`, `safety_margin_factor = 4.0` → `FlushLatencyTarget = 2000 ms`. A coalesced `currency` + `level_progress` snapshot at MVP scale (~1–3 KB serialized) completes in ~8–40 ms on Mali-G52 eMMC; the loose target absorbs Android scheduler jitter and eMMC wear-leveling stalls without spamming advisories.

**Rationale:** Mobile I/O is non-deterministic; a T2-aligned 100 ms target would trigger constantly under normal OEM conditions. The real guarantee ("save completes before OS kill") is served by D.2's synchronous path. This advisory only exists to surface a broken write pipeline.

**Validation:** CI stress test — drive all 6 MVP coalescing keys at max frequency for 60 s on a Mali-G52 emulator image; assert p99 async flush latency < 2000 ms.

### D.2 PauseFlushTimeout

Hard synchronous write budget on `OnApplicationPause(true)` / `OnApplicationFocus(false)`. The upper bound the system must complete within before the OS process-kills the app.

`PauseFlushTimeout = platform_grace_ms * os_safety_margin`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Platform OS grace period | `platform_grace_ms` | int (ms) | 1000–5000 | iOS ~5000 (APFS, consistent); Android OEM-dependent 1000–5000 |
| OS safety margin | `os_safety_margin` | float | 0.3–0.6 | Fraction of grace reserved for the write; remainder absorbs Unity teardown + OEM variance |
| Pause flush timeout | `PauseFlushTimeout` | int (ms) | 300–2500 | Hard wall; flush must complete or abandon within this window |

**Output Range:** 300–2500 ms. Default: 500 ms (= 1000 Android worst case × 0.5). Enforced by `CancellationTokenSource` with `PauseFlushTimeout` delay. On timeout the in-progress Task is cancelled; the atomic-write pattern means `save.json` is never partially written; the in-memory state is preserved for the next resume-then-flush attempt if the process survives.

**Example:** Android OEM worst case: `platform_grace_ms = 1000`, `os_safety_margin = 0.5` → `PauseFlushTimeout = 500 ms`. At MVP with 6 coalescing keys and a 20–40 KB payload: main-thread serialization adds 1–5 ms; Task.Run I/O (tmp write + fsync + File.Replace) on Mali-G52 eMMC runs 15–80 ms under normal conditions. 500 ms provides 6–33× headroom and absorbs an eMMC stall of up to ~420 ms before aborting.

**Rationale:** iOS is reliably ~5 s; designing to that wastes budget. Android OEM is the binding constraint — Samsung mid-range measured at 1.0–1.2 s, some Xiaomi skins at ~800 ms. Design to the conservative 1 s floor with 50% OS margin. A per-platform relaxation to 800 ms on iOS via `#if UNITY_IOS` is permitted once shipping profiling demonstrates headroom.

**Validation:** Integration test — enqueue a pending write, simulate `OnApplicationPause(true)`, assert either `save.json` updated within 500 ms OR `save.json` unchanged (not partially written) when simulated I/O stalls exceed the timeout.

### D.3 SaveFileSizeBudget

Soft on-disk ceiling for `save.json` + `save.bak` + `settings.json` combined. Diagnostic canary for the R9 state-vs-rules boundary — a breach signals that a persistable type is holding gameplay values it should be reading from Data Registry at runtime.

`SaveFileSizeBudget = (N_biomes * N_levels * bytes_per_level_record) + (N_tools * bytes_per_tool_record) + (N_progression_nodes * bytes_per_node_record) + bytes_overhead`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Biomes authored | `N_biomes` | int | 1–20 | Biomes with level records in save.json |
| Levels per biome | `N_levels` | int | 1–60 | Max levels per biome with persisted completion state |
| Bytes per level record | `bytes_per_level_record` | int | 8–32 | Star count (1) + completion flag (1) + best-time (4) + padding |
| Tools persisted | `N_tools` | int | 1–20 | Tool + consumable ownership records |
| Bytes per tool record | `bytes_per_tool_record` | int | 40–80 | GUID (32) + owned flag (1) + charge count (4) + overhead |
| Progression nodes | `N_progression_nodes` | int | 1–100 | Unlock-state nodes |
| Bytes per node record | `bytes_per_node_record` | int | 36–48 | GUID (32) + unlocked flag (1) + overhead |
| Fixed overhead | `bytes_overhead` | int | 512–4096 | Root envelope, currency fields, session, settings.json |
| File-size budget | `SaveFileSizeBudget` | int (bytes) | — | Soft ceiling |

**Output Range:** Unbounded by the formula (it produces expected size for a given scope). The ceiling is derived: full v1.0 scope × 2 JsonUtility verbosity headroom = **~64 KB**. Breach logs `[SaveManager] Size budget exceeded: {actual}B > {budget}B` — ADVISORY in debug, BLOCKING in release CI.

**Example (MVP, 5 biomes authored, 1 active):**
- Level records: 5 × 40 × 16 = 3 200 bytes
- Tool records: 4 × 60 = 240 bytes (3 tools + 1 consumable)
- Progression: 10 × 40 = 400 bytes
- Overhead: 1 024 bytes
- Raw expected: ~4 864 bytes (~5 KB). With 2× JsonUtility verbosity: ~10 KB — well under 64 KB.
- Full v1.0 scope: `5 × 40 × 16 + 4 × 60 + 50 × 40 + 1024 = 6 464` raw, ~13 KB serialized.

**Rationale:** JsonUtility file I/O at 64 KB has negligible runtime cost on any supported device. The ceiling exists as a canary: the rules-vs-state boundary (R9) is easy to violate accidentally; size explosion is the symptom. Catching it pre-ship beats debugging it in production.

**Validation:** CI job measures serialized size across `save.json` + `save.bak` + `settings.json` after a representative EditMode fixture run. Release builds treat `>SaveFileSizeBudget` as a blocking failure; debug builds log and continue.

### D.4 CoalescingCapacity

Total coalescing queue capacity across all keys. The per-key cap (2 payloads: `in-flight` + `next-pending`) is a design axiom from R7; total capacity is a bounded constant. A per-frame flush-request count guard catches runaway upstream producers.

```
TotalQueueCapacity = N_coalescing_keys * max_pending_per_key
MaxFlushRequestsPerFrame = N_coalescing_keys * flush_requests_per_key_per_frame_limit
```

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Coalescing keys | `N_coalescing_keys` | int | 6–9 | MVP 6 (`level_progress`, `currency`, `biome_unlock`, `tool_ownership`, `progression`, `session`); VS +1 (`cosmetic_ownership`); v1.0 +1 (`analytics_flush`) |
| Max payloads per key | `max_pending_per_key` | int | 2 (fixed) | R7 hard cap: 1 in-flight + 1 next-pending |
| Total queue capacity | `TotalQueueCapacity` | int | 12–18 | Statically allocated at startup; no runtime growth |
| Per-key per-frame limit | `flush_requests_per_key_per_frame_limit` | int | 1–3 | Legitimate burst ceiling; >3 signals runaway producer |
| Max flush requests / frame | `MaxFlushRequestsPerFrame` | int | 6–27 | Frame-wide guard; breach triggers debug assertion, not a fault |

**Output Range:** `TotalQueueCapacity` = 12 (MVP), 14 (VS), 16 (v1.0). `MaxFlushRequestsPerFrame` = 18 (MVP). Excess `RequestFlush` calls are silently coalesced by R7 snapshot semantics (no data loss) but the producing system has a bug that must be fixed before ship.

**Example (MVP):**
- Normal level completion fires 3 flushes (`currency`, `level_progress`, `session`) in one frame — all distinct keys, all coalesce cleanly into their `next-pending` slots.
- A buggy Economy system fires `RequestFlush("currency", ...)` 47× in one frame: slot 1 (`in-flight`) holds call 1; slot 2 (`next-pending`) holds call 2; calls 3–47 each overwrite `next-pending` via R7 snapshot semantics. Frame-end count = 47 > 18 → `[SaveManager] Runaway flush producer detected: 47 requests in frame {N}`. No data loss, but the Economy system has a feedback loop to fix.

**Rationale:** R7 makes per-key cap = 2 a design axiom (two slots sufficient because snapshot writes are idempotent; any superseded payload is stale by definition). The per-frame guard is the only meaningful runaway detector — without it, a system emitting 100 flush calls per frame is invisible at runtime, discoverable only via unexplained battery drain in production.

**Validation:** Unit test — call `RequestFlush` on a single key 100× synchronously; assert only 2 payload slots populated and exactly 1 runaway warning logged. Separate test — all 6 MVP keys once per frame for 300 frames; assert capacity never exceeds 12, no warnings.

## Edge Cases

### Storage / Device

- **If `File.Replace(save.tmp, save.json, save.bak)` throws `IOException` because the target volume fills after `save.tmp` was fsynced**: retain `save.tmp` untouched, leave `save.json` as-is, transition to `CorruptedRecovery`, log `[SaveManager] Disk full — save.tmp retained for R8.b recovery`. On next cold boot, R8.b's tmp-promotion path recovers the write. Do NOT delete `save.tmp` to free space — it is recoverable state.

- **If iOS `NSFileProtectionComplete` blocks a write because the device locks before `FlushingSynchronous` completes**: `File.Replace` throws `UnauthorizedAccessException` (not `IOException`). CorruptedRecovery path must catch `UnauthorizedAccessException` separately and must NOT rename `save.json` to `save.corrupt-*` — the file is inaccessible, not corrupt. Retry the write when `OnApplicationFocus(true)` fires next session.

- **If a multi-user Android device switches users mid-session**: the foreground UID changes after `OnApplicationFocus(false)` fires. R11's synchronous flush writes to the OLD user's sandbox, which is still valid at that instant. No special case — R11 is the correct backstop. Explicitly noted so a future reviewer does not "fix" R11 into an async path.

- **If the player clears app data from Android settings while the process is running**: `Application.persistentDataPath` becomes invalid mid-session. Any subsequent write throws `DirectoryNotFoundException`. Catch and transition to `CorruptedRecovery` with `ResetToDefault` semantics; the in-memory state remains valid until scene reload, but no further writes succeed. Log `[SaveManager] Storage path invalidated — user cleared app data`.

### Schema / Migration

- **If the user installs an older app version and loads a save whose `_saveFileVersion` exceeds the code's current version** (downgrade path): the migrator chain is strictly forward-only. Detect `_saveFileVersion > CurrentCodeVersion`, immediately apply R8.d (promote `save.bak` if its version is compatible); if `save.bak` also has a future version, apply R8.e (defaults). Never attempt backward migration.

- **If the migration chain has a gap** (v1 migrator and v3 migrator registered, no v2 migrator): on load of a v1 save, pipeline finds no `(fromVersion: 1) → v2` entry, halts, logs `[MigrationPipeline] No migrator for v1 → v2`, applies R8.d/R8.e. CI build-time assertion catches this: `max(registered migrators) == CurrentCodeVersion` and chain must be gapless from v1.

- **If a successful migration produces a JSON string exceeding `SaveFileSizeBudget`**: reject the migrated result before writing, preserve the original pre-migration `save.json` as `save.corrupt-[timestamp].json`, return R8.e defaults, log the overage with actual byte count. Do NOT write an oversized save — it would compound on the next boot.

- **If the top-level `_saveFileVersion` field is missing from `save.json`** (manual editing, truncated write, earlier serializer version): treat as corrupt per R4.a's rule that `_schemaVersion/_saveFileVersion == 0` is corrupt. Apply R8.d. Never attempt migration — there is no safe version to migrate from.

### Runtime Race Conditions

- **If `OnApplicationPause(true)` fires while `LoadingFromDisk` is still in progress** (slow cold boot on low-end Android during a heavy migration): R11's pause handler detects `State == LoadingFromDisk` and skips the synchronous flush entirely — the coalescing queue is empty and writing stale state would be harmful. Log `[SaveManager] Pause during load — flush skipped`. The OS grace budget is consumed by the load itself.

- **If `SaveManager.Dispose()` fires while a `Task.Run` write is in-flight** (scene reload during async flush): the single-writer `SemaphoreSlim` must be acquired by `Dispose` before proceeding. `Dispose` calls `task.Wait(PauseFlushTimeout)` on the in-flight task; if it times out, the task is abandoned (the tmp file remains, recoverable by R8.b on next boot). Do NOT `task.Cancel()` mid-write — a partial tmp file that is NOT fsynced is worse than an abandoned one.

- **If the player force-quits the app via recent-tasks swipe** (different from `OnApplicationPause`): no callbacks fire. Any pending coalesced writes are lost. The previous successful `save.json` is the authoritative state on next launch. This is **the specific failure mode that R11's aggressive event-driven writes exist to minimize** — if the player has banked treasure and the coalesced write had been dispatched (even asynchronously), it almost certainly completed before the swipe because `Task.Run` I/O on a 20-40 KB file runs in 15-80 ms. The window where data is lost is the interval between the event firing and Task.Run acquiring the file handle.

### Clock / Time

- **If `SessionState.lastPlayedAt` reads back as a future value** (user manipulated system clock, NTP sync rolled backwards): the resume-from-interruption UI check must clamp `if (lastPlayedAt > UtcNow.Ticks) treat as UtcNow.Ticks`. Do NOT suppress the resume message based on "future timestamps are invalid" — show the message conservatively. Never use `lastPlayedAt` for any gameplay-material decision (per R9, it is display-only).

- **If `save.bak`'s file modification time is newer than `save.json`'s** at `LoadingFromDisk` start: this is a diagnostic anomaly, not a load-order signal. ALWAYS attempt `save.json` first per R8.c ordering. The file mtime is not a reliable integrity signal; only JSON parse + schema validation are. Log the anomaly for support diagnostics; do not alter load order.

### Data Integrity / Stale References

- **If a tool ID in `ToolOwnershipState` no longer resolves to any `ToolConfig` in Data Registry** (asset deleted post-release): per R6, log as stale reference, do not error. Tool System must request state normally and defensively: if an owned-tool ID has no corresponding `ToolConfig`, exclude it from available loadouts silently and fire a `gameplay.tool.stale_reference` debug event. The player does not lose currency or progress; the orphaned ID remains in the save file (it may resolve again in a hotfix).

- **If a `BiomeData` asset is deleted but a player has `BiomeUnlockState` for it**: identical stale-reference handling. The unlock record persists; Biome Map renders only biomes present in the current Data Registry catalog. The stale biome is invisible to the player — not a blocker, not cleared. Development builds flag this in a debug panel.

- **If `File.Replace` appears to succeed but an external tool (platform backup, user file-manager app on Android) modifies `save.json` between write and the next load**: no runtime detection is possible short of embedding a content hash. Save / Load does not implement hash verification at MVP — the security surface is low (local single-player, no leaderboards in MVP, banked-count physics makes replay attacks irrelevant per pre-prod ADR-1). If Cloud Save or leaderboards ship post-launch, re-evaluate: a `SaveFileSecurity` ADR may add a content hash under `_fileHash` to the root container.

### Formula Boundaries

- **If `FlushLatencyTarget` is exceeded on a legitimate write under sustained eMMC stall**: the advisory fires but the write completes eventually (no abort). D.1 is explicitly advisory; D.2 is the hard wall. Sustained D.1 advisories in dogfooding builds signal either wear-leveling problems on the test device or an oversized save payload — investigate both before ignoring.

- **If `CoalescingCapacity`'s `MaxFlushRequestsPerFrame` guard fires in a legitimate scenario** (e.g., a batch unlock where 10 progression nodes unlock in one frame from a single "chapter complete" event): the guard is a warning, not a fault. R7 snapshot semantics absorb the excess calls correctly. Evaluate the producing event: if the batch unlock is legitimate design intent, consider issuing a single `gameplay.progression.batch_unlock` event carrying the full snapshot instead of N individual `gameplay.progression.node_unlocked` events. The guard exists to surface this refactor opportunity, not to block the save.

## Dependencies

### Upstream (Save / Load reads from)

| System | Priority | Nature |
|---|---|---|
| **Data Registry** | MVP | Uses GUID `_id` strings as the reference primitive for all cross-asset pointers in save state (owned tools, unlocked nodes, referenced biomes, etc.). Reads catalog on load to validate loaded IDs; stale references logged (R6) but non-fatal. On `FirstRun` (R8.a), Data Registry provides starting-balance defaults (`CurrencyDefinition`) and initial-unlock root flags (`ProgressionNode`). |
| **Event Bus** | MVP | Subscribes to `gameplay.*` durable-consequence events as write triggers. Subscription pattern is `SubscribeAutoManaged<T>(channel, OnEvent, this)`; lifecycle auto-manages via `SubscriptionManager`. Save / Load never emits events — it is a subscriber, never a publisher. |

### Downstream (hard — MVP)

These systems read Save / Load state on scene entry and cannot function without the persistence contract.

| System | Priority | Nature |
|---|---|---|
| Level Runtime | MVP | Reads `SessionState.lastPlayedLevelId` for "continue" flow; emits `gameplay.level.complete` / `.failed` / `.banked` that Save / Load subscribes to |
| Level Scoring | MVP | Reads `LevelProgressState` for star rating; emits via Level Runtime's events (no direct Save / Load emission) |
| Currency & Economy | MVP | Reads `CurrencyState` for current balances; emits `gameplay.currency.changed` (proposed Event Bus addition) |
| Progression / Unlock Gating | MVP | Reads `ProgressionNodeState` for unlocked set; emits `gameplay.progression.node_unlocked` (proposed) |
| Biome Map / Level Select | MVP | Reads `BiomeUnlockState` + `SessionState` for rendering and focus; emits `gameplay.biome.unlocked` (proposed) and `gameplay.session.level_selected` (proposed) |
| Main Menu / Settings / Pause | MVP (minimal) | Reads `SettingsManager.GetCurrent()` (R13 path, not Event Bus); writes via `SettingsManager.Set*()` |
| Tool System | MVP | Reads `ToolOwnershipState` for owned tools + tier indices; emits `gameplay.tool.purchased` / `.upgraded` (proposed) |

### Downstream (phased — dependency activates at system delivery tier)

| System | Delivery Tier | Nature |
|---|---|---|
| Shop & IAP | Vertical Slice (full; MVP stub) | Reads `CurrencyState` (via Currency & Economy); writes transactions through Economy, not directly. MVP stub uses debug-panel UI; writes still flow through Save / Load per R7 snapshot contract. |
| VFX / Juice | Vertical Slice | Does not read or write Save / Load state; listed only for contract clarity (VFX has no persistent state in MVP/VS scope). |
| Accessibility Service | Vertical Slice | Reads reduce-motion flag from `SettingsManager`; no direct Save / Load access. Interface hooks are authored into MVP GDDs (Level Runtime, VFX, HUD) for later consumption. |
| Cosmetic / Skin System | Shippable v1.0 | Reads `CosmeticOwnershipState`; emits `gameplay.cosmetic.purchased` (proposed, inert at MVP) |
| Analytics / Telemetry | Shippable v1.0 | Reads opt-in flag from `SettingsManager`; may persist a local analytics-flush buffer under a new coalescing key (`analytics_flush`). Adds to `SaveFileSizeBudget` formula variables at v1.0. |
| Localization | Shippable v1.0 | Reads locale preference from `SettingsManager` (not `save.json`). |
| Ads SDK | Shippable v1.0 | Reads ad-history state (frequency caps) from `save.json` under a new `ad_history` key. Minor addition; does not require schema redesign. |

### Downstream (soft — reads values but functions if Save / Load lookups return defaults)

| System | Nature |
|---|---|
| Critter AI | Does not read Save / Load directly; all its tuning is in Data Registry. Save / Load inclusion is transitive through Tool System and Hazard System only. |
| Hazard System | Same as Critter AI — no direct Save / Load dependency. |
| HUD | Reads currency balance and star counts via their owning systems (Currency & Economy, Level Scoring), not directly from Save / Load. HUD has no persistent state of its own. |
| Modal / Dialog + Level End | Renders `SaveLoadResult.ResetToDefault` recovery messaging from R8.e on first open after a corruption event; no other dependency. |

### Downstream (post-launch)

| System | Delivery Tier | Nature |
|---|---|---|
| Cloud Save | Post-launch | **Adapter over the same local files.** Subscribes to the same `gameplay.*` events Save / Load listens to, reads the same `save.json` format, uploads to provider (Unity Cloud Save or Firebase — decision deferred to plumbing sprint). Save / Load is intentionally unaware of cloud — the adapter is layered above, not integrated into this system. |
| Leaderboard | Post-launch | Reads banked-count scores from `LevelProgressState`; uploads to leaderboard backend. Banked-count (not physics replay) is the committed ADR-1 model. |

### Cross-References

See **Section C → Interactions with Other Systems** for the interface contract per downstream system (subscription pattern, coalescing key, first-run defaults). This section avoids duplicating that table; the goal here is priority and activation timing.

### Cross-System Ownership Rule

Save / Load owns:
- **File I/O** — all reads/writes to `save.json`, `save.bak`, `settings.json`, `save.tmp`, `save.corrupt-*.json`
- **Schema versioning and migration pipeline**
- **Coalescing queue and atomic-write mechanics**
- **Persistent state caching (the authoritative in-memory copy)**

Downstream systems own:
- **Their own state shape** — consumer systems define their persistable POCO (`LevelProgressState`, `CurrencyState`, etc.) and own its schema evolution
- **Runtime behavior** — when to request a flush, what the state means for gameplay
- **First-run defaults** — each consumer provides a default state factory used on `FirstRun` (R8.a)
- **Event emission** — emitters decide which Event Bus events carry write-trigger semantics

If a value would not survive app uninstall, it does not belong in Save / Load.
If a value is a gameplay tuning knob (cost, threshold, rate), it belongs in Data Registry, not Save / Load (R9).
If a value is an ephemeral per-session log, it belongs in its own domain (e.g., PlaytestLogger's `playtest-log-[date].jsonl`), not Save / Load.

## Tuning Knobs

Save / Load's tuning knobs are operational thresholds — CI gate severities, timing budgets, capacity ceilings. Gameplay tuning (starting currency values, unlock thresholds, quota curves) lives in Data Registry assets per R9, not here.

| Parameter | Current Value | Safe Range | Effect of Increase | Effect of Decrease |
|---|---|---|---|---|
| `pause_flush_timeout_ms` (D.2) | 500 ms | [200, 2500] | More slack for synchronous pause-flush on slow eMMC; risk of OS killing process before budget elapses on aggressive Android OEMs | Tighter sync window; legitimate writes may abort under transient I/O stall, losing last-event payload (recoverable via R8.b on next boot) |
| `safety_margin_factor` (D.1) | 4.0 | [0.1, 4.0] | Advisory `FlushLatencyTarget` ceiling loosens; fewer false-positive debug logs; less signal on pipeline degradation | Tighter advisory; more sensitivity to eMMC stalls; noisier debug logs; earlier warning of broken write pipeline |
| `SaveFileSizeBudget` ceiling (D.3) | 64 KB | [32 KB, 256 KB] | More headroom for future scope additions; masks R9 state-vs-rules boundary violations | Tighter canary; catches accidental gameplay-value persistence earlier; risk of false-positive blocks on legitimate v1.0+ scope growth |
| `flush_requests_per_key_per_frame_limit` (D.4) | 3 | [1, 10] | Higher burst tolerance; fewer runaway-producer warnings; risk of masking real bugs | Tighter; more warnings on legitimate batch unlocks; pushes designers toward single-batched events |
| CI severity — `Resources.Load` / direct `File.*` outside `SaveManager` | **BLOCKING** (R1) | BLOCKING only | N/A | Changing to WARNING permits arbitrary file I/O scattered across gameplay systems; strictly wrong — violates the single-writer invariant |
| CI severity — reflection mutation of loaded state | **BLOCKING** (R2) | BLOCKING only | N/A | Changing to WARNING allows consumers to mutate the authoritative cache; strictly wrong |
| CI severity — delta payload on `RequestFlush` (non-`IPersistableSnapshot`) | **BLOCKING** (R7) | BLOCKING only | N/A | Strictly wrong — delta payloads silently corrupt under coalescing |
| CI severity — migration chain gap | **BLOCKING** (build-time) | BLOCKING only | N/A | Strictly wrong — gap guarantees player-facing load failure at the affected version |
| CI severity — stale-reference count at load | **ADVISORY** (R6) | [ADVISORY, WARNING] | N/A | Upgrading to WARNING once Data Registry asset hygiene is strict may catch asset-rename mistakes earlier |
| CI severity — `SaveFileSizeBudget` breach at test fixture | **BLOCKING** (release) / **ADVISORY** (debug) | [ADVISORY, WARNING, BLOCKING] | N/A | Upgrading debug to BLOCKING flags breaches at iteration time; may produce churn on in-progress persistables |
| Recovery dialog on R8.d (`RecoveredFromBackup`) | Silent + optional light toast | [Silent, Light toast, Modal] | Modal warns player progress was recovered from backup — may alarm for a quiet success | Lighter touch — player doesn't know a recovery happened; acceptable since state is valid |
| Recovery dialog on R8.e (`ResetToDefault`) | **REQUIRED modal** | MODAL only | N/A | Hiding the reset leaves the player confused why their progress vanished; required for trust |
| Migration determinism test iteration count (R5) | 1000 iterations | [100, 10000] | Stronger determinism guarantee; longer CI runtime | Weaker guarantee; faster CI; may miss non-determinism surfacing only in rare conditions |
| Migration JSON size pre-write ceiling (vs `SaveFileSizeBudget`) | 1.0× budget | [0.8×, 1.5×] | Allows migrations that temporarily exceed budget during intermediate steps (v1→v3 via v2); risk of shipping oversized saves | Stricter; catches migration bloat earlier; may force migrators to produce compact output |
| `max_pending_per_key` (D.4) | 2 (fixed) | **Locked** (design axiom) | N/A | Increasing to 3+ breaks R7 snapshot semantics — treat as architecture change, not tuning |
| Root-level `_saveFileVersion` increment policy (R4.b) | One bump per any per-type schema revision | **Locked** | N/A | Sharing version numbers across unrelated schema changes produces ambiguous migration semantics |

### Interaction Between Tuning Knobs

- **Coupled**: `pause_flush_timeout_ms` (D.2) and `safety_margin_factor` (D.1). `FlushLatencyTarget = pause_flush_timeout_ms × safety_margin_factor` — changing one without the other either produces meaningless advisories or suppresses useful ones. Adjust together when tuning for a specific device profile (e.g., minimum-spec Mali-G52 vs. reference iPhone).
- **Coupled**: `flush_requests_per_key_per_frame_limit` (D.4) and the shape of upstream event emission. Tightening the limit without refactoring batch events (e.g., issuing one `gameplay.progression.batch_unlock` instead of N `node_unlocked`) produces warning churn. Change limit only in coordination with the relevant producer GDD.
- **Independent**: `SaveFileSizeBudget` (D.3) and timing budgets (D.1/D.2). Size affects I/O duration only at very large payloads (>256 KB) that are far outside MVP scope.

## Visual/Audio Requirements

**Not applicable.** Save / Load has no runtime visual or audio presence. Two UI-adjacent surfaces are defined elsewhere:

- The `ResetToDefault` recovery modal (AC-INV2) — a required UI element — is owned by the **Modal / Dialog + Level End** GDD (pending), not here. This GDD specifies only that the modal must exist and must block gameplay until acknowledged; the visual treatment and copy are Modal/Dialog's responsibility.
- The `RecoveredFromBackup` optional light toast is a tuning knob (Section G), similarly scoped to the HUD / Modal GDDs.

If a future author considers adding runtime visual/audio to Save / Load (e.g., an in-game "saving…" spinner), route the concern to the HUD GDD. Per R10/R11, normal saves complete in under 100 ms and should not need any indicator; a visible spinner would be a symptom of a broken write pipeline.

## UI Requirements

**No player-facing UI.** Save / Load is runtime infrastructure; players never see its surface directly.

**Editor tooling UX (not a GDD-level concern; noted for scope clarity):**
- Save Manager Editor Window — shows current in-memory state cache per domain, pending coalesced writes, last flush timestamp per key, and `SaveLoadResult.Outcome` from the last load. Mirrors the Data Registry Balance Audit window pattern.
- Mock filesystem toggle — `IFileSystem` injection for PlayMode tests that simulate IOException, disk full, and UnauthorizedAccessException without touching real disk.
- Migration test harness — feeds each registered migrator 1000 fixture inputs (AC-R5) and reports byte-identical output ratio.
- "Reset Save" editor menu command for dev builds — deletes `save.json`, `save.bak`, `settings.json`, and any `save.corrupt-*.json` to force an R8.a FirstRun path.

Editor tooling is implemented alongside Save / Load code and is scoped in Technical Setup / Production — not designed here.

## Cross-References

| This Document References | Target GDD | Specific Element | Nature |
|---|---|---|---|
| Save / Load must handle schema versioning from day one | `design/gdd/data-registry.md` | Section C → Interactions (Save / Load row), soft downstream dependency | Rule dependency — Data Registry declared the requirement; this GDD implements R4.a/R4.b |
| GUID `_id` strings are the stable reference primitive | `design/gdd/data-registry.md` | R4 GUID-based stable IDs | Data dependency — Save / Load uses the `_id` field, never `AssetDatabase` paths |
| Write triggers route through Event Bus Gameplay namespace | `design/gdd/event-bus.md` | R5 Feedback vs Gameplay namespaces; Section C Interactions table | Ownership handoff — Event Bus owns dispatch; Save / Load is a subscriber |
| Auto-managed subscription lifecycle | `design/gdd/event-bus.md` | `SubscribeAutoManaged<T>` pattern | Rule dependency — Save / Load uses the MonoBehaviour-bound auto-unsubscribe path |
| Proposed Event Bus event additions (7 new `gameplay.*` events) | `design/gdd/event-bus.md` | Section C Interactions table | Ownership handoff — this GDD proposes additions; Event Bus GDD must be updated before implementation |
| Non-deterministic physics constrains replay model | `design/gdd/game-concept.md` | Risks → Technical Risks → physics determinism decision | Rule dependency — in-flight Rush Phase state is never persisted; banked-count only |
| 4 Pre-Production ADRs from TD-FEASIBILITY | `design/gdd/game-concept.md` | Pre-Production Action Items #3 | Ownership handoff — ADR-003 (Save Schema Versioning) will formalize this GDD's R4 into wire-format specification |
| Playtest logger is NOT a Save / Load concern | `design/gdd/systems-index.md` | CD-SYSTEMS Playtest Logger resolution | Rule dependency — playtest logger writes `playtest-log-[date].jsonl` directly, outside Save / Load |
| `FlushLatencyTarget` vs Event Bus T1/T2 | `design/gdd/event-bus.md` | D.1 Dispatch Latency Budget | Boundary clarification — Save / Load explicitly does NOT participate in T1 (16.67 ms) or T2 (100 ms) budgets |
| Mali-G52 / 3 GB mobile constraint | `.claude/docs/technical-preferences.md` | Performance Budgets section | Rule dependency — shapes D.2 PauseFlushTimeout and D.4 CoalescingCapacity |

## Acceptance Criteria

34 criteria across 7 categories. Gate classifications: **BLOCKING CI** (must pass for merge) / **ADVISORY** (logged, non-blocking) / **PLAYMODE** (Unity PlayMode runner, CI-safe) / **DEVICE** (requires Mali-G52-class Android or iOS hardware) / **MANUAL** (design review or device observation).

### H.1 Rule Compliance

- **AC-R1 — No main-thread I/O** — GIVEN the full game codebase, WHEN a CI static-analysis grep scans for `File.Write*` / `File.Read*` / `File.Replace` / `FileStream` outside `SaveManager.cs` and `MigrationPipeline.cs`, THEN zero matches appear; any match fails CI. **BLOCKING CI**
- **AC-R2 — Immutable Loaded State** — GIVEN a consumer calls `GetState<LevelProgressState>()` and mutates the returned object, WHEN a second consumer calls `GetState<LevelProgressState>()` in the same session, THEN the second consumer receives the ORIGINAL (unmutated) values; mutation is confined to the returned copy. **PLAYMODE**
- **AC-R3 — Atomic Write Pattern** — GIVEN a `RequestFlush` call completes, WHEN the filesystem is inspected, THEN the write sequence wrote to `save.tmp` first, called `FileStream.Flush(flushToDisk: true)`, then called `File.Replace(save.tmp, save.json, save.bak)`. Verified by mock filesystem intercepting each call in order. **PLAYMODE**
- **AC-R4a — Per-Type Schema Version Present** — GIVEN any persistable POCO type T, WHEN reflection inspects T, THEN `T._schemaVersion` exists as the first declared field, is `public int`, and is not `0`. CI test enumerates all `IPersistableSnapshot` types. **BLOCKING CI**
- **AC-R4b — Root File Version Atomic Advance** — GIVEN a migration run where step 3 of 4 throws, WHEN the load completes, THEN `save.json` is unchanged from pre-migration state (not partially advanced); `_saveFileVersion` on disk still equals the pre-migration value. **BLOCKING CI**
- **AC-R5 — Migration Determinism** — GIVEN each registered migrator, WHEN invoked 1000× on identical input, THEN every invocation produces byte-identical output strings. EditMode unit test per migrator. **BLOCKING CI**
- **AC-R6 — GUID-Only ID References** — GIVEN every persistable field annotated as an asset reference, WHEN inspected by CI grep + regex match on loaded save files, THEN all reference strings match `^[0-9a-f]{32}$`; no path-like strings (containing `/` or `.asset`) appear. **BLOCKING CI**
- **AC-R7 — `IPersistableSnapshot` Marker Enforcement** — GIVEN `RequestFlush<T>`, WHEN T is not tagged with `IPersistableSnapshot`, THEN compilation fails with the generic type constraint. Static analysis additionally flags any payload type whose name contains "Delta", "Diff", "Change", or "Patch" and fails CI. **BLOCKING CI**
- **AC-R10a — Serialization on Main Thread** — GIVEN a `RequestFlush(key, snapshot)` call, WHEN the call stack is traced, THEN `JsonUtility.ToJson(snapshot)` invocation occurs on the Unity main thread (thread ID equals `UnityEngine.PlayerLoopHelper` main); the subsequent `Task.Run` receives only the pre-serialized string. **PLAYMODE**
- **AC-R10b — File I/O Off Main Thread** — GIVEN the Task.Run continuation for a flush, WHEN the file-system operations execute, THEN thread ID is NOT the Unity main thread. Verified by instrumented mock filesystem in PlayMode. **PLAYMODE**
- **AC-R11 — Synchronous Pause Flush** — GIVEN pending coalesced writes and `OnApplicationPause(true)` fires, WHEN the pause handler returns, THEN all pending writes have been serialized to disk OR the `PauseFlushTimeout` elapsed without partially writing `save.json`. **PLAYMODE** (lock correctness); **DEVICE** (actual Android lifecycle timing)
- **AC-R12 — JsonUtility Format Compliance** — GIVEN every persistable POCO type, WHEN round-tripped through `JsonUtility.ToJson` → `FromJson`, THEN field-count parity holds (no silent field-drop), no `Dictionary<>` / `DateTime` / polymorphic / nullable fields exist, no field is `float.NaN` / `Infinity`. EditMode test per type. **BLOCKING CI**
- **AC-R13 — Settings Bypass Isolation** — GIVEN `SettingsManager.SetAudioMaster(0.5f)`, WHEN the Event Bus is inspected, THEN no event is fired and no subscription on the bus receives any notification. `settings.json` is updated directly via Save / Load's file utilities. **PLAYMODE**
- **AC-R9 — State-vs-Rules Boundary** — GIVEN every `IPersistableSnapshot` POCO type, WHEN reviewed by a design-review checklist, THEN no field holds a gameplay tuning value (cost, threshold, rate, duration) — only IDs, counts, flags, timestamps, and balances. Lead-designer signoff at end of Save / Load implementation sprint. **MANUAL**

### H.2 Formula Correctness

- **AC-D1 — FlushLatencyTarget Advisory Only** — GIVEN a simulated 3000 ms I/O stall on a write, WHEN the flush completes, THEN the write succeeds, `[SaveManager] FlushLatency exceeded target` logs in debug, and no exception is thrown. **PLAYMODE**
- **AC-D2 — PauseFlushTimeout Hard Wall** — GIVEN a simulated 800 ms I/O stall during synchronous pause-flush with timeout = 500 ms, WHEN the timeout elapses, THEN the in-progress Task is cancelled, `save.json` is unchanged (not partially written), and control returns to the pause handler. **PLAYMODE**
- **AC-D3a — SaveFileSizeBudget Release-Build Enforcement** — GIVEN a test fixture producing a 70 KB serialized save, WHEN a release build's CI job runs, THEN the job fails with `[SaveManager] Size budget exceeded: 71680 B > 65536 B`. **BLOCKING CI**
- **AC-D3b — SaveFileSizeBudget Debug Advisory** — GIVEN the same 70 KB fixture, WHEN a development build runs, THEN the advisory logs but the build succeeds. **PLAYMODE**
- **AC-D4 — CoalescingCapacity Per-Key Cap** — GIVEN 100 `RequestFlush` calls on a single key in one frame, WHEN the queue state is inspected, THEN exactly 2 payload slots (in-flight + next-pending) are populated and exactly 1 `[SaveManager] Runaway flush producer detected` warning is logged. **PLAYMODE**

### H.3 Edge Cases

- **AC-EC-Storage1 — Disk-Full Retention** — GIVEN `File.Replace` throws `IOException` mid-flush due to disk full, WHEN the handler catches, THEN `save.tmp` is retained (not deleted), `save.json` is unchanged, `[SaveManager] Disk full — save.tmp retained` is logged, and state transitions to `CorruptedRecovery`. **PLAYMODE** (mocked IO)
- **AC-EC-Storage2 — iOS Locked File Distinction** — GIVEN `File.Replace` throws `UnauthorizedAccessException`, WHEN the handler catches, THEN `save.json` is NOT renamed to `save.corrupt-*`; the failure is treated as transient, retry on next focus. **PLAYMODE**
- **AC-EC-Migration1 — Downgrade Detection** — GIVEN `save.json` contains `_saveFileVersion = 99` and code's current version is 3, WHEN load attempts migration, THEN no forward migrator is invoked; R8.d fires if `save.bak` is compatible; R8.e fires otherwise. **BLOCKING CI**
- **AC-EC-Migration2 — Migration Chain Gap** — GIVEN migrators registered for versions 1 and 3 but not 2, WHEN build time runs a chain-completeness check, THEN CI fails with `[MigrationPipeline] No migrator for v1 → v2`. **BLOCKING CI** (build-time)
- **AC-EC-Race1 — Pause During Load** — GIVEN `State == LoadingFromDisk` and `OnApplicationPause(true)` fires, WHEN the pause handler executes, THEN no synchronous flush is attempted, `[SaveManager] Pause during load — flush skipped` logs, and the OS pause callback returns within 50 ms. **PLAYMODE**
- **AC-EC-Race2 — Dispose During In-Flight Write** — GIVEN a Task.Run write is in-flight and `SaveManager.OnDestroy()` fires, WHEN dispose runs, THEN `task.Wait(PauseFlushTimeout)` is called; if it succeeds, disposal proceeds cleanly; if it times out, the task is abandoned (not cancelled) and the tmp file remains on disk for R8.b recovery. **PLAYMODE**
- **AC-EC-Clock — Future Timestamp Clamp** — GIVEN `SessionState.lastPlayedAt = UtcNow.Ticks + 1000000000`, WHEN the resume UI check runs, THEN the effective value is clamped to `UtcNow.Ticks`, and the resume message fires conservatively. **PLAYMODE**
- **AC-EC-Stale — Stale Tool ID Resolution** — GIVEN a save references `ToolConfig` ID `"aaa...000"` which no longer exists in the Data Registry, WHEN `ToolOwnershipState` loads, THEN the ID is retained in state (not deleted), the Tool System filters it from the available loadout silently, and a debug `gameplay.tool.stale_reference` event fires. **PLAYMODE**

### H.4 Performance & Platform

- **AC-P1 — Async Flush Latency p99** — GIVEN all 6 MVP coalescing keys driven at max frequency for 60 seconds on a Mali-G52 emulator image, WHEN measured, THEN p99 async flush latency ≤ 2000 ms. **DEVICE**
- **AC-P2 — Pause Flush Within 500 ms** — GIVEN 6 pending coalesced writes (total ~40 KB) and `OnApplicationPause(true)` on Mali-G52-class hardware, WHEN measured, THEN all writes complete or abort within 500 ms wall-clock. **DEVICE**
- **AC-P3 — Save File Size at MVP Scope** — GIVEN a full MVP save (5 biomes authored, 40 levels completed with 3 stars each, 4 tools owned, all progression unlocked), WHEN serialized, THEN `save.json` size ≤ 32 KB (well under the 64 KB budget). **BLOCKING CI**
- **AC-P4 — JsonUtility Serialization Cost** — GIVEN the full MVP save state, WHEN `JsonUtility.ToJson` is profiled on Mali-G52-class hardware, THEN serialization completes in ≤ 5 ms. **DEVICE**

### H.5 Consumer Integration

- **AC-INT1 — Currency & Economy Round-Trip** — GIVEN Currency & Economy modifies balance and calls `RequestFlush("currency", CurrencyState)`, WHEN scene reloads, THEN the loaded `CurrencyState` matches the pre-flush state field-for-field. **PLAYMODE**
- **AC-INT2 — Progression First-Run Defaults** — GIVEN no save exists (R8.a FirstRun), WHEN Progression requests state, THEN the returned `ProgressionNodeState` contains the initial unlock set declared by `ProgressionNode` root flags in Data Registry. **PLAYMODE**
- **AC-INT3 — Settings Across Sessions** — GIVEN `SettingsManager.SetAudioMaster(0.25f)` is called, WHEN the app is killed and relaunched, THEN `SettingsManager.GetCurrent().AudioMaster == 0.25f`. **PLAYMODE**
- **AC-INT4 — Level Runtime Resume-from-Interruption** — GIVEN a process kill during Rush Phase with last banked event having fired, WHEN the app relaunches, THEN the biome map shows the pre-kill banked currency total, `SessionState.lastPlayedLevelId` points to the in-progress level, and no in-flight Rush state is restored. **PLAYMODE**

### H.6 Invariants (What Must NOT Happen)

- **AC-INV1 — No Silent Data Loss on Recovery** — GIVEN any R8 recovery outcome is triggered, WHEN recovery runs, THEN EITHER (a) `SaveLoadResult.Outcome` reports exactly which path fired (`RecoveredFromTmp` / `RecoveredFromBackup` / `ResetToDefault`) AND the corrupt original is preserved as `save.corrupt-[timestamp].json`, OR (b) `LoadedClean` fires with no renaming. Zero paths discard state silently. **BLOCKING CI**
- **AC-INV2 — R8.e Triggers Required Modal** — GIVEN `SaveLoadResult.Outcome == ResetToDefault`, WHEN the first post-load UI frame renders, THEN a blocking modal is presented to the player referencing the preserved `save.corrupt-*.json` path; gameplay does not advance until the modal is acknowledged. **PLAYMODE**
- **AC-INV3 — No Cross-Writer Race on save.tmp** — GIVEN a flush is in-flight via Task.Run AND `OnApplicationPause(true)` fires, WHEN both attempt to write, THEN only one holds the single-writer lock at any instant; the pause handler `task.Wait(timeout)` before initiating its own write; no interleaved byte-stream corruption occurs in `save.tmp`. **PLAYMODE**
- **AC-INV4 — No Event Bus Subscription Leaks** — GIVEN Save / Load subscribes via `SubscribeAutoManaged`, WHEN `SaveManager` is destroyed and the scene reloads, THEN all subscriptions are unregistered before the next scene's `Awake`; Event Bus debug count shows zero leaked handles from Save / Load. **PLAYMODE**

### H.7 Schema Migration

- **AC-M1 — Current-Schema No-Op** — GIVEN a save written at the current `_saveFileVersion`, WHEN loaded, THEN no migrator is invoked, and the loaded state is byte-identical (after round-trip) to the pre-load state. **BLOCKING CI**
- **AC-M2 — Multi-Step Forward Migration** — GIVEN a save at `_saveFileVersion = 1` and current code version = 3, WHEN loaded, THEN migrators run in order v1→v2, then v2→v3; final state's `_saveFileVersion == 3`; no intermediate state is written to disk until the full chain succeeds. **BLOCKING CI**
- **AC-M3 — Per-Type Version Updated by Migrator** — GIVEN migrator v1→v2 runs on `LevelProgressState` (per-type `_schemaVersion = 1`), WHEN it completes, THEN the resulting state has `_schemaVersion = 2`; a subsequent load does not re-apply the migrator. **BLOCKING CI**
- **AC-M4 — Oversized Migration Rejected** — GIVEN a migrator whose output would exceed `SaveFileSizeBudget`, WHEN the pipeline checks post-migration size, THEN the pipeline aborts, preserves the pre-migration file as `save.corrupt-[timestamp].json`, and returns `ResetToDefault`. **BLOCKING CI**

### Gate Summary

| Gate Level | Count | Coverage |
|---|---|---|
| **BLOCKING CI** | 17 | R1, R4a/b, R5, R6, R7, R12 + migration chain check + R8 five outcomes + INV1 + AC-D3a + AC-P3 + M1-M4 |
| **PLAYMODE** | 13 | R2, R3, R10a/b, R11 (lock), R13, AC-D1, AC-D2, AC-D3b, AC-D4, EC-Storage1/2, EC-Race1/2, EC-Clock, EC-Stale + INT1-4 + INV2/3/4 |
| **DEVICE** | 3 | AC-P1 (flush p99), AC-P2 (pause flush wall), AC-P4 (serialize cost) |
| **MANUAL** | 1 | AC-R9 (state-vs-rules design review) |

**Shift-left note for implementation**: the 17 BLOCKING CI criteria should have stub EditMode test files created at story start, before any `SaveManager` class is authored. The 13 PLAYMODE tests require a test scene fixture with a mockable filesystem layer (`IFileSystem` interface wrapping `File.*` calls) — scaffold this during the same sprint as `SaveManager` implementation, not deferred to QA hand-off.

## Open Questions

| # | Question | Owner | Target Resolution |
|---|---|---|---|
| OQ-1 | **Does level-fail forfeit banked coins from the current Rush Phase?** Save / Load persists whatever Economy commits; the gameplay rule (fail → zero banked, fail → keep banked, fail → keep half banked) is Level Runtime's + Economy's to define. If forfeit, Level Runtime must fire `gameplay.currency.changed` zeroing the current-run banked before firing `gameplay.level.failed`; if keep, it fires them in either order. | game-designer + Level Runtime GDD + Currency & Economy GDD | During Level Runtime + Currency & Economy GDD authoring |
| OQ-2 | **Cloud Save provider choice** — Unity Cloud Save vs. Firebase vs. Google Play Games Services + GameCenter. Decision deferred to plumbing sprint (months 4-5 per producer roadmap). Save / Load's local contract is provider-agnostic; this question only affects the post-launch Cloud Save adapter. | technical-director + producer | Plumbing sprint (months 4-5) |
| OQ-3 | **Content hash in save file root** — add `_fileHash: string` field to detect external modification (platform backup corruption, user file-manager edits, manual hex edits). At MVP the security surface is low (local single-player, no leaderboards); at post-launch (Cloud Save + leaderboards + competitive play) this may matter. A `SaveFileSecurity` ADR would capture the decision. | security-engineer + technical-director | Post-launch, before leaderboard ship |
| OQ-4 | **Proposed Event Bus event additions** — 7 new `gameplay.*` event IDs this GDD assumes (`currency.changed`, `tool.purchased`, `tool.upgraded`, `cosmetic.purchased`, `progression.node_unlocked`, `biome.unlocked`, `session.level_selected`). These must be added to Event Bus's Interactions table before the Save / Load architecture sprint starts. Should this be a Section C addition to Event Bus's GDD, or a separate follow-up patch? | lead-programmer + Event Bus GDD owner | Before Event Bus architecture sprint (likely as a small patch on the already-pending Event Bus review) |
| OQ-5 | **Per-platform `PauseFlushTimeout` relaxation** — iOS reliably gives ~5 s of grace; the 500 ms shared default wastes budget there. Should the default ship per-platform (iOS = 800 ms via `#if UNITY_IOS`, Android = 500 ms)? Decision deferred until post-MVP profiling on target iPhone hardware confirms the headroom. | performance-analyst + unity-specialist | Post-MVP device profiling pass |
| OQ-6 | **Addressables for save migration** — when Full Vision ships (5 biomes × 100 levels + cosmetics + analytics), the migration pipeline itself may become non-trivial to embed in the main binary. Should migrators ship as Addressable assets loadable on demand? Trigger condition: migration pipeline binary weight > 500 KB OR migration test suite > 10 min runtime. | technical-director | Post-launch when v2.x schema evolves |
| OQ-7 | **Ads SDK state storage key** — Ads SDK (v1.0) needs to persist frequency caps + last-shown timestamps. Does this go under a new `ad_history` coalescing key on `save.json`, or as a separate `ad_state.json` file under the same pattern as `settings.json`? `save.json` keeps it backed up to iCloud (user expectation: ad history travels with save); separate file keeps it device-local. | lead-programmer + Ads SDK GDD owner | During Ads SDK GDD authoring (v1.0 scope) |
