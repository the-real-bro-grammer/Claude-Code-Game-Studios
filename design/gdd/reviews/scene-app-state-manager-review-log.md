# Scene / App State Manager — Review Log

Revision history for `design/gdd/scene-app-state-manager.md`. Each pass records verdict, specialists, blocking items, and key changes. Read top-down for chronological context.

---

## Review — 2026-04-27 — Verdict: NEEDS REVISION → APPLIED

**Pass:** 2 (first formal `/design-review`; pass-1 was `/design-system` + CD-GDD-ALIGN gate)
**Scope signal:** M (medium, ~1-1.5 days actual)
**Specialists:** game-designer + systems-designer + qa-lead + unity-specialist + performance-analyst (parallel spawn) + creative-director (synthesis)
**Blocking items:** 12 unique (17 raw findings; deduplicated by 5 convergent overlaps) | **Recommended:** 13 | **Nice-to-have:** 4
**Prior verdict resolved:** Yes — pass-1 CD-GDD-ALIGN APPROVED (2026-04-24) was an architecture/pillar gate; this pass applies stricter implementation-readiness criteria. CD synthesis: "architecture sound; revisions are local, surgical, tractable. Signature of NEEDS REVISION, not MAJOR REVISION. CD-GDD-ALIGN pass-1 stands; /design-review exposes layers pass-1 was not designed to catch."

### Convergent specialist findings (highest confidence)

Five findings independently flagged by 2+ specialists:

1. **D.1 slack double-count** — systems-designer B1 + performance-analyst B-1.
2. **AC-R5 `progress == 0.9` float equality** — qa-lead B2 + unity-specialist B-1 (cited Unity 6 docs).
3. **AC count three-way mismatch (37/42/43)** — qa-lead B1 + systems-designer N2.
4. **AC-R11 p95 of 5 samples = max** — qa-lead R6 + performance-analyst R-4.
5. **E14 FoundationStub under-specified** — game-designer REC-4 + qa-lead R4.

### Per-blocker resolution (all 12 BLOCKING applied)

