# Active Session State

*Last Updated: 2026-04-28 (**Scene / App State Manager APPROVED via pass-6 author-led revision** — pass-5 APPROVE-WITH-CONDITIONS resolved by pass-6 applying all 5 BLOCKING + 16 RECOMMENDED items per pass-5 review log fix-list. **Mandatory unity-specialist Implementation-Contract Verification gate** (first invocation of CD pass-5 strategic recommendation) confirmed AC-BOOT-FEPM bit-encoding (`m_EnterPlayModeOptions: 2` → `1` for `DisableDomainReload`; flag enum: `DisableDomainReload = 1`, `DisableSceneReload = 2`, both = `3`) + `[SetUp]` enforcement + EditorSettings API surface CORRECT/CORRECT on both claims. Auto-promoted to APPROVED per CD pass-6 verdict rule. File metrics: ~880 → ~1080 lines (+~200); 49 → 56 ACs (+7 net); 21 → 21 OQs (R14 closed OQ-1, R15 expanded OQ-21). H subsection counts: H.1 36 + H.2 3 + H.3 9 + H.4 2 + H.5 6 = 56. Gate Summary tier sums: BLOCKING CI 10 + BLOCKING CI PLAYMODE 10 + PLAYMODE 20 + PLAYMODE DEFERRED 3 + PLAYMODE Integration 4 + EDITMODE 3 + DEVICE 2 + DEVICE ADVISORY 1 + ADVISORY 3 = 56. **5 of 20 MVP GDDs now fully approved** (Data Registry + Event Bus + Save/Load + Input System + Scene / App State Manager). All three Section B fantasies preserved across all 6 passes. Methodology lesson encoded: codify the Implementation-Contract Verification gate in /design-system + /design-review for Foundation GDDs with engine-API/platform-lifecycle ACs going forward.)*

## Current Phase

**Systems Design** — **5 of 20 MVP GDDs fully APPROVED**. Status: Data Registry + Event Bus + Save/Load + Input System + **Scene / App State Manager APPROVED pass-6 author-led revision** (2026-04-28). Pass-6 closed pass-5 APPROVE-WITH-CONDITIONS verdict on application of all 5 BLOCKING + 16 RECOMMENDED items per fix-list, with mandatory targeted unity-specialist Implementation-Contract Verification gate (first invocation of CD pass-5 strategic recommendation) returning CORRECT on both AC-BOOT-FEPM claims (bit-encoding + EditorSettings API surface). Next action: **`/design-system audio-bus`** to author Foundation #6 GDD (Audio Bus). After Audio Bus + 2D Physics & Arc Trajectory complete the Foundation layer, MVP Core authoring begins (Level Runtime / Level Scoring / Gopher Spawn / Tool / Critter AI / Hazard / Collection — the Pillar-4 Six). Bidirectional-consistency amendments to Input / Event Bus / Save/Load / Data Registry (Phase 5 work; non-breaking additive edits) remain pending — schedule alongside or before architecture-phase ADR authoring.

### Phase 6B aggregate findings (3 specialists, parallel spawn, 2026-04-24)

**unity-specialist** — CONCERNS. 1 BLOCKING implementation defect + 3 recommended corrections.
- **BLOCKING**: `CopyTo(ProfilerRecorderSample[], int)` overload in AC-R13f spec — signature likely non-existent in Unity 6.3 LTS `ProfilerRecorder` API. Must become `CopyTo(List<ProfilerRecorderSample>)` or `CopyTo(NativeArray<ProfilerRecorderSample>)`.
- RECOMMENDED: add `using Unity.Profiling;` namespace note to AC-R13f; soften B8 ARM "10-15% field measurement" language to "engineering estimate pending OQ-8"; correct B10 prose "Library/ + ProjectSettings/" → "ProjectSettings/ (the define persists in ProjectSettings.asset written by AssetDatabase.SaveAssets())".
- Adjudicated Phase 6A CORRECT on `SetScriptingDefineSymbols` overloads — pass-5 unity-specialist claim that `(NamedBuildTarget, string[])` doesn't exist was WRONG; both overloads exist in Unity 6.3.
- Package pins: `inputsystem@1.11.x` PASS (engine-reference supports); `ugui@2.0.x` CONCERNS (UPM package vs built-in-module ambiguity, correctly flagged UNVERIFIED).

