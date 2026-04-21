# Event Bus — Design Review Log

Tracks `/design-review` passes on `design/gdd/event-bus.md`. Most recent at top.

---

## Review — 2026-04-21 — Verdict: NEEDS REVISION → RESOLVED (pass 1 fix session)
Scope signal: L
Specialists: game-designer, systems-designer, qa-lead, unity-specialist, performance-analyst, creative-director (senior)
Blocking items: 14 | Recommended: 16 | Nice-to-have: 3
Prior verdict resolved: First formal /design-review (CD-GDD-ALIGN resolutions from 2026-04-20 were baked into the draft but not formally reviewed).

### Key findings (adversarial):
- **[game-designer]** AC-F6 only measured VFX rendered frame — Android audio buffer latency (30-80ms) and haptic trigger unmeasured, so Pillar 5 contract was not actually proven. R6 undefined dispatch order created AV desync risk within 100ms envelope. R2 multi-hop chain latency unbudgeted (66ms lost at 30fps before subscribers execute).
- **[systems-designer]** D.2 queue capacity 64 was 20% below GDD's own derived worst case 80. R8 bootstrap buffer 16 not enumerated against MVP boot event count. First-frame bootstrap drain could blow T1 by ~6×. AC-F3 tested .NET List<T> implementation detail that doesn't hold.
- **[qa-lead]** AC-R1/P1 named "Memory Profiler" (snapshot tool, not per-frame counter) — wrong API. Grep regex patterns defective (greedy `.*`, unclosed capture group, CRLF). AC-F4/F5 no hardware specified; "no outliers" unverifiable. AC-F6 describes measurement Unity cannot do in PlayMode. Gate Summary counts wrong (PLAYMODE body=20 vs claim=14; MANUAL=0 claim false).
- **[unity-specialist]** C# 10 `readonly record struct` asserted; engine-reference confirms only C# 9. Bootstrap buffer had no specified storage type — heterogeneous typed structs would require boxing. AssetPostprocessor codegen does not run in headless `-batchmode` CI. Hard contradiction: R8 says buffer, Edge Cases says throw.
- **[performance-analyst]** Queue capacity 64 < 80 peak (convergent with systems-designer). Exception + warning log paths allocate via string interpolation. AC-P3 "T1 not delayed by T2" structurally unenforceable under synchronous dispatch. AC-P2 8 KB budget excludes delegate overhead (~9.6 KB for delegates alone).

### Creative-director senior synthesis:
Verdict NEEDS REVISION (surgical, not structural). "The bones are right. The instrumentation is what's missing." Three independent specialists flagged AC-F6 (highest-confidence blocker). Two independent specialists flagged queue capacity, ProfilerRecorder API, and bootstrap drain. Adjudicated specialist disagreements: FIFO drop retained + telemetry counter; Feedback Group concept for R6 mitigation; AC-F6 split into F6a/b/c; queue capacity 128.

