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

## Review — 2026-04-21 (pass 3) — Verdict: NEEDS REVISION
Scope signal: L (unchanged)
Specialists: game-designer, systems-designer, qa-lead, economy-designer, unity-specialist, performance-analyst, creative-director (synthesis)
Blocking items: 21 (collapsed by creative-director into 8 surgical bundles) | Recommended: ~25 | Nice-to-have: ~15
Summary: First truly independent pass (fresh session, no same-session revision bias). Pass 3 confirmed pass-2 structural and precision fixes genuinely held, but surfaced a third stratum of defects — primarily *downstream-system implications* rather than GDD self-consistency defects. Three-specialist convergence on AC-R8e equivalence (game-designer B1 + systems-designer R5 + economy-designer N2) proves the 60% floor passes a 2.8★-vs-1.1★ loadout pair as "valid Pillar 3" — a participation trophy, not a validation. Two-specialist convergence on D.3 `V_i` gem-conversion undefined (systems-designer B1 + economy-designer B1) blocks the formula at day one. Novel pass-3 findings: (a) ProgressionNode unlock condition-type taxonomy is structurally identical to the reward-type vulnerability pass 2 closed — pay-to-win unlock vector unguarded; (b) ToolConfig monotonic-cost validation upgraded from pass-2 recommended to pass-3 blocking because MVP shop reads `upgradeTierChain` directly, making non-monotonic costs a live gameplay bug; (c) AC-NHV allowlist empty-bypass and CurrencyDefinition earn-rate unit both deferred in pass 2 and still unaddressed — pattern-blindness evidence; (d) Unity Asset Import Protocol gaps: `ImportAsset(ForceUpdate)` does not flush `delayCall`-deferred SetDirty writes (AC-R4 can pass against stale disk), mass-import `delayCall` accumulation risks recursive loop without `StartAssetEditing`/`StopAssetEditing` bracketing, `GUIDFromAssetPath` returns empty-sentinel mid-import for brand-new assets; (e) fast-enter (domain-reload-disabled) PlayMode silently serves stale Dictionary state from EC-R2 lazy-init; (f) `Debug.LogWarning(this)` is the wrong overload (non-clickable console); (g) `IPreprocessBuildWithReport` missing `callbackOrder`; (h) OQ-4 event taxonomy drift — Pillar 5 cracks if hand-written constants misspell taxonomy keys (one CI EditMode AC closes this without source generator); (i) AC-P1 noise-gated without std-dev clamp; AC-P4 per-call vs per-item ambiguity; (j) `ProgressionNode.rewardAmount` has no upper bound (int.MaxValue passes); `redirectAngleRangeDegrees = 0` depresses category mean; (k) qa-lead AC-arithmetic correction: actual = 40 AC identifiers (not 39), sprint scaffolding = 26 (not 28). Creative-director synthesis: "12 → 12 → ~20+ is not a sign of fundamental quality issues; it's the signature of a Foundation document doing its job. What's concerning is not the count but the pattern-blindness of same-session revisions — AC-NHV survived pass 2 despite being flagged." User chose Option B — stop here, revise in fresh session with a targeted checklist.
Prior verdict resolved: Pass 1 structural (12) + pass 2 precision (12) — both RESOLVED. Pass 3 surfaces a new stratum.

### Blocking Items — Tier 1 (6 convergent bundles per creative-director)

