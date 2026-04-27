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

## Review — 2026-04-27 — Verdict: NEEDS REVISION (pass-3)

**Pass:** 3 (second formal `/design-review`; verifies pass-2 fixes + fresh adversarial scrutiny)
**Scope signal:** M (medium, ~2-3 days author-led revision)
**Specialists:** game-designer + systems-designer + qa-lead + unity-specialist + performance-analyst (parallel spawn) + creative-director (synthesis)
**Blocking items:** 12 unique (17 raw findings; 5 convergent dedups) | **Recommended:** 16 | **Nice-to-have:** 13
**Prior verdict resolved:** Yes — pass-2 fixes verified to land correctly across all 5 specialists. Pass-3 finds NEW issues at deeper implementation-contract layer (engine-API, threading, platform-specific behavior).

### CD synthesis verdict (quoted)

> "Pass-2 fixes landed correctly. Pass-3 found 17 raw BLOCKING items (≥12 unique after dedup) — but every one of them is a contract-level or implementation-level defect, not an architectural one. The skeleton is sound. The flesh has hairline fractures. Two of three pillar fantasies have at least one finding that touches the architecture's *promise*, not just its plumbing — that's why this is NEEDS REVISION and not 'minor polish.' But every BLOCKING item is a surgical fix: add a flag, change a tier, swap an API, add a guard, await a result, document a constraint. Estimated total fix time: 2–3 days for a single author, not weeks."

### Convergent specialist findings (highest confidence)

Four findings independently flagged by 2+ specialists:

1. **R10 step 5 silently clears user-pause** — game-designer BLOCK-1 + systems-designer BLOCKING-2.
2. **AC-R10b ADVISORY → BLOCKING tier** — game-designer BLOCK-3 + qa-lead B5 + systems-designer R3.
3. **D.1 `T_slack_consumed` false-negative** — systems-designer BLOCKING-1 + performance-analyst BLOCKING-2.
4. **AC-R11 max-of-5 statistical inadequacy** — game-designer REC-4 + qa-lead B4 (different severities).

### Pillar alignment threats

- **"Cold-start ready"** (Pillar 5 preconditions): iOS `OnApplicationFocus` gap — Section B explicitly names "phone call comes in" as the canonical scenario; spec doesn't handle it on iOS.
- **"Seams are where the chaos isn't"**: R10 user-pause silently cleared on backgrounding violates the third Section B fantasy.
- **"Retry is a beat"** (Pillar 5 + Concept "<2s"): NOT structurally threatened. Issues are budget-diagnostic integrity (D.1) and tooling (5-vs-20 retry coverage). Architecture delivers <2s.

### 12 BLOCKING items (unique after dedup)

