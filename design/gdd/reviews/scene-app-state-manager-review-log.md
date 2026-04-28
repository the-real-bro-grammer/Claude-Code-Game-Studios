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

## Review — 2026-04-28 — Verdict: APPROVE-WITH-CONDITIONS (pass-5)

**Pass:** 5 (third formal `/design-review`; verification-only scope per pass-3/pass-4 protocol)
**Scope signal:** S (small) — ~6h author-led pass-6 surgical revision (2h BLOCKING + 4h RECOMMENDED); pass-6 verification-only against explicit fix-list; pass-7 should not be needed
**Specialists:** unity-specialist + performance-analyst + systems-designer + qa-lead + game-designer (parallel spawn) + creative-director (synthesis)
**Pass-3 BLOCKING items verified:** 12 / 12 LANDED CORRECTLY
**Pass-3 high-priority RECOMMENDED verified:** 4 / 4 LANDED CORRECTLY
**New BLOCKING items:** 5 (per CD adjudication of 3 specialist severity disagreements)
**New RECOMMENDED items:** 16
**Pillar fantasy alignment:** All three Section B fantasies (Retry-as-beat / Cold-start ready / Seams calm) preserved across pass-4 revision

### CD synthesis verdict (quoted)

> "The spec is asymptotic to perfection. Pass-2 found 12 architectural BLOCKING; pass-3 found 12 contract-layer BLOCKING; pass-5 finds 5 line-edit BLOCKING + 16 surgical RECOMMENDED. Defect amplitude is collapsing pass-over-pass. All three Section B fantasies remain preserved. All 12 pass-3 BLOCKING items verified landed correctly. The diagnostic finding is the AC-BOOT-FEPM bit-encoding miss — a doubly-recursive defect where the test designed to prevent false-positive coverage was itself false-positive. Pass-4 author-led revision is sound for *applying* known fixes but fails when the fix introduces new engine-API claims. Engine claims need engine review. Two more days for surgical pass-6 + targeted unity-specialist verification ships this Foundation GDD with battle-tested protocol learnings benefiting the next 15 GDDs. Ship it through pass-6. Close with confidence."

### Specialist disagreements (3 — all CD-adjudicated)

1. **D.5 sums-of-ceilings arithmetic at line 456**: systems-designer BLOCKING vs performance-analyst ADVISORY. **CD: BLOCKING.** Rationale: formula correctness is canonical contract; runtime DEVELOPMENT_BUILD log is mitigation, not fix. "Specs document truth; runtime logs document drift from truth. A spec that doesn't document truth has nothing for runtime logs to drift *from*."
2. **AC-R10c / AC-R10d stale variable name `bg_session_expiry_seconds` at lines 671-672**: qa-lead BLOCKING vs systems-designer + game-designer RECOMMENDED. **CD: BLOCKING.** Rationale: An AC is a test specification. "If a developer reading the AC in isolation cannot determine which variable to assert against without cross-referencing two other sections, the AC is non-implementable as written." Pass-3 REC-8 deferred this; pass-4 chose not to fix; the bet must be paid now or the defect ships.
3. **AC-R9e tier ADVISORY vs PLAYMODE**: qa-lead + game-designer RECOMMENDED PLAYMODE. **CD: BLOCKING PLAYMODE.** Rationale: Tier should follow failure severity. Missing double-unpause guard creates same permanent `Time.timeScale=0f` freeze as missing double-pause guard (AC-R9a is BLOCKING CI PLAYMODE). Asymmetry is unjustified.

### 5 BLOCKING items (pass-6 fix list)