| # | Bundle (convergent specialists) | Fix Prescription |
|---|--------------------------------|------------------|
| T1-1 | D.3 `V_i` gem-conversion undefined [systems-designer B1 + economy-designer B1] | Either (a) ban gem drops at MVP via `OnValidate` reject, (b) specify project-wide coin-per-gem tuning knob, or (c) exclude gems from EV with explicit warning. Formula is unimplementable as written when gem entries exist. |
| T1-2 | StarThresholdConfig denominator anchor unresolved [systems-designer B2 + economy-designer R4] | One sentence in Category #8 Key Validation: "Thresholds are ratios of `LevelData.quota`, not `Ceiling`." Pass-2 deferred forward pointer never landed. |
| T1-3 | ProgressionNode unlock condition-type taxonomy missing [economy-designer B3] | Close condition type to MVP allowlist (`LEVEL_COMPLETE`, `STAR_COUNT`, `COIN_TOTAL`, `TOOL_OWNED`) gated by Monetization Boundary ADR — same pattern pass 2 applied to reward type. Novel pass-3 finding. |
| T1-4 | ToolConfig monotonic-cost validation [systems-designer B4] | `OnValidate` loop asserts `cost[i] ≤ cost[i+1]`; add BLOCKING CI test to AC-R5c. Upgraded from pass-2 recommended: MVP shop reads `upgradeTierChain` directly, non-monotonic = live bug. |
| T1-5 | AC-R8e equivalence clause / spread ceiling [game-designer B1 + systems-designer R5 + economy-designer N2 — 3-specialist convergence] | Add spread ceiling (no single loadout > 80% 3-star while another < 40%); register 60% floor + spread ceiling as tuning knobs. Current AC is participation trophy, not Pillar 3 validation. |
| T1-6 | AC-NHV allowlist empty-bypass [qa-lead B5] | AC fails if `wc -l tools/ci/nhv-allowlist.txt < N`. Pass-2 flagged as recommended and unaddressed — crossed from "recommended" into enforced. |

### Blocking Items — Tier 2 (unified Unity fix + arithmetic)

| # | Bundle | Fix |
|---|--------|-----|
| T2-1 | Unity Asset Import Protocol [unity-specialist B1+B2+B3 unified] | New "Asset Import Protocol" subsection specifying (a) `AssetDatabase.StartAssetEditing()`/`StopAssetEditing()` bracketing around bulk writes, (b) empty-GUID guard before writing `_id` (skip if `"00000000000000000000000000000000"`), (c) explicit flush pattern in AC-R4 test instead of relying on `ImportAsset(ForceUpdate)` alone. |
| T2-2 | AC arithmetic correction [qa-lead count audit] | Gate Summary: 40 AC identifiers (not 39); sprint scaffolding = 26 artifacts (not 28). Self-consistency error survived pass-2 recount. |

### Blocking Items — Additional surgical edits (13)

| # | Item (specialist) | Fix |
|---|-------------------|-----|
| A-1 | Fast-enter mode silently serves stale Dictionary [unity B4] | Add to EC-R2: require Domain Reload enabled in Enter Play Mode Options; document as implementation-sprint DoD constraint. |
| A-2 | `Debug.LogWarning(this)` wrong overload [unity B5] | R5 Layer 1 specifies `Debug.LogWarning(message, this)` two-arg form for clickable Console entries. |
| A-3 | `IPreprocessBuildWithReport` callbackOrder unspecified [unity B6] | Specify `GetPostprocessOrder() ≥ 100` so hook runs after package preprocessors. |
| A-4 | OQ-4 event taxonomy drift [game-designer B2] | Add CI EditMode AC cross-checking declared event ID string constants against the taxonomy SO's registered keys — achievable without source generator; do not defer to architecture. Pillar 5 crack otherwise. |
| A-5 | AC-P1 std-dev clamp [performance-analyst B1] | Add co-condition: `std dev < 20% of median`; rerun up to 30 iterations if unstable. 4ms budget with ±2ms Editor noise is noise-gated. |
| A-6 | AC-P4 per-call vs per-item [performance-analyst B3] | Specify 200ms is per `OnPostprocessAllAssets` invocation regardless of batch size; postprocessor must scope duplicate-ID checks to delta, not full catalog re-scan. |
| A-7 | `ProgressionNode.rewardAmount` upper bound [systems-designer B3] | COINS ∈ [1, 10_000], GEMS ∈ [1, 500]; add to Category #9 Key Validation and Tuning Knobs. |
| A-8 | `redirectAngleRangeDegrees > 0` (and peers) required [systems-designer B5] | Category #1 validation: Ramps `> 0`; Nets `catchZoneAreaTiles > 0`; Guard Posts `coverageRadiusTiles > 0`. Closes `E_norm = 0` degenerate path. |
| A-9 | AC-R4 `IPreprocessBuildWithReport` has no specified assertion [qa-lead B1] | Add explicit assertion on stale-`_id` fixture: hook's `BuildReport` must contain error message naming offending path. |
| A-10 | AC-R8d boundary comparator `>` vs `>=` [qa-lead B2] | Add test at exactly 0.30f and at 31/100 to pin strict-`>` semantics (current fixtures only test fraction-boundary cases). |
| A-11 | AC-R5b fixture existence precheck [qa-lead B3] | CI pre-check asserts the three fixture file paths at `tests/editmode/data-registry/fixtures/layer2/` exist before invoking postprocessor. Precondition must be an assertion. |
| A-12 | AC-EC-A2 MANUAL part underspecified [qa-lead B4] | Specify evidence artifact path + owner, or reclassify as BLOCKING CI only with rename as test-setup step. |
| A-13 | `CurrencyDefinition` earn-rate unit [economy-designer B2] | Define unit (recommend: coins-per-completed-level) or remove field from MVP row. Pass-2 deferred item #8 still unaddressed after two rounds. |