| # | Blocker | Source | Required Fix Direction |
|---|---------|--------|------------------------|
| 1 | R10 step 5 silently clears user-pause | game B1 + sys B2 | `_userPausedBeforeBackground` flag + step 5 guard `Else → IsPaused = _userPausedBeforeBackground`; D.3 pseudocode mirror |
| 2 | AC-R10b ADVISORY tier wrong for P0 contract | game B3 + qa B5 | Promote to BLOCKING CI (PLAYMODE); recount Gate Summary (BLOCKING CI PLAYMODE 8→9; ADVISORY 3→2; total stays 46) |
| 3 | D.1 slack diagnostic false-negative | sys B1 + perf B2 | Add per-term overrun assertions `T_sm_overrun`, `T_lr_overrun`; log unconditionally in DEVELOPMENT_BUILD when either > 0; update AC-D1 |
| 4 | iOS phone-call uses OnApplicationFocus, not OnApplicationPause | unity B4 | Add `OnApplicationFocus(false)` path with `FlushSync` + `AudioListener.pause = true` (NOT `IsPaused`); track as new OQ-21 |
| 5 | R7 async hooks main-thread + R13 Task.Wait deadlock | unity B3 | Add `IPreUnloadHook` contract: NO `ConfigureAwait(false)` / `Task.Run`; R13 quit hooks must be synchronous (Save/Load is via `FlushSync`; EB + Input must expose sync teardown) |
| 6 | AC-EC-E14 PlayMode test physically un-executable | qa B1 | Split into AC-EC-E14a (EditMode — interception) + AC-EC-E14b (PlayMode — post-jump); drop "programmatically trigger Play" |
| 7 | R9 RequestUnpause missing IsPaused==false guard in numbered steps | game B4 | Add as step 1 of `RequestUnpause()`, mirroring `RequestPause()` step 1; promote AC-R9e from coverage gap to ADVISORY |
| 8 | R6/R11 `Resources.UnloadUnusedAssets()` await unspecified | unity B2 | Add explicit `await`; clarify ordering for `LoadSceneMode.Single` (BEFORE `LoadSceneAsync`) vs Additive unload (AFTER `UnloadSceneAsync`) |
| 9 | R2.b rationale wrong | unity B1 | Rewrite rationale: real FEPM hazard is stale `AsyncOperation` in static field, not Awake "set and use" restriction; failure mode "silently ignored" is fabricated. Prescription correct (`BeginTransitionAsync` after Awake) — only the *why* needs fixing |
| 10 | `_pendingPause` no destination-scene guard | sys B3 | Add to R12: validate `AppState ∈ {Level}` before executing deferred `_pendingPause`; discard with ADVISORY log otherwise; track AC-R12d coverage gap |
| 11 | R9 PauseOverlay RequestUnpause bypasses `Resources.UnloadUnusedAssets()` | perf B1 | Add `await Resources.UnloadUnusedAssets()` after `UnloadSceneAsync` in `RequestUnpause()`; new memory-stability AC verifying `GC.GetTotalMemory` delta after 10 cycles |
| 12 | R14.e Boot scene EventSystem requires InputSystemUIInputModule | unity B5 | Add to R14.e: "MUST use `InputSystemUIInputModule`; `StandaloneInputModule` must not be present." AC-INV2 condition (d): scene-validator asserts module |
| 13 | Prep Phase ejected at 60s general threshold | game B2 | Add THIRD tier `bg_expiry_in_prep_seconds` (default 600s); CD adjudicated game-designer (1800s general) vs systems-designer (third tier) → third tier wins (more surgical) |

### Recommended (16 items — apply at pass-4 author's discretion)

Highest priority (pass-4 should fix):
- **REC-1**: Header "7 severity tiers" → "9" reconciliation (qa B3)
- **REC-2**: AC-BOOT-FEPM CI configuration prerequisite documentation (qa B2)
- **REC-3**: D.3 pseudocode `else` branch missing `AudioListener.pause = false` (sys B4)
- **REC-4**: AC-R11 raise to 10 retries for BLOCKING; track 20-retry soak as P1 post-Alpha (qa B4 — CD recommended deferral)

Lower priority (defer if time pressure):
- REC-5: Test file naming convention (qa R4)
- REC-6: AC-R12c sequence-counter mechanism spec (qa R2)
- REC-7: AC-INT-FF integration test file path (qa R1)
- REC-8: AC-R10c/R10d use stale single-threshold variable (qa N1)
- REC-9: R10 step 4 LevelRuntime compile-safety pattern (sys R2)
- REC-10: D.5 sums-of-ceilings strict-less-than (perf REC-3)
- REC-11: AC-P-UnityBoot Mali-G52 estimate may be underestimate (perf REC-2)
- REC-12: D.1 cold-cache 250ms hooks inconsistent with D.2 budgets (perf REC-1)
- REC-13: OQ-1 closeable now via Unity 6 docs (unity REC-5)
- REC-14: AC-R9e upgrade from coverage gap to ADVISORY (qa R3)
- REC-15: Battery/thermal retry-rate constraint (perf REC-6)
- REC-16: ADR-003 interim QA fallback policy (qa R5)

### Specialist disagreements

