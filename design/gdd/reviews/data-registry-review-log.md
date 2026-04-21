# Data Registry — Review Log

## Review — 2026-04-21 — Verdict: MAJOR REVISION NEEDED → REVISED (same session)
Scope signal: L
Specialists: game-designer, systems-designer, qa-lead, economy-designer, unity-specialist, performance-analyst, creative-director (synthesis)
Blocking items: 12 | Recommended: 14 | Nice-to-have: ~6
Summary: First-pass review of Foundation system #1. Three independent classes of blockers converged: (a) Unity platform non-functional code — `[DefaultExecutionOrder]` on ScriptableObjects is a no-op, `AssetDatabase` in `OnValidate` lacked `#if UNITY_EDITOR` guard; (b) formula/schema contradictions — R4 "populated once" contradicted AC-EC-A2 "updates on rename," AC-D2 used out-of-range `C_base = 0.5`, AC-D1 used R5-rejected `W_i = 0`, C_rate had three incompatible authoritative ranges; (c) Pillar 3 audit targeted within-category per-tool dominance while Pillar 3 itself is about cross-loadout viability. Creative-director synthesis: "The specialists are not quibbling; they are pointing at the same three load-bearing beams from six angles." All 12 blockers resolved in same-session revision pass (user selected Option A "Revise now"). Major recommended items also applied: edge cases labeled EC-A1..F6, 30% modal threshold promoted to tuning knob with rationale, performance ACs tightened (P1: 50ms→5ms, P3 excludes audio, P4 added for Layer 2 latency), 40%+ of AC surface rewritten for independent testability (R1 split, R5 split, R8c artifact-gated, R8d scoped to decision logic, NHV uses allowlist file), AC-C1..C11 marked DEFERRED to consumer-system sprints. Revision awaits independent re-review in a fresh session.
Prior verdict resolved: First review.

### Blocking Items Addressed

| # | Blocker (source specialist) | Fix Applied |
|---|------------------------------|-------------|
| 1 | `[DefaultExecutionOrder]` on SO is a no-op [unity-specialist] | EC-R2 rewritten to lazy-init pattern on first `GetById` / `All` call |
| 2 | Missing `#if UNITY_EDITOR` guard on `AssetDatabase` [unity-specialist] | R4 now explicitly requires the guard |
| 3 | R4 "populated once" vs AC-EC-A2 contradiction [systems-designer + unity-specialist] | R4 rewritten as always-check-on-OnValidate model; EC-A1 and EC-A2 aligned |
| 4 | AC-D2 uses C_base=0.5, out of schema range [systems-designer] | Fixtures rewritten to `E_norm=0.5, C_base=5` (in-range) |
| 5 | AC-D1 uses W_i=0, which R5 rejects [systems-designer] | Replaced with `[0.01, 99.99, 0.01]` epsilon fixture |
| 6 | C_rate authority conflict (3 ranges) [systems-designer] | Resolved: fixed tuning constant (default 0.6; tunable [0.5, 0.75]) |
| 7 | EfficiencyRatio does not measure Pillar 3 [game-designer + economy-designer] | D.2 carries explicit scope disclaim; AC-R8e added for VS playtest gate |
| 8 | LootTable `guaranteedDrop` phantom spec [economy-designer] | Removed from MVP schema (deferred to Collection System if needed) |
| 9 | Qty range distribution unspecified [economy-designer] | Specified as uniform-integer draw |
| 10 | ShopProductConfig missing at MVP [economy-designer] | Clarified: MVP shop reads `ToolConfig.upgradeTierChain` directly |
| 11 | 40%+ of ACs untestable (NHV, R1, R5, R8c, R8d) [qa-lead] | R1/R5 split per mechanism, R8c artifact-gated, R8d scoped to decision logic, NHV uses allowlist file |
| 12 | AC-C1..C11 depend on undesigned systems [qa-lead] | All tagged DEFERRED to respective consumer-system sprints |