### Specialist Disagreement (for user adjudication)

- **performance-analyst B2 (Mali-G52 scaling) vs creative-director**: performance-analyst argues 2.27× Editor→IL2CPP scaling is pessimistic (realistic 1.2–1.5×), meaning 44ms Editor → ~57ms on-device, already exceeding AC-P2's 50ms budget; proposes tightening on-device budget to ≤ 60ms. Creative-director partially disagrees — says the fix is a hard VS deadline split (AC-P2a sprint ADVISORY / AC-P2b VS BLOCKING) rather than renegotiating the number without device data. Both positions are defensible; user should adjudicate during the fix pass.

### Recommended Items (not blocking pass-3 fix, but worth addressing)

- Unity R1-R4: atlas membership API unspecified, `GetPostprocessOrder()` for Data Registry postprocessor, AC-P4 batch-import path, `[CreateAssetMenu]` normatively required
- Systems-designer R1-R3: `W_i` upper bound, small-N (≤5) dominance-evasion documentation, ProgressionNode cycle detection algorithm
- Economy R1-R5: ADR gate CI enforcement (Roslyn `DR0002`), coin-vs-gem `C_base` mixing, 30%-modal text committed in pass 2 but not landed, StarThresholdConfig→Level Scoring forward pointer, Tier 0 / base-tool `upgradeTierChain` boundary
- Performance R1-R3: OQ-5 disjunctive trigger ownership gap, Mali-G52 VS deadline split, GC allocation during Dictionary build untracked
- Game-designer R1-R5: E_norm signal-limits annotation, event taxonomy regex digit decision, Pillar 3 bridge note (D.2 MVP → AC-R8e VS), `_dominanceJustified` sub-30% escape path, GEMS earn-cap ADR cross-reference
- QA-lead R1-R5: AC-P1 regression baseline, AC-P2 owner, AC-R8c sign-off content spec, AC-D2 justified-tool mean assertion, AC-R5c subclass enumeration

### Nice-to-Have

- EC-A7 advisory should use `P_i < 0.01` (effective probability) not `W_i < 0.01` (raw weight) to close normalization evasion
- Event taxonomy regex digits: expand to `[a-z0-9_]+` or lock rationale (blocking ergonomics for every emitter GDD — worth resolving now)
- L_budget clamp step shown in D.3 formula expression
- Per-frame `GetById` prohibition rule in Tuning Knobs
- Domain-reload time-cost note in AC-P2 commentary
- `_excludeFromQuotaSimulation` reference-level migration to GopherConfig or EC-F4 double-booking doc
- "Stub" IAP product ID semantics (null vs empty vs placeholder)
- Shop access gating via `TOOL_UNLOCK` documented as primary mechanism