**One minor — bg_expiry tier proposal**: game-designer wanted to widen general threshold to 1800s; systems-designer wanted a third tier specifically for Prep Phase. Creative-director adjudicated in favor of **third tier** (more surgical, preserves 60s default for true MainMenu/idle, gives Prep Phase the protection Section B fantasy demands). Resolution applied as BLOCKING #13.

No other specialist disagreements. Findings were complementary across 5 critique vectors.

### Strategic recommendation (CD)

Foundation GDDs (Asset Loading is on deck) should spawn unity-specialist + performance-analyst by default in `/design-system`, not just `/design-review`. **10 of 17 pass-3 BLOCKING items are findings only those specialists can produce** — Foundation systems are 80% engine-contract, 20% design. Update Foundation review protocol before next Foundation GDD authoring.

### Verification criterion for pass-4

Pass-4 should be a verification pass — no fresh adversarial spawn, just confirm the 12 BLOCKING + 4 high-priority RECOMMENDED items landed correctly. If pass-4 surfaces NEW issues (vs verifying pass-3 fixes), the question shifts to "is the spec asymptotic to perfection or genuinely incomplete?" — at that point, escalate to CD for protocol-level review.

### File metrics

- BLOCKING items found pass-3: 17 raw, 12 unique
- Recommended: 16
- Nice-to-have: 13
- Pass-2 fixes verified holding: 8 (D.1 reformulation, AC-R5 float, R9 atomic, R13 Editor guard, AC count math, R12 firing order, two-tier bg_expiry, UnloadUnusedAssets in R6)
- New OQs proposed: 1 (OQ-21 iOS OnApplicationFocus)
- OQs closeable: 1 (OQ-1 LoadSceneParameters per unity-specialist verification)

### Next step

**Pass-4 author-led revision in fresh session** (`/clear` recommended; this session ran 5 parallel specialists + CD synthesis). Apply all 12 BLOCKING + 4 high-priority RECOMMENDED items. Pass-4 then runs as verification-only re-review. Architecture is stable; pass-3 verifies implementation-contract layer the same way pass-2 verified formula-coherence layer.

---

## Review — 2026-04-27 — Verdict: REVISED-pending-pass-5-verification (pass-4)

**Pass:** 4 (author-led revision; no fresh adversarial spawn per pass-3 verification protocol)
**Scope signal:** M (medium, single-session author-led)
**Specialists:** None (author-led revision only — pass-3 specialists already adjudicated all 12 BLOCKING + 16 RECOMMENDED items)
**Blocking items applied:** 12 of 12 | **Recommended applied:** 4 of 16 (high-priority subset per pass-3 review log triage; remaining 12 RECOMMENDED deferred to pass-5 or implementation time)
**Prior verdict resolved:** Pending (pass-5 verification will confirm)

### Pass-4 protocol followed

Per pass-3 review log "Verification criterion for pass-4": this pass applied all 12 BLOCKING + 4 high-priority RECOMMENDED items via direct edits, with no fresh adversarial spawn. The next session will run pass-5 as verification-only — confirming that fixes landed correctly, NOT scanning for new issues. Per pass-3 CD strategic recommendation: if pass-5 surfaces NEW issues vs verifying pass-4 fixes, escalate to CD for protocol-level review.

### 12 BLOCKING items applied