### Recommended Items Addressed

- Edge Case labels added (EC-A1..A7, EC-B1..B3, EC-R1..R3, EC-F1..F6) — closes traceability gap between ACs and Edge Cases
- EC-F5 minimum-parameter achievability trap newly documented
- EC-F6 Net E_norm cross-biome ambiguity resolved (max-of-biome denominator)
- 30% justification modal threshold promoted to tuning knob with rationale and interaction note
- AC-P1 tightened: 50ms → 5ms at MVP scale (500-asset assumption corrected to actual MVP counts)
- AC-P2 reclassified PLAYMODE → ADVISORY (Mali-G52 hardware not guaranteed)
- AC-P3 scope narrowed: excludes AudioClip PCM data; OQ-5 Addressables trigger re-scoped
- AC-P4 added: AssetPostprocessor Layer 2 latency ≤ 200ms for designer iteration UX
- Gate summary recounted and corrected (38 ACs total)

### Remaining Open (deferred, not blocking revision)

- Cross-loadout viability mechanism lives in AC-R8e playtest gate, owned by Tool System GDD consumption of EfficiencyRatio and VS playtest round
- CurrencyDefinition missing fields (carry cap, cross-currency conversion) — handed to Currency & Economy GDD per Cross-References
- CosmeticConfig schema-evolution procedure — to be documented at VS when ShopProductConfig lands
- Dictionary pre-sized capacity (nice-to-have — pattern noted in AC-P1)
- Hot-path query policy (nice-to-have — noted in EC-R2 commentary)
- Player Fantasy section framing critique (nice-to-have — not a material risk)

### Specialists' Key Quotes

- **[creative-director]**: "Three independent classes of blockers — Unity non-functional code, formula/schema contradictions, and audit targeting the wrong pillar quantity — each justify rejection alone. The specialists are not quibbling; they are pointing at the same three load-bearing beams from six angles."
- **[unity-specialist]**: "Two separate load-bearing Unity patterns are specified incorrectly. An implementer executing the GDD verbatim produces broken code."
- **[game-designer]**: "The EfficiencyRatio audit looks within categories, but Pillar 3 violation happens across categories."
- **[qa-lead]**: "40%+ of the AC surface is not independently testable as written."

## Review — 2026-04-21 (pass 2) — Verdict: NEEDS REVISION → REVISED (same session)
Scope signal: L (unchanged)
Specialists: game-designer, systems-designer, qa-lead, economy-designer, unity-specialist, performance-analyst, creative-director (synthesis)
Blocking items: 12 | Recommended: 16 | Nice-to-have: ~12
Summary: Independent re-review in a fresh session validated pass 1's 12 blockers genuinely resolved, but found a second stratum of 12 new blockers across three convergent themes: (a) Unity 6.3 platform precision gaps — `AssetPathToGUID` is the wrong API name for 6.3 (canonical is `GUIDFromAssetPath(path).ToString()`), OnValidate does NOT reliably fire on asset folder-move (rename only), and `_id` writes in OnValidate never persist to disk without `EditorUtility.SetDirty` + `EditorApplication.delayCall` — any of these three silently corrupts the always-check model; (b) day-one implementer gaps — catalog population mechanism completely unspecified (R3 says what a catalog IS but never HOW entries get added), AC-R5b fixture has no existence AC (test has no runnable substrate), AC-D2 synthetic mean = 0.05 had no peer-tool definition (test unimplementable); (c) AC precision failures — AC-P1 measures OnEnable under EC-R2's lazy-init pattern which builds nothing (test cannot detect regressions), AC-P1/P2 budgets arithmetically inconsistent (11 × 5ms = 55ms > 50ms aggregate), AC-D1 epsilon fixture `[0.01, 99.99, 0.01]` unsafe in float32 at ±0.0001 tolerance, AC-R1b grep pattern cannot resolve receiver types (needs Roslyn), gate count wrong in three places simultaneously (36/38/40), EC-F1 listed as separate gate but its AC body says "covered in AC-D3," ProgressionNode reward type open-ended (pay-to-win vector). Creative-director synthesis: "The revision is progress, not a rewrite — the GDD's skeleton, schemas, and audit strategy survive this review. What fails is precision in roughly a dozen ACs and three sections." All 12 blockers resolved in same-session revision pass (user selected Option A "Revise now"). Revision awaits third-pass independent re-review in a fresh session.
Prior verdict resolved: Yes — pass 1 MAJOR REVISION NEEDED → RESOLVED (12 structural blockers); pass 2 NEEDS REVISION → RESOLVED (12 precision blockers).