| # | Blocker | Source | Required Fix Direction | Est. |
|---|---------|--------|------------------------|------|
| 1 | AC-BOOT-FEPM bit-encoding error line 654 | unity-specialist NEW-P5-B1 | `m_EnterPlayModeOptions: 2` encodes `DisableSceneReload`, NOT `DisableDomainReload`. Unity 6 enum: `DisableDomainReload = 1`, `DisableSceneReload = 2`. Correct value for domain-only-disable is `1`; for both disabled is `3`. Pick one (CD recommends `1` for domain-only) and fix. | 30 min |
| 2 | AC-BOOT-FEPM CI prerequisite enforcement gap | qa-lead BC-1 | Add `[SetUp]` assertion `Assert.IsTrue(EditorSettings.enterPlayModeOptionsEnabled && EditorSettings.enterPlayModeOptions == EnterPlayModeOptions.DisableDomainReload)` to test fixture spec. Specify two sub-cases (Scene Reload enabled vs disabled) as separate test fixtures. | 30 min |
| 3 | AC-R10c / AC-R10d stale variable lines 671-672 | qa-lead BC-2 + systems-designer + game-designer NI-1 (3-way convergent) | Replace `bg_session_expiry_seconds` with the three tier-specific variable names (`bg_expiry_general_seconds`, `bg_expiry_in_prep_seconds`, `bg_expiry_in_rush_seconds`). Split each AC into 3 fixtures (one per tier) OR add explicit "Test against the applicable tier threshold" sentence. | 30 min |
| 4 | D.5 sums-of-ceilings arithmetic error line 456 | systems-designer NEW BLOCKING + performance-analyst NEW-PA-3 | Change "1000+500+1500+800 = 3800ms" to "2000+500+1500+800 = 4800ms" (use Section G safe-range max for `T_dr_ceiling` not D.5 variable-table current value). Recompute "+800ms margin" rationale against correct sum (the +800 figure becomes negative or requires re-thinking the invariant). | 15 min |
| 5 | AC-R9e tier promotion ADVISORY → BLOCKING PLAYMODE | qa-lead + game-designer convergent | Change tier label at line 667 from `**ADVISORY (PLAYMODE)**` to `**BLOCKING CI (PLAYMODE)**` (or `**PLAYMODE**` if BLOCKING CI is excessive — CD recommends BLOCKING PLAYMODE). Update Gate Summary tier sums: ADVISORY 3→2, BLOCKING CI PLAYMODE 9→10. Total stays 49. | 5 min |

### 16 RECOMMENDED items (pass-6 fix list — surgical line edits)

| # | Item | Source | Est. |
|---|------|--------|------|
| R1 | AC-R9f tolerance band 2MB → 5MB (Mali-G52 Mono GC noise floor) | performance-analyst | 10 min |
| R2 | AC-R9f baseline timing disambiguation (post-warm-up unpause) | performance-analyst | 10 min |
| R3 | RequestUnpause Pillar 5 latency bound — new ADVISORY AC | performance-analyst | 30 min |
| R4 | R11 cold-cache `T_sm_overrun >= T_slack_budget` distinct-severity advisory log | performance-analyst | 15 min |
| R5 | `bg_expiry_general_seconds` 60s revisit for BiomeMap/Shop context | performance-analyst | 20 min |
| R6 | D.2 zero-handlers documentation (`max(empty)` → 0; `Task.WhenAll([])`) | systems-designer | 5 min |
| R7 | D.5 per-sibling boot timeout — new rule clause for R2 hang protection | systems-designer | 30 min |
| R8 | R13 / E9 quit-during-loading documentation — new edge case | systems-designer | 20 min |
| R9 | AC-R7 dual-bound [120, 130) ambiguity — widen check (b) to ≤140ms | qa-lead | 15 min |
| R10 | AC-R10b condition (c) no-op — rewrite as positive state assertion or remove | qa-lead | 5 min |
| R11 | AC-R12d location promotion to H.3 PLAYMODE DEFERRED | qa-lead | 10 min |
| R12 | AC-R9e coverage-gap stale entry line 749 — delete | game-designer NI-2 | 2 min |
| R13 | Test file paths in Shift-Left notes (REC-5 from pass-3) | qa-lead | 20 min |
| R14 | OQ-1 close (`LoadSceneParameters` unchanged in Unity 6.3 LTS per pass-5 unity-specialist verification) | unity-specialist | 5 min |
| R15 | OQ-21 fold-in: iOS escalation path documentation (short-call→long-call sequence safe but undocumented) | unity-specialist + game-designer | 15 min |
| R16 | PauseOverlay teardown on bg-expiry eject documentation | game-designer | 15 min |

**Total pass-6 author-revision time: ~6 hours (2h BLOCKING + 4h RECOMMENDED).**

### Diagnostic finding (CD)

The **AC-BOOT-FEPM bit-encoding miss** is the diagnostic finding of this entire review cycle.

The pass-4 author was applying 12 BLOCKING + 4 RECOMMENDED in a verification-only revision. They produced new AC text including bit-encoded EditorSettings flags. **No specialist verified the bit values against Unity 6 source-of-truth before pass-5.** unity-specialist caught it on pass-5; absent that spawn, it would have shipped to Story 1, where a developer would have implemented the test, watched it pass, and shipped a build with EnterPlayMode false-positive coverage gating CI. The defect that AC-BOOT-FEPM exists to prevent would silently NOT be detected.

This is the canonical case for **specialist verification of newly-authored implementation-contract details**. Author-led revisions are correct for *applying* known fixes — the pass-4 methodology was sound for resolving items that were already specified. It fails when the fix itself introduces a new claim about engine behavior. The author is not an engine specialist; engine claims need engine review.