| # | Blocker | Pass-3 source | Pass-4 fix landed |
|---|---------|---------------|-------------------|
| 1 | R10 step 5 silently clears user-pause | game B1 + sys B2 | R10 step 2 captures `_userPausedBeforeBackground = IsPaused` BEFORE mutation; resume step 5 sets `IsPaused = _userPausedBeforeBackground` (mirrors capture). D.3 pseudocode + variable table updated; new sub-flag table row in Detailed Design. AC-R10b verifies the preservation. |
| 2 | AC-R10b ADVISORY tier wrong for P0 contract | game B3 + qa B5 | AC-R10b promoted to BLOCKING CI (PLAYMODE). Gate Summary tier counts updated (BLOCKING CI PLAYMODE 8→9; ADVISORY net same — AC-R9e takes the freed slot). Total recounted at 49 (was 46) — see net change explanation in BLOCKING #6 below. |
| 3 | D.1 slack diagnostic false-negative | sys B1 + perf B2 | Added `T_sm_overrun = max(0, T_sm_actual - T_sm_budget)` and `T_lr_overrun = max(0, T_lr_actual - T_lr_budget)` per-term assertions. DEVELOPMENT_BUILD logs each unconditionally when > 0, independent of `T_slack_consumed > T_slack_budget`. New worked example shows the false-negative case (1300ms T_sm + 400ms T_lr would have appeared as 0ms slack consumed under naive subtraction). AC-D1 rewritten to verify unconditional logging. |
| 4 | iOS phone-call uses OnApplicationFocus, not OnApplicationPause | unity B4 | New iOS-only path in R10: `OnApplicationFocus(false)` gated by `Application.platform == RuntimePlatform.IPhonePlayer` calls `SaveManager.FlushSync()` + sets `AudioListener.pause = true` (NOT `IsPaused = true` — focus loss is not a user-intended pause). On focus return: `AudioListener.pause = false`. New OQ-21 tracks device verification across iOS focus-loss scenarios (phone call, banner notification, Control Center, lock-screen-without-background). |
| 5 | R7 async hooks main-thread + R13 Task.Wait deadlock | unity B3 | R7 `IPreUnloadHook` async-contract constraints added: `ConfigureAwait(false)` and `Task.Run` prohibited (Save/Load is the documented exception with isolated background thread). R13 rewritten to use synchronous teardown methods exclusively — Foundation siblings MUST expose: `SaveManager.FlushSync()` (already exists), `EventBus.DisposeSync()` (cross-GDD obligation), `InputDispatcher.TeardownSync()` (cross-GDD obligation). No `await` on the quit path in either Editor or Player builds for uniformity. Bidirectional consistency amendments updated to capture Event Bus + Input new sync-teardown obligations. |
| 6 | AC-EC-E14 PlayMode test physically un-executable | qa B1 | AC-EC-E14 split into AC-EC-E14a (EDITMODE — interception via `playModeStateChanged` callback + `EditorPrefs` storage) + AC-EC-E14b (PLAYMODE — post-jump behavior with Play mode already entered). Pass-2's "programmatically trigger Play" step dropped — that side now belongs to E14a in EDITMODE. H.3 count 7→8; tier counts updated: EDITMODE 2→3 (+E14a); PLAYMODE 15→16 (+E14b, -E14, +R9f). Net total +3 from all pass-3 changes. |
| 7 | R9 RequestUnpause missing IsPaused==false guard in numbered steps | game B4 | RequestUnpause step 1 added: "If `IsPaused == false` → no-op (mirrors RequestPause step 1)". AC-R9e promoted from coverage gap to ADVISORY (PLAYMODE) verifying the full sequence including the new step 1 guard. Coverage gap entry retained for change-log traceability with note to remove at pass-5. |
| 8 | R6/R11 `Resources.UnloadUnusedAssets()` await unspecified | unity B2 | R6 rewritten with explicit mode-dependent ordering: `LoadSceneMode.Single` auto-reclaims (NO redundant manual call); additive unload requires explicit `await Resources.UnloadUnusedAssets()` AFTER `await UnloadSceneAsync(...)`. Pass-2's redundant placement of UnloadUnusedAssets inside `BeginTransitionAsync` corrected. R11 budget breakdown updated to remove the spurious "+ UnloadUnusedAssets 70ms" line item. D.1 worked examples recomputed accordingly. R9 RequestUnpause step 3 explicitly awaits the call. |
| 9 | R2.b rationale wrong | unity B1 | R2.b prescription preserved (`allowSceneActivation` not in Awake, defer to async post-Awake `BeginTransitionAsync`). Rationale rewritten: real failure mode is stale `AsyncOperation` reference held in a static field across FEPM cycles when domain reload disabled, causing `MissingReferenceException` on the second cycle. Pass-2's "silently ignored on second FEPM cycle" claim removed (it was fabricated — the API is not silent, it produces undefined results). R2.a `Reset()` discipline backstops by clearing static refs. Unity 6 docs link retained. |
| 10 | `_pendingPause` no destination-scene guard | sys B3 | R12 firing-order block extended: BEFORE firing `_pendingPause` after a transition completes, validate the new `AppState ∈ {Level}` (MVP — only state where pause is valid; E7 already prohibits Boot pause; same logic for MainMenu/BiomeMap/Shop). On invalid destination: discard `_pendingPause`, log ADVISORY in DEVELOPMENT_BUILD, no error. AC-R12d added to Known coverage gaps with full GIVEN/WHEN/THEN spec for Story 2 implementation time. |
| 11 | R9 PauseOverlay RequestUnpause bypasses `Resources.UnloadUnusedAssets()` | perf B1 | RequestUnpause step 3 explicitly awaits `Resources.UnloadUnusedAssets()` after `await UnloadSceneAsync("PauseOverlay")` — additive unload does not auto-reclaim per Unity 6 contract. New AC-R9f memory-stability AC: 10 pause/unpause cycles after a warm-up cycle, `GC.GetTotalMemory(true)` delta within `[baseline - 1MB, baseline + 2MB]` (asymmetric tolerance). Failure mode without the fix: 5–20MB drift across 10 cycles. |
| 12 | R14.e Boot scene EventSystem requires InputSystemUIInputModule | unity B5 | R14.e extended: Boot-scene EventSystem MUST use `InputSystemUIInputModule`; `StandaloneInputModule` MUST NOT be present on the same GameObject. Mixed-module setup silently drops UI events because the two modules race. AC-INV2 condition (d) added: scene-validator asserts both module presence (c) + module absence (d). |
| 13 | Prep Phase ejected at 60s general threshold | game B2 | Third tier `bg_expiry_in_prep_seconds` (default 600s, safe-range [300, 1200]) added to R10 step-5 threshold logic, D.3 pseudocode + variable table, Section G tuning knobs. New `_backgroundedDuringPrep` flag captured at OS-background time (R10 step 5). New worked example: 8-min Prep Phase interruption resumes in-place (480s < 600s); 12-min ejects to MainMenu. Three-tier threshold replaces pass-2's two-tier. CD adjudicated in favor of the third-tier approach (more surgical) over widening general to 1800s (game-designer's alternative). |

