# Save / Load

> **Status**: Revised pass 3 — pending targeted pass-4 verification or direct acceptance per CD Event Bus precedent
> **Author**: Robert Michael Watson + game-designer / systems-designer / unity-specialist / gameplay-programmer / qa-lead / creative-director
> **Last Updated**: 2026-04-21 (pass-3 fix pass — all pass-1 + pass-2 blockers addressed; see [review log](reviews/save-load-review-log.md))
> **Implements Pillar**: Foundation for Pillar 3 (Mastery Through Economy — progression must persist) and Pillar 4 (Prep + React — level retry state must be durable). No direct player fantasy; enables every persistent system.
>
> **Creative Director Review (CD-GDD-ALIGN)**: APPROVED 2026-04-20 — no concerns. "One more level without the asterisk" framing is the strongest Pillar 3 articulation in the repo; R9 (state-vs-rules boundary) protects no-pay-to-win at MVP; Modal punt correctly scoped to Modal/Dialog GDD; T1/T2 latency exclusion architecturally correct. OQ-3 (save tamper-proofing) flagged for re-evaluation before any competitive/leaderboard feature ships — future gate, not current block.
>
> **Pass-3 Revision Summary (2026-04-21)**: R10 rewritten (off-main serialize + I/O; POCO self-containment rule R10.a added); R11 converted to pure semaphore-only pause path (no `task.Wait()`); R12 banned-list expanded to exhaustive; R8.b added stale-tmp version-ordering guard; R3 fsync caveat documented with architectural mitigation; R7.a producer coherency rule added; R4 disambiguated (root-driven + invariant table R4.c); D.1/D.2/D.3 formulas re-derived; D.3 split into FileSizeEstimate + FileSizeBudget; Player Fantasy rewritten in career-progress framing; OQ-1 closed (keep-banked committed); OQ-4 closed (7 events patched into Event Bus GDD in same session); AC table regenerated to 60 body-verified identifiers with R9/R11 splits and 7 coverage-gap ACs. See [review log pass-3 revision entry](reviews/save-load-review-log.md).

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

What Save / Load actually delivers is **trust in ownership across sessions**. Every biome unlocked, every tool purchased, every coin banked — those are permanent. The smug little heist accumulates. Pass-2 correction (career-progress framing, not single-run state): Money Miner's non-deterministic 2D physics means no in-flight Rush Phase can be faithfully replayed across a process kill, and the Resume Policy (Section C) lands the player on the Biome Map with banked state intact rather than mid-run. The fantasy is NOT "the level I was playing resumes where I left off" — the fantasy is "the work I had already finished is still mine."

