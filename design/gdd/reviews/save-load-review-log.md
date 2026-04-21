# Save / Load — Review Log

Revision history for `design/gdd/save-load.md`. Each entry captures verdict, specialists consulted, blocker counts, and a summary of findings so re-reviews can track resolution.

---

## Review — 2026-04-21 — Verdict: MAJOR REVISION NEEDED

**Scope signal**: L
**Specialists**: game-designer, systems-designer, unity-specialist, performance-analyst, qa-lead, creative-director (senior synthesis)
**Blocking items**: 10 | **Recommended**: 17+ | **Nice-to-have**: 6
**Prior verdict resolved**: First review (CD-GDD-ALIGN APPROVE at authoring time was alignment-only, not an independent review pass)

### Summary

GDD is engineering-forward with unusually thorough detail for a Foundation system, but has load-bearing defects spanning three classes: formula correctness (D.1/D.2/D.3 output ranges inconsistent with variable-table inputs; D.3 conflates size estimate with budget), factual engine claims (R10's JsonUtility thread-safety rationale is false — thread-safe for POCOs since Unity 2020.1, which cascades into a 30ms unbudgeted main-thread frame cost), and a Player Fantasy promise ("the run resumes cleanly") that the Resume Policy cannot deliver (player lands on Biome Map).

Creative-director synthesis: **"R10's rationale is factually wrong, and that correction may collapse the entire main-thread contention problem."** Sequence revision as: (a) re-ground R10, (b) re-derive formulas + R8.b version guard, (c) rewrite Player Fantasy + split AC-R9 into CI-enforceable + advisory halves, (d) coordinate Event Bus additions and name reference device before re-review.

### Top Blockers

1. **[unity-specialist]** R10 JsonUtility thread-safety rationale is factually wrong (dominates architecture)
2. **[performance-analyst]** Unbudgeted main-thread frame cost for multi-key flushes (up to 30ms / 180% of 60fps frame)
3. **[systems-designer]** D.1/D.2/D.3 formula boundaries broken in both directions
4. **[systems-designer, unity-specialist]** R8.b tmp-promotion is silent stale-state vector (no version ordering guard)
5. **[game-designer]** Player Fantasy ("run resumes cleanly") contradicts Resume Policy (lands on Biome Map)
6. **[qa-lead]** AC-R9 MANUAL gate for no-pay-to-win anti-pillar — needs CI grep split (AC-R9a BLOCKING + AC-R9b advisory)
7. **[performance-analyst, qa-lead]** No named reference device for DEVICE-tier ACs; AC-P1 emulator usage contradictory
8. **[unity-specialist]** Generic `[Serializable] struct KVPair<K,V>` silently drops fields under JsonUtility
9. **[unity-specialist]** `task.Wait()` on main thread deadlock risk under IL2CPP
10. **[game-designer]** OQ-1 (level-fail banking) is load-bearing, not open — must be committed in this GDD

### Pillar Alignment

- **Pillar 3 (Mastery Through Economy)**: Served in intent; R9's manual enforcement leaves anti-pillar (no-pay-to-win) without mechanical teeth
- **Pillar 4 (Prep + React)**: At risk — Resume Policy landing on Biome Map compromises "React" promise for mid-run interruptions
- **Player Fantasy**: Not defensible as written

### Specialist Disagreements

None on load-bearing facts. Specialists differed only on emphasis. All agreed the GDD is not implementation-ready.

### Action Items for Revision

1. Re-derive R10 rationale from real constraints (determinism, allocation profile, pause-handler synchrony) or relax to permit off-main serialization
2. Add explicit serialization frame budget (if R10 stands) to Formulas or Tuning Knobs
3. Fix D.1 output range (current 50–2000 inconsistent with inputs 200–1000 × 0.1–4.0)
4. Fix D.2 output range (max 5000×0.6=3000 exceeds stated 2500 ceiling)
5. Separate D.3 into `FileSizeEstimate` formula + `FileSizeBudget` constant with independent derivation
6. Add version-ordering guard to R8.b (`tmp._saveFileVersion >= json._saveFileVersion`)
7. Rewrite Player Fantasy to match Resume Policy, OR change Resume Policy to mid-level continuity (scope-significant)
8. Split AC-R9 into AC-R9a (BLOCKING CI grep: Cost/Price/Threshold/Rate/Duration/Quota/Multiplier/Limit) + AC-R9b (ADVISORY semantic review)
9. Commit OQ-1 answer in this GDD (banked-during-rush vs end-of-rush)
10. Block DEVICE ACs on ADR-003 naming a reference device; remove emulator from AC-P1
11. Replace generic `KVPair<K,V>` with concrete non-generic structs; extend R12 ban list (HashSet, nested Lists, readonly fields, record types)
12. Replace `task.Wait()` pause pattern with pure synchronous pause-path writes (eliminates IL2CPP deadlock vector)
13. Add 7 coverage-gap ACs (FirstRun, RecoveredFromTmp, ConfigureAwait, playtest-log isolation, OnDestroy ordering, Settings schema version, Uninitialized guard)
14. Coordinate OQ-4's 7 Event Bus channel additions into Event Bus GDD before implementation
15. Add R8.d "light toast required" default (remove silent option); spec R8.e modal tone contract

### Next Re-Review

Run `/design-review design/gdd/save-load.md` in a fresh session after revisions are complete. Expect at least one more full-review cycle before this GDD is implementation-ready.

---

## Review — 2026-04-21 — Verdict: MAJOR REVISION NEEDED (Pass 2 — Independent Validation)

**Scope signal**: L
**Specialists**: game-designer, systems-designer, unity-specialist, performance-analyst, qa-lead, creative-director (senior synthesis)
**Blocking items**: 15 | **Recommended**: 14 | **Nice-to-have**: 5
**Prior verdict resolved**: No — GDD unchanged since pass-1 (git: single commit 506a4b9 added the GDD; no revisions). 0 of 10 pass-1 blockers resolved. Pass-2 ran against the same document pass-1 reviewed.

### Summary

Independent adversarial pass confirms every pass-1 blocker and adds new structural findings. Five specialists converged with unusually high agreement — four of five independently flagged the R10 JsonUtility thread-safety factual error, R12 generic `KVPair<K,V>` field-drop bug, R8.b tmp-promotion version-guard gap, and R11 `task.Wait()` IL2CPP deadlock vector. Two flagged D.1 formula range error. Pass-2 additionally surfaces: (a) Gate Summary arithmetic wrong — header claims 34 ACs, body contains 43; (b) R4 migrator-chain indexing ambiguity between root `_saveFileVersion` and per-type `_schemaVersion` with no CI enforcement; (c) R7 producer-side coherency race on shared coalescing keys; (d) R12 banned-type list materially incomplete (missing `HashSet<T>`, `readonly`, `record`, properties, interface fields, nested `List<List<T>>`); (e) `SemaphoreSlim` disposal missing from `OnDestroy`; (f) R3 `fsync` durability overstated for certain Android OEMs; (g) multi-key main-thread flush cost unbudgeted against 16.67 ms frame (resolves automatically when R10 is corrected); (h) AC-P3 32 KB ceiling unanchored in D.3 math; (i) D.1 advisory target 2000 ms is 50–250× above expected latency, masking pipeline regressions.

### Top Blockers

1. **[unity-specialist, performance-analyst]** R10 JsonUtility thread-safety rationale is factually wrong (POCOs thread-safe since Unity 2020.1) — keystone blocker; correcting it collapses multiple downstream findings (unresolved pass-1)
2. **[unity-specialist, systems-designer]** R12 generic `KVPair<K,V>` silently drops all fields under JsonUtility — must be concrete non-generic structs (unresolved pass-1)
3. **[unity-specialist]** R12 banned-type list incomplete — missing HashSet, readonly, record, properties, interfaces, `List<List<T>>` (**new pass-2 finding**)
4. **[unity-specialist, systems-designer, performance-analyst]** R11/AC-EC-Race2 `task.Wait()` deadlocks on IL2CPP if any internal continuation lacks `ConfigureAwait(false)`; no enforcement AC (unresolved pass-1)
5. **[systems-designer, game-designer]** D.1 output range [50–2000] mathematically wrong — actual [20, 4000]; default 500×4.0=2000 coincidentally matched ceiling, disguising error (unresolved pass-1)
6. **[systems-designer]** D.2 ceiling 2500 ms violated by max inputs (5000×0.6=3000) (unresolved pass-1)
7. **[systems-designer, unity-specialist]** R8.b tmp-promotion missing version-ordering guard; stale tmp can silently downgrade save (unresolved pass-1)
8. **[game-designer]** Player Fantasy ("run resumes cleanly", "loadout...exactly how they left it") contradicts Resume Policy (lands on Biome Map) (unresolved pass-1)
9. **[game-designer]** OQ-1 (level-fail banking) is load-bearing; current subscription model forces "keep banked" answer unless rollback added — must be committed in this GDD (unresolved pass-1)
10. **[qa-lead, game-designer]** AC-R9 MANUAL gate for no-pay-to-win anti-pillar violates project "ACs must be independently testable" standard; needs AC-R9a CI allowlist + AC-R9b advisory split (unresolved pass-1)
11. **[qa-lead]** AC-R11 dual PLAYMODE+DEVICE tag is one AC with two incompatible test surfaces; split required (unresolved pass-1)
12. **[qa-lead, performance-analyst]** No named reference device for AC-P1/P2/P4; AC-P1 "Mali-G52 emulator image" is internally contradictory (unresolved pass-1)
13. **[qa-lead]** Gate Summary arithmetic wrong — header says 34 ACs, actual body count is 43; coverage column lists nonexistent R8a–R8e (**new pass-2 finding**)
14. **[performance-analyst]** Multi-key main-thread flush: 3-key level-complete = 15 ms (90% of frame); 6-key = 30 ms (180%) — unbudgeted; resolves automatically when R10 corrected (unresolved pass-1)
15. **[unity-specialist]** R3 `File.Replace` + `fsync` durability overstated for Android OEMs (Samsung A-series OneUI 3.x returns fsync() before eMMC write cache flush) (**new pass-2 finding**)

### Pillar Alignment

- **Pillar 3 (Mastery Through Economy)**: Intent preserved; R9 MANUAL enforcement leaves anti-pillar (no-pay-to-win) unprotected mechanically. AC-R9a/R9b split is non-negotiable from pillar-integrity standpoint, independent of technical blockers.
- **Pillar 4 (Prep + React)**: Not actively undermined, but Player Fantasy prose overpromises beyond what Pillar 4 requires. Fantasy vs Resume Policy gap is where player trust erodes.
- **Player Fantasy**: Not defensible as written.

### Specialist Disagreements

**None substantive.** Unusually high convergence across 5 independent adversarial passes — all five agreed this GDD is not implementation-ready. Only tension is framing: unity-specialist treats R10 as correctness, performance-analyst treats it as frame-budget; both resolve identically. One user design call required: OQ-1 ("keep banked" vs "forfeit"). game-designer argues architecture forces "keep banked"; systems/qa neutral.

### Resolution Sequence (CD senior synthesis)

Pass-1's sequence remains correct, with one insertion:

1. **Re-ground R10** (JsonUtility POCO threading). Keystone — collapses frame-budget + GC + D.2 example downstream.
2. **Fix R12** (concrete non-generic structs + expanded banned-type list including HashSet, readonly, record, properties, interfaces, List<List>).
3. **Add `ConfigureAwait(false)` enforcement AC** for R11 and pause handler path.
4. **Re-derive D.1/D.2/D.3/D.4** with correct ranges, inline worked examples, anchor AC-P3 to D.3 formula, separate FileSizeEstimate from FileSizeBudget.
5. **Resolve OQ-1** — commit "keep banked"; document kill-app exploit as accepted or mitigated.
6. **Rewrite Section B Player Fantasy** to match Resume Policy (career-progress framing, not single-run state).
7. **Regenerate AC table**: split R9→R9a/R9b, split R11→R11a/R11b, add R8a–R8e per outcome, add 7 coverage-gap ACs from prior action 13 (FirstRun, RecoveredFromTmp, ConfigureAwait, playtest-log isolation, OnDestroy ordering, Settings schema version, Uninitialized guard), name reference device from ADR-003, correct header count to true total.
8. **Patch OQ-4** — coordinate 7 proposed `gameplay.*` events into approved Event Bus GDD before Save/Load implementation sprint.

### Escalation Note

Not yet warranted — pass-2 ran against the original authored GDD by explicit user sequencing (Data Registry + Event Bus revised first; Save/Load queued). If pass-3 returns with the same blockers, escalate to producer for Foundation-#3 sequencing impact on downstream progression/economy systems.

### Next Re-Review

Revise in a **FRESH session** per CD recommendation (40–60% rewrite; clean context budget needed for GDD + prior review log + Data Registry + Event Bus + Unity 6.3 reference docs, ~25–35k tokens input before writing). Run `/design-review design/gdd/save-load.md` after revision completes for pass-3 validation.

---

## Revision — 2026-04-21 — Pass-3 Fix Pass (NOT adversarial — revision against pass-1 + pass-2 blockers)

**Scope**: All 10 pass-1 blockers + 15 pass-2 blockers addressed. User chose "Skip re-review, revise now" in current session (noting CD had recommended fresh session; user acknowledged session-state warning and proceeded). 3 design decisions pre-answered via single `AskUserQuestion` widget: (1) R10 threading = **off-main serialize + I/O**, (2) Reference device = **defer to ADR-003 with placeholder**, (3) OQ-4 = **patch Event Bus GDD now in this session**.

### Applied Changes

**Core rules reground:**

- **R10 rewrite** — corrected false thread-safety premise. New rule: snapshot build on main; `JsonUtility.ToJson` + file I/O both run on `Task.Run`. Added R10.a POCO self-containment (no `UnityEngine.Object`-derived fields in persistable types) with BLOCKING CI grep (AC-R10c). Added `.ConfigureAwait(false)` enforcement (AC-R10d). Pause-path explicitly documented as exception (synchronous serialize on main during pause, cheap under OS grace budget).
- **R11 rewrite** — eliminated `task.Wait()` deadlock vector entirely. Pause handler now uses `_writeLock.Wait(pause_flush_timeout_ms)` — the **synchronous** `SemaphoreSlim` overload — which does not engage async/await machinery and cannot IL2CPP-deadlock. Semaphore disposal in `OnDestroy` added (AC-EC-Race3).
- **R12 rewrite** — concrete non-generic `{Domain}KVPair` structs with example naming convention. Banned-type list expanded to exhaustive: `HashSet`/`SortedSet`/`Queue`/`Stack`, `List<List<>>`, `readonly`/`init` fields, properties, `record`, interface-typed fields, `DateTime`/`TimeSpan`/`Guid`, nullable value types, `UnityEngine.Object`-derived types. Test requirement upgraded to worker-thread round-trip.
- **R8.b tmp-promotion** — added version-ordering guard (`tmp._saveFileVersion >= json._saveFileVersion`) with stale-tmp rejection branch (AC-R8b-reject) preserving rejected tmp as `save.stale-tmp-*.json`.
- **R3 atomic write** — added OneUI 3.x/eMMC durability caveat; architectural mitigation via R8.b tmp-promotion clearly documented.
- **R7.a producer coherency** — new sub-rule requiring producers to use `GetState<>()` before building snapshots, with PLAYMODE AC-R7a verifying cache reflects `RequestFlush` update synchronously.
- **R4 disambiguation** — root `_saveFileVersion` is migration driver; per-type `_schemaVersion` is consistency assertion. R4.c invariant table introduced; AC-R4c enforces gaplessness and per-type-version assertion at load time. R5 migrator signature aligned to `(rootFrom → rootTo)`.

**Formulas re-derived:**

- **D.1** — range corrected (was 50–2000, now 30–3000). `safety_margin_factor` range fixed to [0.1, 1.0] (was [0.1, 4.0] which produced non-signal). Default changed to 250 ms (was 2000 ms). Boundary examples added.
- **D.2** — upper bound widened to 3000 ms (was 2500; violated by 5000 × 0.6). Sync path re-derived under off-main-serialize model: multi-key pause cost 54–258 ms, comfortable under 500 ms default. Enforcement switched to semaphore not cancellation token.
- **D.3 split** — D.3a FileSizeEstimate (formula) vs D.3b FileSizeBudget (constant 64 KB with independent derivation). AC-P3 32 KB anchored to D.3a (3× headroom against ~10 KB MVP estimate). `SaveFileSizeBudget` renamed to `FileSizeBudget` throughout.
- **D.4** — unchanged; multi-key pause cost concern resolved automatically by R10 correction.

**Pseudocode updated:**

- Coalescing queue map now holds POCOs, not serialized strings (serialization deferred to worker).
- Dispatch loop shows `await _writeLock.WaitAsync().ConfigureAwait(false)` + `JsonUtility.ToJson` on worker.
- Pause path uses `_writeLock.Wait(PauseFlushTimeout)` — synchronous semaphore, not `task.Wait()`.

**Player Fantasy rewritten:**

- Career-progress framing replaces single-run state promise. Lines that contradicted Resume Policy ("the run resumes cleanly", "the loadout they were tuning is exactly how they left it") replaced with explicit "banked + unlocked = permanent; in-flight Rush Phase = accepted physics-constraint loss". Biome Map landing is now explicitly reflected in the prose.

**Open Questions closed:**

- **OQ-1** — committed "keep banked" on level-fail; kill-app exploit documented as accepted trade. Captured in Section C interaction rules.
- **OQ-4** — 7 new `gameplay.*` events committed by name and patched into Event Bus GDD's Interactions table in this session. Save/Load Interactions table updated from "PROPOSED addition" → "Exists in Event Bus GDD (pass-3 addition 2026-04-21)".

**AC table regenerated (60 identifiers, body-verified):**

- Split AC-R9 into AC-R9a (BLOCKING CI grep on banned field-name regex) + AC-R9b (ADVISORY semantic review).
- Split AC-R11 into AC-R11a (PLAYMODE lock correctness) + AC-R11b (DEVICE timing on ADR-003 reference device).
- Expanded R8 coverage into AC-R8a (FirstRun), AC-R8b (RecoveredFromTmp with version guard), AC-R8b-reject (stale-tmp rejection), AC-R8c (LoadedClean), AC-R8d (RecoveredFromBackup + toast), AC-R8e (ResetToDefault + modal).
- Added 7 coverage-gap ACs: AC-R13a (Settings schema version), AC-EC-Race3 (SemaphoreSlim disposal), AC-EC-Uninit (uninitialized guard), AC-EC-PlaytestIsolation (playtest logger isolation via CI grep), AC-R10c (POCO self-containment), AC-R10d (ConfigureAwait), AC-R4c (root-per-type invariant table).
- DEVICE tier ACs (AC-R3b device portion, AC-R11b, AC-P1, AC-P2, AC-P4) all reference "ADR-003 reference Android device" — named device selection added to ADR-003 scope.
- Gate Summary body-verified and reconciled: BLOCKING CI 21 + PLAYMODE 33 + DEVICE 5 + ADVISORY 1 = **60**.

**Tuning Knobs updated:**

- `safety_margin_factor` range/default corrected; knob entries added for R10c/R10d CI severity, R9a banned-name grep, playtest-logger isolation; R8.d default tightened to "light toast required" (silent option removed per pass-1 action #15); R8.e modal tone contract added.
- Reference device naming authority = ADR-003 (locked).

**Event Bus GDD patched:**

- 7 new rows added to Section C Interactions table below `gameplay.level.quota_progress`: `currency.changed`, `tool.purchased`, `tool.upgraded`, `cosmetic.purchased`, `progression.node_unlocked`, `biome.unlocked`, `session.level_selected`. All T2 tier. Each with typed payload shape and subscriber list that includes Save/Load. Note appended referencing Save/Load pass-3 OQ-4 closure.

### Systems-Index + Review Log Updates

- Systems-index updated from "MAJOR REVISION NEEDED (pass 2)" → "Revised pass 3" pending targeted pass-4 verification or direct acceptance per CD policy.
- Dashboard metrics updated.
- This pass-3 revision entry appended to review log.

### Next Step (per CD pass-2 policy)

Per CD pass-2 guidance on Event Bus ("targeted pass-4 verification is the next step"), Save/Load has the same option: either (a) run targeted `/design-review` pass-4 in a fresh session that spot-checks the applied fixes without re-running full adversarial specialists, or (b) accept the revision as-is and promote to Approved (as was done for Event Bus). Recommendation: given the volume of applied changes, a targeted pass-4 is lower-risk than direct acceptance.

### Escalation Note

Not warranted at this stage. Pass-3 revision landed with design decisions explicitly made (not deferred); no systemic disagreement pattern remains.