### 4 high-priority RECOMMENDED applied

| # | Recommended | Pass-3 source | Pass-4 fix landed |
|---|-------------|---------------|-------------------|
| REC-1 | "7 severity tiers" → "9 severity tiers" | qa B3 | Section H opening paragraph + Gate Summary header updated. Subsection counts updated: H.1 28→30; H.3 7→8. Total 46→49. Verified via grep — no stale "7 severity tiers" or "46 ACs" references remain. |
| REC-2 | AC-BOOT-FEPM CI prerequisite documentation | qa B2 | AC-BOOT-FEPM expanded with CI prerequisite block: `ProjectSettings/EditorSettings.asset` MUST contain `m_EnterPlayModeOptionsEnabled: 1` AND `m_EnterPlayModeOptions: 2` (DisableDomainReload encoding) before the test can run. Without the prerequisite, test runs in default-domain-reload mode with false-positive coverage. |
| REC-3 | D.3 pseudocode `else` branch missing `AudioListener.pause = false` | sys B4 | D.3 pseudocode rewritten with `AudioListener.pause = false` as Step 0 (unconditional, BEFORE the threshold branch) — ensures audio restore even when the resume path branches to `BeginTransitionAsync`. R10 step 1 of `paused == false` branch already does this; the pseudocode now matches. |
| REC-4 | AC-R11 raise to 10 retries; track 20-retry as P1 post-Alpha | qa B4 | AC-R11 sample size raised from 5 → 10 retries for the BLOCKING DEVICE tier (n=5 was statistically thin; n=10 reliably surfaces 1-in-100 tail events). 20-retry soak deferred to P1 post-Alpha sprint task (`production/sprints/post-alpha/perf-soak-20-retry.md`) — NOT a blocking gate. CD recommended this deferral. |