It is 11:47 at night and the player has already said "one more level" for the fourth time. They complete the level. They bank. They tap the next level button. The phone drops to 3% battery. iOS force-quits the app. They plug in, they relaunch — Biome Map, coin count right, biome just unlocked still unlocked, tools still owned, star counts intact. The in-progress level they had just started is gone (it was 30 seconds in and the physics weren't deterministic anyway). But everything banked and unlocked is waiting for them. The heist crew didn't evaporate just because the phone did — they're a professional operation, and the vault is exactly where the player left it.

The fantasy is that the player can lean in on the career arc. Money Miner's core loop ("I'm too clever for them") only works when the smug little heist gets to accumulate — three stars here, a new tool purchased there, a biome unlocked, a shop balance slowly climbing. Every one of those moments is a tiny ownership contract between the player and the game, and Save / Load is the bookkeeper that makes each contract real instead of performative. Without it, the failure mode is specific and corrosive: thirty minutes of accumulated career progress evaporates to an OS memory kill, and now every future "one more level" decision is shadowed by a quiet *"…but what if."* The smugness curdles into caution — and caution is the death of mobile casual engagement.

The narrow loss-window — the seconds between event emission and `Task.Run` completion of an in-flight level-complete write — is architecturally minimized by R11 (synchronous pause flush), R3 (atomic tmp+rename), and the fact that Task.Run I/O on a 10–30 KB file runs in 15–80 ms on reference hardware. The overwhelming common case is that **any event-marked progress survives**: a banked level, an unlocked biome, a purchased tool. Only the literal current-frame of an in-progress Rush Phase is lost, and only when a force-quit beats the pause handler. Players will not perceive this as a save-system failure — they will perceive it as having left a level unfinished and needing to restart it, which is an entirely normal mobile interruption.

Players won't articulate it. They'll just keep tapping the icon, trusting without thinking, committing without flinching on purchases and unlocks. Save / Load succeeds when **"one more" never comes with an asterisk on the career** — the single-run asterisk (mid-level state) is accepted as a physics-constraint cost, documented, and bounded by the Resume Policy.

## Detailed Design

### Core Rules

**R1. No main-thread file I/O.** All `File.*`, stream, and disk operations execute on `Task.Run` threads. Main-thread code calls `SaveManager.RequestFlush(key, payload)` and `SaveManager.GetState<T>()` only — never direct file access. Enforced by CI static-analysis grep banning `File.Write*` / `File.Read*` outside `SaveManager.cs` and `MigrationPipeline.cs`.

**R2. Immutable loaded state.** On load, each persistable domain is deserialized once into `SaveManager`'s internal authoritative cache. `GetState<T>()` returns a copy (deep-clone or re-deserialization), never the authoritative reference. Consumers may mutate their copy and pass it back via `RequestFlush`; they may not mutate what they received. Enforced by unit test that mutates a returned state and asserts the cached version is unchanged.

**R3. Atomic write with 1-deep backup.** Every flush sequence is: serialize → write to `save.tmp` → `FileStream.Flush(flushToDisk: true)` → close → `File.Replace(save.tmp, save.json, save.bak)`. `save.bak` holds the previous good copy, rotated on every successful replace. Any exception before `File.Replace` returns leaves `save.json` untouched. Enforced by integration test that aborts the write thread mid-flush and verifies `save.json` remains the prior valid version.

**Durability caveat — best-effort on mobile.** `FlushToDisk: true` maps to `fsync` on iOS APFS (reliable for the app sandbox) and Android ext4/F2FS (reliable on Pixel / stock kernels), but Samsung OneUI 3.x–4.x on eMMC-backed A-series devices is known to return from `fsync()` before the eMMC write cache is flushed (the kernel honors the syscall contract for the mounted filesystem, but the underlying eMMC controller batches writes with its own cache). Consequence: an `fsync`-acknowledged `save.tmp` may still be lost if the OS force-kills within ~20–80 ms of the apparent flush. Mitigation is **architectural not syscall-level**: (a) R8.b tmp-promotion recovers any in-flight tmp on next cold boot; (b) R11's synchronous pause handler trades latency for durability by holding the main thread until the full atomic sequence completes; (c) the kill-app-via-recent-tasks exploit documented in Edge Cases already accepts this window. No code-level `fsync`-retry loop is added — it would not change the OEM behavior. Flagged for monitoring: if post-launch telemetry shows Samsung A-series OEM showing disproportionate save-loss events, revisit with an eMMC-aware durability option (possible: a double-write pattern to a secondary path).

**R4. Two-level schema versioning — root is the migration driver; per-type is a consistency assertion.**

- **R4.a Per-type schema version.** Every persistable POCO declares `public int _schemaVersion` as its first field. Serializer asserts the field is present and non-zero at both write and load time; `_schemaVersion = 0` is treated as corrupt. Per-type versions are **assertion-only** — they are NOT migration drivers. They exist so that a consumer system can detect a migration bug (e.g., v2→v3 migrator forgot to bump `LevelProgressState._schemaVersion` from 1 to 2) at load time and fail loud rather than silently proceed on stale data.
- **R4.b Root-level file version — the migration driver.** The top-level `SaveFileRoot` container declares `public int _saveFileVersion`, incremented by one whenever ANY per-type schema is revised (see R4.c for the invariant table). The migration pipeline keys solely on `_saveFileVersion`: migrators are registered as `(rootFrom: int) → (rootTo: int)` with the signature `MigrateVN_to_VM(string json) → string json`. Each migrator internally patches whichever per-type POCOs changed at that root version, bumping each per-type `_schemaVersion` to the values declared in the R4.c invariant table. This eliminates the ambiguity of "which version does the migrator key on" (answer: only root).
- **R4.c Root-to-per-type version invariant table.** For every value of `_saveFileVersion`, a static compile-time table declares the expected per-type `_schemaVersion` of every `IPersistableSnapshot` type. Example:

  | `_saveFileVersion` | `LevelProgressState._schemaVersion` | `CurrencyState._schemaVersion` | ... |
  |---|---|---|---|
  | 1 | 1 | 1 | ... |
  | 2 | 2 | 1 | ... (only LevelProgressState changed at v2) |
  | 3 | 2 | 2 | ... (only CurrencyState changed at v3) |

  On load, after `_saveFileVersion` migration completes, the pipeline cross-checks every per-type `_schemaVersion` against this table. A mismatch triggers `ResetToDefault` (R8.e) and preserves the original as `save.corrupt-[timestamp].json` for support diagnostics — the migration is broken and no further load attempts are safe. CI build-time assertion: the table is gapless from v1 to `CurrentCodeVersion` and every declared `IPersistableSnapshot` type has an entry at every version.

- **R4.d Atomic version advance.** A migration run that fails mid-way does not partially commit. Either all per-type migrations succeed and `_saveFileVersion` is advanced atomically on write, or the original `save.json` is preserved and the run aborts to `CorruptedRecovery`. This prevents the mixed-schema save file that no migrator can handle.

**R5. Migration determinism.** Each migrator is keyed `(rootFromVersion: int) → (rootToVersion: int)` per R4.b with signature `MigrateVN_to_VM(string json) → string json`. Each migrator is a pure function: no `UnityEngine.Random`, no `DateTime.Now`, no `UnityEngine.Object` access, no I/O, no `PlayerPrefs`. Enforced by unit test that runs each migrator 1000× on identical input and asserts byte-identical output.

**R6. ID references, never asset paths.** Every cross-reference to a Data Registry asset uses the GUID `_id` string (e.g., `"a1b2c3d4e5f6..."`). `AssetDatabase.GetAssetPath` does not exist at runtime and is banned in save-domain files by CI grep. Runtime assertion in `SaveManager.Load()` validates every loaded ID string against the GUID regex `^[0-9a-f]{32}$`. An ID that does not resolve to a loaded Data Registry asset is logged as a *stale reference*, not an error — the save is valid, but the referenced entity no longer exists (e.g., a tool renamed after release).

**R7. Snapshot-based coalescing — never deltas.** Write payloads must represent the FULL current state of their domain, not a delta against the prior state. Consumer systems build a complete `ToolOwnershipState` / `CurrencyState` / etc. and pass it to `RequestFlush`; the save manager stores it as the in-flight payload for that coalescing key and discards any earlier pending payload without logging. The discarded payload is superseded, not lost — but only because the newer payload is a complete snapshot. Delta-shaped payloads passed to `RequestFlush` are a contract violation caught by a compile-time type constraint (`IPersistableSnapshot` marker interface) and by a CI static-analysis rule.

**R7.a Producer-side coherency for shared coalescing keys.** Multiple producer systems may target the same coalescing key (e.g., `gameplay.level.complete` and `gameplay.level.banked` both write the `currency` key). Each producer MUST build its snapshot from the **current authoritative state** (via `GetState<CurrencyState>()` to obtain the latest cached value, then apply its own mutation, then pass the resulting full snapshot to `RequestFlush`). Two producers emitting within the same frame on the same key therefore observe read-modify-write semantics through the cache: the second producer sees the first's mutation because `SaveManager` updates its authoritative cache on every `RequestFlush` acceptance (snapshot-semantic in-memory as well as on-disk). A producer that builds its snapshot from its own local shadow copy instead of calling `GetState<>()` risks overwriting the other producer's change. Enforced by design-review inspection at each producer's implementation story (not CI-enforceable; producers are Currency & Economy / Level Runtime / Tool System / etc. — each story must reference R7.a in its acceptance criteria). This rule is the reason coalescing is safe even when multiple unrelated producers share a key.

**R8. Corrupted-save recovery is non-destructive and surfaceable.** The load pipeline distinguishes five distinct outcomes, each represented in `SaveLoadResult`:
- **R8.a `FirstRun`** — no `save.json`, no `save.tmp`, no `save.bak`. Fresh install or post-delete-data. Consumers receive default states; UI may show onboarding/welcome flow.
- **R8.b `RecoveredFromTmp`** — `save.tmp` exists at cold boot (proof of a process kill during the rename window) AND parses as valid, well-formed JSON with a valid `_saveFileVersion` **AND `save.tmp._saveFileVersion >= save.json._saveFileVersion`** (stale-tmp guard: rejects an abandoned tmp left over from a prior code version). Only when all three conditions hold is `save.tmp` promoted to `save.json`. If `save.json` is unreadable (absent, corrupt JSON, unreadable file), the version-ordering comparison is skipped and the tmp is accepted provided it parses valid with a version ≤ the current code version. Version-ordering guard prevents a stale tmp from silently downgrading a successful later save. Recovers a complete write that would otherwise be silently discarded.
- **R8.c `LoadedClean`** — normal path. `save.json` loads, migrates, validates.
- **R8.d `RecoveredFromBackup`** — `save.json` fails schema validation or JSON parse. Original is renamed to `save.corrupt-[unix-timestamp].json`; `save.bak` is promoted to `save.json`; load retries. UI may surface a mild "some progress recovered" message.
- **R8.e `ResetToDefault`** — both `save.json` and `save.bak` fail to load. Defaults returned for all domains. UI MUST surface a clear "progress was reset, corrupted save preserved for support at `save.corrupt-*.json`" message. Save / Load never silently discards progress.

Enforced by integration tests for each outcome branch.

**R9. Save / Load owns state, not rules.** Save / Load stores *which* progression nodes are unlocked, *which* tools are owned, *what* currency balance is — it never stores unlock thresholds, prices, or quota values. Those live in Data Registry. If a Data Registry asset changes its unlock cost after a save was written, the player's already-unlocked state is unaffected; the new cost applies only to future purchases. Enforced by design-review grep: no Data Registry tuning values (costs, thresholds, quotas) may appear in any persistable POCO type.

**R10. Snapshot build on main; serialization AND I/O off main.** `JsonUtility.ToJson` and `FromJson` ARE thread-safe for pure POCOs — types with no `UnityEngine.Object` references, no `AssetDatabase` access, and no other main-thread-only Unity API dependencies. This has been a documented Unity invariant since 2020.1 (JsonUtility is a pure reflection walker over serializable fields with no internal main-thread coupling on POCO types). Pass-2 review corrected a prior authored rationale on this point; this revision adopts off-main serialization as the governing contract.

The handler sequence is:

- **Main thread — snapshot build only.** Read live runtime state (`Transform`, `ScriptableObject` fields, `ToolConfig` references resolved to GUID `_id` strings, etc.) into a plain-C# POCO that satisfies R12 + R10.a below. No `JsonUtility` call on main.
- **`Task.Run` — serialize + write.** The POCO (plus target filename) is handed to a background worker. `JsonUtility.ToJson(snapshot)` runs off-main, producing the JSON string; the string is written through the atomic tmp→fsync→Replace pattern (R3).
- **Continuations inside the worker use `.ConfigureAwait(false)`.** Enforced by CI grep (AC-R10d): every `await` in `SaveManager.cs` and `MigrationPipeline.cs` must be followed by `.ConfigureAwait(false)` unless the preceding comment explicitly justifies a main-thread continuation. This prevents IL2CPP deadlock when the pause handler (R11) holds the main thread while a Task.Run flush completes.

**R10.a POCO self-containment (load-bearing).** Any persistable field whose declared type derives from `UnityEngine.Object` (including `GameObject`, `Component`, `ScriptableObject`, and asset references such as `ToolConfig`) MUST be resolved to its GUID `_id` string on the main thread during snapshot build, and stored in the POCO as `string`. No `UnityEngine.Object`-derived type appears in any persisted field. Enforced at two layers:

1. **CI static-analysis (AC-R10c — BLOCKING CI).** Scan every `IPersistableSnapshot`-implementing type for fields whose declared type derives from `UnityEngine.Object`. Zero matches.
2. **EditMode round-trip (AC-R12 extension).** Every persistable POCO round-trips through `JsonUtility.ToJson` + `FromJson` on a worker thread (`Task.Run`) in the test; a main-thread-only dependency surfaces as a deserialization exception or field-drop. Failure blocks merge.

**Pause-path exception.** R11's synchronous pause flush runs on the main thread under the OS grace window (the main thread is blocked on `OnApplicationPause` anyway). There, `JsonUtility.ToJson` + write execute inline on main — this is allowed because serialization is thread-safe (so running it on main is trivially safe) and the OS grace budget of 1–5 seconds absorbs the cost. The rule "off-main normally" applies to the async path; the pause path is a bounded synchronous exception.

**R11. Pause/focus triggers synchronous flush under a single-writer lock — no `task.Wait()`.** `OnApplicationPause(true)` and `OnApplicationFocus(false)` drain the coalescing queue **synchronously** on the main thread. The OS gives approximately 1–5 seconds of CPU time on mobile before kill; async bridging on this path is unsafe under IL2CPP.

**Pause-path contract:**

- The `SaveManager` holds a single `SemaphoreSlim(1, 1)` instance (`_writeLock`) that gates all writes to `save.tmp` / `save.json` / `save.bak`. All async flush paths (Task.Run) acquire `_writeLock` via `await _writeLock.WaitAsync().ConfigureAwait(false)` before touching the filesystem and release in `finally`.
- The pause handler calls `_writeLock.Wait(pause_flush_timeout_ms)` — the **synchronous** overload — on the main thread. This blocking acquire does NOT spin up `async`/`await` machinery and does NOT capture `SynchronizationContext`, so it cannot deadlock against an off-main Task.Run continuation that uses `ConfigureAwait(false)` per R10. Critically, `task.Wait()` on an async Task is NOT used anywhere in the pause path — the IL2CPP deadlock vector is eliminated architecturally.
- On acquire, the pause handler iterates the flush queue and performs `JsonUtility.ToJson` + atomic-write inline on the main thread (serialization is thread-safe per R10, so running it on main is trivially safe; the OS grace budget absorbs the cost — see D.2 worked example).
- On timeout (semaphore not acquired within `pause_flush_timeout_ms`), the pause handler abandons the wait and yields control back to the OS. The in-flight Task.Run continues; if it completes before kill, the pending payload is durable; if it does not, the previous `save.json` remains intact per R3 atomic-write and R8.b's tmp-promotion recovers the in-progress write on next cold boot.
- `_writeLock` MUST be disposed in `SaveManager.OnDestroy()` after a final synchronous drain (the `Disposed` state transition). Failure to dispose leaks a kernel handle per `SemaphoreSlim` instance across domain reloads in the Unity Editor and is caught by the `Dispose()`-path integration test (AC-EC-Race3 below).

The single-writer lock prevents two threads from racing on `save.tmp`. No path in the system calls `Task.Wait()` on a Task whose completion depends on a continuation being posted to the main thread.

**R12. Persistable schema format constraints.** Every field of every persistable type must be directly serializable by Unity `JsonUtility`. `JsonUtility` uses the same reflection rules as the inspector — silent field-drop (the top failure mode) occurs whenever a declared field fails Unity's serialization criteria. The banned-list below is exhaustive for MVP + VS + v1.0 scope; additions require an explicit pass-review.

**Banned declared field forms — required replacements:**

- **Dictionary**: `Dictionary<K, V>` → `List<FooKVPair>` where `FooKVPair` is a **concrete, non-generic** `[Serializable]` struct (e.g., `[Serializable] struct ToolOwnershipKVPair { public string _id; public int tier; }`). Do NOT use `KVPair<K, V>` — **generic struct type parameters are dropped by `JsonUtility` at runtime**, producing a silently empty object. Every Dictionary-replacement pair type is declared once per consuming domain with a unique concrete name.
- **HashSet / SortedSet / generic collections other than `List<T>`**: `HashSet<T>` / `SortedSet<T>` → `List<T>` + application-level uniqueness invariant (asserted in EditMode tests). `Queue<T>` / `Stack<T>` → `List<T>` ordered by index; enqueue/dequeue semantics reconstructed by consumer.
- **Nested lists**: `List<List<T>>` is silently dropped at the outer level by `JsonUtility`. Replace with a flattened `List<FooRow>` where `FooRow` is a concrete `[Serializable]` struct holding a `List<T>` field.
- **Polymorphic base-class fields**: `AbstractBase` references → flat discriminated-union struct with an `int _kind` tag and all union-member fields declared inline (union-member values default to their type's zero if `_kind` does not match).
- **`readonly` / `init` fields**: `JsonUtility` silently skips `readonly` and `init` fields on deserialize. All persistable fields are plain `public` mutable fields (backing-store style); assertion via `FieldInfo.IsInitOnly == false` in the schema-validation test.
- **Properties (auto-implemented or otherwise)**: `JsonUtility` does NOT serialize properties, period. Every persistable member is a `public` field. Properties in a persistable type are allowed only as convenience accessors over fields, and MUST be tagged `[field: NonSerialized]` to avoid accidental confusion.
- **`record` / `record struct` types**: `record` types rely on compiler-generated positional constructors and `init`-only properties; incompatible with `JsonUtility`'s field-reflection model. Banned.
- **Interface-typed fields** (`IPersistable`, `IFoo`): fields typed as an interface are silently dropped (`JsonUtility` cannot instantiate an interface). Use concrete types or the `_kind`-tagged union pattern.
- **`DateTime` / `DateTimeOffset` / `TimeSpan`**: → `long` ticks (UTC for `DateTime`; `(offset * TimeSpan.TicksPerHour)` paired field for `DateTimeOffset`; raw `.Ticks` for `TimeSpan`).
- **`Guid`**: → `string` (32-char lowercase hex, format `"N"`), matching Data Registry R4 `_id` format.
- **Nullable value types** (`int?`, `long?`, `float?`): → sentinel values (`-1`, `long.MinValue`, `float.NegativeInfinity` explicitly clamped before serialize) documented per-field. The sentinel is part of the persistable schema contract — changing a sentinel is a schema bump.
- **`float.NaN` / `float.PositiveInfinity` / `float.NegativeInfinity`**: banned as persisted values. MUST be clamped to valid finite values (typically `0f` or a per-field default) before serialize; JSON-encoded `NaN`/`Infinity` is not portable across parsers and is silently corrupted on round-trip. The schema-validation test fails the POCO on any field value matching these.
- **`UnityEngine.Object`-derived types** (`GameObject`, `ScriptableObject`, `Component`, `ToolConfig`, any asset reference): banned per R10.a. Substitute the GUID `_id` string at snapshot-build time on the main thread.

**Concrete KVPair naming convention.** One concrete struct per Dictionary-replacement site:

```csharp
[Serializable] public struct ProgressionNodeKVPair { public string _id; public bool unlocked; }
[Serializable] public struct ToolOwnershipKVPair   { public string _id; public int tier; }
[Serializable] public struct CurrencyBalanceKVPair { public string currencyId; public long balance; }
```

Naming: `{DomainName}KVPair`. Declared in the same file as the owning POCO (e.g., `ToolOwnershipState.cs` declares `ToolOwnershipKVPair`).

Enforced by a schema-validation EditMode test (AC-R12) that round-trips each persistable type through `JsonUtility.ToJson` / `FromJson` **on a Task.Run worker thread** (catches main-thread-dependency leaks per R10.a) and asserts field-count parity with the live object on a per-field-name basis. Silent field-drop is the top JsonUtility failure mode and is the reason R12's banned-list is exhaustive rather than illustrative.

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

1. **Coalescing map** — `Dictionary<CoalescingKey, PendingWrite>`, where `PendingWrite` holds the POCO snapshot (`IPersistableSnapshot`, NOT a serialized string), an `InFlight` bool, and an optional `NextPendingPayload` POCO (held when a new snapshot arrives during an in-flight write). **Serialization is deferred to the worker thread per R10; the map holds POCOs, not strings.**
2. **Flush queue** — `Queue<CoalescingKey>` of distinct keys with pending payloads; ordered FIFO per key insertion.

**Enqueue rule (main thread):**

```
On RequestFlush(key, snapshot):                  // snapshot is POCO already built on main
  if coalescing_map[key].InFlight:
    coalescing_map[key].NextPendingPayload = snapshot
  else:
    coalescing_map[key].Payload = snapshot       // overwrite, not append
  if key not already in flush_queue:
    flush_queue.Enqueue(key)
  // No JsonUtility call on main here — deferred to the worker per R10
```

**Dispatch rule (background worker — Task.Run):**

```
Loop:
  key = flush_queue.Dequeue()                                 // blocks if empty
  entry = coalescing_map[key]
  await _writeLock.WaitAsync().ConfigureAwait(false)         // R11 single-writer
  entry.InFlight = true
  try:
    serialized = JsonUtility.ToJson(entry.Payload)            // off-main per R10
    AtomicWrite(key.FilePath, serialized)                     // tmp → fsync → Replace
  finally:
    entry.InFlight = false
    _writeLock.Release()
  if entry.NextPendingPayload != null:
    entry.Payload = entry.NextPendingPayload
    entry.NextPendingPayload = null
    flush_queue.Enqueue(key)                                  // re-enqueue for next cycle
```

**Pause-path override (main thread — R11):**

```
On OnApplicationPause(true) or OnApplicationFocus(false):
  if State == LoadingFromDisk:
    log "Pause during load — flush skipped"
    return                                                    // EC-Race1
  if _writeLock.Wait(PauseFlushTimeout):                      // synchronous acquire; NOT task.Wait()
    try:
      while flush_queue not empty:
        key = flush_queue.Dequeue()
        entry = coalescing_map[key]
        serialized = JsonUtility.ToJson(entry.Payload)        // on main — R10 exception path
        AtomicWrite(key.FilePath, serialized)                 // on main
    finally:
      _writeLock.Release()
  else:
    log "PauseFlushTimeout elapsed — yielding without flush"
    // in-flight Task.Run continues on worker; save.json intact per R3
```

**Latency budget:** event-to-disk target ≤ `FlushLatencyTarget` (D.1 default 250 ms) under normal I/O. Advisory, not a real-time guarantee — the hard guarantee ("save completes before OS kill") is served by **R11's synchronous pause-flush**. For T1/T2 Event Bus consumers, Save / Load never participates in the T1 (16.67 ms) or T2 (100 ms) budget; Save / Load's handlers only store a POCO reference in the coalescing map and return (zero serialize cost on the dispatch path), freeing the dispatch thread well under T2's 100 ms wall.

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
| `gameplay.currency.changed` | `CurrencyState` | `currency` | Exists in Event Bus GDD (pass-3 addition 2026-04-21) |
| `gameplay.tool.purchased` | `ToolOwnershipState`, `CurrencyState` | `tool_ownership`, `currency` | Exists in Event Bus GDD (pass-3 addition 2026-04-21) |
| `gameplay.tool.upgraded` | `ToolOwnershipState`, `CurrencyState` | `tool_ownership`, `currency` | Exists in Event Bus GDD (pass-3 addition 2026-04-21) |
| `gameplay.progression.node_unlocked` | `ProgressionNodeState` | `progression` | Exists in Event Bus GDD (pass-3 addition 2026-04-21) |
| `gameplay.biome.unlocked` | `BiomeUnlockState` | `biome_unlock` | Exists in Event Bus GDD (pass-3 addition 2026-04-21) |
| `gameplay.session.level_selected` | `SessionState` (last-played level ID) | `session` | Exists in Event Bus GDD (pass-3 addition 2026-04-21) |
| `gameplay.cosmetic.purchased` (VS) | `CosmeticOwnershipState`, `CurrencyState` | `cosmetic_ownership`, `currency` | Exists in Event Bus GDD (inert at MVP — no emitter in MVP scope) |

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

**Level-fail banking policy (committed, pass-3):** Level-fail **keeps** the banked coins accumulated during that Rush Phase. Save / Load persists the banked total at the moment `gameplay.level.banked` fires; a subsequent `gameplay.level.failed` does NOT zero the banked total. Rationale: (a) the subscription architecture forces this outcome unless Level Runtime additionally fires `gameplay.currency.changed` with a rollback payload before `gameplay.level.failed` — a coordination burden not justified by design intent; (b) the Player Fantasy ("work you already finished stays yours") is directly undermined by forfeit semantics at the banked level; (c) the force-quit-exploit concern (player deliberately force-quits to keep banked on an impending fail) is bounded — banking-during-rush is a deliberate decision point, so a player who pre-banks then fails was playing the system correctly. The kill-app-just-before-fail exploit exists but is low-yield (the player could have just banked anyway) and is accepted as a design trade. Level Runtime and Currency & Economy GDDs consume this commitment without re-deriving it.

**Resume-from-interruption policy:** Process kill during Rush Phase → on relaunch, player lands on the **Biome Map** with the last banked-coin state intact. No in-flight Rush Phase state is preserved (physics non-determinism per pre-prod ADR-1 makes this impossible regardless). UI surfaces a neutral "picked up where you left off" message if `SessionState.lastPlayedAt` is within the last session. No data-loss dialog — the banked state IS the saved state.

## Formulas

Save / Load's formulas are operational budgets and capacities, not gameplay balance formulas. Gameplay balance lives in consuming systems (Economy, Progression, Level Scoring).

### D.1 FlushLatencyTarget

Advisory event-to-disk latency ceiling for a coalesced async save write. Not a real-time guarantee; not tied to Event Bus T1/T2 tiers. Pass-2 correction: prior formulation produced a 50× non-signal target. Revised to sit at half the sync budget so the advisory surfaces meaningful degradation.

`FlushLatencyTarget = pause_flush_timeout_ms * safety_margin_factor`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Pause flush timeout | `pause_flush_timeout_ms` | int (ms) | 300–3000 | Hard synchronous budget from D.2 (range widened per D.2 correction) |
| Safety margin factor | `safety_margin_factor` | float | 0.1–1.0 | Fraction of sync budget where async should comfortably complete; capped at 1.0 (advisory cannot exceed sync wall — doing so would mean async routinely misses a budget that the pause path enforces) |
| Flush latency target | `FlushLatencyTarget` | int (ms) | 30–3000 | Advisory ceiling; async flushes should complete within this under normal I/O |

**Output Range:** 30 ms (= 300 × 0.1) to 3000 ms (= 3000 × 1.0). Default: 250 ms (= 500 × 0.5). Not clamped at runtime; a sustained write exceeding the target logs `[SaveManager] FlushLatency exceeded target` in debug only — advisory, not a fault.

**Example (default):** `pause_flush_timeout_ms = 500`, `safety_margin_factor = 0.5` → `FlushLatencyTarget = 250 ms`. A coalesced `currency` + `level_progress` snapshot at MVP scale (~1–3 KB serialized) completes in ~8–40 ms on the reference Mali-G52-class device (per ADR-003) across the Task.Run serialize + atomic-write path. 250 ms advisory fires when the write pipeline has degraded 5–30× from baseline — meaningful signal on eMMC wear-leveling problems, scheduler starvation, or payload-size explosion.

**Example (boundary — min):** `pause_flush_timeout_ms = 300`, `safety_margin_factor = 0.1` → `FlushLatencyTarget = 30 ms`. Tight; appropriate for a reference device with consistent eMMC behavior. Below this the advisory fires on normal scheduler jitter.

**Example (boundary — max):** `pause_flush_timeout_ms = 3000`, `safety_margin_factor = 1.0` → `FlushLatencyTarget = 3000 ms` ≡ sync budget. Advisory collapses to the same wall as the sync timeout; use only when deliberately disabling the signal (e.g., during early prototyping).

**Rationale:** Mobile I/O is non-deterministic; a T2-aligned 100 ms target would trigger constantly under normal OEM conditions. The real guarantee ("save completes before OS kill") is served by D.2's synchronous path. This advisory only exists to surface a broken write pipeline — it must fire at degradation levels humans would notice (laggy saves during rush phase), not at every Android scheduler hiccup. 250 ms default is tuned to exactly that: 5–10× normal latency = meaningful; 30–50× = the sync path itself is taking over.

**Validation:** CI stress test — drive all 6 MVP coalescing keys at max frequency for 60 s on the ADR-003 reference device; assert p95 async flush latency < `FlushLatencyTarget` (default 250 ms).

### D.2 PauseFlushTimeout

Hard synchronous write budget on `OnApplicationPause(true)` / `OnApplicationFocus(false)`. The upper bound the system must complete within before the OS process-kills the app.

`PauseFlushTimeout = platform_grace_ms * os_safety_margin`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Platform OS grace period | `platform_grace_ms` | int (ms) | 1000–5000 | iOS ~5000 (APFS, consistent); Android OEM-dependent 1000–5000 (Samsung mid-range ~1.0–1.2 s; Xiaomi skins ~800 ms; stock AOSP ~3–5 s) |
| OS safety margin | `os_safety_margin` | float | 0.3–0.6 | Fraction of grace reserved for the write; remainder absorbs Unity teardown + OEM variance |
| Pause flush timeout | `PauseFlushTimeout` | int (ms) | 300–3000 | Hard wall; flush must complete or abandon within this window |

**Output Range:** 300 ms (= 1000 × 0.3) to 3000 ms (= 5000 × 0.6). Pass-2 correction: prior range stated 2500 ms upper; 5000 × 0.6 = 3000, so upper bound widened to 3000. Default: 500 ms (= 1000 Android worst case × 0.5). Enforced by `SemaphoreSlim.Wait(PauseFlushTimeout)` on the main thread (R11) — NOT by `CancellationTokenSource` on the in-flight Task (that would require `task.Wait()` which is banned per R11). If the semaphore is not acquired within the timeout, the pause handler yields without initiating its own writes; any in-flight Task.Run continues to completion if the OS grants time. `save.json` is never partially written per R3.

**Example (default, MVP):** Android OEM worst case: `platform_grace_ms = 1000`, `os_safety_margin = 0.5` → `PauseFlushTimeout = 500 ms`. With R10's off-main serialize: the pause-path does both serialize and write synchronously on main (exception path per R10). For 6 coalescing keys × 1–10 KB payload each:

- `JsonUtility.ToJson` on main per key: 1–3 ms × 6 = **6–18 ms total**
- Per-key tmp-write + `FlushToDisk(true)` + `File.Replace` on ADR-003 reference Android device eMMC: 8–40 ms × 6 = **48–240 ms total**
- Combined pause-path synchronous cost: **54–258 ms**
- Headroom to 500 ms budget: 242–446 ms (1.9×–9.3× margin)

The headroom absorbs a 200–400 ms eMMC wear-leveling stall on any single key before the semaphore wait gives up on the remaining keys.

**Example (boundary — min, tight Xiaomi profile):** `platform_grace_ms = 1000` (1 s grace), `os_safety_margin = 0.3` → `PauseFlushTimeout = 300 ms`. Tight; 54–258 ms expected consumption leaves 42–246 ms margin. Single eMMC stall > 100 ms can exhaust budget — semaphore times out, remaining keys flush on next focus.

**Example (boundary — max, iOS deliberate):** `platform_grace_ms = 5000` (iOS grace), `os_safety_margin = 0.6` → `PauseFlushTimeout = 3000 ms`. Luxurious; only applies if a per-platform override is activated via `#if UNITY_IOS` (deferred per OQ-5). MVP does not ship per-platform defaults.

**Rationale:** iOS is reliably ~5 s; designing to that wastes budget on the shared default. Android OEM is the binding constraint — Samsung mid-range measured at 1.0–1.2 s, some Xiaomi skins at ~800 ms. Design to the conservative 1 s floor with 50% OS margin. A per-platform relaxation to 800 ms on iOS via `#if UNITY_IOS` is permitted once shipping profiling demonstrates headroom — deferred per OQ-5.

**Pause-path synchronous cost shifted from async in pass-3.** Because R10 moved serialization off-main on the async path, the pause-path is now the ONLY place `JsonUtility.ToJson` runs on main thread. Multi-key pause cost of 54–258 ms is cheap compared to the OS grace window — the pause budget easily absorbs it. The main-thread frame-budget concern that haunted pass-1 (30 ms for 6-key flush at 180% of 60 fps frame) is resolved because async flushes never hit main-thread serialize anymore.

**Validation:** Integration test — enqueue 6 pending writes, simulate `OnApplicationPause(true)`, assert either (a) all 6 `save.json` files updated within 500 ms OR (b) some updated and the pause handler yielded at the 500 ms wall with `save.json` never partially written for any key. Instrument per-key timing.

### D.3a FileSizeEstimate (the formula)

Expected serialized size of `save.json` + `save.bak` + `settings.json` combined for a given scope. Pass-2 correction: prior D.3 conflated the estimating formula with the budget constant; they are now split.

`FileSizeEstimate = (N_biomes * N_levels * bytes_per_level_record) + (N_tools * bytes_per_tool_record) + (N_progression_nodes * bytes_per_node_record) + bytes_overhead`

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
| Raw estimate | `FileSizeEstimate_raw` | int (bytes) | 600–70,000 | Pre-verbosity-multiplier expected size |
| Serialized estimate | `FileSizeEstimate` | int (bytes) | ≈ 2 × raw | Apply JsonUtility verbosity factor (~2×) to raw |

**Example (MVP, 5 biomes authored, 1 active, 4 tools, 10 progression nodes):**
- Level records: 5 × 40 × 16 = 3,200 bytes
- Tool records: 4 × 60 = 240 bytes
- Progression: 10 × 40 = 400 bytes
- Overhead: 1,024 bytes
- Raw: 4,864 bytes (~5 KB). Serialized (≈2×): **~10 KB**. Matches AC-P3 fixture expectation (see below).

**Example (full v1.0 scope, 5 biomes × 60 levels, 20 tools, 100 progression):**
- Level: 5 × 60 × 20 = 6,000 bytes
- Tool: 20 × 60 = 1,200 bytes
- Progression: 100 × 40 = 4,000 bytes
- Overhead: 2,048 bytes
- Raw: 13,248 bytes (~13 KB). Serialized: **~26 KB**. Well under D.3b's 64 KB budget.

**Example (boundary — max authored):** `N_biomes=20, N_levels=60, N_tools=20, N_progression=100` — `20 × 60 × 32 = 38,400` + `20 × 80 = 1,600` + `100 × 48 = 4,800` + `4,096` = **48,896 bytes raw**, ~98 KB serialized. Exceeds D.3b 64 KB budget — this is the ceiling beyond which the R9 canary must fire and the design must either reduce scope or revisit the budget.

### D.3b FileSizeBudget (the constant)

Soft on-disk ceiling against which `FileSizeEstimate` and actual serialized size are compared. Diagnostic canary for the R9 state-vs-rules boundary — a breach signals that a persistable type is holding gameplay values it should be reading from Data Registry at runtime.

`FileSizeBudget = 65,536 bytes (64 KB)`

**Derivation (independent of D.3a).** The 64 KB ceiling is chosen as: (a) 2.4× the v1.0 serialized estimate (26 KB) to leave headroom for 1 major additive schema revision pre-ship; (b) small enough that any accidental persistence of a `Dictionary<GUID, ToolConfig>`-style value (where an owning system drags a reference asset through JSON instead of its `_id`) produces a 5–50 KB overage and trips the canary; (c) negligible I/O cost at 64 KB on any supported mobile eMMC (write time ~10–40 ms — within D.1 advisory budget). The number is a budget, not a formula output.

**AC-P3 32 KB fixture threshold** is derived FROM D.3a: the MVP-scope fixture (5 biomes / 40 completed levels at 3 stars / 4 tools / 10 progression nodes) produces ~10 KB serialized per D.3a; AC-P3 asserts ≤ 32 KB to provide 3× headroom against D.3a prediction drift. 32 KB is explicitly NOT derived from D.3b — it is a tighter ceiling specifically for the MVP-scope fixture, sitting at half of D.3b as an early-warning tripwire.

**Breach behavior:** `[SaveManager] Size budget exceeded: {actual}B > 65536B` — ADVISORY in debug builds, BLOCKING in release CI (AC-D3a / AC-D3b).

**Rationale:** JsonUtility file I/O at 64 KB has negligible runtime cost on any supported device. The ceiling exists as a canary: the rules-vs-state boundary (R9) is easy to violate accidentally; size explosion is the symptom. Catching it pre-ship beats debugging it in production.

**Validation:** CI job measures serialized size across `save.json` + `save.bak` + `settings.json` after a representative EditMode fixture run. Release builds treat `> FileSizeBudget` as a blocking failure; debug builds log and continue. AC-P3 independently enforces the tighter 32 KB fixture ceiling.

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

- **If a successful migration produces a JSON string exceeding `FileSizeBudget`**: reject the migrated result before writing, preserve the original pre-migration `save.json` as `save.corrupt-[timestamp].json`, return R8.e defaults, log the overage with actual byte count. Do NOT write an oversized save — it would compound on the next boot.

- **If the top-level `_saveFileVersion` field is missing from `save.json`** (manual editing, truncated write, earlier serializer version): treat as corrupt per R4.a's rule that `_schemaVersion/_saveFileVersion == 0` is corrupt. Apply R8.d. Never attempt migration — there is no safe version to migrate from.

### Runtime Race Conditions

- **If `OnApplicationPause(true)` fires while `LoadingFromDisk` is still in progress** (slow cold boot on low-end Android during a heavy migration): R11's pause handler detects `State == LoadingFromDisk` and skips the synchronous flush entirely — the coalescing queue is empty and writing stale state would be harmful. Log `[SaveManager] Pause during load — flush skipped`. The OS grace budget is consumed by the load itself.

- **If `SaveManager.Dispose()` fires while a `Task.Run` write is in-flight** (scene reload during async flush): `Dispose` calls `_writeLock.Wait(PauseFlushTimeout)` — the **synchronous semaphore** overload, NOT `task.Wait()` on any async Task (`task.Wait()` is banned per R11 to prevent IL2CPP deadlock). If the semaphore is acquired before timeout, dispose proceeds with a synchronous queue drain and then disposes `_writeLock`. If timeout elapses, the in-flight task is left running on the worker (it will complete or be cut off by the scene reload); the tmp file remains on disk and is recoverable by R8.b on next boot. Do NOT `task.Cancel()` mid-write — a partial tmp file that is NOT fsynced is worse than an abandoned one.

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
| Currency & Economy | MVP | Reads `CurrencyState` for current balances; emits `gameplay.currency.changed` |
| Progression / Unlock Gating | MVP | Reads `ProgressionNodeState` for unlocked set; emits `gameplay.progression.node_unlocked` |
| Biome Map / Level Select | MVP | Reads `BiomeUnlockState` + `SessionState` for rendering and focus; emits `gameplay.biome.unlocked` and `gameplay.session.level_selected` |
| Main Menu / Settings / Pause | MVP (minimal) | Reads `SettingsManager.GetCurrent()` (R13 path, not Event Bus); writes via `SettingsManager.Set*()` |
| Tool System | MVP | Reads `ToolOwnershipState` for owned tools + tier indices; emits `gameplay.tool.purchased` / `.upgraded` |

### Downstream (phased — dependency activates at system delivery tier)

| System | Delivery Tier | Nature |
|---|---|---|
| Shop & IAP | Vertical Slice (full; MVP stub) | Reads `CurrencyState` (via Currency & Economy); writes transactions through Economy, not directly. MVP stub uses debug-panel UI; writes still flow through Save / Load per R7 snapshot contract. |
| VFX / Juice | Vertical Slice | Does not read or write Save / Load state; listed only for contract clarity (VFX has no persistent state in MVP/VS scope). |
| Accessibility Service | Vertical Slice | Reads reduce-motion flag from `SettingsManager`; no direct Save / Load access. Interface hooks are authored into MVP GDDs (Level Runtime, VFX, HUD) for later consumption. |
| Cosmetic / Skin System | Shippable v1.0 | Reads `CosmeticOwnershipState`; emits `gameplay.cosmetic.purchased` (channel exists in Event Bus GDD; inert at MVP — no MVP emitter) |
| Analytics / Telemetry | Shippable v1.0 | Reads opt-in flag from `SettingsManager`; may persist a local analytics-flush buffer under a new coalescing key (`analytics_flush`). Adds to `FileSizeBudget` formula variables at v1.0. |
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
| `pause_flush_timeout_ms` (D.2) | 500 ms | [300, 3000] | More slack for synchronous pause-flush on slow eMMC; risk of OS killing process before budget elapses on aggressive Android OEMs | Tighter sync window; legitimate writes may abort under transient I/O stall, losing last-event payload (recoverable via R8.b on next boot) |
| `safety_margin_factor` (D.1) | 0.5 | [0.1, 1.0] | Advisory `FlushLatencyTarget` loosens (e.g., 1.0 collapses advisory to sync wall); less sensitivity to pipeline degradation | Tighter advisory (0.1 × 500 = 50 ms); more sensitivity to eMMC stalls; noisier debug logs; earlier warning of broken write pipeline |
| `FileSizeBudget` ceiling (D.3b) | 64 KB | [32 KB, 256 KB] | More headroom for future scope additions; masks R9 state-vs-rules boundary violations | Tighter canary; catches accidental gameplay-value persistence earlier; risk of false-positive blocks on legitimate v1.0+ scope growth |
| `flush_requests_per_key_per_frame_limit` (D.4) | 3 | [1, 10] | Higher burst tolerance; fewer runaway-producer warnings; risk of masking real bugs | Tighter; more warnings on legitimate batch unlocks; pushes designers toward single-batched events |
| CI severity — `Resources.Load` / direct `File.*` outside `SaveManager` | **BLOCKING** (R1) | BLOCKING only | N/A | Changing to WARNING permits arbitrary file I/O scattered across gameplay systems; strictly wrong — violates the single-writer invariant |
| CI severity — reflection mutation of loaded state | **BLOCKING** (R2) | BLOCKING only | N/A | Changing to WARNING allows consumers to mutate the authoritative cache; strictly wrong |
| CI severity — delta payload on `RequestFlush` (non-`IPersistableSnapshot`) | **BLOCKING** (R7) | BLOCKING only | N/A | Strictly wrong — delta payloads silently corrupt under coalescing |
| CI severity — migration chain gap | **BLOCKING** (build-time) | BLOCKING only | N/A | Strictly wrong — gap guarantees player-facing load failure at the affected version |
| CI severity — stale-reference count at load | **ADVISORY** (R6) | [ADVISORY, WARNING] | N/A | Upgrading to WARNING once Data Registry asset hygiene is strict may catch asset-rename mistakes earlier |
| CI severity — `FileSizeBudget` breach at test fixture | **BLOCKING** (release) / **ADVISORY** (debug) | [ADVISORY, WARNING, BLOCKING] | N/A | Upgrading debug to BLOCKING flags breaches at iteration time; may produce churn on in-progress persistables |
| CI severity — POCO self-containment (R10.a — `UnityEngine.Object`-derived field in persistable) | **BLOCKING** (AC-R10c) | BLOCKING only | N/A | Relaxing enables silent main-thread dependency leakage; catastrophic for off-main serialization model |
| CI severity — bare `await` without `.ConfigureAwait(false)` in `SaveManager`/`MigrationPipeline` | **BLOCKING** (AC-R10d) | BLOCKING only | N/A | Relaxing reintroduces the IL2CPP deadlock vector that R11's pure-semaphore pause path was designed to eliminate |
| CI severity — R9 banned field name grep | **BLOCKING** (AC-R9a) | BLOCKING only | N/A | Relaxing leaves the no-pay-to-win anti-pillar without mechanical enforcement — AC-R9b semantic review is advisory and cannot substitute for the CI gate |
| CI severity — playtest logger touching save.json APIs | **BLOCKING** (AC-EC-PlaytestIsolation) | BLOCKING only | N/A | Relaxing allows playtest data to contaminate durable save state |
| Recovery UI on R8.d (`RecoveredFromBackup`) | **Light toast required (default)** | [Light toast, Modal] — **Silent NOT permitted** | Modal warns player progress was recovered from backup — may alarm for a quiet success | Lighter touch — but silent is pulled from the menu per pass-1 action #15; minimum acknowledgment is the toast. Maintaining the silent option leaves the recovery invisible and undermines player trust when UI surfaces a mild "some progress recovered" message mismatches a missing state |
| Recovery UI on R8.e (`ResetToDefault`) | **REQUIRED modal** | MODAL only | N/A | Hiding the reset leaves the player confused why their progress vanished; required for trust |
| R8.e modal tone contract | **Reassuring, apologetic, non-alarming** (e.g., "We couldn't load your progress, but we've kept the original file for support. Resuming from a fresh start.") | Locked contract | Stronger wording (e.g., "DATA CORRUPTED") exceeds scope — Save/Load does not know whether the user actually lost meaningful progress | Softer wording may not convey the need for the user to seek support via the preserved `save.corrupt-*.json` file |
| Migration determinism test iteration count (R5) | 1000 iterations | [100, 10000] | Stronger determinism guarantee; longer CI runtime | Weaker guarantee; faster CI; may miss non-determinism surfacing only in rare conditions |
| Migration JSON size pre-write ceiling (vs `FileSizeBudget`) | 1.0× budget | [0.8×, 1.5×] | Allows migrations that temporarily exceed budget during intermediate steps (v1→v3 via v2); risk of shipping oversized saves | Stricter; catches migration bloat earlier; may force migrators to produce compact output |
| `max_pending_per_key` (D.4) | 2 (fixed) | **Locked** (design axiom) | N/A | Increasing to 3+ breaks R7 snapshot semantics — treat as architecture change, not tuning |
| Root-level `_saveFileVersion` increment policy (R4.b) | One bump per any per-type schema revision | **Locked** | N/A | Sharing version numbers across unrelated schema changes produces ambiguous migration semantics |
| Reference device naming authority | **ADR-003** | Locked | N/A | Naming a device inline in this GDD without ADR ratification couples the design doc to hardware; ADR-003 is the canonical authority |

### Interaction Between Tuning Knobs

- **Coupled**: `pause_flush_timeout_ms` (D.2) and `safety_margin_factor` (D.1). `FlushLatencyTarget = pause_flush_timeout_ms × safety_margin_factor` — changing one without the other either produces meaningless advisories or suppresses useful ones. Adjust together when tuning for a specific device profile (e.g., minimum-spec Mali-G52 vs. reference iPhone).
- **Coupled**: `flush_requests_per_key_per_frame_limit` (D.4) and the shape of upstream event emission. Tightening the limit without refactoring batch events (e.g., issuing one `gameplay.progression.batch_unlock` instead of N `node_unlocked`) produces warning churn. Change limit only in coordination with the relevant producer GDD.
- **Independent**: `FileSizeBudget` (D.3) and timing budgets (D.1/D.2). Size affects I/O duration only at very large payloads (>256 KB) that are far outside MVP scope.

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

**60 criteria across 7 categories** (body-verified count, pass-3 regeneration). Gate classifications: **BLOCKING CI** (must pass for merge) / **ADVISORY** (logged, non-blocking) / **PLAYMODE** (Unity PlayMode runner, CI-safe) / **DEVICE** (requires the reference device named in **ADR-003 — Save Schema Versioning & Reference Device**) / **MANUAL** (design review or device observation).

> **Reference device note**: all DEVICE-tier ACs cite "ADR-003 reference device". Pass-2 identified that "Mali-G52 emulator image" (the prior phrasing) is internally contradictory — an emulator cannot honor Mali-G52 timing. ADR-003 (pre-production action item from TD-FEASIBILITY) is hereby expanded to include selection of a named Android reference device AND a named iOS reference device. Until ADR-003 lands, DEVICE ACs are gated pending — not deleted, not skipped, and not runnable on emulator substitutes.

### H.1 Rule Compliance

- **AC-R1 — No main-thread I/O** — GIVEN the full game codebase, WHEN a CI static-analysis grep scans for `File.Write*` / `File.Read*` / `File.Replace` / `FileStream` outside `SaveManager.cs` and `MigrationPipeline.cs`, THEN zero matches appear; any match fails CI. **BLOCKING CI**
- **AC-R2 — Immutable Loaded State** — GIVEN a consumer calls `GetState<LevelProgressState>()` and mutates the returned object, WHEN a second consumer calls `GetState<LevelProgressState>()` in the same session, THEN the second consumer receives the ORIGINAL (unmutated) values; mutation is confined to the returned copy. **PLAYMODE**
- **AC-R3a — Atomic Write Pattern** — GIVEN a `RequestFlush` call completes, WHEN the filesystem is inspected, THEN the write sequence wrote to `save.tmp` first, called `FileStream.Flush(flushToDisk: true)`, then called `File.Replace(save.tmp, save.json, save.bak)`. Verified by mock filesystem intercepting each call in order. **PLAYMODE**
- **AC-R3b — fsync Best-Effort Instrumentation** — GIVEN the atomic-write completes successfully, WHEN run against the ADR-003 reference Android device, THEN the `FlushToDisk(true)` call is verified to have been invoked (instrumentation counter ≥ 1 per successful flush). The Samsung OneUI eMMC caveat (R3) is architecturally mitigated by R8.b's tmp-promotion; this AC only asserts the syscall is made, not that the eMMC honored it. **DEVICE** (ADR-003) + **PLAYMODE** (counter-only)
- **AC-R4a — Per-Type Schema Version Present** — GIVEN any persistable POCO type T, WHEN reflection inspects T, THEN `T._schemaVersion` exists as the first declared field, is `public int`, and is not `0`. CI test enumerates all `IPersistableSnapshot` types. **BLOCKING CI**
- **AC-R4b — Root File Version Atomic Advance** — GIVEN a migration run where step 3 of 4 throws, WHEN the load completes, THEN `save.json` is unchanged from pre-migration state (not partially advanced); `_saveFileVersion` on disk still equals the pre-migration value. **BLOCKING CI**
- **AC-R4c — Root-to-Per-Type Invariant Table Enforced** — GIVEN the R4.c invariant table, WHEN CI build runs, THEN the table is gapless from v1 to `CurrentCodeVersion`, every `IPersistableSnapshot` type has an entry at every version, and a load-time cross-check fails with `[MigrationPipeline] Per-type version mismatch at root v{N}` if any per-type `_schemaVersion` does not match the table after migration. **BLOCKING CI** (build-time chain + load-time assert)
- **AC-R5 — Migration Determinism** — GIVEN each registered migrator, WHEN invoked 1000× on identical input, THEN every invocation produces byte-identical output strings. EditMode unit test per migrator. **BLOCKING CI**
- **AC-R6 — GUID-Only ID References** — GIVEN every persistable field annotated as an asset reference, WHEN inspected by CI grep + regex match on loaded save files, THEN all reference strings match `^[0-9a-f]{32}$`; no path-like strings (containing `/` or `.asset`) appear. **BLOCKING CI**
- **AC-R7 — `IPersistableSnapshot` Marker Enforcement** — GIVEN `RequestFlush<T>`, WHEN T is not tagged with `IPersistableSnapshot`, THEN compilation fails with the generic type constraint. Static analysis additionally flags any payload type whose name contains "Delta", "Diff", "Change", or "Patch" and fails CI. **BLOCKING CI**
- **AC-R7a — Producer Read-Modify-Write Cache Coherency** — GIVEN two producers on the same coalescing key call `RequestFlush` in the same frame, WHEN the second producer calls `GetState<>()` before its `RequestFlush`, THEN the returned state reflects the first producer's mutation (the cache updated synchronously on the first `RequestFlush` accepted). Enforces R7.a. **PLAYMODE**
- **AC-R8a — FirstRun Outcome** — GIVEN no `save.json`, no `save.tmp`, and no `save.bak` exist at `Load()`, WHEN load runs, THEN `SaveLoadResult.Outcome == FirstRun` is raised, every `GetState<T>()` returns the consumer-supplied default state, and no file is written until a `RequestFlush` fires. **PLAYMODE**
- **AC-R8b — RecoveredFromTmp with Version Guard** — GIVEN `save.tmp` exists at `Load()` AND parses valid AND has `_saveFileVersion >= save.json._saveFileVersion` (or `save.json` is unreadable AND `save.tmp._saveFileVersion <= CurrentCodeVersion`), WHEN load runs, THEN `SaveLoadResult.Outcome == RecoveredFromTmp`, `save.tmp` is promoted to `save.json`, and the prior `save.json` (if any) is rotated to `save.bak`. The stale-tmp rejection case (`save.tmp._saveFileVersion < save.json._saveFileVersion`) is covered separately below. **PLAYMODE**
- **AC-R8b-reject — RecoveredFromTmp Stale-Tmp Rejection** — GIVEN `save.tmp._saveFileVersion < save.json._saveFileVersion`, WHEN load runs, THEN `save.tmp` is NOT promoted, `save.tmp` is preserved as `save.stale-tmp-[timestamp].json` for support diagnostics, `save.json` loads normally, and `[SaveManager] Stale tmp rejected by version guard: tmp=v{A} < json=v{B}` logs. **PLAYMODE**
- **AC-R8c — LoadedClean No-Rename** — GIVEN `save.json` parses valid at the current schema version with no migration needed, WHEN load runs, THEN `SaveLoadResult.Outcome == LoadedClean`, no files are renamed, no `save.corrupt-*` is created, and `save.bak` is unchanged. **PLAYMODE**
- **AC-R8d — RecoveredFromBackup + Optional Toast** — GIVEN `save.json` fails schema validation or JSON parse AND `save.bak` parses valid, WHEN load runs, THEN `SaveLoadResult.Outcome == RecoveredFromBackup`, `save.json` is renamed to `save.corrupt-[timestamp].json`, `save.bak` is promoted to `save.json`, and the UI receives a light-toast signal (tuning-knob default — see Tuning Knobs). **PLAYMODE**
- **AC-R8e — ResetToDefault + Required Modal** — GIVEN both `save.json` and `save.bak` fail load, WHEN load completes, THEN `SaveLoadResult.Outcome == ResetToDefault`, both originals are preserved as `save.corrupt-*` and `save.bak.corrupt-*`, defaults are returned for all domains, and a blocking modal is scheduled per AC-INV2. **PLAYMODE**
- **AC-R9a — State-vs-Rules CI Grep (Banned Field Names)** — GIVEN any field in an `IPersistableSnapshot` POCO type, WHEN a CI static-analysis grep scans for field names matching `(?i)(cost|price|threshold|rate|duration|quota|multiplier|limit|multiplier|percent|bonus|penalty)`, THEN zero matches appear in any persistable POCO file. Any match fails CI with `[SaveManager] R9 banned field name: {FieldName} in {Type}`. **BLOCKING CI**
- **AC-R9b — State-vs-Rules Semantic Design Review** — GIVEN every `IPersistableSnapshot` POCO type, WHEN reviewed by a design-review checklist at the end of each sprint that touches a persistable type, THEN no field holds a gameplay tuning value semantically (AC-R9a is name-only; R9b catches renamed violations such as `_coinPricePerUnit` → `_coinUnit`). Lead-designer signoff required. **ADVISORY** (MANUAL review; complements AC-R9a CI grep)
- **AC-R10a — Snapshot Build on Main Thread** — GIVEN a handler for a Save/Load-subscribed event, WHEN the main-thread snapshot build runs, THEN the POCO is fully populated from live Unity state on the main thread (thread ID equals `UnityEngine.PlayerLoopHelper` main) before the POCO is handed to `Task.Run`. Any `UnityEngine.Object` read after the Task.Run boundary fails the test. **PLAYMODE**
- **AC-R10b — Serialize and I/O Off Main Thread** — GIVEN the `Task.Run` worker for a flush, WHEN `JsonUtility.ToJson` and all file-system operations execute, THEN thread ID is NOT the Unity main thread. Verified by instrumented `IFileSystem` and a thread-id capture inside the serialize lambda in PlayMode. **PLAYMODE**
- **AC-R10c — POCO Self-Containment (UnityEngine.Object-Free)** — GIVEN every `IPersistableSnapshot`-implementing type, WHEN reflection inspects declared field types, THEN zero fields have a declared type deriving from `UnityEngine.Object` (including `GameObject`, `Component`, `ScriptableObject`, or any asset-reference subclass). Any match fails CI with `[SaveManager] R10.a violation: {FieldName} in {Type} is UnityEngine.Object-derived`. **BLOCKING CI**
- **AC-R10d — ConfigureAwait(false) Enforcement** — GIVEN `SaveManager.cs` and `MigrationPipeline.cs`, WHEN a CI grep scans for `\bawait\s+[^;]+` lines, THEN every match is followed by `.ConfigureAwait(false)` on the same statement OR is preceded by a line-comment beginning `// await-main-thread-context:` that justifies the exception. Zero unjustified bare `await` statements. **BLOCKING CI**
- **AC-R11a — Synchronous Pause-Flush Lock Correctness** — GIVEN pending coalesced writes and `OnApplicationPause(true)` fires, WHEN the pause handler executes, THEN `_writeLock.Wait(PauseFlushTimeout)` is used (not `Task.Wait()` on any Task), on acquire the handler drains the flush queue inline on main, and on timeout the handler yields without initiating writes. The mock-filesystem test asserts no `Task.Wait()` call appears on the pause-handler code path via method-instrumentation. **PLAYMODE**
- **AC-R11b — Pause-Flush Wall-Clock on Reference Device** — GIVEN 6 pending coalesced writes (combined ~30–60 KB serialized) and `OnApplicationPause(true)` on the ADR-003 reference Android device, WHEN measured, THEN all writes complete OR the pause handler yields within 500 ms wall-clock, `save.json` is never partially written for any key, and instrumented serialize+write time is logged per key for diagnostics. **DEVICE** (ADR-003)
- **AC-R12 — JsonUtility Format Compliance (Worker-Thread Round-Trip)** — GIVEN every persistable POCO type, WHEN round-tripped through `JsonUtility.ToJson` → `FromJson` **on a `Task.Run` worker thread** (not the test main thread), THEN field-count parity holds (no silent field-drop); reflection asserts zero fields of banned types (`Dictionary<,>`, `HashSet<>`, `SortedSet<>`, `Queue<>`, `Stack<>`, `List<List<>>`, `readonly`, `init`, interface-typed, `record`, properties, `DateTime`, `TimeSpan`, `Guid`, nullable value types, `UnityEngine.Object`-derived); every `float` field is finite (not `NaN` / `Infinity`) after a test-mutation sweep. **BLOCKING CI**
- **AC-R13 — Settings Bypass Isolation** — GIVEN `SettingsManager.SetAudioMaster(0.5f)`, WHEN the Event Bus is inspected, THEN no event is fired and no subscription on the bus receives any notification. `settings.json` is updated directly via Save / Load's file utilities. **PLAYMODE**
- **AC-R13a — Settings Schema Version** — GIVEN `settings.json`, WHEN inspected, THEN the root contains `_settingsVersion: int` (non-zero) and a per-field-compatible migration pipeline (simpler than the R4 migrator chain: one step per version bump) is registered. On load of a `settings.json` at an older `_settingsVersion`, a migrator advances the file before `SettingsManager` consumers see values. **PLAYMODE**

### H.2 Formula Correctness

- **AC-D1 — FlushLatencyTarget Advisory Only** — GIVEN a simulated 500 ms I/O stall (2× default `FlushLatencyTarget` of 250 ms) on a write, WHEN the flush completes, THEN the write succeeds, `[SaveManager] FlushLatency exceeded target` logs in debug, and no exception is thrown. **PLAYMODE**
- **AC-D2 — PauseFlushTimeout Hard Wall via Semaphore** — GIVEN a simulated 800 ms I/O stall during synchronous pause-flush with timeout = 500 ms, WHEN the timeout elapses, THEN `_writeLock.Wait(500)` returns `false`, the pause handler yields without initiating its own writes, the in-flight Task.Run continues to completion on the worker, `save.json` is not partially written for any key. No `Task.Wait()` is invoked anywhere on this path. **PLAYMODE**
- **AC-D3a — FileSizeBudget Release-Build Enforcement** — GIVEN a test fixture producing a 70 KB serialized save, WHEN a release build's CI job runs, THEN the job fails with `[SaveManager] Size budget exceeded: 71680 B > 65536 B`. **BLOCKING CI**
- **AC-D3b — FileSizeBudget Debug Advisory** — GIVEN the same 70 KB fixture, WHEN a development build runs, THEN the advisory logs but the build succeeds. **PLAYMODE**
- **AC-D4 — CoalescingCapacity Per-Key Cap** — GIVEN 100 `RequestFlush` calls on a single key in one frame, WHEN the queue state is inspected, THEN exactly 2 payload slots (in-flight + next-pending) are populated and exactly 1 `[SaveManager] Runaway flush producer detected` warning is logged. **PLAYMODE**

### H.3 Edge Cases

- **AC-EC-Storage1 — Disk-Full Retention** — GIVEN `File.Replace` throws `IOException` mid-flush due to disk full, WHEN the handler catches, THEN `save.tmp` is retained (not deleted), `save.json` is unchanged, `[SaveManager] Disk full — save.tmp retained` is logged, and state transitions to `CorruptedRecovery`. **PLAYMODE** (mocked IO)
- **AC-EC-Storage2 — iOS Locked File Distinction** — GIVEN `File.Replace` throws `UnauthorizedAccessException`, WHEN the handler catches, THEN `save.json` is NOT renamed to `save.corrupt-*`; the failure is treated as transient, retry on next focus. **PLAYMODE**
- **AC-EC-Migration1 — Downgrade Detection** — GIVEN `save.json` contains `_saveFileVersion = 99` and code's current version is 3, WHEN load attempts migration, THEN no forward migrator is invoked; R8.d fires if `save.bak` is compatible; R8.e fires otherwise. **BLOCKING CI**
- **AC-EC-Migration2 — Migration Chain Gap** — GIVEN migrators registered for versions 1 and 3 but not 2, WHEN build time runs a chain-completeness check, THEN CI fails with `[MigrationPipeline] No migrator for v1 → v2`. **BLOCKING CI** (build-time)
- **AC-EC-Race1 — Pause During Load** — GIVEN `State == LoadingFromDisk` and `OnApplicationPause(true)` fires, WHEN the pause handler executes, THEN no synchronous flush is attempted, `[SaveManager] Pause during load — flush skipped` logs, and the OS pause callback returns within 50 ms. **PLAYMODE**
- **AC-EC-Race2 — Dispose During In-Flight Write (No task.Wait)** — GIVEN a Task.Run write is in-flight and `SaveManager.OnDestroy()` fires, WHEN dispose runs, THEN `_writeLock.Wait(PauseFlushTimeout)` is called (synchronous semaphore acquire, NOT `task.Wait()`); if acquired before timeout, disposal proceeds cleanly after a synchronous queue drain; if timeout elapses, the in-flight task is left running and the tmp file remains on disk for R8.b recovery. **PLAYMODE**
- **AC-EC-Race3 — SemaphoreSlim Disposed in OnDestroy** — GIVEN `SaveManager.OnDestroy()` completes, WHEN the test inspects the instance's `_writeLock` field via reflection, THEN `_writeLock.SafeWaitHandle.IsClosed == true` (disposal invoked). Verifies no kernel-handle leak across scene reload. **PLAYMODE**
- **AC-EC-Uninit — Uninitialized State Throws** — GIVEN `SaveManager.Awake()` completes but `Load()` has not been called, WHEN a consumer invokes `GetState<LevelProgressState>()` or `RequestFlush(key, snapshot)`, THEN `InvalidOperationException` is raised with message `[SaveManager] Call Load() before using GetState/RequestFlush`. **PLAYMODE**
- **AC-EC-Clock — Future Timestamp Clamp** — GIVEN `SessionState.lastPlayedAt = UtcNow.Ticks + 1000000000`, WHEN the resume UI check runs, THEN the effective value is clamped to `UtcNow.Ticks`, and the resume message fires conservatively. **PLAYMODE**
- **AC-EC-Stale — Stale Tool ID Resolution** — GIVEN a save references `ToolConfig` ID `"aaa...000"` which no longer exists in the Data Registry, WHEN `ToolOwnershipState` loads, THEN the ID is retained in state (not deleted), the Tool System filters it from the available loadout silently, and a debug `gameplay.tool.stale_reference` event fires. **PLAYMODE**
- **AC-EC-PlaytestIsolation — Playtest Logger Does Not Touch save.json** — GIVEN the playtest logger (`playtest-log-[date].jsonl`) is active in a DEVELOPMENT_BUILD, WHEN a CI grep scans `PlaytestLogger.cs` for references to `save.json`, `save.bak`, `save.tmp`, `settings.json`, or any `SaveManager.*` API, THEN zero matches appear. Playtest logger owns its own file I/O outside Save/Load per CD-SYSTEMS. **BLOCKING CI**

### H.4 Performance & Platform

- **AC-P1 — Async Flush Latency p95 on Reference Device** — GIVEN all 6 MVP coalescing keys driven at max frequency for 60 seconds on the ADR-003 reference Android device, WHEN measured, THEN p95 async flush latency ≤ `FlushLatencyTarget` (default 250 ms). **DEVICE** (ADR-003)
- **AC-P2 — Pause Flush on Reference Device** — GIVEN 6 pending coalesced writes (combined ~30–60 KB) and `OnApplicationPause(true)` on the ADR-003 reference Android device, WHEN measured, THEN all writes complete or the handler yields within 500 ms wall-clock with no partial `save.json`. **DEVICE** (ADR-003)
- **AC-P3 — Save File Size at MVP Scope** — GIVEN a full MVP save (5 biomes authored, 40 levels completed with 3 stars each, 4 tools owned, all progression unlocked) derived from D.3a's formula (~10 KB expected serialized), WHEN serialized, THEN `save.json` size ≤ 32 KB (3× headroom against D.3a prediction; half of D.3b's 64 KB budget). **BLOCKING CI**
- **AC-P4 — JsonUtility Serialization Cost on Reference Device** — GIVEN the full MVP save state, WHEN `JsonUtility.ToJson` is profiled on the ADR-003 reference Android device on a `Task.Run` worker thread, THEN serialization completes in ≤ 5 ms p95 across 100 iterations. **DEVICE** (ADR-003)

### H.5 Consumer Integration

- **AC-INT1 — Currency & Economy Round-Trip** — GIVEN Currency & Economy modifies balance and calls `RequestFlush("currency", CurrencyState)`, WHEN scene reloads, THEN the loaded `CurrencyState` matches the pre-flush state field-for-field. **PLAYMODE**
- **AC-INT2 — Progression First-Run Defaults** — GIVEN no save exists (R8.a FirstRun), WHEN Progression requests state, THEN the returned `ProgressionNodeState` contains the initial unlock set declared by `ProgressionNode` root flags in Data Registry. **PLAYMODE**
- **AC-INT3 — Settings Across Sessions** — GIVEN `SettingsManager.SetAudioMaster(0.25f)` is called, WHEN the app is killed and relaunched, THEN `SettingsManager.GetCurrent().AudioMaster == 0.25f`. **PLAYMODE**
- **AC-INT4 — Level Runtime Resume-from-Interruption** — GIVEN a process kill during Rush Phase with last banked event having fired, WHEN the app relaunches, THEN the biome map shows the pre-kill banked currency total, `SessionState.lastPlayedLevelId` points to the in-progress level, and no in-flight Rush state is restored. **PLAYMODE**

### H.6 Invariants (What Must NOT Happen)

- **AC-INV1 — No Silent Data Loss on Recovery** — GIVEN any R8 recovery outcome is triggered, WHEN recovery runs, THEN EITHER (a) `SaveLoadResult.Outcome` reports exactly which path fired (`RecoveredFromTmp` / `RecoveredFromBackup` / `ResetToDefault`) AND the corrupt original is preserved as `save.corrupt-[timestamp].json`, OR (b) `LoadedClean` fires with no renaming. Zero paths discard state silently. **BLOCKING CI**
- **AC-INV2 — R8.e Triggers Required Modal** — GIVEN `SaveLoadResult.Outcome == ResetToDefault`, WHEN the first post-load UI frame renders, THEN a blocking modal is presented to the player referencing the preserved `save.corrupt-*.json` path; gameplay does not advance until the modal is acknowledged. **PLAYMODE**
- **AC-INV3 — No Cross-Writer Race on save.tmp (Semaphore-Only)** — GIVEN a flush is in-flight via Task.Run AND `OnApplicationPause(true)` fires, WHEN both attempt to write, THEN only one holds `_writeLock` at any instant; the pause handler uses `_writeLock.Wait(timeout)` synchronously (NOT `task.Wait()`); no interleaved byte-stream corruption occurs in `save.tmp`. **PLAYMODE**
- **AC-INV4 — No Event Bus Subscription Leaks** — GIVEN Save / Load subscribes via `SubscribeAutoManaged`, WHEN `SaveManager` is destroyed and the scene reloads, THEN all subscriptions are unregistered before the next scene's `Awake`; Event Bus debug count shows zero leaked handles from Save / Load. **PLAYMODE**

### H.7 Schema Migration

- **AC-M1 — Current-Schema No-Op** — GIVEN a save written at the current `_saveFileVersion`, WHEN loaded, THEN no migrator is invoked, and the loaded state is byte-identical (after round-trip) to the pre-load state. **BLOCKING CI**
- **AC-M2 — Multi-Step Forward Migration** — GIVEN a save at `_saveFileVersion = 1` and current code version = 3, WHEN loaded, THEN migrators run in order v1→v2, then v2→v3; final state's `_saveFileVersion == 3`; no intermediate state is written to disk until the full chain succeeds. **BLOCKING CI**
- **AC-M3 — Per-Type Version Updated by Migrator** — GIVEN migrator v1→v2 runs on `LevelProgressState` (per-type `_schemaVersion = 1`), WHEN it completes, THEN the resulting state has `_schemaVersion = 2` as declared in the R4.c invariant table; a subsequent load does not re-apply the migrator. **BLOCKING CI**
- **AC-M4 — Oversized Migration Rejected** — GIVEN a migrator whose output would exceed `FileSizeBudget`, WHEN the pipeline checks post-migration size, THEN the pipeline aborts, preserves the pre-migration file as `save.corrupt-[timestamp].json`, and returns `ResetToDefault`. **BLOCKING CI**

### Gate Summary

Body-verified counts (2026-04-21 pass-3 regeneration). Every identifier below appears in the AC body above. Totals reconcile: **21 + 33 + 5 + 1 = 60.**

| Gate Level | Count | Identifiers |
|---|---|---|
| **BLOCKING CI** | **21** | AC-R1, AC-R4a, AC-R4b, AC-R4c, AC-R5, AC-R6, AC-R7, AC-R9a, AC-R10c, AC-R10d, AC-R12, AC-D3a, AC-EC-Migration1, AC-EC-Migration2, AC-EC-PlaytestIsolation, AC-P3, AC-INV1, AC-M1, AC-M2, AC-M3, AC-M4 |
| **PLAYMODE** | **33** | AC-R2, AC-R3a, AC-R7a, AC-R8a, AC-R8b, AC-R8b-reject, AC-R8c, AC-R8d, AC-R8e, AC-R10a, AC-R10b, AC-R11a, AC-R13, AC-R13a, AC-D1, AC-D2, AC-D3b, AC-D4, AC-EC-Storage1, AC-EC-Storage2, AC-EC-Race1, AC-EC-Race2, AC-EC-Race3, AC-EC-Uninit, AC-EC-Clock, AC-EC-Stale, AC-INT1, AC-INT2, AC-INT3, AC-INT4, AC-INV2, AC-INV3, AC-INV4 |
| **DEVICE (ADR-003)** | **5** | AC-R3b, AC-R11b, AC-P1, AC-P2, AC-P4 |
| **ADVISORY** | **1** | AC-R9b |

**Shift-left note for implementation**: the 21 BLOCKING CI criteria should have stub EditMode test files created at story start, before any `SaveManager` class is authored. The 33 PLAYMODE tests require a test scene fixture with a mockable filesystem layer (`IFileSystem` interface wrapping `File.*` calls) — scaffold this during the same sprint as `SaveManager` implementation, not deferred to QA hand-off. The 5 DEVICE ACs are gated on ADR-003 completion (reference device selection); the spec files can be authored now but test runs are parked until ADR-003 lands.

## Open Questions

| # | Question | Owner | Target Resolution |
|---|---|---|---|
| OQ-2 | **Cloud Save provider choice** — Unity Cloud Save vs. Firebase vs. Google Play Games Services + GameCenter. Decision deferred to plumbing sprint (months 4-5 per producer roadmap). Save / Load's local contract is provider-agnostic; this question only affects the post-launch Cloud Save adapter. | technical-director + producer | Plumbing sprint (months 4-5) |
| OQ-3 | **Content hash in save file root** — add `_fileHash: string` field to detect external modification (platform backup corruption, user file-manager edits, manual hex edits). At MVP the security surface is low (local single-player, no leaderboards); at post-launch (Cloud Save + leaderboards + competitive play) this may matter. A `SaveFileSecurity` ADR would capture the decision. | security-engineer + technical-director | Post-launch, before leaderboard ship |
| OQ-5 | **Per-platform `PauseFlushTimeout` relaxation** — iOS reliably gives ~5 s of grace; the 500 ms shared default wastes budget there. Should the default ship per-platform (iOS = 800 ms via `#if UNITY_IOS`, Android = 500 ms)? Decision deferred until post-MVP profiling on target iPhone hardware confirms the headroom. | performance-analyst + unity-specialist | Post-MVP device profiling pass |
| OQ-6 | **Addressables for save migration** — when Full Vision ships (5 biomes × 100 levels + cosmetics + analytics), the migration pipeline itself may become non-trivial to embed in the main binary. Should migrators ship as Addressable assets loadable on demand? Trigger condition: migration pipeline binary weight > 500 KB OR migration test suite > 10 min runtime. | technical-director | Post-launch when v2.x schema evolves |
| OQ-7 | **Ads SDK state storage key** — Ads SDK (v1.0) needs to persist frequency caps + last-shown timestamps. Does this go under a new `ad_history` coalescing key on `save.json`, or as a separate `ad_state.json` file under the same pattern as `settings.json`? `save.json` keeps it backed up to iCloud (user expectation: ad history travels with save); separate file keeps it device-local. | lead-programmer + Ads SDK GDD owner | During Ads SDK GDD authoring (v1.0 scope) |

### Closed in Pass-3 Revision (2026-04-21)

- **OQ-1 (level-fail banking)** — closed. Committed "keep banked" per Section C (Interactions → Level-fail banking policy) and pass-2 CD synthesis. Force-quit exploit documented as accepted design trade.
- **OQ-4 (Event Bus event additions)** — closed. The 7 new `gameplay.*` events (`currency.changed`, `tool.purchased`, `tool.upgraded`, `cosmetic.purchased`, `progression.node_unlocked`, `biome.unlocked`, `session.level_selected`) are committed as named proposals and patched into `design/gdd/event-bus.md` in this pass-3 revision. Save / Load is the subscriber contract reference; Event Bus GDD is the dispatch authority.