### Specialists' Key Quotes (pass 3)

- **[creative-director]**: "12 → 12 → ~20+ is **not** a sign of fundamental quality issues; it's the signature of a Foundation document doing its job. What's concerning is not the count but the *pattern-blindness* of same-session revisions: AC-NHV survived pass 2 despite being flagged. The pass-3 fix session should be in a **fresh** session with a targeted checklist, not a continuation."
- **[game-designer]**: "The 60% floor that passes 2.8★ vs 1.1★ loadouts is not a Pillar 3 validation; it's a participation trophy."
- **[systems-designer]**: "MVP shop reads `ToolConfig.upgradeTierChain` directly. A non-monotonic chain is not a future problem — it is immediately purchasable by a player at MVP."
- **[economy-designer]**: "The reward-type enum got closed; the condition-type taxonomy has no analogous constraint. This is the same anti-pattern pass 2 fixed on the reward side."
- **[qa-lead]**: "Two deferred items from pass 2 (AC-NHV empty allowlist, CurrencyDefinition earn-rate unit) remain unaddressed in pass 3. This is pattern-blindness, not scope."
- **[unity-specialist]**: "`ImportAsset(ForceUpdate)` does not flush `delayCall`-deferred SetDirty writes. AC-R4 passes against in-memory state while the on-disk `.asset` still carries the stale `_id`."
- **[performance-analyst]**: "A 4ms budget with ±2ms Editor noise is noise-gated, not performance-gated. Without a std-dev clamp, a 3× regression still passes."

### Estimated Fix Effort

Creative-director estimate: ~2–4 hours of focused GDD surgery in a fresh session. Fixes are textually small, non-structural, and most have clear prescriptions embedded in specialist reports. Creative-director specifically recommends AGAINST a full pass-4 adversarial review — recommends a targeted pass-4 verifying only the Tier-1 bundles after the fix, not a full re-review.

## Revision — 2026-04-21 (pass 3 fix pass, fresh session per CD recommendation)

User selected Option 1 (revise now) and adjudicated the performance-analyst-vs-creative-director disagreement on Mali-G52 scaling in favor of **Option Y** (hard-deadline split: AC-P2a Editor-only sprint ADVISORY + AC-P2b Mali-G52 on-device VS BLOCKING). All 21 pass-3 blocking items resolved in this revision pass.

### Blocking Items Addressed (pass 3)

**Tier 1 — 6 convergent bundles**

| # | Bundle | Fix Applied |
|---|--------|-------------|
| T1-1 | D.3 `V_i` gem-conversion undefined | `EV_loot` definition narrowed to coin-denominated values only; gem entries excluded with `OnValidate` WARNING on any `LevelData` whose referenced LootTable contains gems. Forward promotion path documented: when Currency & Economy GDD lands at VS, `V_i` formula promotes to `V_i_coins + (V_i_gems * gemToCoinRate)` with the rate stored in `CurrencyDefinition` gated by a Monetization Boundary ADR. |
| T1-2 | StarThresholdConfig denominator anchor unresolved | Category #8 row adds "Thresholds are ratios of `LevelData.quota`, not of `QuotaAchievabilityCeiling`" + forward pointer to Level Scoring GDD when authored. |
| T1-3 | ProgressionNode unlock condition-type taxonomy missing | Category #9 closes `unlockConditionType` to enum `{LEVEL_COMPLETE, STAR_COUNT, COIN_TOTAL, TOOL_OWNED}`, gated by Monetization Boundary ADR for extensions. Same pattern pass 2 applied to reward type — pay-to-win unlock vector closed. |
| T1-4 | ToolConfig monotonic-cost validation | Category #1 adds "Upgrade costs strictly monotonic non-decreasing" normative; new AC-R5d BLOCKING CI with three-case parameterised test (decreasing fails, non-decreasing passes, equal-edge passes). Upgraded from pass-2 RECOMMENDED because MVP shop reads `upgradeTierChain` directly. |
| T1-5 | AC-R8e equivalence clause / spread ceiling | AC-R8e rewritten as two gates that must both pass: 60% 1-star floor + 40pp 3-star spread ceiling. Both values registered in Tuning Knobs with interaction note ("floor alone passes 2.8★-vs-1.1★ pair; ceiling alone passes two weak loadouts; both must pass for Pillar 3 validation"). |
| T1-6 | AC-NHV allowlist empty-bypass | AC-NHV adds `wc -l < tools/ci/nhv-allowlist.txt` non-empty-bypass guard asserting ≥ 7 non-comment lines; allowlist shrinkage below MVP floor requires explicit ADR entry. |