### 12 RECOMMENDED deferred

REC-5 through REC-16 (test file naming, AC sequence-counter spec, AC-INT-FF integration test path, AC-R10c/d stale variable refs, R10 LevelRuntime compile pattern, D.5 strict-less-than, AC-P-UnityBoot Mali-G52 estimate, D.1 cold-cache 250ms hooks, OQ-1 closeable, AC-R9e tier upgrade — already covered by pass-4 BLOCKING #7, battery/thermal retry rate, ADR-003 interim QA fallback) deferred to pass-5 verification or implementation-time discovery. Pass-3 review log triaged these as "lower priority — defer if time pressure."

### File metrics

- Lines: 727 → ~880 (+153) — primarily from R10 expansion (iOS path + three-tier + user-pause), R7 async-contract block, R13 sync-teardown contract, AC-EC-E14 split (one AC → two), new AC-R9e/AC-R9f.
- ACs: 46 → 49 (+3 net: +AC-R9e, +AC-R9f, +AC-EC-E14a, +AC-EC-E14b, -AC-EC-E14)
- OQs: 20 → 21 (+1: OQ-21 iOS focus-loss device verification)
- Modified rules: R2.b, R6, R7, R9 (RequestUnpause), R10, R11, R12, R13, R14.e
- Modified formulas: D.1 (per-term overrun), D.3 (three-tier + user-pause + Step 0)
- Modified ACs: AC-D1, AC-INV2, AC-R10b (tier), AC-R11, AC-BOOT-FEPM
- Split ACs: AC-EC-E14 → AC-EC-E14a (EDITMODE) + AC-EC-E14b (PLAYMODE)
- New ACs: AC-R9e (ADVISORY), AC-R9f (PLAYMODE)
- New OQ: OQ-21
- Bidirectional amendments: Event Bus + Input now require sync teardown methods (was: pre-unload hooks only)
- Stale references checked + updated: zero "7 severity tiers" remnants; zero "46 ACs" remnants; un-suffixed "AC-EC-E14" references in tooling table + shift-left notes + E14 edge case prose all updated to E14a/E14b

### Specialist disagreements

None at pass-4 (no fresh spawn). Pass-3 single disagreement (bg_expiry tier proposal, BLOCKING #13) was already CD-adjudicated in favor of three-tier approach; pass-4 implements the adjudicated outcome.

### Next step

**Pass-5 verification-only re-review in fresh session** (`/clear` recommended). Pass-5 spawns specialists ONLY to verify the pass-4 fixes landed correctly — NOT to scan for new issues. Per pass-3 CD: "if pass-5 surfaces NEW issues vs verifying pass-4 fixes, escalate to CD for protocol-level review." If pass-5 finds the fixes hold, GDD status moves to APPROVED.

**Validation criteria for pass-5:**
- All 12 BLOCKING fixes from pass-3 review log table verified present in current GDD body.
- All 4 high-priority RECOMMENDED fixes verified.
- AC count math reconciled across header + subsection counts + Gate Summary tier sums (49 / 30+3+8+2+6 / 9+9+16+2+4+3+2+1+3).
- No internal contradictions between R10 step numbering, D.3 pseudocode, sub-flag table, Section G tuning knobs, and worked examples.
- Cross-GDD obligations (Event Bus `DisposeSync`, Input `TeardownSync`) flagged for Phase 5 amendment work but not yet executed (pass-5 verifies they are correctly TRACKED, not yet APPLIED).

---