**performance-analyst** — REJECT. 2 BLOCKING domain errors + 3 concerns.
- **BLOCKING**: AC-R13c specifies `temperature_c: float` in thermal-state triple, cites `PowerManager.ThermalStatus` — but that Android API (API 29+) returns discrete severity enum (0–6), not Celsius. Same class of domain error as pass-4's GPU→CPU figure. Fix: replace with `thermal_severity_level: int`.
- **BLOCKING**: iOS `ProcessInfo.thermalState` requires native iOS plugin (Foundation Obj-C API, not accessible from managed C#) — undisclosed Plumbing Sprint dependency not in Section F table/ADR list.
- CONCERNS: `v_lateLift` range device-DPI-dependent but presented device-independent (no DPI conversion disclosure — 0.1 pt/ms = 15.9 mm/s at 320 dpi Android); `v_lateLift` safe-range floor 0.05 not enforced by OnValidate; `0.1 ms residual` noise budget contradicts pass-3's own 0.4-0.6 ms Stopwatch empirical finding (environmental noise unchanged by instrumentation).
- Budget arithmetic (AC-P3 = 4.6 ms thermal ceiling via 4.0 × 1.15) coherent IF residual noise is really 0.1 ms — but B9 finding invalidates that assumption. Dev-vs-Release cross-tier flaky-CI risk for handlers in 0.8–1.0 ms range.

**accessibility-specialist NEW (first consultation across all 6 passes)** — CONCERNS. 3 BLOCKING/HIGH items + 3 advisory + 1 recommendation. **6 pass-1-through-5 oversights flagged.**
- **BLOCKING (citation error)**: B16 cites "48 dp per WCAG 2.1 SC 2.5.5" — SC 2.5.5 is AAA, not AA. Governing AA criterion is WCAG 2.2 SC 2.5.8 (24×24). Fix: re-cite correctly.
- **BLOCKING (self-contradiction)**: OQ-12 claims Section F phased table updated to REQUIRED-at-VS for `input.gesture.longpress.unhandled` Haptics subscription — Section F (GDD line ~153) still shows only `input.tap.main` with NO REQUIRED marker + NO longpress row. Normative section contradicts OQ claim. Fix: add second Haptics row.
- **HIGH (timing contradiction)**: AC-EC-CB3 is MVP MANUAL but OQ-2 (copy-authoring task) is VS-scoped. If AC tests at MVP, copy must exist at MVP. Fix in one direction.
- Advisory OQs to add: (a) mid-session accessibility settings access gap — production players mid-session have NO path (only `DEVELOPMENT_BUILD`-gated pause button); (b) minimum Android API floor for VibrationEffect (API 26+) unspecified — if Android min-target < 26, haptic third channel silently fails on low-end Android = SC 1.3.3 (A) violation re-opening deaf-blind gap.
- RECOMMENDATION (non-blocking): promote `hit_target_scale` to MVP as `PlayerPrefs` key (UI slider can defer to VS), since fixed 1.4× is design assumption not user accommodation. Users with tremor/CP/fine-motor may need 1.6-2.0×.
- EAA Annex I scope: mobile games are AMBIGUOUS / likely OUT for pure entertainment but IN via IAP checkout flow (Annex I Section IV e-commerce). GDD's blanket "EAA applies to mobile apps" overstates — applies to IAP-specific surfaces.
- EN 301 549 → WCAG 2.1 AA correct (harmonized); WCAG 2.2 AA stronger but not yet mandated for 2026.

## Input System GDD — REVISED (pass-5 Phase 6A), Phase 6B pending

**File**: `design/gdd/input-system.md` (~900 lines post Phase 6A; 71 ACs: 10 BLOCKING CI + 51 PLAYMODE + 1 EDITMODE + 4 DEVICE + 5 ADVISORY — body-verified count updated pass-5 Phase 6A per BLOCKING #3 reconciliation).
**Status header**: REVISED (pass-5 Phase 6A) 2026-04-24.
**Cross-document edits**: `game-concept.md` Pillar 5 text unchanged from pass-3 sync ("100 ms at 60 fps target / 133 ms at 30 fps floor"). `design/gdd/systems-index.md` Input row updated to REVISED (pass-5 Phase 6A); Progress Tracker summary updated.
**Review log**: `design/gdd/reviews/input-system-review-log.md` — Phase 6A revision entry appended with per-BLOCKING-item resolution (B1–B18), user design decisions captured, Phase 6B + 6C spawn instructions.
**CD-GDD-ALIGN trajectory**: pass-1 APPROVED → pass-2 CONCERNS → pass-3 CONCERNS → pass-4 CONCERNS → pass-5 REJECT → **pass-5 Phase 6A REVISED** (2026-04-24) pending Phase 6B/6C restoration.

### User design decisions captured this session (multi-tab widget, all recommended options)

- **B1 — `input.contact.began` HUD subscriber promotion**: **Rush-phase REQUIRED** (not all-scenes-REQUIRED, not keep-advisory). Added AC-R3g-required BLOCKING CI; HUD REQUIRED on Rush-phase-capable scenes (scenes spawning `LevelRuntime`).
- **B16 — Accessibility activation path**: **Persistent "Accessibility" button on title screen** (single-tap, 48 dp, 3:1 contrast per WCAG 2.1 SC 1.4.11/2.5.5). Rapid 5-tap path REPEALED per pass-5 BLOCKING #16. Main Menu GDD owns button visual spec.
- **B18 — Haptic third channel**: **Document gap in OQ-12, haptic at VS REQUIRED** (cert-gating). Haptics elevated from opt-in VS to REQUIRED VS on `input.gesture.longpress.unhandled` third-channel tick; closes deaf-blind gap at VS.
- **B7 — `v_lateLift` default**: **0.1 pt/ms + OQ-11 empirical calibration**. Above 0.05 floor of specialist-cited 0.05–0.8 pt/ms motor-impaired range; below upper end. Safe range updated `[0.01, 0.1]` → `[0.05, 0.5]`.

### 18 BLOCKING items — Phase 6A resolution summary

All 18 resolved author-led with citations; detail in review log 2026-04-24 Phase 6A entry.

**Pillar 5 contract cluster (B1–B3)**: R3.e REQUIRED on Rush-phase scenes (B1, new AC-R3g-required); R3.c language "guaranteed non-visual fail-safe" → "publish-path fail-safe" (B2); Gate Summary arithmetic reconciled to 71 (B3).

**Formula / budget structural cluster (B4–B7)**: D.3 explicit top-to-bottom priority + `InProgress otherwise` catch-all (B4); H.4 cross-tier amnesty closed via B11 thermal-state discriminator (B5); OQ-13 Rush-peak T1 aggregate empirical validation (B6); `v_lateLift` 0.02 → 0.1 pt/ms + OQ-11 (B7).

**Unity 6.3 reality-alignment cluster (B8–B11)**: AC-P3 ceiling 5.0 → 4.6 ms CPU-domain re-derivation (B8, Cortex-A53/A55 10–15% citation); ProfilerMarker noise-floor correction + `ProfilerCategory.Scripts` + `capacity: 1000` (B9, Unity 6.2 docs cited); CI injection rewrite with separate `unity-builder@v4` step + read-modify-write `SetScriptingDefineSymbols(NamedBuildTarget, string)` pattern (B10); AC-R13c thermal-state annotation + 3-precondition downgrade gate (B11).

**CI operability / AC hygiene cluster (B12–B15)**: CI Gate Operability Status column added (B12); AC-R3d-required phantom-type cleanup + AC-R3d-audio-xref new ADVISORY + AC-INT3/INT5 divergence disclosure (B13); AC-D3-margin-validate 6-step mechanism spec (B14, `SerializedObject` + `AssetDatabase.SaveAssets` + `LogAssert.Expect`); AC-INV5 tautology demoted to ADVISORY dashboard rollup (B15).

**Accessibility / regulatory cluster (B16–B18)**: Rapid 5-tap REPEALED + persistent title-screen button (B16); OQ-6 VS-sprint-gating + accessibility-specialist REQUIRED reviewer + EAA/UK-Equality-Act citations (B17); OQ-12 deaf-blind gap + Haptics REQUIRED at VS cert-gating (B18).

### Citations used in Phase 6A

- [Unity Scripting API `PlayerSettings.SetScriptingDefineSymbols`](https://docs.unity3d.com/ScriptReference/PlayerSettings.SetScriptingDefineSymbols.html)
- [Unity Manual Custom scripting symbols (6.3)](https://docs.unity3d.com/6000.3/Documentation/Manual/custom-scripting-symbols.html)
- [Unity Scripting API `ProfilerRecorder.StartNew`](https://docs.unity3d.com/6000.2/Documentation/ScriptReference/Unity.Profiling.ProfilerRecorder.StartNew.html)
- [Unity Manual Read frame timing data with ProfilerRecorder](https://docs.unity3d.com/6000.2/Documentation/Manual/frame-timing-manager-record-timing-data.html)
- [Unity Manual Visualizing profiler counters](https://docs.unity3d.com/6000.2/Documentation/Manual/profiler-creating-custom-counters.html)
- [game-ci/unity-test-runner docs](https://game.ci/docs/github/test-runner/)
- [ARM Cortex-A55 sustained-performance blog](https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/arm-cortex-a55-efficient-performance-from-edge-to-cloud)
- [WCAG 2.1 SC 1.4.11 Non-text Contrast (W3C)](https://www.w3.org/WAI/WCAG21/Understanding/non-text-contrast.html)
- [EAA June 28 2025 effective date (accessible-eu-centre.ec.europa.eu)](https://accessible-eu-centre.ec.europa.eu/content-corner/news/eaa-comes-effect-june-2025-are-you-ready-2025-01-31_en)
- [UK Gov accessibility + Equality Act guidance](https://www.gov.uk/guidance/meet-the-requirements-of-equality-and-accessibility-regulations)
- Touchscreen motor-impairment kinematic-theory research (CHI Conference, Wobbrock lab) — for `v_lateLift` 0.05–0.8 pt/ms range

## Next session: Phase 6B (parallel specialist re-spawn) + Phase 6C (CD synthesis)

### Phase 6B — Focused specialist re-spawn (parallel, <1 hour)

Spawn THREE specialists via PARALLEL `Task` calls (single message, multiple tool uses):

1. **`unity-specialist`** — verify B8 + B9 + B10 against Unity 6.3 LTS reference docs (`docs/engine-reference/unity/modules/input.md`, `physics.md`, `ui.md`). Also verify Unity 6.3 LTS `com.unity.inputsystem@1.11.x` + `com.unity.ugui@2.0.x` package pins.
2. **`performance-analyst`** — verify B7 + B11 thermal + CI-infra fixes + B9 noise-floor arithmetic correction.
3. **`accessibility-specialist`** (**NEW — never previously consulted on this GDD**) — review B16 + B17 + B18 + OQ-6 + R9 + R17 in-menu 600 ms warning copy + R19 `hit_target_scale` 4th accommodation dimension.

**Do NOT re-spawn** game-designer/systems-designer/qa-lead/ux-designer unless Phase 6B output surfaces design-layer changes.

### Phase 6C — CD synthesis + pass-6 verdict

After Phase 6B returns, spawn `creative-director` with all three specialist findings + the pass-5 Phase 6A revision diff (appended to review log). CD renders pass-6 verdict. Target: **APPROVED** + CD-GDD-ALIGN **restored from REJECT to APPROVED**.

## CD-GDD-ALIGN trajectory

pass-1 APPROVED → pass-2 CONCERNS → pass-3 CONCERNS → pass-4 CONCERNS → pass-5 REJECT → **pass-5 Phase 6A REVISED** → Phase 6B + 6C pending. Document is architecturally sound; Phase 6A addressed the implementation-contract layer REJECT was on; Phase 6B + 6C verify the citations land and restore APPROVED.

<!-- STATUS -->
Epic: Systems Design
Feature: Scene / App State Manager GDD — APPROVED pass-6 author-led revision (2026-04-28)
Task: **Pass-6 COMPLETE — Scene Manager APPROVED**. All 5 BLOCKING + 16 RECOMMENDED items applied per pass-5 fix-list. Mandatory Implementation-Contract Verification gate (first invocation) confirmed AC-BOOT-FEPM bit-encoding `m_EnterPlayModeOptions: 1` + `[SetUp]` API surface CORRECT. 56 ACs (was 49); 19 edge cases (was 18); D.5 sums-of-ceilings arithmetic corrected; 6 tier-specific R10c/R10d fixtures replace stale `bg_session_expiry_seconds`. **Next: `/design-system audio-bus`** (Foundation #6). Then 2D Physics & Arc Trajectory (Foundation #7). Bidirectional amendments to Input / Event Bus / Save/Load / Data Registry remain pending — schedule alongside architecture-phase ADR authoring.
<!-- /STATUS -->

## Completed This Session (2026-04-28 pass-6 author-led revision — Scene / App State Manager APPROVED)

- [x] Read pass-5 review log fix-list (5 BLOCKING + 16 RECOMMENDED) + current GDD body + systems-index.md Scene Manager row + active.md Phase context.
- [x] **Spawned unity-specialist Implementation-Contract Verification gate ping** (background) for narrow AC-BOOT-FEPM bit-encoding + EditorSettings API surface verification — first invocation of CD pass-5 strategic recommendation. **Returned CORRECT on both claims** in ~10 min.
- [x] **Applied 5 BLOCKING fixes**:
  - **#1 + #2 (combined)**: AC-BOOT-FEPM bit-encoding `m_EnterPlayModeOptions: 2` → `1` (DisableDomainReload only) per Unity 6 flag enum; added `[SetUp]` enforcement assertion (`Assert.IsTrue(EditorSettings.enterPlayModeOptionsEnabled) + bitwise mask`); split into Sub-case A (DisableDomainReload only, harder static-field-persistence case) and Sub-case B (both reloads disabled, plus scene state persistence).
  - **#3**: AC-R10c + AC-R10d each split into 3 tier-specific fixtures (Rush / Prep / General) replacing stale `bg_session_expiry_seconds`. Each fixture is a separate `[Test]` method against the applicable tier threshold variable.
  - **#4**: D.5 sums-of-ceilings `1000+500+1500+800 = 3800ms` → `2000+500+1500+800 = 4800ms` (using safe-range MAXIMA per Section G); re-derived target invariant `sum(T_x_ceiling) ≤ T_boot_target × 1.6 = 4800 ms`.
  - **#5**: AC-R9e tier ADVISORY → BLOCKING CI PLAYMODE (per failure-severity asymmetry — same `Time.timeScale = 0f` permanent freeze as AC-R9a).
- [x] **Applied 16 RECOMMENDED fixes**: AC-R9f tolerance band 2MB → 5MB (R1); AC-R9f baseline timing post-warm-up-unpause anchor (R2); new AC-R9g RequestUnpause Pillar 5 latency ADVISORY (R3); R11 cold-cache distinct-severity advisory log when `T_sm_overrun >= T_slack_budget` (R4); `bg_expiry_general_seconds` 60s revisit deferred to live-ops (R5); D.2 zero-handlers documentation (R6); D.5 per-sibling boot timeout `T_sibling_max = T_x_ceiling × 2` for R2 hang protection (R7); E19 quit-during-loading edge case (R8); AC-R7 dual-bound widening 120ms → 140ms eliminates dead-zone (R9); AC-R10b condition (c) rewritten as positive PauseOverlay.isLoaded assertion (R10); AC-R12d formal H.3 entry promotion + coverage-gap entry replaced with breadcrumb (R11); AC-R9e stale coverage-gap entry deleted (R12); test file paths added to all Shift-Left Implementation Notes (R13); OQ-1 closed (`LoadSceneParameters` unchanged in Unity 6.3 LTS) (R14); OQ-21 expanded with iOS short-call→long-call escalation path documentation (R15); D.3 PauseOverlay teardown on bg-expiry eject sub-section (R16).
- [x] **Status header updated** to APPROVED (pass-6 author-led revision 2026-04-28); AC count opening paragraph updated (49 → 56); H.1 / H.3 subsection counts updated (30 → 36, 8 → 9); Gate Summary tier sums recounted; Open Questions table totals updated (R14 closed OQ-1).
- [x] **Appended pass-6 review log entry** to `design/gdd/reviews/scene-app-state-manager-review-log.md` (per-BLOCKING-item resolution table + per-RECOMMENDED-item resolution table + Implementation-Contract Verification gate result + methodology validation + pillar fantasy carry-forward + validation criteria self-audit).
- [x] **Updated `design/gdd/systems-index.md`**: Last Updated line; Scene Manager row status `APPROVE-WITH-CONDITIONS (pass-5 /design-review)` → **`APPROVED (pass-6 author-led revision)`**; Progress Tracker counts (Design docs reviewed, Design docs approved 4 → 5, MVP systems designed 4 fully approved + 1 conditional → 5 fully approved); Review Gate History new row for /design-review (Scene / App State Manager) pass-6 APPROVED.
- [x] **Updated `production/session-state/active.md`**: Last Updated line; Current Phase; STATUS block; Completed This Session (this entry); Next Tasks updated.

## Completed earlier sessions (2026-04-24 /design-system scene-app-state-manager)

- [x] **Phase 2 — Context gathering + Feasibility Brief**: Read game-concept.md, systems-index.md, entities.yaml (9 registry entries), all 4 approved Foundation GDDs (Data Registry, Event Bus, Save/Load, Input System) with targeted Grep for scene-related references. Flagged 7 UNVERIFIED Unity 6.3 scene-API claims (no `docs/engine-reference/unity/modules/scene.md` exists; LLM cutoff ~Unity 2022 LTS).
- [x] **Phase 3 — Skeleton created** at `design/gdd/scene-app-state-manager.md`.
- [x] **Section A (Summary + Overview)**: Framing widget (3 tabs: Framing / ADR ref / Fantasy) captured — all 3 recommended picks (data/infrastructure framing, no ADR ref, no direct fantasy). Surfaced six-state MVP enumeration (Boot/MainMenu/BiomeMap/Level/Shop/Paused) + Prep→Rush-is-NOT-a-Scene-Manager-event assumption.
- [x] **Section B (Player Fantasy)**: creative-director consulted for infrastructure fantasy framing. Three-framing draft approved (Retry-is-a-beat lead + cold-start support + interruption closer), with "What this fantasy excludes" anti-pillar block per CD tonal guidance.
- [x] **Section C (Detailed Design — 14 Core Rules + States and Transitions + Interactions)**: 4 design decisions captured via multi-tab widget (async-only LoadSceneAsync, dedicated Boot scene, additive pause overlay, Prep→Rush not SM event). 3 parallel specialists consulted (systems-designer primary + gameplay-programmer + engine-programmer supporting per Foundation/Infrastructure routing). 4 synthesis decisions captured (3-channel event taxonomy, typed IPreUnloadHook + Event Bus trigger, separate OS-background forced-pause path, 1100/600/300 ms retry budget).
- [x] **Section D (Formulas — D.1–D.5)**: Retry budget decomposition; Pre-unload aggregate window (max-not-sum); OS-background session-expiry threshold; Activation handshake timeout; Bootstrap cumulative init duration. Each with variable table + ranges + worked examples + edge cases.
- [x] **Section E (Edge Cases — E1–E18)**: Synthesized from systems-designer G1–G6 gaps + gameplay-programmer 5 pitfalls + engine-programmer 5 gotchas. Narrative `**If X**: Y` format.
- [x] **Section F (Dependencies)**: 13 cross-system rows (Foundation siblings + undesigned downstream); 6 platform-dependency rows; 3 pending ADRs (ADR-003 shared + ADR-008 + ADR-009 new); 4 bidirectional-consistency amendments flagged for Input / Event Bus / Save/Load / Data Registry.
- [x] **Section G (Tuning Knobs)**: 13 knobs across 6 categories (timing budgets, OS-lifecycle threshold, bootstrap advisory, pre-unload ordering, cross-GDD sortingOrder budget, configurable constants).
- [x] **Section H (Acceptance Criteria — 36 ACs)**: qa-lead consulted per skill protocol; ACs proposed across 5 gate tiers with Shift-Left Implementation Notes + Gate Summary table with Operability Status column (matches Input System pass-5 B12 pattern). 6 known coverage gaps flagged for story-time addition.
- [x] **Optional sections**: Visual/Audio brief ("no direct ownership — see downstream"), UI Requirements brief + UX Flag note ("N/A — no user-facing screens; downstream UI GDDs each trigger /ux-design"), Open Questions (15 OQs: 7 Unity 6.3 API verification → Phase 6 unity-specialist; 5 cross-GDD provisional; 2 deferred design; 1 project-level ADR-003).
- [x] **Phase 5a — Self-Check PASS**: 651 lines, all 12 sections (8 required + 4 optional/header) have real content, zero `[To be designed]` placeholders.
- [x] **Phase 5a-bis — CD-GDD-ALIGN**: creative-director verdict APPROVE-WITH-CONDITIONS (3 conditions). Conditions 1 (header pillar-claim distinction) + 2 (AC-INT-FF added to Section H.5 bringing total to 37 ACs) applied inline. Condition 3 (ADR-008 gating dependency) logged in Next Tasks. Auto-promoted to APPROVED per pass-6 CD verdict rule. CD review rated Player Fantasy section EXEMPLARY; found no pillar violations across all 14 Rules + 18 Edge Cases + 5 Formulas + 37 ACs.
- [x] **Phase 5b — Registry update**: 3 new channel constants registered (`gameplay.scene.will_unload`, `gameplay.scene.load_ready`, `gameplay.app.state_changed`); 3 existing entries `referenced_by` updated (`t2_latency_max`, `DeviceDisplayMetrics`, `gameplay_channel_regex`).
- [x] **Phase 5d — Systems index + session state updated**: Scene Manager row status `Not Started` → **Approved (pass-1 post-CD-GDD-ALIGN)** with full detail; Progress Tracker counts 4/20 → 5/20 MVP systems; Review Gate History new row for CD-GDD-ALIGN (Scene / App State Manager) pass-1 APPROVED.

## Completed earlier sessions (2026-04-24 Phase 6D — Input System)

- [x] Read pass-6 CD verdict (review log lines 1303–1429) + Phase 6A methodology entry (lines 697–812) + Phase 6B aggregate findings + current GDD structural map (R9, AC-R13c, AC-R13f, Section C subscribers, Section F Dependencies, Section G Tuning Knobs, OQ table).
- [x] **Executed Phase 6D targeted author revision** applying all 3 BLOCKING items with specialist-scripted exact fix language:
  - **BLOCKING #1 (unity-specialist)**: AC-R13f `CopyTo(ProfilerRecorderSample[], int)` → `CopyTo(List<ProfilerRecorderSample>)` (MVP default, managed — acceptable for 1000-event post-run analysis) with `CopyTo(NativeArray<ProfilerRecorderSample>)` zero-alloc alternative documented. `using Unity.Profiling;` namespace note added at AC-R13f + R13.d prose block (line 121 area).
  - **BLOCKING #2 (performance-analyst)**: AC-R13c `temperature_c: float` → `thermal_severity_level: int` with Android `[0, 6]` per `THERMAL_STATUS_*` (API 29+; sub-API-29 sentinel `-1`) + iOS `[0, 3]` per `ProcessInfo.ThermalState` + `is_throttled = (level >= 2)` derivation on both platforms. Section F new subsection `Native Platform Plugins` with 4-row table (iOS `ProcessInfo.thermalState`, iOS `CoreHaptics`, Android `PowerManager.ThermalStatus` JNI, Android `VibrationEffect` + `Vibrator.vibrate()` JNI) all scoped to Plumbing Sprint Months 4–5. ADR-007 trigger logged.
  - **BLOCKING #3a (accessibility-specialist)**: R9 primary activation path WCAG re-citation "48 × 48 dp per WCAG 2.2 SC 2.5.8 AA (24 × 24 px minimum); exceeds WCAG 2.1 SC 2.5.5 AAA" with W3C citations; OQ-6 body updated consistent.
  - **BLOCKING #3b (accessibility-specialist)**: Section C subscribers table second Haptics row `input.gesture.longpress.unhandled` added (closes OQ-12 self-contradiction).
  - **BLOCKING #3c (accessibility-specialist)**: OQ-2 promoted VS → MVP (direction: promote OQ-2, not demote AC-EC-CB3) preserving pass-5 ux-designer R17 MVP-copy decision.
- [x] **Applied 2 advisory OQs**: OQ-14 (mid-session Accessibility settings access gap — known accessibility gap until VS Main Menu / Settings / Pause overlay expansion); OQ-15 (Android `VibrationEffect` API 26+ floor with two acceptable resolutions — state min-target ≥ 26 OR Haptics GDD specifies `Vibrator.vibrate()` fallback).
- [x] **Applied 6 recommended items inline**: AC-P3 "10–15% CPU field measurement" → "engineering estimate pending OQ-8 on-device calibration" (honesty-of-provenance); B10 prose "Library/ + ProjectSettings/" → "ProjectSettings/ (define persists in `ProjectSettings.asset` written by `AssetDatabase.SaveAssets()`)"; Section G `v_lateLift` DPI conversion disclosed (0.1 pt/ms ≈ 15.9 mm/s at 320 dpi; full 160/320/480 dpi coverage); R13.d `0.1 ms residual` noise labeled "design target pending OQ-7 empirical validation"; Section G `v_lateLift` `OnValidate` floor `>= 0.05` enforcement added; `hit_target_scale` promoted to MVP `PlayerPrefs` key `MM.A11y.HitTargetScale` (R9 struct 3 → 4 fields + Section G Accessibility / Platform Knobs new row).
- [x] **Updated GDD status header** to APPROVED (pass-6 post-Phase 6D) 2026-04-24 with Phase 6D resolution summary + CD-GDD-ALIGN auto-restore note.
- [x] **Appended Phase 6D revision entry** to `design/gdd/reviews/input-system-review-log.md` (parallel structure to Phase 6A entry at lines 697–812) documenting each BLOCKING + advisory + recommended item addressed + status change + next-step routing.
- [x] **Updated `design/gdd/systems-index.md`**: Input row status `APPROVED-WITH-CONDITIONS (pass-6)` → **`APPROVED (pass-6)`** reflected in Last Updated line + Design docs reviewed + Design docs approved + MVP systems designed; Review Gate History new row for CD-GDD-ALIGN (Input) pass-6 post-Phase 6D APPROVED.
- [x] **Updated `production/session-state/active.md`**: Last Updated line, Current Phase, STATUS block, Completed This Session (this entry), Next Tasks.

## Completed earlier sessions (2026-04-24 Phase 6C)

- [x] Read Phase 6B review log entries (unity-specialist lines 816–1043, performance-analyst lines 1046–1194, accessibility-specialist NEW lines 1195–1290) + systems-index.md Input row + active.md Phase 6B scope.
- [x] **Phase 6C CD synthesis rendered**: pass-6 verdict **APPROVED-WITH-CONDITIONS**, CD-GDD-ALIGN restored from REJECT to APPROVED-WITH-CONDITIONS. Both auto-restore to full APPROVED on Phase 6D merge — no pass-7 required.
- [x] Specialist disagreements adjudicated (3): `SetScriptingDefineSymbols(string[])` overload existence (Phase 6A correct, pass-5 specialist wrong); performance-analyst REJECT vs unity-specialist CONCERNS severity (same class at document-architecture level); accessibility WCAG AA-vs-AAA (BLOCKING upheld as regulatory-defensibility-grade).
- [x] Defect-class recurrence assessment: ~75% attenuation vs pass-5 (1 cross-domain API-shape recurrence vs 4 pass-5 convergent findings). Phase 6A citation-heavy methodology verified directionally correct; Phase 6D continues same methodology with added habit of verifying API return-type/shape on cross-domain claims.
- [x] Appended CD pass-6 verdict section to `design/gdd/reviews/input-system-review-log.md` (parallel structure to passes 1–5: verdict + CD-GDD-ALIGN disposition + 3 BLOCKING Phase 6D items + recommended items + strengths preserved + specialist disagreements adjudicated + defect-class recurrence + scope signal + next step + recommendation to user).

## Completed earlier sessions (2026-04-24 Phase 6B)

- [x] Read `production/session-state/active.md` Phase 6A entry + Phase 6A review log (lines 697–812) + targeted GDD sections for specialist scoping.
- [x] **Spawned 3 Phase 6B specialists in parallel via single-message multi-Task call** (unity-specialist + performance-analyst + accessibility-specialist NEW) with adversarial prompts + explicit instructions to prove citations wrong.
- [x] Unity-specialist completed + self-wrote findings to review log lines 816–1043. Verdict: CONCERNS (1 BLOCKING CopyTo + 3 recommended).
- [x] Performance-analyst completed inline; findings recorded to review log lines 1046–1194. Verdict: REJECT (2 BLOCKING domain errors + 3 concerns).
- [x] Accessibility-specialist first spawn truncated mid-read; re-spawned fresh with tightened targeted prompt. Second run completed; findings recorded to review log lines 1195–1290. Verdict: CONCERNS (3 BLOCKING/HIGH + 3 advisory + 1 recommendation; 6 pass-1-through-5 oversights flagged).
- [x] Aggregate Phase 6B: **3 BLOCKING (unity CopyTo, perf thermal_c field + iOS plugin, a11y WCAG citation + Section F Haptics + AC-EC-CB3 timing) + 11 concerns/gaps**.
- [x] Session-state updated with Phase 6B summary + Phase 6D task list + Phase 6C pending.

## Completed earlier sessions

- [x] Read `production/session-state/active.md` pass-5 entry + full pass-5 review log + current `design/gdd/input-system.md` (788 lines pre-Phase 6A).
- [x] Gathered Unity 6.3 citations via parallel WebSearch/WebFetch: `SetScriptingDefineSymbols` overloads, `ProfilerRecorder.StartNew` + `ProfilerCategory.Scripts`, game-ci unity-test-runner + unity-builder, Cortex-A53/A55 CPU thermal scaling, WCAG 2.1 SC 1.4.11, EAA June 2025, UK Equality Act Section 29, motor-impaired lift velocity research.
- [x] Captured 4 user design decisions via AskUserQuestion multi-tab widget (B1 Rush-phase REQUIRED, B16 persistent A11y button, B18 OQ-12 + haptic at VS, B7 `v_lateLift = 0.1 pt/ms`).
- [x] Revised `design/gdd/input-system.md` addressing all 18 BLOCKING items across status header, Summary, Quick reference, R3.c, R3.e, Downstream tables (Section C + F), R9, R13.b/d, D.1, D.3 (rule block + variable table + worked examples), Section G tuning knobs (v_lateLift + CI injection + PhaseTransitionSuppressDuration), Section H ACs (AC-R3d-required rewrite + new AC-R3g-required BLOCKING CI + new AC-R3d-audio-xref ADVISORY + AC-R13c thermal-state + AC-P3 CPU-domain + AC-D3-margin-validate mechanism + AC-INV5 demotion + AC-INT3/INT5 divergence disclosure + AC-EC-CB3 MVP copy promotion), Gate Summary (reconciliation + CI Operability Status column), Shift-Left Implementation Notes, Section G Cross-Reference table, Open Questions table (+OQ-11 + OQ-12 + OQ-13 + OQ-6 updated).
- [x] Updated `design/gdd/systems-index.md` Input row to REVISED (pass-5 Phase 6A); updated Progress Tracker summary.
- [x] Appended Phase 6A revision entry to `design/gdd/reviews/input-system-review-log.md` with per-BLOCKING-item resolution + user design decisions + Phase 6B/6C spawn instructions.

## Next Tasks (for next session)

- [x] **`/design-review design/gdd/scene-app-state-manager.md` pass-6 author-led revision** (2026-04-28, this session): **COMPLETE — APPROVED**. Status moved from APPROVE-WITH-CONDITIONS (pass-5) → APPROVED (pass-6) via fix-list application + Implementation-Contract Verification gate. 5 / 20 MVP GDDs now fully approved.
- [ ] **`/design-system audio-bus`** — Foundation #6 GDD (Audio Bus). Per recommended design order in systems-index.md.
- [ ] **`/design-system 2d-physics-arc-trajectory`** — Foundation #7 GDD. After Audio Bus.
- [ ] **4 bidirectional-consistency amendments** to existing Foundation GDDs (non-breaking additive edits, do NOT require full re-review): Input System (add Scene Manager row + `InputDispatcher.TeardownSync()` per pass-3 BLOCKING #5); Event Bus (add Scene Manager as publisher of `gameplay.scene.*` + `gameplay.app.state_changed` channels + `IPreUnloadHook` priority 10 + `EventBus.DisposeSync()`); Save/Load (add Scene Manager as `FlushSync()` caller + `IPreUnloadHook` priority 20); Data Registry (add Scene Manager as unload-sequence driver, informational).
- [x] **`/design-system scene-app-state-manager`** (2026-04-24): **COMPLETE**. Foundation #5 GDD authored via full skill run (Phase 2 context + Phase 2e feasibility brief + Phase 3 skeleton + Sections A/B/C/D/E/F/G/H + 3 optional + Phase 5 finalization). 3 parallel specialists consulted for Section C (systems-designer + gameplay-programmer + engine-programmer); 1 for Section B (creative-director fantasy framing); 1 for Section H (qa-lead ACs); 1 for Phase 5a-bis (creative-director CD-GDD-ALIGN). CD verdict APPROVE-WITH-CONDITIONS → APPROVED on condition application. 7 UNVERIFIED Unity 6.3 scene-API claims flagged for Phase 6 `unity-specialist` adversarial review.
- [x] **`/consistency-check design/gdd/scene-app-state-manager.md`** (2026-04-24, this session): **COMPLETE — Verdict PASS**. 6 Scene-Manager-relevant registry entries verified (`t2_latency_max`=100ms 3-doc agreement with Input/Event Bus, `DeviceDisplayMetrics` consistent, `gameplay_channel_regex` compliant for 3 new channels, `gameplay.scene.will_unload` + `gameplay.scene.load_ready` + `gameplay.app.state_changed` internally consistent across R5/R8/R9/R10/AC-R5/AC-R8/AC-R8a/E8/Section F). AC-INT-FF cross-reference to Input AC-P1 ("Feedback Floor End-to-End (Pillar 5)") verified valid. ⚠️ Pending bidirectional amendments to Event Bus / Save/Load / Input / Data Registry confirmed as known-deferred (not a conflict per skill ontology — registry's forward-looking `referenced_by` claims are flagged in registry comments + Scene Manager Section F lines 419-425). No 🔴 CONFLICTS → no `docs/consistency-failures.md` reflexion-log append. No registry corrections needed.
- [ ] **`/design-review design/gdd/scene-app-state-manager.md` in a fresh session** — full adversarial specialist pass (game-designer, systems-designer, qa-lead, engine-programmer, unity-specialist, performance-analyst). First system in project to introduce Fast Enter Play Mode Reset() pattern (R2.a + AC-BOOT-FEPM) — unity-specialist review especially load-bearing for the 7 UNVERIFIED Unity 6.3 claims.
- [ ] **4 bidirectional-consistency amendments** to existing Foundation GDDs (non-breaking additive edits, do NOT require full re-review):
  - **Input System** — add Scene Manager row to Section F Dependencies
  - **Event Bus** — add Scene Manager as publisher of `gameplay.scene.*` + `gameplay.app.state_changed` channels in Section C Interactions + Section F Dependencies; add `IPreUnloadHook` priority 10 registration rule to R-rules
  - **Save/Load** — add Scene Manager as `FlushSync()` caller on OS background + quit in Section F; add `IPreUnloadHook` priority 20 registration
  - **Data Registry** — add Scene Manager as unload-sequence driver (informational only)
- [ ] **ADR-008 (Scene Loading Strategy)** — per CD Condition 3, this ADR must be authored BEFORE the first Scene Manager implementation story (async-only LoadSceneAsync + activation-handshake contract + Addressables deferral rationale). Pre-production scope.
- [ ] **ADR-009 (Bootstrap Architecture)** — dedicated Boot scene + BootLoader single-construction authority + Foundation Reset() discipline for Fast Enter Play Mode. Pre-production scope.
- [ ] **`/architecture-decision ADR-007`** (prior trigger from Input System) — stub CoreHaptics / `ProcessInfo.thermalState` / `PowerManager.ThermalStatus` / `VibrationEffect` native-plugin bridge. Trigger logged; full ADR authored at pre-production (Plumbing Sprint scoping task).
- [ ] Continue authoring MVP Foundation GDDs: Audio Bus (#6), 2D Physics & Arc Trajectory (#7).
- [ ] Then MVP Core: Level Runtime, Level Scoring, Gopher Spawn, Tool System, Critter AI, Hazard System, Collection System.
- [ ] Then MVP Feature: Currency & Economy, Progression.
- [ ] Then MVP Presentation: HUD (must absorb T2 dual-channel REQUIRED subscriber + contact.began REQUIRED on Rush-phase scenes per pass-5 B1 + haptic third channel at VS per B18), Biome Map, Main Menu/Settings/Pause (must absorb persistent title-screen Accessibility button per pass-5 B16 + VS-sprint-gating Accessibility Settings UI per B17), Modal/Dialog+Level End.
- [ ] Run `/review-all-gdds` when all MVP GDDs are complete.
- [ ] Run `/gate-check systems-design` before advancing to Architecture phase.

## Active Files

- `design/gdd/game-concept.md` — concept doc (Pillar 5 dual-threshold text stable since pass-3)
- `design/art/art-bible.md` — art bible (approved)
- `design/gdd/systems-index.md` — systems decomposition (Scene Manager row updated to Approved pass-1 this session; Review Gate History row added)
- `design/gdd/data-registry.md` — Foundation #1 GDD (APPROVED pass 4)
- `design/gdd/event-bus.md` — Foundation #2 GDD (APPROVED + pass-4 cross-coordination amendment + pass-4 Input-coordination Interactions additions — APPROVED preserved; pending bidirectional-consistency amendment to absorb Scene Manager as publisher of 3 new channels + IPreUnloadHook priority 10)
- `design/gdd/save-load.md` — Foundation #3 GDD (APPROVED pass 4; pending bidirectional-consistency amendment to absorb Scene Manager as FlushSync caller + IPreUnloadHook priority 20)
- `design/gdd/input-system.md` — Foundation #4 GDD (APPROVED pass-6 post-Phase 6D 2026-04-24, 71 body-verified ACs; pending bidirectional-consistency amendment to absorb Scene Manager as sceneLoaded-hook provider + IPreUnloadHook priority 30)
- `design/gdd/scene-app-state-manager.md` — **Foundation #5 GDD NEW this session** (APPROVED pass-1 post-CD-GDD-ALIGN 2026-04-24, 651 lines, 37 ACs, 14 Rules + 5 Formulas + 18 Edge Cases + 15 OQs)
- `design/gdd/reviews/input-system-review-log.md` — Phase 6A/6B/6C/6D entries (prior sessions)
- `design/registry/entities.yaml` — **UPDATED this session**: 3 new constants (`gameplay.scene.will_unload`, `gameplay.scene.load_ready`, `gameplay.app.state_changed`); 3 referenced_by updates (`t2_latency_max`, `DeviceDisplayMetrics`, `gameplay_channel_regex`)
- `.claude/docs/technical-preferences.md` — engine + platform config
- `docs/engine-reference/unity/VERSION.md` — Unity 6.3 LTS pin (2026-02-13) + knowledge gap warning
- `docs/engine-reference/unity/modules/input.md` — Input System reference
- `docs/engine-reference/unity/modules/physics.md` — Physics reference (3D only; Physics2D/Box2D status for B10 RECOMMENDED)
- `docs/engine-reference/unity/modules/ui.md` — UI reference (UI Toolkit + UGUI)
- `CLAUDE.md` — references Unity 6.3 LTS reference docs

## Key Decisions (recorded this session)

- **Phase 6A methodology**: author-led revision with inline Unity 6.3 LTS doc citations + accessibility-standards citations. Not author-alone (pass-4's methodology that produced pass-5's defect class) — citations are inline and verifiable against source URLs, so Phase 6B specialist verification is checking the citations land, not the claims themselves.
- **4 design decisions captured via multi-tab widget**: Rush-phase REQUIRED for contact.began (B1), persistent title-screen button (B16), OQ-12 deaf-blind gap + Haptics REQUIRED at VS (B18), `v_lateLift = 0.1 pt/ms` (B7). All 4 matched recommended options.
- **Scope discipline**: Haptics at VS (not MVP) per B18 user decision — adds REQUIRED cert-gating at VS rather than scope-bloating MVP to include Haptics GDD authoring. Deaf-blind gap documented as known regression at MVP, closes at VS.
- **AC-P3 ceiling 4.6 ms (not 5.0 ms)**: CPU-domain re-derivation per Cortex-A53/A55 10–15% sustained-load scaling citation. Pass-4's 5.0 ms was 0.4 ms too loose — a real Pillar 5 false-negative under thermal stress.
- **AC-R13c correlated-tail downgrade gated by 3 preconditions** (not 1): Release-tier failure pattern + thermal-state correlation ≥ 80% + ADR-003 device precondition. Simultaneous non-thermal 0.3 ms regressions in both PreFire and T1Chain no longer get merge amnesty.
- **AC-INV5 demoted from BLOCKING CI to ADVISORY**: pass-5 B15 tautology finding sustained — "summary" AC passes iff its three components pass; no independent test logic. Demotion opens the BLOCKING CI slot for AC-R3g-required (B1).

## Pre-Production Action Items (unchanged from prior session)

1. Write Leaderboard Model ADR (banked-count, non-deterministic physics OK)
2. Write URP Mobile Settings ADR (no HDR, no MSAA, disabled 2D shadows, Forward renderer)
3. Write Save Schema Versioning + Reference Device ADR-003 (JSON format, migration rules, named Android + iOS reference devices — Save/Load + Input System both block DEVICE ACs on this)
4. Write Monetization Boundary ADR (IAP catalog, ad placement rules, no pay-to-win)
5. Build "Stress Scene" in Week 2 of prototype (5 holes × 40 Rigidbody2D on target Android device) — includes Input pipeline budget validation per Input D.1 / AC-P1/P2/P3 + Event Bus AC-F4-aggregate per-frame T1 aggregate validation + **OQ-13 Rush-peak T1 aggregate empirical measurement** (new pass-5 B6 trigger)
6. Schedule Plumbing Sprint at Months 4-5 (IAP + ads + analytics + cloud save)

## Open Questions (project-level; separate from per-GDD OQs)

- First biome choice for v1.0 second slot: Meadow + Orchard vs. Meadow + Mines
- Energy system model: lives-regen vs. ad-gated retries vs. cooldown-free (deferred to soft-launch A/B test)
- Leaderboard cadence: daily/weekly/monthly/season (deferred to post-launch)
