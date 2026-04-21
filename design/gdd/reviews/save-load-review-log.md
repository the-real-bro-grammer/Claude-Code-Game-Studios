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
