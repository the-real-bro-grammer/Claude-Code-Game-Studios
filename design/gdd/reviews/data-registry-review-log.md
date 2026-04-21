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
