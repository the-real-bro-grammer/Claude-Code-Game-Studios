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