### Blocking Items Addressed (pass 2)

| # | Blocker (source specialist) | Fix Applied |
|---|------------------------------|-------------|
| 1 | Catalog population mechanism unspecified [unity-specialist] | R3.a added — AssetPostprocessor auto-scan; postprocessor iterates `importedAssets[]` and auto-inserts `DataRegistryEntry` subclasses into the matching `[Category]Catalog` |
| 2 | AC-P1 measures wrong code path (OnEnable, not lazy-init first-GetById) [performance-analyst + unity-specialist + qa-lead] | AC-P1 rewritten to call `GetById` on freshly-instantiated Catalog whose Dictionary is still null; median of 10 runs + 3 warmups |
| 3 | AC-P1 + AC-P2 arithmetic inconsistent (11 × 5ms > 50ms) [performance-analyst] | Per-Catalog budget lowered to 4ms; 11 × 4 = 44ms fits AC-P2's 50ms aggregate with 6ms headroom |
| 4 | R4 claim that always-check handles move is incorrect [unity-specialist] | R4 split into Mechanism A (OnValidate — rename only) + Mechanism B (AssetPostprocessor movedAssets[] — folder-move); EC-A2 updated to reflect both paths |
| 5 | `EditorUtility.SetDirty` + save pattern unspecified [unity-specialist] | R4 Persistence pattern added: `EditorUtility.SetDirty` via `EditorApplication.delayCall` + `AssetDatabase.SaveAssetIfDirty`; AC-R4 forces `ImportAsset(ForceUpdate)` before assertion + wired into `IPreprocessBuildWithReport` |
| 6 | Wrong Unity 6.3 API name `AssetPathToGUID` [unity-specialist] | Replaced with `AssetDatabase.GUIDFromAssetPath(path).ToString()` throughout R4, EC-A1, EC-A2, EC-B2, AC-R4, AC-EC-B2 |
| 7 | Gate count wrong in 3 places (36/38/40); EC-F1 listed as separate gate but covered in AC-D3 [game-designer + systems-designer + qa-lead, triple convergence] | Gate summary recounted to 39 AC identifiers (19 BLOCKING CI + 6 ADVISORY + 11 DEFERRED + 3 MANUAL); EC-F1 re-labeled as fixture row inside AC-D3, producing no separate test file |
| 8 | AC-D2 synthetic category fixture unimplementable [systems-designer + qa-lead] | Full 3-tool fixture spelled out: T_A(E_norm=0.5, C_base=5, ratio=0.10) + T_B(0.3, 10, 0.03) + T_C(0.1, 5, 0.02), mean = 0.05; boundary assertions at exactly-2× (not flagged, strict `>` comparator), just-above-2× (flagged), justified (not flagged) |
| 9 | AC-R1b grep cannot resolve receiver types [qa-lead] | Rewritten as Roslyn analyzer AC — `DataRegistryImmutabilityAnalyzer` emitting diagnostic `DR0001`; grep explicitly rejected as implementation approach |
| 10 | AC-D1 float32 precision undefined [systems-designer] | Epsilon fixture swapped from `[0.01, 99.99, 0.01]` to `[1, 9999, 1]` (float32-exact representable values); normalization specified in `double`, cast to `float` for runtime consumers |
| 11 | AC-R5b fixture has no existence AC [qa-lead] | Named on-disk fixture path `tests/editmode/data-registry/fixtures/layer2/` with 3 required fixture types (duplicate-ID pair, broken cross-catalog reference, atlas-discipline violator); fixture authored in sprint's first story before any formula tests |
| 12 | ProgressionNode reward type open-ended (pay-to-win vector) [economy-designer] | Enum closed to MVP allowlist `{COINS, GEMS, TOOL_UNLOCK, LEVEL_UNLOCK, BIOME_UNLOCK}`; extensions require Monetization Boundary ADR entry before enum may be widened |