| # | Blocker | Source | Fix |
|---|---------|--------|-----|
| 1 | D.1 slack double-count | systems B1 + perf B-1 | Reformulated to Option B (uncapped T_lr + T_slack as derived diagnostic). Example 2 recomputed: 1730ms (was erroneously 1810ms). New pathological example (combined overage > 300ms) added. |
| 2 | UnloadSceneAsync no auto-reclaim | unity B-3 | `Resources.UnloadUnusedAssets()` added to R6 (legal entry-point), R11 (retry budget), D.1 worked examples. Unity 6 docs cited: only `LoadSceneAsync` in `LoadSceneMode.Single` auto-reclaims; additive unload requires manual call. |
| 3 | AC-R5 float equality | qa B2 + unity B-1 | Changed `progress == 0.9` → `progress >= 0.9f` in R5 + R6 + AC-R5 with rationale comment (IEEE 754 + Unity progress can step 0.85→0.95 in one frame). |
| 4 | AC-P-Boot t=0 mismatch | perf B-2 | t=0 reset to `BootLoader.Awake()` start (matches D.5's actual budget). Added new ADVISORY `AC-P-UnityBoot` for end-to-end visibility (OS-launch → BootLoader.Awake) — dashboard only, no gate. |
| 5 | bg_session_expiry 300s | game B-1 + perf R-5 | Two-tier: `bg_expiry_general_seconds = 60s` (non-gameplay contexts) + `bg_expiry_in_rush_seconds = 1800s` (mid-Rush). `_backgroundedDuringRush` boolean captured at OS-background time per R10 step 4. D.3 worked examples and Section G tuning knobs fully recomputed. |
| 6 | R9 timeScale phantom-gameplay window | game B-2 | Atomic `IsPaused = true; Time.timeScale = 0f` at step 3 (was step 5 after sceneLoaded). Eliminates 100-200ms phantom-gameplay window during Rush-phase pause. Single-frame visual gap accepted as correct expression of "game is paused now." |
| 7 | AC count three-way mismatch | qa B1 + systems N2 | Full recount: 28+3+7+2+6 = 46 ACs across 9 tiers. H.1/H.3/H.4/H.5 headers + Gate Summary tier sums + Total all reconciled. |
| 8 | R13 Task.Wait Editor crash | unity B-4 | Added `#if UNITY_EDITOR` synchronous-only path. AC-R13 condition 1 reflects the Editor/Player split. |
| 9 | allowSceneActivation in Awake | unity B-2 | New R2.b rule with Unity 6 docs citation. R2 + E14 paths annotated. |
| 10 | AC-R7/R8/R9b/R13 untestable | qa B3-B6 | Each rewritten with concrete falsifiable criteria: AC-R7 max-with-tolerance + sum-bound proof; AC-R8 asymmetric tolerance windows [150,400]ms / [400,700]ms; AC-R9b sequence counters not wall-clock timestamps; AC-R13 explicit "call BeginTransitionAsync after Disposed → assert rejected" step. |
| 11 | T_slack overage > 300ms undefined | systems B2 | D.1 explicit "behavior when exceeds T_slack_budget" branch + pathological worked example showing AC-R11 fail with attribution log. |
| 12 | D.5 sums-of-ceilings invariant | systems B3 | Added invariant annotation (sum(T_x_ceiling) ≤ T_boot_target + 800ms safety margin) + DEVELOPMENT_BUILD advisory log. |

### Recommended applied (11 of 13)

- **REC-2** (game) — Header pillar claim "Implements Pillar 5" → "Enables Pillar 5 preconditions at scene boundaries" (architectural truth: Scene Manager doesn't produce feedback, only ensures Input is wired before first tap).
- **R1** (systems) — R14.c invariant rewording: "Paused" is a flag (R4), not a state. R10's `IsPaused = true` during Loading is a lawful safety-flag composite, not a state-machine transition.
- **R2** (systems) — R12 firing order specified: `_pendingTransition` fires FIRST; `_pendingPause` defers to subsequent transition. New AC-R12c verifies.
- **R-3** (perf) — Save/Load contention risk surfaced as OQ-18 (cross-GDD). Save/Load GDD bidirectional amendment must document read-during-write safety.
- **R4 + REC-4** (qa + game) — E14 FoundationStub DISCARDED. Replaced with `[InitializeOnLoad]` `BootEnforcer` Editor script: intercepts Play-from-non-Boot, stores target scene in `EditorPrefs`, relaunches via Boot.unity. Production fidelity guaranteed. AC-EC-E14 fully rewritten.
- **R5** (qa) — AC-INT-FF single-owner clarified: Scene Manager owns the integration test (readiness assertion); Input AC-P1 owns the latency sub-assertion. Failure-attribution path documented.
- **N1** (systems) — R5 single-subscriber via Event Bus `ExclusiveSubscribe` facet (not Scene Manager introspecting bus internals). Tracked as OQ-16 for Phase 5 Event Bus amendment.
- **R-1** (perf) — R11 cold-cache reality note: 1100ms `T_sm_budget` is warm-cache target; cold-cache may reach 1300-1500ms; T_slack absorbs the overrun within 2000ms `T_retry_max` gate.
- **R-5** (unity) — Addressables 2.4 `SceneInstance.Activate()` API surface delta noted in ADR-008 trigger. Tracked as OQ-17.
- **AC-INV1** (qa R1) — Best-effort grep scope acknowledged; reflection/wrapper bypass covered by code review + E16 runtime assertion.
- **N2/N3** (qa) — New ACs: AC-EC-E9 (OS background mid-transition) + AC-EC-E18 (storage invalidated). Both PLAYMODE.

### Recommended deferred (2 of 13)

- **AC-R10a 6-condition split** (qa R3) — left as future cosmetic refactor. The 6-condition assertion is functionally correct; named NUnit sub-assertions can be added at implementation time.
- **REC-1 T_retry_target = 1000ms ADVISORY** (game) — Section B already names "1.5s = let's go again." Adding a separate ADVISORY duplicates the design intent. Concept doc and Section B together provide the positive target without GDD-level formula change.

### New ACs added in pass-2

- AC-EC-E9 — OS background mid-transition (PLAYMODE)
- AC-EC-E18 — storage invalidated during OS background (PLAYMODE)
- AC-R12c — pending queue firing order (PLAYMODE)
- AC-P-UnityBoot — end-to-end cold-boot visibility (DEVICE ADVISORY)

### OQ resolutions in pass-2

**6 of 7 pass-1 Unity API OQs CLOSED** via unity-specialist Unity 6 doc verification:
- OQ-2 — `Application.focusChanged` is NOT preferred over `OnApplicationPause`. R10 unchanged.
- OQ-3 — `BeforeSplashScreen` IS reliable; R3's prohibition is architectural, not reliability-based.
- OQ-4 — FEPM Domain Reload + Scene Reload toggles confirmed independent. AC-BOOT-FEPM implementable.
- OQ-5 — No native pre-unload callback in Unity 6 SceneManagement. `IPreUnloadHook` pattern stands.
- OQ-6 — `UnloadSceneAsync` does NOT auto-reclaim; `LoadSceneAsync(Single)` DOES. (Drove BLOCKING #2.)
- OQ-7 — No native "backgrounded N minutes" API. `DateTime.UtcNow` polling correct.

**OQ-1 remains open** (low risk — `LoadSceneParameters` field changes; defer to first implementation story).

**5 new pass-2 OQs added:** OQ-16 (ExclusiveSubscribe Phase 5), OQ-17 (Addressables 2.4 API delta), OQ-18 (Save/Load read-during-write), OQ-19 (two-tier expiry retention validation), OQ-20 (cold-cache T_sm overshoot device measurement).

### File metrics

- Lines: 651 → 727 (+76)
- ACs: 37 reported / 42 actual → 46 reconciled
- OQs: 15 → 20 (6 closed, 5 added; net active 13)
- New rules: R2.b
- New ACs: 4 (AC-EC-E9, AC-EC-E18, AC-R12c, AC-P-UnityBoot)
- New OQs: 5 (OQ-16 through OQ-20)
- Modified rules: R5, R6, R9, R10, R11, R12, R13, R14.c
- Modified formulas: D.1 (full reformulation), D.3 (two-tier), D.5 (sums invariant)

### Specialist disagreements

None. Five complementary critique vectors (fantasy/tone, formula coherence, testability, engine-API, budget realism). Convergent findings reinforced rather than contested each other. Minor philosophical alignment between game-designer BLOCK-1 (raise expiry) and performance-analyst R-5 (300s probably moot under Android OEM kill) — both said the default was poorly calibrated.

### Next step

**Pass-3 re-review in a fresh session** (`/clear` recommended; this session used substantial context across 5 specialists + heavy editing). Pass-3 will spawn fresh specialists to verify the fixes land. The architecture is stable; pass-3 verifies implementation-contract layer.

**Validation criterion for pass-3 (per CD synthesis):** D.1 cold-cache example internally consistent; ACs run green on game-ci without flake across 5 consecutive runs (when implemented); Mali-G52 retry loop holds memory steady across 10 retries (when DEVICE testing begins).

---