### Resolution (pass-1 fix session):
User adjudicated four key design decisions:
1. **Payload pattern**: `readonly struct` (C# 9-safe fallback, losing auto `Equals`/`ToString` — implemented manually via `IEquatable<T>`).
2. **Bootstrap buffer**: Per-channel typed ring buffers (capacity 4 each) — eliminates boxing at cost of ~120 slots scene-wide (negligible memory).
3. **Codegen CI**: Commit generated `.cs` files to VCS + staleness-check CI workflow (PR-visible Analytics subscribers; avoids AssetPostprocessor headless unreliability).
4. **Logging**: `#if UNITY_EDITOR` only (not DEVELOPMENT_BUILD) — preserves AC-R1 zero-alloc in dev builds; release-build telemetry counters (R4.a + R8.b) replace on-device logging.

Plus CD-adjudicated: FIFO drop + telemetry, Feedback Group concept (R6.a + D.4 + AC-F4b), AC-F6 split, queue capacity 128 + floor 64.

### Fixes summary (14 blocking resolved):
1. C# version → `readonly struct` + OQ-7 logged for possible C# 10 revert
2. Queue capacity 64→128; safe-range floor 32→64
3. R8/Edge-Case contradiction reconciled; broken R5 ref fixed; R7 renumbered in sequence
4. Bootstrap buffer → per-channel typed (no boxing)
5. Codegen → committed files + staleness CI
6. First-frame drain → R8.a time-slicing + AC-BOOT
7. AC-F3 → constructor-time pre-allocation (`new List<T>(16)`)
8. AC-F6 → split F6a (auto bus latency) + F6b (device visual) + F6c (device audio)
9. AC-R1/P1 → `ProfilerRecorder` + `GC.Alloc` marker; steady state = 5s post-load
10. Grep patterns → POSIX-compatible, named CI workflow
11. AC-F4/F5/P3 → CI hardware P95/P99; Mali-G52 deferred to AC-F6b/c MANUAL
12. Logging → `#if UNITY_EDITOR` only + release counters
13. AC-P3 → reworded for subscriber-discipline dependency (≤5ms per T2 subscriber)
14. R6 → Feedback Group concept (R6.a + D.4 + AC-F4b aggregate latency)

### 16 recommended + 3 nice-to-have also integrated:
Runtime telemetry (R4.a, R8.b, AC-REL); multi-hop chain latency (D.5); depth-guard reset contract (R2.a + AC-R2b); PlayerLoop hook point (R2.b); string payload ban (R7.a); subscribers-list concrete-typing contract; ChannelID typed-struct spec; SubscribeAutoManaged pool-object allocation note; SubscriptionHandle.Unsubscribe() (not IDisposable); D.3 subscriber capacity 8→16; AC-P2 memory budget recalculated 8→24 KB; AC-EC7 mid-dispatch scene unload; AC-R7a string-field ban; R7 Roslyn analyzer scoped to two-tier grep; T1 fixed wall-clock clarification for 30fps floor; GopherLaunchedPayload string→enum migration.

### Gate Summary (recounted pass-1):
- BLOCKING CI: 12 (was claimed 10, actual 10)
- PLAYMODE: 13 (was claimed 14, actual 20 — now 13 after AC merges)
- ADVISORY: 1 (was 3, AC-R6 promoted to PLAYMODE)
- MANUAL: 2 (was claimed 0 — AC-F6b + AC-F6c require Mali-G52 device capture)

### Status after pass 1:
All 14 blockers resolved; revised GDD awaits independent /design-review pass-2 verification in a fresh session. Per CD synthesis: "Do NOT advance to downstream GDDs until resolved — 12 systems inherit these decisions."

---

## Review — 2026-04-21 — Verdict: NEEDS REVISION (pass 2, fresh session)
Scope signal: L
Specialists: game-designer, systems-designer, qa-lead, unity-specialist, performance-analyst, creative-director (senior)
Blocking items: 20 + 1 structural amendment | Recommended: 16 | Nice-to-have: 3
Prior verdict resolved: Partially — pass-1's 14 items held, but pass-2 surfaced 20 NEW blockers (test instrumentation precision, measurement-contract gaps, Pillar 5 metric reformulation) that pass-1 did not examine.

### Key findings (adversarial, pass 2):
- **[game-designer]** D.4 `group_latency = max(completion) - fire` measures aggregate completion, NOT subscriber ONSET. 80ms AV desync can pass 100ms-aggregate AC; perceptual synchrony threshold ~22ms. R6.a called this "the measurable Pillar 5 guarantee" but it can't catch perceptible desync. FIFO drop has no DEVELOPMENT_BUILD warning visible during playtest — Pillar 5 silent violations.
- **[systems-designer]** D.2 queue 128 derivation wrong: 40 RB2D × 2 emits × 2 namespaces = 160 worst case, not 80. 128 is BELOW dual-namespace worst case. D.5 chain latency at 30fps: 3 hops = 99.99ms, zero headroom. D.4 semantic gap for async Analytics subscribers (R9 PostLateUpdate vs synchronous handler return). AC-BOOT 8ms budget not derived (66μs/event requires stub subscribers). Gate Summary count 12 vs body 14.
- **[qa-lead]** ProfilerRecorder cannot scope GC.Alloc to `EventBus.Fire<T>` call-stack (AC-R1/P1 untestable). AC-R2b `Thread.Sleep(50)` stall is tautological. AC-R3 regex misses `$"..."`. AC-R7 Python filter script unnamed. AC-R7a multi-line grep unspecified. AC-F3 reflection into `List<T>._items` is BCL-fragile. AC-F6b/c P99 on N=20 = P100 (statistically invalid). AC-REL overflow/exception injection has no fixture spec.
- **[unity-specialist]** R2.b PlayerLoop "injected into PreUpdate.EventBusPlayerLoopSystem" conflates type identity with node path. R2.b physics T1 claim wrong (FixedUpdate fires 0–N per rendered frame). R4.a Dictionary has no pre-population contract — lazy key insert allocates. R8 SO bootstrap buffer has no reset-on-OnEnable (cross-scene contamination). AC-F6a uses `Time.realtimeSinceStartup` float (0.12–0.61ms LSB precision). R9 staleness CI missing explicit `AssetDatabase.ImportAsset()` (silent false-pass). OQ-7 closeable now: Unity 6.3 ships C# 9 only.
- **[performance-analyst]** AC-P2 24 KB derivation arithmetically wrong (delegate size vs slot size confusion). AC-BOOT 8ms excludes subscriber execution (120×6×50μs=36ms reality). AC-R1/P1 10-second window misses rarely-fired channels + JIT cold-path. R4.a Dictionary pre-pop convergent with unity-specialist. R9 Analytics batch buffer size/resize policy unspecified.

### Creative-director senior synthesis:
Verdict NEEDS REVISION (surgical, not structural — but with one structural amendment required). "The bones remain correct. Pass-1 fixed the right things; pass-2 found measurement-contract precision gaps and one Pillar 5 metric reformulation." Pass-2 found ~20 surgical blockers concentrated in four areas: test instrumentation API (ProfilerRecorder scope), measurement contract gaps (what ACs actually prove vs claim), missing init/reset contracts (Dictionary pre-pop, SO buffer reset), and Pillar 5 metric reformulation (onset-spread, not just completion). **Structural amendment**: D.4 + R6.a + AC-F6 + R2.b must be reframed — add `intra_group_onset_spread ≤ 22ms` formula; reclassify physics-originated events as T2 or carve out; clarify Analytics is T3 (not in Feedback Groups). None require rethinking architecture or pillar set. FRESH SESSION STRONGLY RECOMMENDED for revision (20+ blockers needs clean context). Validation criterion: pass 3 should be gate-check-ready, not a third adversarial round — if pass 3 finds >3 new blockers, escalate to structure review.

### Adjudicated specialist disagreements:
1. **P99 sample size (AC-F6b/c)**: P95@N=20 vs P99@N=100 → CD calls **P95@N=20** (statistically honest, cheaper).
2. **R8.a drain cap**: tuning knob row vs hardcoded → CD calls **hardcoded invariant** (tuning knob on invariant defeats purpose).
3. **AC-R1 vs AC-P1 overlap**: merge vs annotate → CD calls **merge** (duplication, not defense-in-depth).
4. **Queue capacity**: raise 128→256 vs correct derivation to justify 128 → no default call; either path acceptable pending empirical data.

### Next action (user-adjudicated):
User selected Option [A]: stop + /clear + revise in fresh session. Systems-index updated to NEEDS REVISION (pass 2). Pass-3 revision session will fix 20 blockers + structural D.4 amendment in prioritized order: (1) structural amendment first, (2) Tier 1 convergent bundles (B1–B4), (3) test-harness blockers, (4) single-specialist architecture blockers, (5) Tier 3 sweep.

---

## Revision — 2026-04-21 (pass 3 fix pass, fresh session per CD recommendation)

User ran `/design-review design/gdd/event-bus.md` in a fresh session (initially selected Data Registry by mistake; pivoted to Event Bus; selected "Pass-3 fix revision" instead of another adversarial review round per CD pass-2 guidance that "Pass 3 should be gate-check-ready, not a third adversarial round"). All 20 pass-2 blocking items + 1 structural D.4 amendment resolved in this revision pass.

### User-adjudicated design decisions (pass-3)

| # | Decision | Rationale / Outcome |
|---|----------|---------------------|
| 1 | Queue capacity fix | **Raise to 256**. Dual-namespace worst case (160) × 1.6× safety margin. Derivation corrected in D.2; Tuning Knobs updated; R2 text updated. |
| 2 | Chain latency constraint | **Hard 2-hop cap**. `MAX_CHAIN_DEPTH = 2` as architectural rule; any 3+ hop chain blocks implementation until fan-out restructure. AC-F5 enforces via static analysis. "Pillar 5 known risk with playtest sign-off" escape hatch removed. |
| 3 | Dev-build overflow visibility | **Once-per-session DEV_BUILD log + counter**. Preserves R1 zero-alloc steady-state (first overflow allocates one log string; subsequent increments counter-only). Playtesters see single actionable error, not log flood. |
| 4 | R2.b physics wording | **Reframe as wall-clock latency**. R2.b rewritten: synchronous dispatch model; T1 = wall-clock ≤ 16.67ms from Fire to last subscriber return; "same rendered frame" retired as a semantic guarantee. |

### Blocking Items Addressed (pass 3)

**Structural amendment (D.4 Pillar 5 metric reformulation)**

| # | Specialist | Fix Applied |
|---|-----------|-------------|
| D.4-AM | game-designer pass-2 #1 (3-specialist convergence) | Added `intra_group_onset_spread = max(entry_ts) - min(entry_ts) ≤ 22ms` as a second independent metric alongside `group_latency`. Both must pass. Resolves 80ms AV desync hiding inside 100ms aggregate budget. R6.a, AC-F4b, Tuning Knobs, Interactions table all updated. Analytics explicitly excluded from Feedback Groups as T3 tier. |

**Tier-1 convergent bundles (5)**

| # | Bundle | Fix Applied |
|---|--------|-------------|
| B1 | D.5 chain latency at 30fps (3 specialists) | Hard 2-hop cap rule; D.5 table enumerates declared chains (Collection→Currency→HUD, Hazard→Currency→HUD both 2-hop, 72ms at 30fps); AC-F5 elevated to BLOCKING CI with chain-depth static-analysis check + latency validation. |
| B2 | R4.a Dictionary pre-pop contract (unity+perf) | Pre-population contract added: `new Dictionary<ChannelID, int>(N, ChannelIDComparer.Default)` at Awake with explicit `_counts[id] = 0` loop; eliminates lazy-key-insert allocation; preserves R1 steady-state. |
| B3 | AC-R1/P1 ProfilerRecorder scope (qa+perf) | Merged into unified AC-R1 per CD adjudication. Three-phase protocol: scene-load (0-3s) → cold-fire JIT-warmup sweep + GC.Collect (3-5s) → steady-state window (5-20s). Primary signal: thread-local `GC.GetAllocatedBytesForCurrentThread()` delta per Fire call. Secondary: ProfilerRecorder GC.Alloc frame-granularity. Both must read 0. AC-P1 marker retained as merge pointer. |
| B4 | Gate Summary count mismatch (sd+qa) | Recounted by body enumeration to 36 AC identifiers: 16 BLOCKING CI / 16 PLAYMODE / 2 ADVISORY / 2 MANUAL. Table rebuilt; pass-2 body-vs-claim inconsistency closed. |
| B5 | D.4 Pillar 5 metric (covered in structural amendment above) | — |

**Tier-2 + surgical (15 single-specialist blockers)**

| # | Item | Fix |
|---|------|-----|
| T2-1 | D.2 queue 128 < 160 dual-namespace worst | Raised 128→256 with corrected derivation showing 40 RB2D × 2 emits × 2 namespaces = 160 worst; 256 = 1.6× margin. R2 text, D.2, Tuning Knobs all updated. |
| T2-2 | R2.b PlayerLoop identity vs path conflation | Split into explicit subsystem type (`EventBusPlayerLoopSystem` struct with `delegate*<void>`) + path (`PreUpdate > EventBusPlayerLoopSystem` last child) + `[RuntimeInitializeOnLoadMethod]` timing. |
| T2-3 | R2.b physics T1 wrong at 30fps | Rewritten as wall-clock latency framing; Fire is synchronous; physics events dispatch at FixedUpdate call site; PlayerLoop hook is ONLY for deferred-queue drain. |
| T2-4 | R8 SO bootstrap buffer reset contract missing | OnEnable reset sequence added: read/write indices zeroed + `Array.Clear(slots, 0, 4)`. Closes cross-scene contamination path. |
| T2-5 | AC-F6a Time.realtimeSinceStartup float precision | Replaced with `Stopwatch.GetTimestamp()` long ticks + explicit rejection rationale (0.12–0.61ms LSB precision at runtime ages unsuitable for 2ms AC). |
| T2-6 | R9 staleness CI missing ImportAsset | Added explicit `AssetDatabase.ImportAsset(path, ImportAssetOptions.ForceUpdate)` for both taxonomy SOs before codegen trigger. Closes silent-false-pass path. |
| T2-7 | AC-R3 regex misses `$"..."` | Added `\$?` to regex class for interpolated-string coverage; self-verifying fixture at `tests/editmode/event-bus/fixtures/string-literal-violations/`. |
| T2-8 | AC-R7 Python script unnamed | Named committed script: `tools/ci/eventbus-linq-on-payload-filter.py` with own unit tests at `tests/editmode/ci-scripts/test-eventbus-linq-filter.py`. |
| T2-9 | AC-R7a multi-line grep unspecified | Replaced grep with committed Python scanner `tools/ci/eventbus-string-field-scanner.py`; grep rejected as approach; unit tests cover single-line, multi-line-decl, generic-constraint cases. |
| T2-10 | AC-F3 reflection into List._items fragile | Replaced with owned `ChannelSubscriberList<T>` wrapper + internal `BackingArrayRef` + `SubscriberCapacity_TestOnly` via `[InternalsVisibleTo]`. No BCL reflection. |
| T2-11 | AC-F6b/c P99@N=20 statistically invalid | Changed to P95@N=20 per CD pass-2 adjudication; explicit rationale inline. |
| T2-12 | AC-REL fixture undefined | Three named fixture files at `tests/playmode/event-bus/fixtures/release-telemetry/`: `ZeroSubscriberFireInjector.cs`, `QueueOverflowInjector.cs`, `RecursiveFireInjector.cs` + `MockAnalyticsService.cs`. Elevated from PLAYMODE to BLOCKING CI. |
| T2-13 | AC-BOOT 8ms excludes subscriber cost | Split AC-BOOT-A (bus-overhead-only ≤ 8ms, BLOCKING CI) + AC-BOOT-B (total-with-realistic-subscribers ≤ 40ms, ADVISORY). Subscriber cost correctly attributed to consumer-system GDDs. |
| T2-14 | AC-P2 24KB math wrong | Rederived per-component table: 128-byte backing array + 384-byte delegates + 128-byte bootstrap + 200-byte SO overhead = ~850 bytes/channel × 30 = ~25.5 KB; ceiling raised to 30 KB with 7.5× margin. "40 bytes/delegate" phrasing corrected. |
| T2-15 | R9 batch buffer size/resize unspecified | Fixed-capacity `AnalyticsEventSlot[512]` at 80 bytes/slot = 40 KB; NO RESIZE policy (overflow drops new event + increments `analytics_batch_overflow_count`); flush at `PostLateUpdate > AnalyticsBatchFlushSystem`. |

**Tier-3 sweep (3 items)**

| # | Item | Fix |
|---|------|-----|
| T3-1 | OQ-7 (C# 10 consideration) | Closed — Unity 6.3 ships C# 9 only per engine-reference; `readonly struct` is final. |
| T3-2 | R8.a drain cap tuning knob | Removed from Tuning Knobs table per CD pass-2 adjudication; hardcoded invariant with explicit rationale ("making it tunable defeats spike-prevention contract"). |
| T3-3 | AC-R2b tautological Thread.Sleep | Replaced with deterministic `DepthGuard_TestOnly` int accessor + explicit `yield return new WaitForSeconds(0.5f)` coroutine-tolerance probe. |

### Specialist disagreements — all resolved

| # | Disagreement | Resolution (CD pass-2 adjudication) | Applied pass-3? |
|---|--------------|---------------------------------------|------------------|
| 1 | P95@N=20 vs P99@N=100 (AC-F6b/c) | **P95@N=20** (statistically honest, cheaper) | ✓ |
| 2 | R8.a drain cap tuning knob vs hardcoded | **Hardcoded invariant** | ✓ |
| 3 | AC-R1 / AC-P1 overlap — merge vs annotate | **Merge** | ✓ |
| 4 | Queue capacity raise vs redrivation | No CD default — pass-3 user adjudication chose raise to 256 | ✓ |
| 5 | R2.b physics framing (pass-3 net-new) | User adjudication chose wall-clock latency reframing | ✓ |
| 6 | Chain depth (pass-3 net-new) | User adjudication chose hard 2-hop cap | ✓ |
| 7 | Dev-build overflow visibility (pass-3 net-new) | User adjudication chose once-per-session DEV_BUILD log | ✓ |

### Remaining Open (deferred RECOMMENDED items — not blocking pass-4 verification)

Pass-2's 16 recommended items were not systematically re-addressed in this revision pass (focus was the 20 blocking + 1 structural amendment). Recommended items can be swept in post-sprint backlog review. No nice-to-have items were landed in pass-3.

### Specialist Key Quotes (pass 3 revision)

- **[creative-director, pass-2 synthesis recalled]**: "Pass 3 should be gate-check-ready, not a third adversarial round. If pass 3 finds >3 new blockers, escalate to structure review." Pass-3 fix pass resolved all 20 pass-2 blockers + 1 structural amendment; targeted pass-4 verification (not full adversarial) is the expected next step per this guidance.
- **[user]**: four design decisions resolved via pre-revision AskUserQuestion widget (queue 256, 2-hop cap, once-per-session DEV_BUILD log, wall-clock R2.b framing). All four chose the CD/specialist-recommended option.

### Estimated next step

Per CD pass-2 recommendation: **targeted pass-4 verification** in a fresh session — verify the D.4 structural amendment landed + 5 Tier-1 convergent bundles landed + 15 Tier-2 surgical edits landed. Explicitly NOT a full adversarial pass-4 review. If verification clean, promote systems-index to APPROVED. If verification finds > 3 regressions, escalate to structure review.

---

## Review — 2026-04-21 — Verdict: APPROVED (pass-3 revision accepted without further review)

User selected **Option B (Accept revisions and mark Approved)** after the pass-3 fix pass completed. Rationale aligned with CD pass-2 guidance: "Pass 3 should be gate-check-ready, not a third adversarial round." Pass-3 resolved all 20 pass-2 blockers + the structural D.4 Pillar 5 amendment via a fresh-session fix pass with 4 pre-adjudicated design decisions; continuing to adversarial pass-4 was deemed lower-value than moving to Input System (next Foundation GDD in design order).

**Risk accepted:** 20+ fixes land without independent verification. Mitigation: the fixes all have prescribed specifications in pass-3 revision entry; implementation-sprint story-stubs will surface any remaining ambiguity before code is written. Targeted pass-4 remains an option if a later consistency-check or architecture review surfaces suspect revisions.

**Final gate totals (pass-3):** 16 BLOCKING CI + 16 PLAYMODE + 2 ADVISORY + 2 MANUAL = 36 AC identifiers. 25 test scaffolding units.

**Prior verdicts resolved:** pass-1 14 structural + pass-2 20 precision + D.4 structural amendment — all resolved. Event Bus promoted to Approved status alongside Data Registry; both Foundation #1 and Foundation #2 are now ready for architecture phase. Save/Load (Foundation #3) still NEEDS REVISION per its own review log.