### Remaining Open (deferred RECOMMENDED items — not blocking implementation sprint)

- E_norm signal-limits disclaim — D.2 audit should explicitly acknowledge E_norm proxies miss duration / cooldown / placement dimensions; position audit as coarse filter, not authoritative pillar gate
- AC-R8e equivalence clause — current 60% floor test passes two loadouts at 2.8★ vs 1.1★ (violates Pillar 3); add spread ceiling + register 60% as tuning knob
- `_excludeFromQuotaSimulation` reference-level migration — flag should move from shared `LootTable` to the `GopherConfig` that references it, or EC-F4 must document the double-booking hazard
- StarThresholdConfig denominator anchor — hand to Level Scoring GDD via explicit forward pointer (normalized against quota vs ceiling is load-bearing)
- ToolConfig monotonic-cost validation — upgrade tier chain should assert costs non-decreasing; currently Tier 2 cheaper than Tier 3 passes
- Event taxonomy regex digit decision — current `[a-z_]+` disallows `ramp_2` naming; either add `0-9` or lock rationale
- AC-R8c escalation path — sign-off has no fallback if lead designer unavailable during sprint
- CurrencyDefinition earn-rate unit — "earn-rate baseline > 0" unimplementable without unit; either define here or remove from MVP row
- AC-NHV allowlist completeness enforcement — no AC verifies the allowlist is non-empty or reviewed at sprint boundary
- AC-P4 batch-import stall — git-pull-of-30-assets × 200ms = 6s stall; budget applies to single-asset save only as written
- OQ-5 disjunctive trigger ownership gap — 3.9MB schema + 28.1MB audio neither fires; combined 32MB threshold has no owner
- 30% modal evasion path — C_base nudge by 1 coin drops under 2× threshold; document as known limitation in D.2 scope disclaim

### Specialists' Key Quotes (pass 2)

- **[creative-director]**: "The first pass closed 12 load-bearing structural blockers. That work was real. But six specialists independently hit a second stratum of defects that a programmer implementing the GDD verbatim would stub, guess, or get wrong. This revision is progress, not a rewrite."
- **[unity-specialist]**: "Two previously claimed fixes (EC-R2 lazy-init, `#if UNITY_EDITOR` guard) are correctly resolved. However, four new BLOCKING issues were found — particularly the dirty-mark gap and missing catalog population mechanism, which are the first things a programmer will hit on day one."
- **[performance-analyst]**: "AC-P1 tells the test to explicitly trigger OnEnable — but under EC-R2's lazy-init, OnEnable with no prior GetById call builds nothing. A slow Dictionary build would pass AC-P1 every time."
- **[systems-designer]**: "AC-D2 says 'synthetic category whose mean is 0.05' but provides zero definition of what populates that mean. An implementer cannot write this test without inventing data the GDD does not authorize."
- **[qa-lead]**: "The gate count is load-bearing — sprint close depends on it, and it is wrong in the document. Three different numbers for the same field."
- **[economy-designer]**: "ProgressionNode reward type open-ended is the same anti-pattern as leaving `_dominanceJustified` without a 30-char note — it only feels safe until someone tests its edges."