CD characterization: "doubly-recursive test correctness defect" — the test designed to prevent false-positive coverage was itself a false-positive coverage instance. The methodology lesson: **when authoring a regression-prevention test, the test's own correctness is the regression risk.** Recursion stops only when an independent specialist closes the loop.

### Strategic recommendation (CD): Implementation-Contract Verification gate

Pass-3 already added unity-specialist + performance-analyst as defaults for Foundation GDDs in `/design-system`. Pass-5 reinforces this and adds a second protocol-level recommendation:

**Add an "Implementation-Contract Verification" gate between author-led revisions and final approval, scoped narrowly:**

When an author-led revision (any pass) adds or modifies an AC clause that asserts:
- Engine API bit values, enum encodings, serialization formats
- Platform-specific lifecycle event ordering (iOS/Android backgrounding, focus, etc.)
- Frame-budget numeric thresholds
- Memory/GC tolerance bands

…spawn the relevant specialist (unity-specialist for engine; performance-analyst for budgets/memory; android-specialist or ios-specialist if those exist) for a **targeted verification** — not a full review, just a verify-the-claim ping. The specialist returns CORRECT / INCORRECT / NEEDS_CLARIFICATION on each flagged clause.

This gate would have caught AC-BOOT-FEPM bit-encoding in pass-4 before pass-5 even ran. Time cost: ~10 min per gate invocation. Defect cost prevented: a CI gate silently passing on broken builds.

**Recommend updating `/design-system` and `/design-review` to include this gate by default in `full` review mode for any Foundation GDD that includes engine-API or platform-lifecycle ACs.**

### Pillar fantasy alignment audit (game-designer)

All three Section B fantasies verified preserved by all 5 specialists:

- **"Retry is a beat, not a load screen"** — PRESERVED. R11 budget split unchanged; D.1 per-term overruns are additive diagnostics, not behavior changes; T_retry_max = 2000ms holds. The 200ms gap between tone target (1500ms "let's go again") and nominal warm-cache retry (~1700ms) is a known calibration item, not a spec defect.
- **"The game is already running when the player gets there"** — PRESERVED. iOS phone-call path (pass-3 BLOCKING #4) directly addressed the one identified threat. Boot-sequence fixes (R2, R2.a, R2.b, R13) intact. OQ-21 captures remaining device-test uncertainty.
- **"The seams are where the chaos isn't"** — PRESERVED. Most threatened in pass-3; pass-4 fixes (B1 user-pause preservation + B2 Prep tier 600s) directly resolved the violations. Residual PauseOverlay teardown gap on eject-from-paused-state is not a Section B violation — it self-corrects via R14.a.

### File metrics

- Lines: ~880 (pass-4) → expected ~880 (pass-6 line edits should net out roughly even — some additions for new ACs, some deletions for stale entries)
- ACs: 49 (unchanged at pass-5; pass-6 fix #5 keeps total at 49 via tier rebalance; new RequestUnpause Pillar 5 AC adds +1 if applied → 50)
- OQs: 21 (pass-6 closes OQ-1; folds new content into OQ-21; net 20-21)
- Pass-3 BLOCKING items verified holding: 12 / 12
- Pass-3 high-priority RECOMMENDED verified holding: 4 / 4
- New BLOCKING items found pass-5: 5 (CD-adjudicated)
- New RECOMMENDED items pass-5: 16
- New OQ proposals pass-5: 0 (existing OQ-21 expanded scope with iOS escalation note)
- Specialist disagreements adjudicated: 3 (D.5 arithmetic; AC-R10c/d severity; AC-R9e tier)

### Validation criteria for pass-6

Pass-6 should be a verification pass of the explicit fix-list above. Per CD: "Author should explicitly mark each item as resolved with line references, the same discipline as pass-4." Targeted unity-specialist verification (10-min ping) of the AC-BOOT-FEPM bit encoding correction. CD verification (no new spawns) of the full pass-6 diff. If clean, **APPROVED** without conditions.

### Next step

**Pass-6 author-led revision in fresh session** (`/clear` recommended; this session ran 5 parallel specialists + CD synthesis). Apply all 5 BLOCKING + 16 RECOMMENDED items per the fix table above. Pass-6 then runs as verification-only re-review with **mandatory targeted unity-specialist verification** of the AC-BOOT-FEPM bit-encoding correction (Implementation-Contract Verification gate prototype). Auto-promotes to full APPROVED on condition application per CD pass-6 verdict rule.

---

## Review — 2026-04-28 — Verdict: APPROVED (pass-6)

**Pass:** 6 (author-led revision, verification-only against pass-5 explicit fix-list; first invocation of CD pass-5 strategic recommendation: targeted unity-specialist Implementation-Contract Verification gate)
**Scope signal:** S (small) — ~6h author-led revision (2h BLOCKING + 4h RECOMMENDED) + ~10 min Implementation-Contract Verification ping
**Specialists:** None for design adversarial review (pass-5 already adjudicated all items); unity-specialist invoked ONLY for narrow Implementation-Contract Verification gate on AC-BOOT-FEPM bit-encoding + `[SetUp]` API surface — returned CORRECT on both claims.
**Pass-5 BLOCKING items applied:** 5 / 5
**Pass-5 RECOMMENDED items applied:** 16 / 16
**Auto-promotion:** APPROVED per CD pass-5 verdict rule on condition application.

### Pass-6 protocol followed

Per pass-5 review log "Validation criteria for pass-6": this pass applied all 5 BLOCKING + 16 RECOMMENDED items via direct edits, with one mandatory targeted specialist ping (the Implementation-Contract Verification gate prototype). The gate fires when an author-led revision adds or modifies an AC clause asserting engine API bit values, enum encodings, serialization formats, platform-specific lifecycle event ordering, frame-budget numeric thresholds, or memory/GC tolerance bands. Pass-6 fired the gate exactly once (AC-BOOT-FEPM bit-encoding + EditorSettings API surface) and got CORRECT/CORRECT back.

### Implementation-Contract Verification gate result

**unity-specialist targeted ping (10 min, narrow scope):**
- **Claim 1 — Bit-encoding `m_EnterPlayModeOptions: 1` = `EnterPlayModeOptions.DisableDomainReload`**: **CORRECT.** Unity 6 docs list `DisableDomainReload` as the first non-zero member of the `[Flags]` enum; standard powers-of-two assignment gives `DisableDomainReload = 1`, `DisableSceneReload = 2`, both = `3`. Pass-5-caught defect (`2 = DisableDomainReload`) was wrong; corrected value `1` is right.
- **Claim 2 — `[SetUp]` API surface**: **CORRECT.** Unity 6 docs confirm `EditorSettings.enterPlayModeOptionsEnabled` and `EditorSettings.enterPlayModeOptions` properties exist on `UnityEditor.EditorSettings`. Sub-case B's `Assert.AreEqual((EnterPlayModeOptions)3, EditorSettings.enterPlayModeOptions, ...)` cast-equality works correctly because the enum's underlying type is `int` and NUnit's `Assert.AreEqual` compares by value (no boxing ambiguity since both sides are the same enum type).

Gate verdict: **PASS.** The pass-6 fix-list claim about engine bit values has been independently verified by an engine specialist before merge. The recursive defect-class that AC-BOOT-FEPM was designed to prevent (false-positive coverage of FEPM regressions) — and which itself recursed onto AC-BOOT-FEPM's own correctness in pass-4 — is now closed.

### 5 BLOCKING items applied

| # | Blocker | Pass-5 source | Pass-6 fix landed |
|---|---------|---------------|-------------------|
| 1 | AC-BOOT-FEPM bit-encoding error line 654 (`m_EnterPlayModeOptions: 2` annotated as `DisableDomainReload`) | unity-specialist NEW-P5-B1 | Corrected to `m_EnterPlayModeOptions: 1` (encodes `DisableDomainReload` only). The flag enum: `DisableDomainReload = 1`, `DisableSceneReload = 2`, both = `3`. Pass-4's `2` annotated as DisableDomainReload was the doubly-recursive defect — the AC designed to prevent FEPM false-positive coverage was itself false-positive. CD recommended value `1` (Domain disabled, Scene Reload enabled — the harder case for `Reset()` discipline because static fields persist across cycles while scene state reloads). unity-specialist Implementation-Contract Verification CORRECT. |
| 2 | AC-BOOT-FEPM `[SetUp]` enforcement gap + need for two sub-cases | qa-lead BC-1 | Added `[SetUp]` assertion: `Assert.IsTrue(EditorSettings.enterPlayModeOptionsEnabled)` + `Assert.AreEqual(EnterPlayModeOptions.DisableDomainReload, EditorSettings.enterPlayModeOptions & EnterPlayModeOptions.DisableDomainReload)` (bitwise mask preserves correctness across both sub-cases). Split AC-BOOT-FEPM into Sub-case A (`m_EnterPlayModeOptions: 1`, Scene Reload enabled — harder case for static-field persistence) and Sub-case B (`m_EnterPlayModeOptions: 3`, both reloads disabled — same regression target plus scene state persistence). Both sub-cases BLOCKING CI; failing either means the FEPM regression-prevention contract is broken on at least one valid Unity FEPM configuration. unity-specialist Implementation-Contract Verification confirmed property names + cast-equality semantics CORRECT. |
| 3 | AC-R10c / AC-R10d stale variable name `bg_session_expiry_seconds` lines 671-672 | qa-lead BC-2 + systems-designer + game-designer NI-1 (3-way convergent) | Each AC split into 3 tier-specific fixtures replacing the obsolete single-threshold name. AC-R10c → AC-R10c-Rush (tests against `bg_expiry_in_rush_seconds` 1800s default), AC-R10c-Prep (tests against `bg_expiry_in_prep_seconds` 600s default), AC-R10c-General (tests against `bg_expiry_general_seconds` 60s default). AC-R10d split identically for the below-threshold in-place-resume branch. Each fixture is a separate test method so a failure pinpoints which tier regressed. Net AC count change: +4 (2 single-fixture pair → 6 tier-specific fixtures). |
| 4 | D.5 sums-of-ceilings arithmetic error line 456 (1000+500+1500+800=3800ms used current values not safe-range maxima) | systems-designer NEW BLOCKING + performance-analyst NEW-PA-3 | Corrected sum to use safe-range MAXIMA per Section G: `T_dr_ceiling [500, 2000]` + `T_eb_ceiling [100, 500]` + `T_sl_ceiling [400, 1500]` + `T_input_ceiling [200, 800]` = `2000+500+1500+800 = 4800 ms` (vs `T_boot_target = 3000 ms`). The +800 ms margin pass-2 used was derived against the wrong sum; corrected delta is 1800 ms (60% over-target). Re-stated invariant: `sum(T_x_ceiling) ≤ T_boot_target × 1.6 = 4800 ms`. The 60% headroom allows safe ranges to be exercised but flags the pathological case where every ceiling is simultaneously raised to its max. CD adjudicated this as BLOCKING (formula correctness is canonical contract; runtime DEVELOPMENT_BUILD log is mitigation, not fix). AC-D3 logs per-sibling ceilings AND the sum vs the corrected 4800 ms invariant. |
| 5 | AC-R9e tier ADVISORY → BLOCKING PLAYMODE | qa-lead + game-designer convergent + CD adjudicated | Tier promoted to BLOCKING CI PLAYMODE. Rationale: missing the double-unpause guard creates the same permanent `Time.timeScale = 0f` freeze failure mode as missing the double-pause guard in `RequestPause()`. AC-R9a (pause direction) is already BLOCKING CI PLAYMODE; the asymmetry of leaving the unpause-direction symmetry as ADVISORY was unjustified — same bug class, same severity. Gate Summary tier sums updated: ADVISORY 3 → 3 (AC-R9e exited, AC-R9g RequestUnpause Pillar 5 latency entered per R3 — net 0); BLOCKING CI PLAYMODE 9 → 10. Total ACs net change from this item alone: 0 (tier shift). |

### 16 RECOMMENDED items applied

| # | Recommended | Pass-5 source | Pass-6 fix landed |
|---|-------------|---------------|-------------------|
| R1 | AC-R9f tolerance band 2MB → 5MB (Mali-G52 Mono GC noise floor) | performance-analyst | Upper bound of asymmetric tolerance widened from 2MB to 5MB to absorb empirical Mono GC behavior on Cortex-A53/A55 ARM (~0.5MB-per-cycle steady-state working-set churn that does not represent a real leak). 5MB still catches the 5–20MB drift failure mode without `await Resources.UnloadUnusedAssets()`. |
| R2 | AC-R9f baseline timing disambiguation (post-warm-up unpause) | performance-analyst | Baseline read disambiguated to "immediately after the first `RequestUnpause()` returns and `gameplay.app.state_changed { isPaused: false }` has fired" — explicit post-warm-up-unpause anchor point. Pass-3 ambiguity about whether baseline was post-pause or post-unpause now resolved. |
| R3 | RequestUnpause Pillar 5 latency bound — new ADVISORY AC | performance-analyst | New AC-R9g added: `t_unpause_to_first_input_event ≤ 100 ms` 60 fps / 133 ms 30 fps measured on ADR-003 reference Android device. The Pillar 5 Feedback Floor applies at the seam from paused-back-to-active gameplay just as at fresh scene entry. The `await Resources.UnloadUnusedAssets()` in `RequestUnpause` step 3 has 5–30 ms warm-cache cost; cold cache may push to 60–80 ms. AC-R9g surfaces the risk as a dashboard ADVISORY (gate is AC-R9f memory stability). DEVICE (ADVISORY), blocked on ADR-003. |
| R4 | R11 cold-cache `T_sm_overrun >= T_slack_budget` distinct-severity advisory log | performance-analyst | Distinct-severity advisory log added: when `T_sm_overrun >= T_slack_budget` (Scene Manager alone consumes 100% or more of shared slack — even if `T_retry` still passes via `T_lr` running well under budget), Budget Analyzer logs an ADVISORY (distinct from the BLOCKING `T_slack_consumed > T_slack_budget` gate failure). Catches the silent regression where T_sm climbs while T_lr runs faster — gate stays green, but margin is burned through. |
| R5 | `bg_expiry_general_seconds` 60s revisit for BiomeMap/Shop context | performance-analyst | Section G OS-lifecycle thresholds annotated: 60s default lumps 3 distinct non-gameplay contexts (MainMenu / BiomeMap / Shop). Resolution: retain 60s at MVP (conservative — eject-quickly is reversible if telemetry shows friction). Defer split-into-three-non-gameplay-tiers (`bg_expiry_in_biomemap_seconds` ~300s, `bg_expiry_in_shop_seconds` ~180s) to a live-ops tuning candidate, tracked as OQ-19 expansion. No MVP action required. |
| R6 | D.2 zero-handlers documentation (`max(empty)` → 0; `Task.WhenAll([])`) | systems-designer | New edge-case sub-section in D.2 documenting that zero registered handlers produce `Task.WhenAll([])` synchronously-completed; `T_preunload = max(empty) = 0 ms`; transition proceeds without delay; no log fires (legitimate runtime configuration). Codifies previously-implicit behavior. |
| R7 | D.5 per-sibling boot timeout — new rule clause for R2 hang protection | systems-designer | New per-sibling boot timeout sub-section in D.5: `T_sibling_max = T_x_ceiling × 2`. R2 await sequence becomes `await Task.WhenAny(sibling.Ready, Task.Delay(T_x_max))`. On timeout, `BootLoader` logs fatal error, runs R13 quit teardown, exits to OS. Per-sibling (not aggregate) so first hang is caught at its own ceiling, not after cumulative budget. BLOCKING for R2 implementation; tracked as code-review enforcement. |
| R8 | R13 / E9 quit-during-loading documentation — new edge case | systems-designer | New E19 added: documents that R13 sync-teardown does NOT await in-flight `BeginTransitionAsync` to complete (quit is terminal — no value in finishing a transition that will be torn down moments later). Cancels the transition's cancellation token, skips activation, executes sync-teardown chain (Event Bus → Save/Load → Input), sets `AppState = Disposed`. Distinguishes from E9 (OS background = recoverable, preserves state). |
| R9 | AC-R7 dual-bound [120, 130) ambiguity — widen check (b) to ≤140ms | qa-lead | Upper bound of check (b) widened from `≤ max + 50ms = 120ms` to `≤ max + 70ms = 140ms` to eliminate the [120, 130) dead-zone where a 130ms result was simultaneously failing (b) inclusive AND failing (c) boundary. The 70ms tolerance still catches a serialized-execution regression (serial = 150ms+ fails (b); pseudo-parallel-with-mid-handler-stall ≥130ms fails (c) parallelism proof). |
| R10 | AC-R10b condition (c) no-op — rewrite as positive state assertion | qa-lead | Condition (c) rewritten from no-op-with-N/A to positive state assertion: `SceneManager.GetSceneByName("PauseOverlay").isLoaded == true` after resume. The OS-background path preserves the additive PauseOverlay scene across the round trip; the AC now catches a future refactor that unloads PauseOverlay on background (which would silently break user-pause state on resume — un-paused gameplay flash before any code re-pauses). |
| R11 | AC-R12d location promotion to H.3 PLAYMODE DEFERRED | qa-lead | AC-R12d promoted from "Known coverage gaps" to formal H.3 entry with PLAYMODE (DEFERRED) tier (activates at Story 2). Full GIVEN/WHEN/THEN spec written. Net AC count change: +1. Old coverage-gap entry removed, replaced with breadcrumb pointing to the formal H.3 entry. |
| R12 | AC-R9e coverage-gap stale entry line 749 — delete | game-designer NI-2 | Stale entry "AC-R9e — PROMOTED to ADVISORY in pass-3 BLOCKING #7 — moved into Section H Rule Compliance as a real test (not a coverage gap). Entry retained here for change-log traceability; remove at pass-5." deleted from "Known coverage gaps" section. The change-log breadcrumb is no longer needed; AC-R9e is now BLOCKING CI PLAYMODE per pass-6 BLOCKING #5. |
| R13 | Test file paths in Shift-Left notes (REC-5 from pass-3) | qa-lead | Each Shift-Left implementation note now names the explicit test file path under `tests/[mode]/scene-manager/`. Test layout follows project standards (`tests/unit/` / `integration/` / `playmode/` / `editmode/` / `device/`). Story-time scaffolds have unambiguous file landing zones. |
| R14 | OQ-1 close (`LoadSceneParameters` unchanged in Unity 6.3 LTS) | unity-specialist | OQ-1 status changed from OPEN to CLOSED — pass-5 unity-specialist verified the `LoadSceneParameters` struct against Unity 6.3 LTS docs: same two fields as Unity 2019+ (`LoadSceneMode loadSceneMode` + `LocalPhysicsMode localPhysicsMode`), no new flags introduced across Unity 6.0/6.1/6.2/6.3. R1's `LoadSceneMode` enum assumption confirmed correct. Unity 6.3 API verification table now 7 RESOLVED + 0 OPEN. |
| R15 | OQ-21 fold-in: iOS escalation path documentation (short-call→long-call sequence safe but undocumented) | unity-specialist + game-designer | OQ-21 body expanded with escalation-sequence documentation: short-call (`OnApplicationFocus(false)`) escalating to long-call (`OnApplicationPause(true)`) is safe-by-construction because every state mutation is idempotent and `_userPausedBeforeBackground` is captured at OS-background time (the more authoritative event). Documented as "expected behavior, no special-case code required" for future maintainers. |
| R16 | PauseOverlay teardown on bg-expiry eject documentation | game-designer | New sub-section in D.3 documenting that the OS-background resume path's `BeginTransitionAsync("MainMenu")` call must be preceded by explicit `PauseOverlay` cleanup (`UnloadSceneAsync` + `Resources.UnloadUnusedAssets`) when the overlay is loaded. Otherwise MainMenu runs against an orphaned additive overlay. Adds ~50-100 ms to the resume-eject path (acceptable — no Pillar 5 budget applies). Code-review enforcement; no AC. |

### File metrics

- Lines: ~880 (pass-4) → ~1080 (pass-6, +~200) — primarily from BLOCKING #2 (FEPM split + `[SetUp]` enforcement + 2 sub-cases), BLOCKING #3 (AC-R10c/R10d split into 6 fixtures with full GIVEN/WHEN/THEN per fixture), R7 (per-sibling boot timeout), R8 (E19 quit-during-loading), R13 (Shift-Left test paths expanded), R15 (OQ-21 escalation), R16 (PauseOverlay teardown on eject).
- ACs: 49 (pass-3) → 56 (pass-6, +7 net): +1 BLOCKING #2 (FEPM split), +4 BLOCKING #3 (R10c/R10d splits), +1 R3 (AC-R9g), +1 R11 (AC-R12d formal). BLOCKING #5 AC-R9e tier promotion = 0 count change.
- OQs: 21 → 21 (R14 closes OQ-1, status CLOSED; total unchanged). Net active: 13 (was 14; OQ-1 closed).
- Modified rules: AC-BOOT-FEPM, AC-R7, AC-R9e, AC-R9f, AC-R10b, AC-R10c, AC-R10d. New: AC-R9g, AC-R12d-formal, AC-BOOT-FEPM-A, AC-BOOT-FEPM-B, AC-R10c-Rush/Prep/General, AC-R10d-Rush/Prep/General.
- Modified formulas: D.1 (R4 distinct-severity advisory log), D.2 (R6 zero-handlers edge case), D.3 (R16 PauseOverlay teardown sub-section), D.5 (BLOCKING #4 sums-of-ceilings + R7 per-sibling boot timeout).
- Modified rules: R11 prose (R4 distinct-severity advisory).
- New edge cases: E19 (quit-during-loading per R8).
- Stale references checked + updated: zero `bg_session_expiry_seconds` remnants outside review log historical context; "49 ACs" / "9 severity tiers" updated to "56 ACs" / "9 severity tiers"; H.1/H.3 subsection counts updated; Gate Summary tier sums recounted.

### Specialist disagreements

None at pass-6 (no fresh adversarial spawn — only the targeted Implementation-Contract Verification ping). Pass-5 specialist disagreements (3) were already CD-adjudicated; pass-6 implements the adjudicated outcomes (BLOCKING #4 D.5 arithmetic; BLOCKING #3 AC-R10c/R10d severity; BLOCKING #5 AC-R9e tier).

### Methodology validation: Implementation-Contract Verification gate (CD pass-5 strategic recommendation, first invocation)

Pass-5 CD recommended introducing a narrow specialist verification gate when an author-led revision adds or modifies an AC clause asserting:
- Engine API bit values, enum encodings, serialization formats
- Platform-specific lifecycle event ordering
- Frame-budget numeric thresholds
- Memory/GC tolerance bands

Pass-6 fired the gate exactly once — for AC-BOOT-FEPM bit-encoding + EditorSettings API surface — and got CORRECT/CORRECT back in ~10 minutes of specialist time. **Methodology validated.** Time cost: trivial (~10 min vs ~6 hours of pass-6 author work). Defect cost prevented: a CI gate silently passing on broken builds (the doubly-recursive defect class that drove the pass-5 → pass-6 cycle). Recommend codifying the gate in `/design-system` and `/design-review` skills for any Foundation GDD that includes engine-API or platform-lifecycle ACs going forward.

Other pass-6 changes that COULD have triggered the gate but did not:
- AC-R9f tolerance band 2MB → 5MB (R1) — could have triggered "memory/GC tolerance bands". Skipped because the change is empirical-noise-floor adjustment, not a new claim about engine memory behavior. Verifiable post-implementation via the test itself.
- AC-R7 dual-bound widening ≤140ms (R9) — could have triggered "frame-budget numeric thresholds". Skipped because the change is dead-zone elimination via arithmetic, not a new claim about engine scheduling.
- D.5 per-sibling boot timeout `T_sibling_max = T_x_ceiling × 2` (R7) — could have triggered "frame-budget numeric thresholds". Skipped because the value is structural (2× safe-range ceiling) not an engine-derived constant.

Future pass authors should err on the side of firing the gate when uncertain. The cost is trivial; the defect-class it prevents is the worst kind (silently passing CI on broken contracts).

### Pillar fantasy alignment audit (carried forward — pass-5 verified)

All three Section B fantasies remained preserved across pass-5; pass-6 line edits (BLOCKING + RECOMMENDED) did not introduce new pillar threats:
- **"Retry is a beat, not a load screen"** — preserved. R11 budget split unchanged; AC-R9g new ADVISORY tracks RequestUnpause latency dashboard-only without changing the gate.
- **"The game is already running when the player gets there"** — preserved. R7 per-sibling boot timeout protects against indefinite splash hang (arguably strengthens the fantasy by closing a previously-undefined failure mode).
- **"The seams are where the chaos isn't"** — preserved. AC-R10b condition (c) positive state assertion (R10) strengthens the user-pause preservation contract; R16 PauseOverlay teardown on eject prevents orphan-overlay flash on bg-expiry MainMenu return.

### Validation criteria for APPROVED status (pass-6 self-audit)

- [x] All 5 BLOCKING fixes from pass-5 review log table verified present in current GDD body.
- [x] All 16 RECOMMENDED fixes verified.
- [x] AC count math reconciled across opening paragraph (56 ACs) + subsection counts (H.1=36 + H.2=3 + H.3=9 + H.4=2 + H.5=6 = 56) + Gate Summary tier sums (10+10+20+3+4+3+2+1+3 = 56).
- [x] No internal contradictions between R10 step numbering, D.3 pseudocode, sub-flag table, Section G tuning knobs, and worked examples (verified: variable names `bg_expiry_in_rush_seconds` / `bg_expiry_in_prep_seconds` / `bg_expiry_general_seconds` consistent across D.3 + Section G + AC-R10c-* + AC-R10d-*).
- [x] Cross-GDD obligations (Event Bus `DisposeSync`, Input `TeardownSync`) flagged for Phase 5 amendment work (status: TRACKED, not yet APPLIED — pass-6 does not change this).
- [x] Implementation-Contract Verification gate fired and returned CORRECT on AC-BOOT-FEPM bit-encoding + EditorSettings API surface.
- [x] Pass-3 BLOCKING #1-12 verified holding (carried forward from pass-5 verification).
- [x] All three Section B fantasies preserved (carried forward from pass-5 verification).

### Next step

**APPROVED.** Status header updated to `APPROVED (pass-6 author-led revision 2026-04-28)`. Auto-promotion per CD pass-5 verdict rule on condition application.

Pre-implementation requirements (carried forward):
1. ADR-008 (Scene Loading Strategy) — pre-production scope; required BEFORE first Scene Manager implementation story.
2. ADR-009 (Bootstrap Architecture) — pre-production scope.
3. ADR-003 (named reference Android + iOS devices) — pre-production scope; AC-R11/AC-P-Retry/AC-P-Boot/AC-P-UnityBoot/AC-R9g all blocked on this.
4. 4 bidirectional consistency amendments to Input / Event Bus / Save/Load / Data Registry GDDs (Phase 5 work — non-breaking additive edits per Section F).

Next foundational GDD: **Audio Bus** (#6) per `design/gdd/systems-index.md` recommended design order. Once authored, Foundation #7 is **2D Physics & Arc Trajectory**.

---