**Tier 2 — 2 unified fixes**

| # | Bundle | Fix Applied |
|---|--------|-------------|
| T2-1 | Unity Asset Import Protocol | New normative subsection R4.a added after R4 specifying three protocol rules: (a) `AssetDatabase.StartAssetEditing()` / `StopAssetEditing()` bracketing around all bulk writes; (b) empty-GUID sentinel guard (`"00000000000000000000000000000000"`) before writing `_id`; (c) explicit `delayCall` drain in tests (`ImportAsset(ForceUpdate)` does not flush delayCall queues). R4, AC-R4, AC-EC-A2 Part 2 updated to reference the protocol. |
| T2-2 | AC arithmetic correction | Gate Summary recounted to 42 distinct AC identifiers (was self-inconsistent at 39/38/40); scaffolding = 26 units (was 28). Reason for pass-3 growth vs. pass 2 (+3): new AC-R5d, new AC-R7a, AC-P2 split into P2a+P2b; EC-F1 remains folded into AC-D3. |

**13 surgical edits applied (A-1 through A-13)**

| # | Item | Fix |
|---|------|-----|
| A-1 | Fast-enter mode stale Dictionary | EC-R2 requires Domain Reload enabled; DoD checklist item `EnterPlayModeOptionsEnabled = 0`. |
| A-2 | `Debug.LogWarning(this)` wrong overload | R5 Layer 1 specifies `Debug.LogWarning(message, this)` two-arg form with clickable-Console rationale. |
| A-3 | IPreprocessBuildWithReport callbackOrder | AC-R4 specifies `callbackOrder = 100` (runs after package preprocessors at default 0). |
| A-4 | OQ-4 event taxonomy drift | New AC-R7a BLOCKING CI test (`EventIdConstantDriftTest`) cross-checks `public const string` event-ID declarations against taxonomy SO keys; closes Pillar 5 crack without source generator. |
| A-5 | AC-P1 std-dev clamp | Added noise-stability co-condition: std dev < 20% of median; test extends to 30 iterations if unstable; fails if 30 still unstable. |
| A-6 | AC-P4 per-call vs per-item | 200ms budget is per-invocation regardless of batch size; delta-only checks required; full Catalog re-scan explicitly out of scope. |
| A-7 | ProgressionNode rewardAmount upper bound | COINS ∈ [1, 10_000]; GEMS ∈ [1, 500] enforced at OnValidate per Category #9. |
| A-8 | Tool E_norm denominator > 0 | Category #1 requires Ramps `redirectAngleRangeDegrees > 0`, Nets `catchZoneAreaTiles > 0`, Guard Posts `coverageRadiusTiles > 0`; closes `E_norm = 0` degenerate. |
| A-9 | AC-R4 BuildReport path assertion | AC-R4 specifies IPreprocessBuildWithReport loads stale-`_id` fixture at `tests/editmode/data-registry/fixtures/stale-id/` and asserts `BuildReport.steps[]` contains error message naming the offending path. |
| A-10 | AC-R8d `>` vs `>=` comparator | AC-R8d adds boundary pins at exactly `0.30f` (return false) and 31/100 (return true); pins strict `>` semantic independent of fraction-boundary float representation. |
| A-11 | AC-R5b fixture precheck | AC-R5b test's `[OneTimeSetUp]` hard-asserts each of 3 expected fixture paths exists via `File.Exists` before invoking postprocessor; missing fixture produces named diagnostic instead of silent "no log entries" failure. |
| A-12 | AC-EC-A2 MANUAL underspec | Split into Part 1 MANUAL (evidence artifact path `production/qa/evidence/data-registry-rename-move-[YYYY-MM-DD].md`, owner = sprint's assigned engineer, pre-/post-rename `_id` values + screenshots) + Part 2 BLOCKING CI (automated RenameAsset/MoveAsset test). |
| A-13 | CurrencyDefinition earn-rate unit | Field renamed to `earnRateBaselineCoinsPerCompletedLevel` with explicit MVP unit (coins earned per average 2-star level completion); gem currency may set 0. |

### Specialist Disagreement — Resolution

- **performance-analyst B2 vs creative-director on Mali-G52 scaling**: User selected Option Y (CD's hard-deadline split). AC-P2 split into P2a (Editor ADVISORY, Data Registry sprint gate, ≤ 50ms) + P2b (on-device IL2CPP Mali-G52 BLOCKING, VS gate, ≤ 100ms). A `hw-dependency: mali-g52` backlog ticket MUST be created before Data Registry sprint close naming the VS playtest session that will verify P2b.

### Remaining Open (pass-3 RECOMMENDED items deferred — not blocking)

Per creative-director's pass-3 summary, ~25 RECOMMENDED items and ~15 nice-to-have items remain deferred to post-sprint or consumer-system GDDs. Not reproduced here — see the pass 3 entry above for the full list. Targeted pass-4 should verify only the Tier-1 bundles landed correctly; it is explicitly NOT a full re-review.

## Review — 2026-04-21 (pass 4 — targeted verification) — Verdict: APPROVED
Scope signal: L (unchanged)
Specialists: none (lean-depth single-session analysis per creative-director's pass-3 recommendation — explicitly NOT a full re-review)
Blocking items: 0 | Recommended: 2 | Nice-to-have: 2
Summary: Targeted pass-4 verifies the pass-3 fix pass landed correctly. All 6 Tier-1 convergent bundles (D.3 V_i gem exclusion + OnValidate warning, StarThreshold quota-anchor, ProgressionNode condition-type enum closure, ToolConfig monotonic-cost + AC-R5d, AC-R8e two-gate floor+ceiling, AC-NHV non-empty-bypass) verified in the GDD text with correct wording, cross-references, and tuning-knob registrations. Both Tier-2 unified fixes verified: R4.a Asset Import Protocol subsection (StartAssetEditing/StopAssetEditing bracketing + empty-GUID sentinel guard + explicit delayCall drain in tests) landed at lines 61-67; Gate Summary recount to 42 AC identifiers + 26 scaffolding units is internally arithmetic-consistent (22 BLOCKING CI + 6 ADVISORY + 11 DEFERRED + 2 VS-BLOCKING + 2 MANUAL = 43; −1 for AC-EC-A2 two-row double-count = 42). All 13 surgical edits (A-1 through A-13) verified present with specified content. Specialist disagreement resolution (Mali-G52 Option Y — hard-deadline split AC-P2a Editor ADVISORY / AC-P2b on-device VS BLOCKING) landed with mandatory `hw-dependency: mali-g52` sprint DoD ticket requirement. Two minor RECOMMENDED items surfaced: scaffolding count narrative phrasing is parseable two ways (21 vs 22 distinct BLOCKING CI test files depending on how "shares a test file" is read — could use one-line clarification), and front-matter status header is now 12 lines across three revision passes (dense but not blocking). Neither item blocks implementation sprint start. Data Registry is APPROVED for architecture / implementation phase.
Prior verdict resolved: Pass 3 NEEDS REVISION → RESOLVED in pass-3 fix pass; pass 4 targeted verification confirms the fix pass landed correctly.

### Pass-3 Blocking Items — Verification Table

| Pass-3 Item | GDD Location | Verified |
|---|---|---|
| T1-1 D.3 V_i gem exclusion + warning + forward promotion | data-registry.md:207 | ✓ |
| T1-2 StarThreshold denominator = LevelData.quota | data-registry.md:95 | ✓ |
| T1-3 ProgressionNode unlockConditionType enum closed | data-registry.md:96 | ✓ |
| T1-4 ToolConfig monotonic upgrade costs + AC-R5d | data-registry.md:88, :404 | ✓ |
| T1-5 AC-R8e 60% floor + 40pp spread ceiling | data-registry.md:422-426 + Tuning Knobs | ✓ |
| T1-6 AC-NHV wc-l ≥ 7 non-empty-bypass | data-registry.md:505 | ✓ |
| T2-1 R4.a Asset Import Protocol subsection | data-registry.md:61-67 | ✓ |
| T2-2 Gate Summary 42 AC / 26 scaffolding | data-registry.md:507-518 | ✓ |
| A-1 EC-R2 Domain Reload DoD | data-registry.md:261 | ✓ |
| A-2 Debug.LogWarning(message, this) two-arg | data-registry.md:68 | ✓ |
| A-3 IPreprocessBuildWithReport callbackOrder=100 | data-registry.md:398 | ✓ |
| A-4 AC-R7a event-ID constant ↔ taxonomy drift | data-registry.md:410 | ✓ |
| A-5 AC-P1 std-dev clamp (<20% of median) | data-registry.md:467 | ✓ |
| A-6 AC-P4 per-invocation + delta-only check | data-registry.md:475 | ✓ |
| A-7 ProgressionNode rewardAmount bounds | data-registry.md:96 | ✓ |
| A-8 E_norm denominator parameters > 0 | data-registry.md:88 | ✓ |
| A-9 AC-R4 BuildReport path assertion | data-registry.md:398 | ✓ |
| A-10 AC-R8d strict `>` pinning at 0.30f and 31/100 | data-registry.md:420 | ✓ |
| A-11 AC-R5b OneTimeSetUp fixture precheck | data-registry.md:401 | ✓ |
| A-12 AC-EC-A2 Part 1 MANUAL + Part 2 BLOCKING CI | data-registry.md:461-463 | ✓ |
| A-13 CurrencyDefinition earnRateBaselineCoinsPerCompletedLevel | data-registry.md:94 | ✓ |
| Option Y: AC-P2a Editor ADVISORY / AC-P2b VS BLOCKING | data-registry.md:469-471 | ✓ |

### Recommended Items (non-blocking, advisory)

- Gate Summary scaffolding-count sentence at line 518 is parseable two ways ("22 BLOCKING CI test files (EC-A2 Part 2 shares a test file with AC-R4...)" — reading A gives 22 distinct files, reading B gives 21). One-line clarification at story-stub time would close the ambiguity.
- Front-matter status header is now 12 lines across three revision passes. Post-sprint, consider moving pass-1/pass-2 summaries into the review log and keeping only pass-3/pass-4 verdicts in the GDD header to keep the reader oriented.

### Nice-to-Have (carry into post-sprint backlog)

- EC-A7 advisory threshold could migrate from `W_i < 0.01` (raw weight) to `P_i < 0.01` (effective probability) to close the normalization-evasion path flagged in pass 3 nice-to-haves.
- Event taxonomy regex digit decision (`[a-z_]+` → `[a-z0-9_]+`) still open and is adjacent to OQ-4; may resolve naturally when OQ-4 source-generator decision lands at architecture phase.
