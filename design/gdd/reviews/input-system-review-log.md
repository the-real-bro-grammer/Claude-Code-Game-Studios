# Input System GDD — Review Log

Revision history for `design/gdd/input-system.md`. Each entry records an adversarial `/design-review` pass: verdict, scope, specialists consulted, and a brief synthesis. Prior CD-GDD-ALIGN gates (pillar alignment only) are not logged here — see `design/gdd/input-system.md` status header for those.

---

## Review — 2026-04-22 — Verdict: NEEDS REVISION

Scope signal: L
Specialists: game-designer, systems-designer, unity-specialist, performance-analyst, ux-designer, qa-lead, creative-director (senior synthesis)
Blocking items: 6 | Recommended: 11 | Nice-to-have: 7
Prior verdict resolved: First review

Summary: The GDD is architecturally strong — D.3 piecewise classifier, R10.d suppression priority stack, R11 pause-gesture cancellation, R4 UI-first hit resolution, R8 zero-feedback architectural boundary, and registry co-ownership between CI severity knobs and ACs are all preserved. However, the document is not implementable as written: subscriber arithmetic in Section G contradicts the T1Chain sub-budget (8×2=16ms vs 8ms ceiling); R13 and AC-P1 double-count the 16.67ms frame budget as two different clocks; R3.c LongPress silent discard violates Pillar 5 for the accessibility population; AC-P3 uses `mean` where every other latency AC uses p99; R4's `EventSystem.IsPointerOverGameObject(touchId)` overload is unverified against Unity 6.3's new Input System + UI Toolkit; R8's `UnityEngine.InputSystem.XInput.Haptics.*` grep is gamepad-only and never matches on mobile targets; max-accessibility tuning `(d_slop=20, d_early=19)` produces a classifier dead zone at `d=15`. CD synthesis adjudicated a unique game-designer finding (R3.a tap-on-`ended` as Pillar 5 violation) as RECOMMENDED not BLOCKING — resolves via opt-in `input.contact.began` T2 event rather than moving tap-firing to `began` (which would break tap-vs-drag disambiguation). Prior CD-GDD-ALIGN APPROVE from 2026-04-22 remains valid for pillar alignment; this review is orthogonal. Estimated revision effort per CD: 2-4 hours of author time.

### Blocking items (merge-blocking — must resolve before implementation)

1. [systems-designer + performance-analyst] Subscriber arithmetic contradiction in Section G — pick one of: `SubscriberCountMax = 4`, `PerHandlerCostThreshold_ms = 1`, or rewrite bound against `BudgetCeiling_ms = 16.67`.
2. [performance-analyst] Frame-budget double-count between R13.c (main-thread slice) and AC-P1 (end-to-end 100ms) — restate Input's main-thread ceiling as ≤4ms sub-budget; declare AC-P1's 100ms as a separate clock.
3. [game-designer + ux-designer] R3.c LongPress silent discard violates Pillar 5 — publish `input.gesture.longpress.unhandled` T2 event OR route to `input.tap.main` with `wasLongPress=true` flag.
4. [performance-analyst + qa-lead] AC-P3 uses `mean` not p99; 30s window insufficient for Mali-G52 thermal throttling — rewrite to p99 over ≥90s sustained load.
5. [unity-specialist] Two unverified Unity API claims: (a) R4's `EventSystem.IsPointerOverGameObject(touchId)` overload + new-Input-System finger ID mapping; (b) R8's `UnityEngine.InputSystem.XInput.Haptics.*` grep is gamepad-only. Pin `com.unity.inputsystem` + `com.unity.ugui` versions in Section F; rewrite R8 grep against `Handheld.Vibrate` / `CoreHaptics` / Android JNI paths.
6. [systems-designer] Max-accessibility tuning dead zone + undefined classifier statefulness — add margin invariant `d_early ≤ d_slop - 3` OR document Path B degradation as accepted trade-off; explicitly state classifier is stateful (no Drag → Tap/LongPress re-classification).

### Notable recommended items (important, not blocking)

- AC-P1 test harness unspecified (render frame submission timestamp ownership).
- AC-EC-HT2 release-build parity (telemetry event in release, not assertion-only).
- AC-INT3/INT4/INT5 stub contracts missing (depend on un-authored Modal/Dialog + Level Runtime GDDs).
- AC-INV4 `EventBus.DebugSubscriberCount` dependency unconfirmed.
- R13 latency ACs lack CI determinism (pinned runner, deterministic clock, flakiness threshold).
- `input.contact.began` T2 advisory event for Rush-phase pending-state visuals (creative-director adjudication of game-designer's R3.a finding).
- R9 MVP accessibility practically absent (PlayerPrefs not player-accessible on mobile pre-VS) — bias defaults toward accommodation OR ship debug menu.
- R6 shared `HitBoundsCalculator` utility to prevent consumer drift (6+ consumers implementing 1.4× inflation independently).
- `PhaseTransitionSuppressDuration` timer must use wall-clock accumulator (not `StartCoroutine`, which fails at `Time.timeScale == 0`).
- `.asmdef` boundary for `src/Input/` (compile-error R1 enforcement, not just CI grep).
- Continue-button render-gating coordinated with Input state transition (cross-GDD handoff to HUD).

### Strengths to preserve (unchanged in revision)

- D.3 piecewise classifier with explicit `<` / `≤` boundary semantics — QA-test-derivable.
- R10.d suppression priority stack (Loading > Modal > PhaseTransition) + EC-SM4 re-entrancy snapshot rule.
- R11 pause-gesture cancellation (Unity synthetic-`ended` hazard — rarely handled in mobile games).
- R4 UI-first hit resolution + sortingOrder scene validator (dev-build-only).
- R8 zero-feedback-from-Input architectural boundary.
- Section H → Section G CI severity co-ownership table.
- R6.a `DeviceDisplayMetrics` registry entry as cross-system contract for pixel/pt/dp conversion.

### Specialist disagreement (adjudicated)

game-designer alone flagged R3.a tap-on-`ended` as BLOCKING Pillar 5 violation (no on-contact feedback for 80-120ms lifts). Creative-director adjudication: concern is real but framing too strict. Resolution — expose `input.contact.began` as T2 advisory event for opt-in pending-state consumers; do NOT move tap-firing to `began`; document Pillar 5's 100ms floor as "contact → meaningful response." Downgraded to RECOMMENDED.

---

## Revision — 2026-04-22 — Pass-1 (addresses all 6 blockers from 2026-04-22 review)

Scope signal: L (same as review).
Author: Robert Michael Watson. Specialists re-consulted: unity-specialist (Unity 6.3 API verification for R4 + R8 — BLOCKING #5). No re-spawn of the full 7-specialist adversarial panel in this pass; that comes on `/design-review` re-run in the next fresh session.
Blocking items resolved: 6/6. Recommended items addressed in this pass: partial (listed below). Nice-to-have items: deferred.

**Revision log (per-blocker resolution):**

1. **BLOCKING #1 + #2 (coupled) — subscriber arithmetic + frame-budget double-count** → RESOLVED via coupled 4 ms sub-budget rewrite.
   - R13 rewritten: Input's main-thread sub-budget is 4 ms Release / 5 ms Dev (inside the 16.67 ms frame). PreFire ≤ 2 ms, T1Chain ≤ 2 ms. SubscriberCountMax = 4 (was 8). PerHandlerCostThreshold = 0.5 ms (was 2 ms). Coupling invariant `4 × 0.5 = 2.0 ≤ T1Chain = 2.0` ✓ (binding).
   - D.1 `BudgetCeiling_ms` updated to {4.0, 5.0}; variable ranges, worked example, rationale, and validation thresholds rewritten to the new sub-budget framing. D.1 now explicitly distinguishes three independent clocks: R13.c's 4 ms Input main-thread sub-budget; the registry `t1_latency_max = 16.67 ms` Event-Bus per-channel T1 window; AC-P1's 100 ms end-to-end Pillar 5 Feedback Floor.
   - Section G Pipeline Capacity Knobs table updated (SubscriberCountMax=4 Locked, PerHandlerCostThreshold=0.5ms range [0.25, 1.0]) and the Interaction coupling paragraph restated against the new 2.0 ms T1Chain bound.
   - AC-R13a/b/c/d/e, AC-D1-release, AC-D1-dev, and AC-P1 all updated to the new thresholds; AC-P1 explicitly declares the 100 ms as a separate clock from R13.c and from the registry `t1_latency_max`.
   - User decision recorded: chose "4 ms sub-budget, max = 4 subs" from three options (vs. max=8 @ 0.25 ms per-handler, or 16.67 ms full-frame ceiling). Consequence: current 8-subscriber roster on `input.tap.main` must use scene/phase-scoped subscription discipline so peak-concurrent count stays ≤ 4 — flagged in Section G prose and for downstream-consumer GDDs to honour.

2. **BLOCKING #3 — R3.c LongPress silent discard (Pillar 5 violation)** → RESOLVED via new T2 event.
   - R3.c rewritten: LongPress now publishes `input.gesture.longpress.unhandled` on a T2 Feedback channel with payload `LongPressUnhandledPayload(Vector2 ScreenPosition, float DurationMs, GameObject? Target)`. No MVP consumer is required; HUD / Accessibility Feedback is the opt-in subscriber.
   - D.3 GestureClassification Path C worked example and `GestureClass` enum table row updated to reflect T2 publish.
   - Section C Interactions downstream table: new row for HUD opt-in subscription on `input.gesture.longpress.unhandled` at T2.
   - AC-R3d (BLOCKING) and AC-D3-longpress rewritten to assert the T2 publish; AC-INV2 updated to include the new channel in the "no events in Suppressed state" invariant.
   - Event Bus GDD (`design/gdd/event-bus.md`): new row added to Interactions table under Input System emitter for the T2 channel.
   - User decision recorded: chose "new T2 event channel" over "flag on input.tap.main" (preserves tap-vs-drag semantic cleanliness).
   - Creative-director's R3.a adjudication (`input.contact.began` T2 advisory for Rush-phase pending-state visuals) is **NOT** bundled into this pass — remains RECOMMENDED; a future revision will add it when Rush-phase visual consumers require it.

3. **BLOCKING #4 — AC-P3 mean → p99, 30 s → ≥ 90 s** → RESOLVED.
   - AC-P3 rewritten: 99th-percentile `TotalLatency_ms < 3.2 ms` (80% of 4.0 ms R13.c sub-budget) over ≥ 90 seconds sustained load (Mali-G52 thermal-throttling window). Explicit statement that mean-based / ≤ 30 s measurements do NOT satisfy this AC. Subscriber count in the scenario reduced from 8 to 4 to match the new R13.d cap.

4. **BLOCKING #5 — Unity API unverified claims** → RESOLVED via unity-specialist consultation; two UNVERIFIED engineering-time tasks remain flagged in the GDD.
   - R4 Pass 1 rewritten: drop `EventSystem.IsPointerOverGameObject(int touchId)`; use `results.Count > 0` after `EventSystem.current.RaycastAll` as the authoritative UI-hit signal. The prior overload does not map `EnhancedTouch.Touch.touchId` to `InputSystemUIInputModule`'s internal pointer IDs in Unity 6.3 (module synthesises its own IDs). UI Toolkit `PanelRaycaster` participation in `EventSystem.RaycastAll` is flagged UNVERIFIED — engineering PlayMode test required on first implementation.
   - R8 haptics grep rewritten: replaced `UnityEngine.InputSystem.XInput.Haptics.*` (gamepad-only, no-op on mobile) with `Handheld\.Vibrate|AndroidJavaObject|AndroidJavaClass|CoreHaptics|VibrationEffect|\.Haptics\b` — over-inclusive mobile-haptics pattern covering Unity built-in, iOS CoreHaptics, and Android JNI paths.
   - Section F: new subsection "Unity Package Pins (R4 / R8 API surface)" pinning `com.unity.inputsystem @ 1.11.x`, `com.unity.ugui @ 2.0.x`, `com.unity.ui @ 2.0.x` (all UNVERIFIED — confirm against actual 6.3 LTS manifest on project creation).
   - AC-R4-ui-first updated to assert the `results.Count > 0` decision path and that `IsPointerOverGameObject(int)` is NOT called.
   - AC-R8 and Section G R8 CI-severity row updated with the new grep pattern.
   - Three engineering-time validation tasks remain as UNVERIFIED in the GDD (package manifest version check, UI Toolkit PanelRaycaster participation PlayMode test, GC.Alloc profile of first RaycastAll call) — to be resolved during implementation sprint.

5. **BLOCKING #6 — classifier dead zone + statefulness undefined** → RESOLVED.
   - D.3 Boundary semantics extended with explicit "Statefulness" subsection: classifier is stateful per-gesture; once `input.drag.start` publishes, the gesture is locked into Drag until `ended`/`canceled` — cannot re-classify as Tap or LongPress under finger-jitter or floating-point noise.
   - D.3 extended with "Classifier-knob margin invariant" subsection specifying the dead-zone failure at `(d_slop=20, d_early=19, d=15, t=160ms, ended=false)` and the `d_early ≤ d_slop − 3` OnValidate fix.
   - D.3 variable table: `d_early` range tightened from `6 – d_slop` to `6 – (d_slop − 3)`.
   - Section G Coupling invariant paragraph and Interaction "Coupled — d_slop and d_early" paragraph rewritten with the 3-unit margin.
   - Two new ACs added to H.2 Formula Correctness: AC-D3-stateful (PLAYMODE — Drag lock across post-lock sample permutations) and AC-D3-margin (PLAYMODE — dead-zone avoidance + OnValidate rejection of margin-violating configs). Total ACs 61 → 63; PLAYMODE 47 → 49.

**Recommended items addressed in this pass (partial):**
- AC-P1 test harness ownership noted inline (render-frame-submission timestamp is the AC-P1 harness's responsibility, not InputDispatcher's) — harness spec remains a RECOMMENDED follow-up.

**Recommended items NOT yet addressed (deferred to future revisions or implementation):**
- AC-EC-HT2 release-build parity (telemetry event in release vs dev-only assertion).
- AC-INT3/INT4/INT5 stub contracts for un-authored Modal/Dialog + Level Runtime GDDs.
- AC-INV4 `EventBus.DebugSubscriberCount` API confirmation.
- R13 latency ACs CI determinism (pinned runner, deterministic clock, flakiness threshold).
- `input.contact.began` T2 advisory event (R3.a CD adjudication).
- R9 MVP accessibility debug menu.
- Shared `HitBoundsCalculator` utility.
- `PhaseTransitionSuppressDuration` wall-clock accumulator vs coroutine.
- `.asmdef` boundary for `src/Input/`.
- HUD Continue-button render-gating cross-GDD handoff.

**Next step:** `/design-review design/gdd/input-system.md` in a fresh session. Specialist panel may surface new findings on the delta (4ms sub-budget framing, T2 longpress channel, the margin invariant) and may flag anything this revision pass introduced — that is expected and the reason the re-review is run in a clean context.

Verdict pending re-review: **REVISED (pass-1), awaiting adversarial re-validation.**

---

## Review — 2026-04-22 — Pass-2 — Verdict: NEEDS REVISION

Scope signal: S-to-M
Specialists: game-designer, systems-designer, performance-analyst, unity-specialist, qa-lead, ux-designer, creative-director (senior synthesis)
Blocking items: 5 (CD-adjudicated from 14 specialist-flagged) | Recommended: 12 | Nice-to-have: 7
Prior verdict resolved: **Partial** — 2 truly resolved, 2 paper-resolved, 2 new blockers introduced

**CD-GDD-ALIGN: reversed from APPROVED to CONCERNS.** Pass-1 approval predicated on Pillar 5 being upheld across all input paths; pass-2 reveals two independent Pillar-5 compliance gaps (T2 LongPress opt-in has no REQUIRED subscriber; AC-P1 100ms floor unholdable at 30fps device floor). Not a rejection — contract is recoverable via explicit declarations. Pillars 1-4 remain aligned. Restore to APPROVED once author (a) names a REQUIRED T2 subscriber or removes channel from MVP, and (b) declares Pillar 5's device-floor contract explicitly in AC-P1.

Summary: The pass-1 revision made genuine architectural progress — the 4ms sub-budget rewrite with three clocks distinguished, the classifier Statefulness + margin invariant, and the Unity 6.3 API verification (R4 RaycastAll + R8 haptics grep) are all well-executed. However, two prior blockers were paper-resolved rather than truly resolved: BLOCKING #3 LongPress Pillar 5 was "fixed" via a T2 channel with no required subscriber (structurally equivalent to silent discard); BLOCKING #1 subscriber arithmetic was "fixed" via SubscriberCountMax=4 while the Downstream table still lists 8 subscribers with no scene-scoped discipline audit. Two new blockers were introduced: the zero-slack `4 × 0.5 = 2.0` binding invariant combined with Stopwatch ~0.2ms noise makes AC-R13b structurally flaky (5-8% false-positive rate per run) and per-handler threshold is not CI-asserted at all; Gate Summary arithmetic is wrong on every row (BLOCKING CI = 7 not 8, PLAYMODE = 50 not 49, total = 65 not 63). Additionally, AC-P1 Pillar 5 100ms end-to-end cannot hold at 30fps floor (render-submission alone = 33.33ms; worst-case end-to-end 100-133ms) — author must declare Pillar 5 as 60fps-target contract with 133ms 30fps relaxation, or as device-agnostic 100ms requiring 60fps shipping-cert. Cross-GDD gap: Input's 4ms sub-budget is uncoordinated with Event Bus's per-frame T1 aggregate across all channels — requires Event Bus GDD re-opening for reconciliation.

### Blocking items (CD-adjudicated; merge-blocking — must resolve before implementation)

1. **[creative-director + game-designer + ux-designer] T2 LongPress has no REQUIRED MVP subscriber** — "opt-in, no consumer required" is structurally equivalent to the silent discard it replaced. Fix: name a REQUIRED MVP subscriber (HUD/Accessibility Feedback with minimal contact-point feedback tick) with an AC verifying ≥1 registered T2 subscriber at scene start, OR remove the channel from MVP and open OQ for post-MVP.

2. **[creative-director + ux-designer] AC-P1 Pillar 5 device-floor ambiguity** — 100ms floor unholdable at 30fps floor. CD recommends declaring Pillar 5 as a 60fps-target contract with 133ms relaxation at 30fps (matches project's "60fps target / 30fps floor" budget language). Either way, AC-P1 must cite which.

3. **[creative-director + systems-designer + performance-analyst] Per-handler 0.5ms threshold is CI-structurally flaky + not asserted** — Stopwatch ~0.2ms noise vs 0.5ms threshold = 2.5:1 SNR produces 5-8% false positives per run; additionally, no AC asserts per-handler cost, only chain sum. CD adjudication: RELAX architecture, don't tier-split. Raise per-handler Release threshold to 0.8ms (SNR 4:1), keep 2.0ms chain ceiling; Dev tier 1.2ms. Add per-handler CI assertion.

4. **[performance-analyst] 4ms sub-budget uncoordinated with per-frame T1 aggregate across all Event Bus channels** — Cross-GDD reconciliation: Event Bus must define a per-frame T1 aggregate ceiling across all channels, not just per-channel window. Input's 4ms must declare itself a slice of that aggregate. Re-opens Event Bus GDD (Foundation #2).

5. **[qa-lead] Gate Summary arithmetic wrong + test-infrastructure gaps** — Recounted: BLOCKING CI = 7 (not 8), PLAYMODE = 50 (not 49), ADVISORY = 4 identifiers (counted as 2), total = 65 (not 63). Plus: AC-R13c must specify 99p-of-sum not sum-of-99p; AC-R3d publish counter fixture must be defined (or `EventBus.DebugSubscriberCount` API confirmed); AC-D3-margin must split PlayMode+EditMode into two ACs (Unity Test Runner cannot run both under one gate).

### Notable recommended items (important, not blocking)

- [performance-analyst] **90s thermal window → ≥120s** (Mali-G52 onset ~60s; stated 90-300s rationale is wrong)
- [game-designer] **AC-P3 cold-start only** — add warm-session window (270s across 3 consecutive levels)
- [performance-analyst] **PreFire 2ms risk under throttle** — AC-R13a must assert on thermally-stabilized device
- [game-designer + ux-designer] **Drag-lock retraction cancel pattern undocumented** — cross-system note for Tool System handoff needed
- [game-designer] **`input.contact.began` Rush-phase 300ms silence** — slow-tap path still 3× Pillar 5 floor; CD's prior adjudication deferred; revisit
- [unity-specialist] **PanelRaycaster authoring requirement** — must be stated constraint in R4, not deferred engineering task. UIDocument phantom-VisualElement edge case unhandled
- [systems-designer] **Dev/Release CI profile selection mechanism unspecified** — Release gate could be satisfied on Dev build
- [ux-designer] **R9 accessibility MVP UI** — CD downgraded from BLOCKING to RECOMMENDED (v1.0 cert risk, not MVP blocker); log as OQ with owner + v1.0-gating ADR trigger
- [performance-analyst] **AC-R13e only at scene-load** — dynamic post-modal-push subscriber registration not gated
- [qa-lead] **AC-D3-stateful "all permutations"** — enumerate 4 equivalence classes
- [qa-lead] **AC-P3 CI operability** — requires dedicated soak-test job with ≥120s timeout; not per-commit gate
- [qa-lead] **AC-INV5 redundant summary** — remove; replace with CI dashboard step

### Strengths preserved (truly resolved from pass-1)

- Unity 6.3 API verification: R4 RaycastAll change, R8 mobile haptics grep (verified against project reference docs `docs/engine-reference/unity/modules/input.md` and `ui.md`)
- Package version pins (`com.unity.inputsystem @ 1.11.x`, `com.unity.ugui @ 2.0.x`, `com.unity.ui @ 2.0.x`) consistent with project reference baseline
- D.3 Classifier Statefulness rule (Drag lock; prevents re-classification under jitter) — architecturally sound
- D.3 Classifier-knob margin invariant `d_early ≤ d_slop − 3` — correctly cures dead-zone InProgress-forever failure
- Three-clocks distinction in D.1 (4 ms Input main-thread / 16.67 ms Event Bus per-channel / 100 ms end-to-end Pillar 5) — improvement over pass-1 conflation

### Specialist disagreements (adjudicated by creative-director)

1. **T2 LongPress severity** — game-designer + ux-designer: BLOCKING; author pass-1 position: RESOLVED. **CD upheld BLOCKING** (paper resolution; requires REQUIRED subscriber contract).
2. **R9 accessibility cert risk** — ux-designer: BLOCKING; creative-director: RECOMMENDED. **CD downgraded to RECOMMENDED** (v1.0 cert concern, not MVP-gating; must be logged as OQ with v1.0 ADR trigger).
3. **Per-handler flakiness remedy** — tier-split vs architectural relaxation. **CD: architectural relaxation** (0.5→0.8ms per-handler Release threshold). SNR problem is physics, not policy.
4. **Gate Summary severity** — drafting error vs structural signal. **CD: BLOCKING** (S-scope fix; reviewer cannot approve a doc whose summary contradicts its body).

### Prior blocker resolution status

| Prior Blocker | Status |
|---|---|
| #1 Subscriber arithmetic | PAPER RESOLVED (cap-vs-list inconsistency; no enforceable scene-scope discipline) |
| #2 Frame-budget double-count | TRULY RESOLVED (three clocks distinguished; 4ms sub-budget framing clean) |
| #3 LongPress Pillar 5 violation | PAPER RESOLVED (T2 channel lacks REQUIRED subscriber) |
| #4 AC-P3 mean/30s | PARTIALLY RESOLVED (rewrote to 99p/90s, but 90s wrong for Mali-G52 onset) |
| #5 Unity API unverified | TRULY RESOLVED (R4 + R8 + package pins verified against project reference docs) |
| #6 Classifier statefulness + margin | TRULY RESOLVED (margin fix + Statefulness rule both architecturally sound) |

Net: 2 truly resolved, 2 paper-resolved, 1 partially, 1 truly resolved (tied to #1). 2 NEW BLOCKERS introduced: R13.b flakiness + Gate Summary arithmetic.

### Scope Signal: S-to-M
Per-handler threshold re-tune + T2 subscriber wiring + Pillar 5 30fps declaration + Gate Summary recount + cross-GDD T1 reconciliation note. ~1-2 day turnaround if author is focused; no architectural rewrites required. Event Bus T1 coordination is the only item that may spill into a cross-GDD decision.

**Next step:** Revise pass-2 in a fresh session after addressing 5 BLOCKING items. Blocker #4 (Event Bus coordination) may require `/consistency-check` or a targeted Event Bus GDD amendment before Input pass-2 can land.

Verdict: **NEEDS REVISION (pass-2).**

---

## Revision — 2026-04-22 — Pass-2 (addresses all 5 blockers from 2026-04-22 pass-2 review)

Scope signal: S-to-M (same as pass-2 review).
Author: Robert Michael Watson. No specialist re-spawn in this pass — all five blockers had CD-adjudicated or empirically-determined resolutions; the pass-3 adversarial panel runs on `/design-review` in the next fresh session.
Blocking items resolved: 5/5. Recommended items addressed in this pass: 1 (AC-P3 90s→120s thermal window + soak-test CI operability note).

**Revision log (per-blocker resolution):**

1. **BLOCKING #1 — T2 LongPress has no REQUIRED MVP subscriber (paper-resolved in pass-1)** → RESOLVED via HUD as REQUIRED MVP subscriber.
   - R3.c rewritten: HUD is now the REQUIRED MVP subscriber rendering a ~200 ms dim/fade tick at `ScreenPosition`. Opt-in framing repealed.
   - Section C Interactions downstream table: new row for HUD T2 subscription with "REQUIRED MVP — Pillar 5 fail-safe" flag. Prior opt-in row removed.
   - Dependencies > Downstream (hard — MVP) HUD row rewritten to mention T2 subscription alongside T1 channels.
   - Overview quick-reference rewritten to replace "opt-in consumers" with "HUD REQUIRED subscriber per R3.c + AC-R3d-required".
   - New AC-R3d-required (BLOCKING CI): asserts ≥ 1 T2 subscriber registered at scene start via `EventBus.DebugSubscriberCount(channel)`; failing scene start fails CI with named message.
   - Event Bus GDD Interactions row updated: subscriber column changed from "HUD / Accessibility Feedback (opt-in)" to "HUD (REQUIRED MVP — Pillar 5 fail-safe per input-system.md R3.c + AC-R3d-required)".
   - User decision recorded: chose "Name HUD as REQUIRED MVP subscriber" over removing channel from MVP or routing through Accessibility Service at MVP. Rationale: minimum scope addition that honours Pillar 5 for the 600 ms accessibility ceiling population without waiting for VS.

2. **BLOCKING #2 — AC-P1 Pillar 5 device-floor ambiguity** → RESOLVED via 60 fps-target / 30 fps-floor dual-threshold contract.
   - AC-P1 rewritten: end-to-end wall-clock ≤ 100 ms at 60 fps target OR ≤ 133 ms at 30 fps floor; linear interpolation between. Frame-rate class determined by `Application.targetFrameRate` + rolling 30-frame `Time.deltaTime` mean in the harness. Input's 4 ms main-thread sub-budget (R13.c) unchanged; the 33 ms relaxation reflects render-submission at the floor.
   - Overview "Implements Pillar" line, R13.c paragraph, D.1 `BudgetCeiling_ms` variable description, AC-R13c, and Section G CI Severity Knobs row updated to reference "100 ms / 133 ms" or "60 fps target / 30 fps floor" where they previously cited a bare 100 ms.
   - User decision recorded: chose "60 fps target with 133 ms relaxation at 30 fps" over device-agnostic-100-ms-with-60fps-cert or two-separate-ACs. Rationale: matches project budget language ("60fps target / 30fps floor" in technical-preferences.md); accommodates Mali-G52 thermal throttling without waiving Pillar 5 or narrowing the hardware target matrix.

3. **BLOCKING #3 — Per-handler 0.5 ms threshold structurally flaky + not CI-asserted** → RESOLVED via CD-adjudicated architectural relaxation + new AC-R13f.
   - R13.d rewritten: per-handler threshold raised from 0.5 ms → 0.8 ms Release / 1.2 ms Dev (4:1 SNR on Stopwatch mobile noise, from 2.5:1). Binding multiplicative coupling invariant (`N × per_handler ≤ chain`) repealed explicitly; per-handler and chain are independent caps. `4 × 0.8 = 3.2 > 2.0` is the expected geometry.
   - Section G Pipeline Capacity Knobs table: old `PerHandlerCostThreshold_ms | 0.5 ms | [0.25, 1.0]` row replaced by two rows (Release 0.8 ms range [0.5, 1.0] + Dev 1.2 ms range [0.8, 1.5]).
   - Section G Interaction Between Tuning Knobs "Coupled — SubscriberCountMax × PerHandler" paragraph rewritten as "Independent caps" with explicit repeal of the binding-invariant framing.
   - New AC-R13f (BLOCKING CI Release + PLAYMODE Dev): measures each subscriber's 99p handler cost individually; fails CI with named subscriber if > 0.8 ms Release / > 1.2 ms Dev.
   - AC-P3 subscriber-count scenario updated to cite 0.8 ms ceiling (was 0.5 ms); AC-P3 window extended 90 s → 120 s per recommended-list item (Mali-G52 onset ~60 s) + CI operability note (dedicated soak-test job, not per-commit).
   - Formula Candidates Moved to Tuning Knobs line updated to match new threshold values.

4. **BLOCKING #4 — 4 ms sub-budget uncoordinated with per-frame T1 aggregate across Event Bus channels** → RESOLVED via cross-GDD amendment.
   - Event Bus GDD D.1 amended (pass-4 cross-coordination amendment 2026-04-22): new locked constant `T1_per_frame_aggregate_ms = 8 ms` (per-frame sum across all T1 channels; distinct from `T1_latency_max = 16.67 ms` per-channel). Rationale: 16.67 ms frame − 8 ms aggregate T1 ≥ 8.67 ms for render/physics/gameplay. Input's 4 ms is 50 % of 8 ms aggregate; remaining ≤ 4 ms shared among Collection / Tool / Hazard / Critter AI.
   - Event Bus GDD new AC-F4-aggregate (BLOCKING CI PlayMode): asserts per-frame sum across all T1 channels ≤ 8 ms at 99p over 1000-frame run; failures attribute cost per channel.
   - Event Bus GDD Gate Summary recounted: 36 → 37 ACs (added AC-F4-aggregate to BLOCKING CI row; 17 BLOCKING CI now).
   - Event Bus GDD status header amended: "APPROVED (with pass-4 cross-coordination amendment 2026-04-22)" — amendment is additive, not revisionary; preserves prior APPROVED.
   - Input GDD R13.b paragraph extended with Cross-GDD aggregate coordination block tying its 4 ms claim to Event Bus's 8 ms aggregate.
   - User decision recorded: chose "Amend Event Bus GDD with per-frame T1 aggregate, cross-ref in Input" over inline-note-only or full Event Bus pass-4 re-review. Rationale: smallest surface that makes Input's 4 ms claim load-bearing; amendment is additive so Event Bus APPROVED status is preserved.

5. **BLOCKING #5 — Gate Summary arithmetic wrong + test-infrastructure gaps** → RESOLVED via body-verified recount + targeted AC fixes.
   - Gate Summary table rewritten: BLOCKING CI 8 → 9 (pre-revision recount yields 7; +2 from AC-R13f + AC-R3d-required); PLAYMODE 49 → 50 (pre-revision recount yields 50; AC-D3-margin split contributes net +0 to PLAYMODE since the EditMode half migrates to new EDITMODE row); new EDITMODE row = 1 (AC-D3-margin-validate); DEVICE = 4 unchanged; ADVISORY 2 → 4 (counts identifiers not "CI-participating gates"). Total 63 → 68.
   - Acceptance Criteria intro paragraph count text updated 63 → 68.
   - Shift-Left Implementation Notes bullet updated 8 BLOCKING → 9 BLOCKING + 47 PLAYMODE → 50 PLAYMODE + 1 EDITMODE.
   - Cross-Reference to Section G table gained two new rows (AC-R13f; AC-R3d-required).
   - AC-R13c rewritten with "99th-percentile of the per-frame-sum `PreFire_ms + T1Chain_ms`" (not sum of separate 99ps); implementer note on the distinction added inline.
   - AC-R3d rewritten with explicit publish-counter fixture using `EventBus.SubscribeAutoManaged` to a `LongPressPublishCounter` helper (not reliance on the debug-build `EventBus.DebugSubscriberCount` API; that API remains the AC-R3d-required fixture and stays flagged UNVERIFIED).
   - AC-D3-margin split into: AC-D3-margin (PLAYMODE — classifier dead-zone avoidance synthetic samples) + AC-D3-margin-validate (EDITMODE — `OnValidate` rejection + `AssetDatabase.SaveAssets` + `Debug.LogError` assertion). Rationale: Unity Test Runner cannot host PlayMode + EditMode under one gate; splitting makes each independently runnable + attributable.
   - AC-EC-OS3 and AC-EC-CB3 explicitly tagged MANUAL in the ADVISORY row listing; the reconciliation note that collapsed them to "count 2" is repealed.

**Recommended items addressed in this pass (partial):**
- AC-P3 thermal window 90 s → 120 s + soak-test CI operability note (recommended-list item from pass-2).

**Recommended items NOT addressed (deferred):**
- AC-P3 warm-session 270 s window across 3 consecutive levels (game-designer pass-2 finding).
- PreFire 2 ms risk under thermal throttling (assert on thermally-stabilised device — performance-analyst pass-2 finding).
- Drag-lock retraction cancel pattern cross-system handoff to Tool System (game-designer + ux-designer pass-2 finding).
- `input.contact.began` Rush-phase 300 ms silence (game-designer pass-2 adjudication revisit).
- PanelRaycaster authoring requirement — should be stated constraint in R4, not deferred engineering task (unity-specialist pass-2 finding).
- UIDocument phantom-VisualElement edge case (unity-specialist pass-2 finding).
- Dev/Release CI profile selection mechanism (systems-designer pass-2 finding).
- R9 accessibility MVP UI OQ with v1.0 ADR trigger (ux-designer pass-2 → CD downgraded to RECOMMENDED).
- AC-R13e dynamic post-modal-push subscriber registration (performance-analyst pass-2 finding).
- AC-D3-stateful "all permutations" 4 equivalence class enumeration (qa-lead pass-2 finding).
- AC-INV5 redundant summary remove + replace with CI dashboard step (qa-lead pass-2 finding).

**Next step:** `/design-review design/gdd/input-system.md` in a fresh session as pass-3 re-review. Specialist panel may surface new findings on the delta (Pillar 5 dual-threshold contract, per-handler independent-caps framing, AC-D3-margin split, cross-GDD T1 aggregate). Verdict target: APPROVED; CD-GDD-ALIGN expected to restore to APPROVED on pillar coverage. If NEEDS REVISION (pass-3), create revision pass-3.

Verdict pending re-review: **REVISED (pass-2), awaiting adversarial re-validation.**

---

## Review — 2026-04-22 — Pass-3 — Verdict: NEEDS REVISION

Scope signal: M
Specialists: game-designer, systems-designer, performance-analyst, unity-specialist, qa-lead, ux-designer, creative-director (senior synthesis)
Blocking items: 8 (CD-adjudicated from 12 specialist-flagged) | Recommended: ~20 (deduplicated across specialists) | Nice-to-have: ~8
Prior verdict resolved: **Partial** — pass-2 blockers #4 (T1 aggregate) + #5 (Gate Summary arithmetic) TRULY resolved; #1 T2 subscriber + #2 Pillar 5 device-floor + #3 per-handler SNR remain PARTIALLY or PAPER-resolved at the perceptual / tail-math / measurement layers.

**CD-GDD-ALIGN: remains CONCERNS.** Pass-2 restoration conditions were (a) REQUIRED T2 subscriber and (b) Pillar 5 device-floor declaration. (a) met structurally but fails at the perceptual layer — HUD's "~200 ms dim/fade at `ScreenPosition`" feedback renders under the thumb that just long-pressed (occluded for the motor-impaired population the channel exists to serve). (b) met in the GDD but the pillar source-of-truth in `game-concept.md` was never synchronized, and the AC-P1 formula has undefined behaviour below 30 fps and above 60 fps. Not a rejection — narrow restoration path: fix Findings #1 and #2 and CD-GDD-ALIGN restores to APPROVED at pass-4.

Summary: The pass-2 revision made genuine architectural progress — three clocks distinguished, classifier Statefulness + margin invariant sound, Unity 6.3 API verification held, cross-GDD Event Bus T1 aggregate cleanly landed, and Gate Summary recount body-verified clean at 9/50/1/4/4 = 68. But pass-2 introduced or left unresolved: (a) HUD T2 feedback geometry violates Pillar 5 for the occluded-thumb case; (b) `game-concept.md` Pillar 5 text still unconditionally "100 ms" contradicting the dual-threshold contract, AC-P1 formula fps-domain gaps below 30 / above 60 unspecified; (c) per-handler 0.8 ms SNR claim was mean-vs-mean; Mali-G52-class 99p Stopwatch noise tail is 0.4–0.6 ms, so 0.8 ms tail SNR ≤ 2:1 and the pass-2 fix was paper-resolved at the tail; (d) Dev/Release CI profile selection mechanism unspecified (load-bearing for BLOCKING CI gates that claim "Release" thresholds); (e) AC-R13f's inline Stopwatch brackets (4 subs × 2 brackets) add 0.4–0.8 ms of measurement overhead inside the measured chain, self-defeating Release; (f) two one-line drafting contradictions — D.3 `GestureClass` variable table still says "opt-in consumers only", Pipeline Capacity Knobs `SubscriberCountMax` row cites the repealed `N × per_handler ≤ chain` invariant; (g) AC-R13a/b/c protocol does not specify shared paired dataset so an implementer could compute from independent captures; (h) the R9 accessibility OQ + v1.0 ADR trigger that the CD pass-2 downgrade was *conditional* on was never logged — downgrade terms not met.

### Blocking items (CD-adjudicated; merge-blocking — must resolve before pass-4 can be APPROVED)

1. **[game-designer + ux-designer — CD UPHELD]** HUD T2 feedback occluded by the finger it serves. Pass-2 spec "~200 ms dim/fade at `ScreenPosition`" renders under the thumb during slow lifts; no spawn-above-finger offset; no minimum sprite diameter / contrast. Fix: route through `R7.GetOcclusionOffset()`, minimum 48 dp diameter + 3:1 contrast, add audio cue as non-visual fail-safe OR alternative channel.

2. **[game-designer + systems-designer + ux-designer — CD UPHELD as consolidated fix]** Pillar 5 three-gap cluster: (a) `game-concept.md` Pillar 5 text still reads unconditional "100 ms" — contradicts AC-P1's dual-threshold; (b) AC-P1 linear-interpolation formula `100 + 33 × (60 − fps) / 30` undefined for `fps < 30` (thermal emergency) or `fps > 60` (120 Hz devices); (c) perceptual 133 ms exceeds Nielsen's "instant" threshold on primary audience device — downgraded to RECOMMENDED (project accepted the floor). Fix: update `game-concept.md` Pillar 5 text + extend AC-P1 with explicit fps-domain behaviour.

3. **[performance-analyst + unity-specialist — CD UPHELD]** Dev/Release CI profile selection mechanism unspecified — BLOCKING CI gates claim "Release" thresholds with no mechanism distinguishing Release from Dev at CI run time. Additionally: AC-R13f's inline Stopwatch brackets add 0.4–0.8 ms overhead *inside* the measured chain in Release, self-defeating the measurement. Fix: name the scripting-define-symbol (`MONEYMINER_RELEASE_GATE`); migrate Release per-handler measurement to sampling `ProfilerRecorder` (no inline brackets); keep inline brackets in Dev under wider ceiling.

4. **[systems-designer + performance-analyst — CD UPHELD]** D.3 `GestureClass` variable table still says "opt-in consumers only per R3.c" — contradicts R3.c REQUIRED MVP subscriber contract. One-line fix.

5. **[systems-designer + performance-analyst — CD UPHELD]** Pipeline Capacity Knobs `SubscriberCountMax` row cites the repealed `N × per_handler ≤ chain` multiplicative invariant. Internal contradiction with Section G "Independent caps" paragraph on the same page. One-line fix.

6. **[systems-designer + performance-analyst — CD UPHELD]** AC-R13a / AC-R13b / AC-R13c joint-satisfiability dataset protocol unspecified — three 99p gates on the same paired data but the document does not state they share it; implementer could compute AC-R13a/b from marginals and AC-R13c from joint samples, passing all three while AC-R13b's 2.0 ms chain ceiling is breached on worst-case frames. Fix: add shared-dataset preamble to H.4.

7. **[ux-designer — CD UPHELD on procedural grounds]** R9 accessibility OQ + v1.0 ADR trigger missing from OQ table. CD's pass-2 downgrade of R9 from BLOCKING to RECOMMENDED was *conditional* on the OQ being logged with owner + v1.0 ADR trigger. Pass-2 revision log committed to it and deferred instead. OQ table ends at OQ-5. Downgrade terms not met — resolution is to log OQ-6 (accessibility-specialist owner, v1.0 ADR gating shipping cert) + add MVP developer-reachable debug menu scaffold for motor-impaired playtesters pre-VS.

8. **[performance-analyst — CD UPHELD (upgraded back from initial downgrade)]** 0.8 ms per-handler Release SNR claim is mean-vs-mean; Mali-G52 99p Stopwatch noise tail is 0.4–0.6 ms, not 0.2 ms mean — pass-2 fix paper-resolved at the tail. Fix: raise per-handler Release to 1.0 ms (tail SNR ~2:1) + Dev to 1.5 ms; log OQ-7 for noise-subtracted measurement as post-MVP refinement.

### Notable recommended items (important, not blocking — ~20 deduplicated)

- Drag-lock retraction cancel pattern → Tool System handoff contract (3× deferred; promote to OQ).
- Modal-push mid-drag Tool System handoff (drag-end not fired; cleanup-leak hazard).
- AC-P3 warm-session 270 s window across 3 consecutive levels (2× deferred; promote to OQ).
- AC-D3-stateful "all post-lock sample permutations" enumerate 4 equivalence classes.
- AC-D3-margin-validate EditMode `AssetDatabase.SaveAssets` import-loop risk + `LogAssert.Expect` mechanism unspecified.
- AC-R3d-required `EventBus.DebugSubscriberCount` API dependency — rewrite to behavioral `LongPressPublishCounter` pattern (removes UNVERIFIED dependency).
- AC-P1 harness spec unowned — promote to BLOCKING at HUD GDD handoff gate.
- AC-INT3 / AC-INT5 stub contracts (Modal/Dialog + Level Runtime unauthored).
- AC-INV5 tautological BLOCKING CI gate (passes iff R1+R12+R4-nonalloc pass).
- Pillar 5 traceability clarity across AC-P1 / AC-R3d / AC-R3d-required / AC-INV2.
- AC-R13e dynamic post-modal-push subscriber registration not gated.
- AC-P3 "typical subscribers sit well below ceiling" prose not testable.
- `Screen.safeArea` staleness on `OnApplicationPause(false)` (Android 15+/iOS 18+ risk).
- UIDocument phantom-VisualElement edge case (2× deferred).
- Script Execution Order attribute-vs-project-settings mechanism unspecified.
- IL2CPP stripping risk on `DebugSubscriberCount` in Release.
- T1 aggregate per-channel apportionment for Collection / Tool / Hazard / Critter AI.
- Mali-G52 may not sustain 30 fps during AC-P3 soak.
- `Physics2D.OverlapPointNonAlloc` Unity 6.3 status unverified (CD DOWNGRADED — 4th UNVERIFIED engineering task).
- PanelRaycaster as R4 stated constraint (CD DOWNGRADED — already flagged UNVERIFIED with PlayMode gate).
- `input.contact.began` Rush-phase advisory event (3× deferred; promote to OQ).

### Strengths truly preserved from pass-2

- Gate Summary body-verified recount clean at 9 + 50 + 1 + 4 + 4 = 68 (qa-lead confirmed).
- Cross-GDD Event Bus `T1_per_frame_aggregate_ms = 8 ms` + AC-F4-aggregate (additive amendment preserves APPROVED).
- D.3 Classifier Statefulness rule + `d_early ≤ d_slop − 3` margin invariant — architecturally sound.
- Three-clocks distinction (4 ms / 16.67 ms / 100 ms–133 ms) survives pass-3 unchanged.
- Unity 6.3 API verification pass (R4 RaycastAll, R8 mobile haptics grep, package pins).
- AC-R3d publish-counter fixture using `EventBus.SubscribeAutoManaged` + `LongPressPublishCounter`.
- AC-D3-margin split into PlayMode + EditMode halves.

### Specialist disagreements (adjudicated by creative-director)

1. **HUD T2 feedback severity** — game-designer + ux-designer: BLOCKING (occluded by thumb); author pass-2 position: RESOLVED. **CD upheld BLOCKING** — paper resolution at the perceptual layer.
2. **Pillar 5 133 ms perceptual threshold** — game-designer: BLOCKING (outside Nielsen's 100 ms instant window). **CD downgraded perceptual-threshold sub-finding to RECOMMENDED**; upheld the pillar-text-sync + fps-domain sub-findings as BLOCKING.
3. **R9 accessibility severity** — ux-designer: BLOCKING (procedural — downgrade conditions not met); CD pass-2 position: RECOMMENDED. **CD pass-3 UPHELD ux-designer's BLOCKING** on procedural grounds — the downgrade was conditional on the OQ logging which did not happen.
4. **AC-R3d-required BLOCKING gate on unverified API** — qa-lead: BLOCKING. **CD DOWNGRADED to RECOMMENDED** — AC has escape clause ("gated pending Event Bus confirmation"); behavioral-fixture rewrite is correct architectural choice but not merge-blocking.
5. **AC-P1 harness spec** — qa-lead: BLOCKING. **CD DOWNGRADED to RECOMMENDED** — known/tracked gap; promote at HUD GDD handoff gate.
6. **Physics2D.OverlapPointNonAlloc** — unity-specialist: BLOCKING. **CD DOWNGRADED to RECOMMENDED** — NonAlloc family still in 6.3 docs per CD research; risk lower than claimed; add 4th UNVERIFIED engineering task.
7. **PanelRaycaster stated R4 constraint** — unity-specialist: BLOCKING. **CD DOWNGRADED to RECOMMENDED** — R4 has PlayMode verification gate already; framing change not contract change.
8. **Per-handler tail SNR** — performance-analyst: BLOCKING. **CD UPHELD BLOCKING (upgraded from initial instinct to accept pass-2)** — tail math is load-bearing; pass-2 mean-SNR reasoning failed.

### Prior blocker resolution status (pass-2 → pass-3)

| Pass-2 Blocker | Pass-3 Status |
|---|---|
| #1 T2 REQUIRED subscriber | PARTIALLY RESOLVED — subscriber exists; feedback occluded |
| #2 Pillar 5 device-floor | PARTIALLY RESOLVED — dual-threshold in GDD; pillar text stale; fps-domain gaps |
| #3 per-handler flakiness | PAPER RESOLVED — mean-SNR adjudication fails at tail; CI profile selection unresolved |
| #4 T1 aggregate | TRULY RESOLVED |
| #5 Gate Summary arithmetic | TRULY RESOLVED (9 + 50 + 1 + 4 + 4 = 68 confirmed) |

### Scope Signal: M

Roughly 2-4 hours author time; mostly surgical / editorial with one design decision (HUD feedback channel resolution) and one cross-document sync (`game-concept.md` Pillar 5). No architectural rewrites required.

**Next step**: Revise pass-3 in a fresh session (or same session if focus holds). Pass-4 should be APPROVED barring new findings on the delta.

Verdict: **NEEDS REVISION (pass-3).**

---

## Revision — 2026-04-22 — Pass-3 (addresses all 8 BLOCKING items from 2026-04-22 pass-3 review)

Scope signal: M (same as pass-3 review).
Author: Robert Michael Watson. No specialist re-spawn in this pass — all eight blockers had CD-adjudicated or user-decision-recorded resolutions; the pass-4 adversarial panel runs on `/design-review` in the next fresh session.
Blocking items resolved: 8/8. Recommended items addressed: 5 (OQ-6/7/8/9/10 promoted from deferred-list to logged OQs; H.4 shared-dataset preamble also resolves the AC-R13a/b/c-vs-AC-R13c mixed-signals finding).

**User decisions recorded this pass (via multi-tab widget 2026-04-22):**
- Finding #1 (HUD T2 feedback occlusion): **Offset visual + audio cue** — HUD renders visual tick at `ScreenPosition + GetOcclusionOffset()` (≥ 48 dp, 200 ms dim/fade, ≥ 3:1 contrast) AND emits audio tick via Audio Bus (`input.longpress.received`). HANDOFF: HUD GDD author owns both channels.
- Finding #3 (CI profile selection): deferred to author recommendation → **scripting-define-symbol `MONEYMINER_RELEASE_GATE`** (cleanest; CI pipeline sets per build variant). Release per-handler measurement migrated to sampling `ProfilerRecorder` (no inline brackets).
- Finding #7 (R9 accessibility): deferred to author recommendation → **OQ-6 logged + MVP developer-reachable debug menu** (dev-build-gated; activation pattern TBD during implementation) + full player-facing UI as v1.0-cert-gating ADR (owner: Accessibility Service + Main Menu / Settings / Pause GDDs).
- Finding #8 (per-handler SNR): deferred to author recommendation → **both: raise Release to 1.0 ms + Dev to 1.5 ms now AND log OQ-7 for noise-subtracted measurement post-MVP**.

**Revision log (per-blocker resolution):**

1. **BLOCKING #1 — HUD T2 feedback occluded by thumb** → RESOLVED via dual-channel contract.
   - R3.c rewritten: HUD renders (1) offset visual tick at `ScreenPosition + InputState.GetOcclusionOffset(ScreenPosition)` with min 48 dp diameter / 200 ms dim-fade / ≥ 3:1 contrast + (2) audio tick via Audio Bus (working slug `input.longpress.received`, tag TBD at Audio Bus GDD). Visual channel prevents thumb occlusion; audio channel is non-visual fail-safe for accessibility population.
   - Section C Interactions downstream HUD T2 row rewritten to dual-channel.
   - Dependencies Downstream (hard — MVP) HUD row rewritten to dual-channel.
   - Overview quick-reference updated to mention dual-channel.
   - D.3 Path C worked example footnote updated (opt-in → HUD REQUIRED dual-channel).
   - D.3 variable table `GestureClass` row updated (also addresses BLOCKING #4).
   - Editor tooling `ProfilerRecorder`/`Stopwatch` split documented.

2. **BLOCKING #2 — Pillar 5 three-gap cluster** → RESOLVED via consolidated fix.
   - `game-concept.md` Pillar 5 text amended: "Every input gets audiovisual feedback within the Feedback Floor — 100 ms at 60 fps target / 133 ms at 30 fps floor per input-system.md AC-P1; linear interpolation between. Sustained fps < 30 is a thermal emergency outside this contract."
   - AC-P1 extended with Out-of-domain behaviour subsection: `fps < 30` → UNMEASURABLE (thermal-emergency counter, run marked INCONCLUSIVE if > 5% samples); `fps > 60` → threshold clamps to 100 ms (no extrapolation).
   - AC-P1 added display-panel-latency exclusion note (8-16 ms OLED/LCD response + panel-refresh alignment outside Unity's instrumentation boundary; acknowledge to playtesters + v1.0 cert review).
   - Status header amended to cite cross-document synchronization with `game-concept.md` Pillar 5.

3. **BLOCKING #3 — Dev/Release CI profile selection + AC-R13f Release self-defeat** → RESOLVED via scripting-define-symbol + measurement-method split.
   - R13.d rewritten: Release uses sampling `ProfilerRecorder` (activated only when `MONEYMINER_RELEASE_GATE` scripting define set); Dev uses inline `Stopwatch.GetTimestamp` brackets. Separation prevents Release measurement from polluting itself.
   - Section G Interaction Between Tuning Knobs paragraph added Dev/Release CI profile selection block with the scripting-define-symbol mechanism specified.
   - AC-R13f body rewritten with measurement method per tier + compile-in rule.
   - Editor tooling section updated to mention the mutually-exclusive Release/Dev measurement paths.

4. **BLOCKING #4 — D.3 variable table still says "opt-in consumers only"** → RESOLVED via one-line edit.
   - D.3 variable table `GestureClass` row rewritten to "HUD REQUIRED MVP subscriber per R3.c + AC-R3d-required, dual-channel: offset visual + audio cue."

5. **BLOCKING #5 — Pipeline Capacity Knobs `SubscriberCountMax` cites repealed invariant** → RESOLVED via row rewrite.
   - Pipeline Capacity Knobs table row rewritten: chain ceiling (AC-R13b) is the authoritative constraint; SubscriberCountMax = 4 is an architectural discipline rule (not derived from repealed multiplicative invariant). Explicit repeal cited.

6. **BLOCKING #6 — AC-R13a/b/c joint-satisfiability dataset protocol unspecified** → RESOLVED via H.4 shared-dataset preamble.
   - New preamble block inserted at start of H.4 stating: AC-R13a/b/c/d/f all operate on a single shared paired dataset (1000 events / 60 frames / paired per-frame tuples); per-AC separate captures not permitted; CI reporter MUST label each 99p with paired-frame index for triage.

7. **BLOCKING #7 — R9 accessibility OQ missing** → RESOLVED via OQ-6 + MVP debug menu note.
   - OQ-6 logged: owner accessibility-specialist + ux-designer (MVP debug menu) + Accessibility Service GDD author (v1.0 ADR); resolution target MVP implementation sprint (debug menu) + v1.0 Accessibility Service GDD authoring (player UI). Full v1.0-cert-gating ADR trigger cited.
   - R9 section body extended with MVP debug menu requirement block.
   - Editor tooling section gained R9 accessibility debug menu bullet.

8. **BLOCKING #8 — 0.8 ms per-handler mean-vs-tail SNR** → RESOLVED via threshold retune + OQ-7.
   - R13.d body rewritten: per-handler Release 0.8 → 1.0 ms (tail SNR ≥ 2:1 at 0.5 ms noise tail); Dev 1.2 → 1.5 ms (absorbs Dev bracket overhead).
   - Section G Pipeline Capacity Knobs per-handler Release + Dev rows updated.
   - Section G Interaction paragraph updated with pass-3 retune rationale.
   - Formula Candidates line updated to 1.0 / 1.5 ms values.
   - AC-R13f body rewritten with new thresholds + attribution rule (when AC-R13b fails and all AC-R13f pass, nominate highest-99p subscriber as optimization target).
   - AC-P3 subscriber-count scenario updated to cite 1.0 ms ceiling + provisional heuristic flagged for OQ-8 calibration.
   - Cross-Reference to Section G table row updated to 1.0 / 1.5 ms.
   - OQ-7 logged (noise-subtracted measurement, post-MVP refinement, performance-analyst owner).

**Recommended items addressed in this pass (promoted to OQ or otherwise logged):**
- OQ-8: AC-P3 thermal-headroom threshold empirical calibration (performance-analyst + technical-director; first on-hardware run post-ADR-003).
- OQ-9: Drag-lock retraction cancel pattern → Tool System handoff (3× deferred; now logged with owner + resolution at Tool System GDD authoring).
- OQ-10: `input.contact.began` Rush-phase advisory event (3× deferred; now logged with re-adjudication trigger).

**Recommended items still deferred (not addressed in pass-3):**
- AC-P3 warm-session 270 s window across 3 consecutive levels (2× deferred; could be rolled into OQ-8 at first on-hardware run).
- AC-D3-stateful 4-equivalence-class enumeration.
- AC-D3-margin-validate `AssetDatabase.SaveAssets` import-loop risk + `LogAssert.Expect` mechanism.
- AC-R3d-required behavioral-fixture rewrite (CD downgraded to RECOMMENDED; escape clause in AC body acceptable).
- AC-P1 harness spec (CD downgraded to RECOMMENDED; promote at HUD GDD handoff).
- AC-INT3 / INT5 stub contracts (pending Modal/Dialog + Level Runtime GDDs).
- AC-INV5 tautological BLOCKING CI summary gate — still counted in 9.
- AC-R13e dynamic post-modal-push subscriber registration not gated.
- AC-P3 "typical subscribers sit well below ceiling" prose clause.
- `Screen.safeArea` staleness on `OnApplicationPause(false)`.
- UIDocument phantom-VisualElement hit.
- Script Execution Order attribute-vs-project-settings mechanism.
- IL2CPP stripping risk on `DebugSubscriberCount`.
- T1 aggregate per-channel apportionment for Collection/Tool/Hazard/Critter AI.
- `Physics2D.OverlapPointNonAlloc` Unity 6.3 status (CD downgraded — 4th UNVERIFIED engineering task should be added).
- PanelRaycaster as R4 stated constraint (CD downgraded — existing PlayMode gate acceptable).

**AC count delta pass-2 → pass-3:** 68 → 68 (no new ACs added; no ACs removed; pass-3 blockers resolved via existing AC text rewrites + cross-references + OQ promotions).

**Next step:** `/design-review design/gdd/input-system.md` in a fresh session as pass-4 re-review. Specialist panel may surface new findings on the delta (HUD dual-channel contract, pillar-text cross-sync, CI profile scripting define, per-handler 1.0/1.5 ms retune + `ProfilerRecorder` migration, H.4 shared-dataset preamble, five new OQs). Verdict target: **APPROVED**; CD-GDD-ALIGN expected to restore to **APPROVED** on pillar coverage.

Verdict pending re-review: **REVISED (pass-3), awaiting adversarial re-validation.**

---

## Review — 2026-04-23 — Pass-4 — Verdict: NEEDS REVISION

Scope signal: M (bordering L)
Specialists: game-designer, systems-designer, performance-analyst, unity-specialist, ux-designer, qa-lead, creative-director (senior synthesis)
Blocking items: 14 (CD-adjudicated from 16 specialist-flagged) | Recommended: ~20 (deduplicated) | Nice-to-have: ~8
Prior verdict resolved: **Partial** — 4/8 pass-3 blockers truly resolved (Pillar 5 pillar-text sync; AC-P1 fps-domain extensions; D.3 variable table row; SubscriberCountMax rewrite); 4/8 paper-resolved (HUD visual half OK but audio half paper-tier; ProfilerRecorder migration mechanically incoherent; OQ-6 activation pattern self-defeating catch-22; per-handler retune OK numerically but AC-P3 ceiling now provisional).

**CD-GDD-ALIGN: CONCERNS (sustained; not restored to APPROVED).** Pass-3 closed two genuine structural gaps (HUD visual half + Pillar 5 pillar-text sync) but paper-resolved four others. Pass-4 surfaces five NEW Pillar-5-adjacent defects: (a) HUD audio-channel paper resolution (no AC, no dependency, namespace-colliding slug); (b) OQ-6 activation catch-22 (motor-impaired users can't access motor-impairment menu); (c) OQ-10 input.contact.began 3× deferred (CD's own pass-1 adjudication condition unfulfilled); (d) AC-P3 "provisional heuristic pending calibration" pre-authorizes relaxation on first failure = not a gate; (e) 10 BLOCKING CI infrastructure items including MONEYMINER_RELEASE_GATE unspecified CI injection path, `EventBus.DebugSubscriberCount` phantom API backing BLOCKING CI gate, ProfilerRecorder "no inline brackets" claim architecturally incoherent, H.4 shared-dataset preamble incompatible with AC-R13f tier-split, AC-P1 INCONCLUSIVE CI reporter behavior unspecified, AC-P1 30fps hysteresis + 120Hz window too narrow, AC-R13c correlated-tail under thermal load, D.1 half-and-half split gate asymmetry, plus two self-inflicted stale-text remnants in D.3 formula block (line 241) + AC-D3-late-lift (line 612) that pass-3 created but did not catch.

### Blocking items (CD-adjudicated; merge-blocking)

1. **[game-designer + systems-designer]** D.3 formula code block (line ~241) still says "opt-in only, no consumer required" — pass-3 self-inflicted stale text; one-line editorial.
2. **[systems-designer]** AC-D3-late-lift body (line ~612) still says "LongPress (discarded)" — contradicts R3.c; one-line editorial.
3. **[unity-specialist + qa-lead — high convergence]** `EventBus.DebugSubscriberCount` is a phantom API backing BLOCKING CI gate (AC-R3d-required). Fix: rewrite AC-R3d-required to `LongPressPublishCounter` behavioral fixture.
4. **[performance-analyst + unity-specialist — high convergence]** `MONEYMINER_RELEASE_GATE` CI injection mechanism unspecified. Fix: name editor-script mechanism + game-ci invocation.
5. **[performance-analyst + unity-specialist]** `ProfilerRecorder` "no inline brackets" claim architecturally incoherent. Fix: retain inline `ProfilerMarker.Begin/End` (lighter than Stopwatch) + restate per-handler budget.
6. **[qa-lead]** H.4 shared-dataset preamble vs AC-R13f measurement-method-split tier incompatibility.
7. **[performance-analyst]** AC-R13c 4.0 ms joint-sum under correlated tails. Fix: correlated-tail routing clause.
8. **[systems-designer]** D.1 half-and-half split gate asymmetry. Fix: rewrite D.1 prose to declare only joint sum is BLOCKING-gated.
9. **[qa-lead]** AC-P1 INCONCLUSIVE CI reporter behavior unspecified. Fix: warning-level annotation, non-merge-blocking, routed to perf-analyst.
10. **[systems-designer + qa-lead]** AC-P1 30-fps hysteresis + 120 Hz rolling-window too narrow. Fix: wall-clock window (≥ 500 ms) + hysteresis band.
11. **[game-designer]** HUD audio channel has no AC and no dependency declaration — paper-resolution recurrence of pass-2 T2-subscriber pattern. Fix: add Audio Bus dependency + AC-R3d-audio + rename slug to 4-segment namespace-compliant form.
12. **[ux-designer]** OQ-6 MVP debug-menu activation = "long-press + three-finger-tap" — accessibility catch-22. User-decision required.
13. **[game-designer + ux-designer]** OQ-10 `input.contact.began` deferred 3× = CD pass-1 adjudication condition unfulfilled. User-decision required.
14. **[game-designer]** AC-P3 3.2 ms ceiling "provisional heuristic pending OQ-8 calibration" = not a gate. User-decision required.

### Specialist disagreements adjudicated

- **AC-R3d-required BLOCKING severity**: qa-lead + unity-specialist BLOCKING (pass-3 CD RECOMMENDED). **CD pass-4 UPHELD BLOCKING** — escape clause on BLOCKING CI gate is structurally contradictory.
- **OQ-10 severity**: game-designer BLOCKING; ux-designer RECOMMENDED. **CD pass-4 UPHELD BLOCKING** — pass-1 CD adjudication condition unfulfilled.
- **Dual-channel 3:1 contrast**: ux-designer BLOCKING. **CD DOWNGRADED to RECOMMENDED** (Input-scope floor; promote to BLOCKING at HUD GDD handoff).
- **H.4 per-event vs per-frame sampling (BLOCKING #7 original)**: systems-designer BLOCKING. **CD DOWNGRADED to RECOMMENDED** — subsidiary to BLOCKING #6.

### Pass-3 honesty check

| Pass-3 Blocker Claim | Pass-4 Verdict |
|---|---|
| #1 HUD dual-channel | PARTIALLY RESOLVED (visual half ✓; audio half paper-tier) |
| #2 Pillar 5 pillar-text sync | TRULY RESOLVED |
| #3 Dev/Release CI profile + AC-R13f | PARTIALLY RESOLVED (ProfilerRecorder mechanism incoherent; CI injection gap) |
| #4 D.3 variable table | TRULY RESOLVED (but formula block + AC-D3-late-lift self-inflicted stale text missed) |
| #5 SubscriberCountMax | TRULY RESOLVED |
| #6 H.4 shared-dataset | PARTIALLY RESOLVED (preamble landed; tier-split created new defect) |
| #7 R9 accessibility OQ-6 | PARTIALLY RESOLVED (OQ logged; activation pattern self-defeating) |
| #8 Per-handler retune | PARTIALLY RESOLVED (numerically OK; AC-P3 ceiling now provisional) |

Net: 3/8 truly resolved, 5/8 partial. CD flags the recurring OQ-promotion-as-deferral pattern (OQ-6, OQ-7, OQ-8, OQ-10) as the structural risk for pass-5: if any pass-4 revision lands with another OQ-promotion on a Pillar-5 blocker, CD-GDD-ALIGN should escalate from CONCERNS to REJECT.

### Scope signal: M (bordering L)

Approximately 4–6 hours author effort: 10 surgical edits + 4 user-decision-required design choices. No architectural rewrites.

**Next step:** Revise pass-4. Four design decisions required via multi-tab widget before drafting (#5 ProfilerRecorder mechanism; #12 OQ-6 activation; #13 OQ-10 contact.began; #14 AC-P3 ceiling).

Verdict: **NEEDS REVISION (pass-4).**

---

## Revision — 2026-04-23 — Pass-4 (addresses all 14 BLOCKING items from 2026-04-23 pass-4 review)

Scope signal: M (same as pass-4 review).
Author: Robert Michael Watson. No specialist re-spawn in this pass — the 14 blockers had CD-adjudicated or user-decision-recorded resolutions; the pass-5 adversarial panel runs on `/design-review` in the next fresh session.
Blocking items resolved: 14/14. Recommended items addressed: 3 (Shift-Left EditMode wording clarified; OQ-10 closed; v_lateLift variable-table row cleaned up per systems-designer nice-to-have).

**User decisions recorded this pass (via multi-tab widget 2026-04-23):**
- Finding #5 (ProfilerRecorder mechanism): **Retain inline ProfilerMarker.Begin/End + restate budget** — acknowledges markers must be inline for per-handler granularity; Release absorbs ~0.2–0.4 ms marker overhead within 1.0 ms ceiling.
- Finding #12 (OQ-6 activation): **Non-impaired activation pattern** — `DEVELOPMENT_BUILD`-gated single-tap pause-menu button (primary) + rapid 5-tap on title-screen logo (secondary, works in shipping MVP).
- Finding #13 (OQ-10): **Implement input.contact.began T2 channel now** — new R3.e + AC-R3g + Event Bus Interactions row + Section C downstream subscriber rows; honours pass-1 CD adjudication condition.
- Finding #14 (AC-P3 ceiling): **Set non-provisional 5.0 ms ceiling now** — derived from Mali-G52 published 25 % sustained-load thermal inflation × 4.0 ms nominal; OQ-8 downgraded to informational data-capture.

**Revision log (per-blocker resolution):**

1. **BLOCKING #1 — D.3 formula code block stale "opt-in only"** → RESOLVED via one-line edit; formula block LongPress branch now reads "HUD REQUIRED MVP subscriber per AC-R3d-required, dual-channel".
2. **BLOCKING #2 — AC-D3-late-lift stale "LongPress (discarded)"** → RESOLVED via AC body rewrite; `v == v_lateLift` now correctly states LongPress publishes on T2 with HUD REQUIRED subscriber. `v_lateLift` variable table row also cleaned up.
3. **BLOCKING #3 — AC-R3d-required phantom API** → RESOLVED via behavioral-fixture rewrite. AC now uses `LongPressPublishCounter` via `EventBus.SubscribeAutoManaged` on a dedicated MonoBehaviour host GameObject; no dependency on unverified `EventBus.DebugSubscriberCount`. AC remains BLOCKING CI and is now operable.
4. **BLOCKING #4 — MONEYMINER_RELEASE_GATE CI injection** → RESOLVED via named mechanism in Section G: `tools/ci/SetReleaseGate.cs` editor script exposing `MoneyMiner.CI.ReleaseGateSetter.EnableReleaseGate()` invoked by `game-ci/unity-test-runner@v4` via `customParameters: -executeMethod` on Release-artifact jobs only. Complementary `DisableReleaseGate()` teardown. Technical Setup sprint MUST land this before first Input CI merge.
5. **BLOCKING #5 — ProfilerRecorder mechanism incoherence** → RESOLVED per user decision: Release retains inline `ProfilerMarker.Begin/End` brackets (lighter than Stopwatch: ~0.05–0.1 ms per bracket vs ~0.1–0.2 ms), read externally via `ProfilerRecorder.StartNew(ProfilerCategory.Scripts, "Input.Handler.{Type}.{Method}", 1)`. Total 4-sub chain overhead ~0.2–0.4 ms absorbed by 1.0 ms ceiling. Pass-3's "no inline brackets" framing explicitly corrected: Release has lighter brackets, not zero brackets. R13.d, AC-R13f, Section G Interaction paragraph, and Editor tooling note all updated.
6. **BLOCKING #6 — H.4 shared-dataset tier-split** → RESOLVED via preamble rewrite: Release-tier ACs share Release-artifact capture, Dev-tier ACs share Dev-build capture, cross-tier informational only. Per-event vs per-frame sampling units also clarified (AC-R13a/b/f per-event 99p over ~1000-event series; AC-R13c per-frame-sum 99p over 60-frame series).
7. **BLOCKING #7 — AC-R13c correlated-tail** → RESOLVED via routing clause: when AC-R13c fails with AC-R13a/b both passing (`correlated-tail-under-thermal`), CI tags the failure, routes to OQ-8, downgrades to warning-level annotation (not silent merge-block); when AC-R13c fails with AC-R13a OR AC-R13b also failing (`single-component-overrun`), merge-block is preserved.
8. **BLOCKING #8 — D.1 half-and-half split** → RESOLVED via D.1 Rationale rewrite: half-and-half split is design-intent load-balancing contract (PLAYMODE via AC-R13a/b), not a BLOCKING CI gate; only the joint sum (AC-R13c) is merge-blocking. Gate asymmetry explicitly acknowledged.
9. **BLOCKING #9 — AC-P1 INCONCLUSIVE CI behavior** → RESOLVED via subsection: `game-ci/unity-test-runner@v4` configured to treat INCONCLUSIVE as warning-level GitHub Actions annotation (non-merge-blocking), routed to perf-analyst via `notify-perf-analyst-on-inconclusive.yml` workflow (to be authored during Technical Setup).
10. **BLOCKING #10 — AC-P1 hysteresis + wall-clock window** → RESOLVED: rolling-mean window restated as wall-clock ≥ 500 ms (replaces 30-frame fixed count); hysteresis band added (once fps < 30, remain UNMEASURABLE until ≥ 32 fps for ≥ 10 consecutive 500 ms windows = ≥ 5 s sustained recovery); `unscaledDeltaTime` used instead of `deltaTime` to survive `timeScale == 0` pollution.
11. **BLOCKING #11 — HUD audio channel** → RESOLVED via three complementary changes: (a) slug renamed `input.longpress.received` → `audio.input.longpress.received` (4-segment Event Bus R5 regex compliant, no Input-namespace collision); (b) Audio Bus added to Input Upstream dependency table as publisher-target (provisional flag pending Audio Bus GDD); (c) new AC-R3d-audio BLOCKING CI asserts audio channel publishes on every LongPress classification with `LongPressUnhandledPayload`, using `LongPressAudioCounter` behavioral fixture. Event Bus GDD Interactions table also updated.
12. **BLOCKING #12 — OQ-6 activation catch-22** → RESOLVED per user decision: non-impaired activation with two paths — `DEVELOPMENT_BUILD`-gated single-tap pause-menu "Accessibility Debug" button (primary MVP playtester path) + rapid 5-tap on title-screen logo within 2 seconds (secondary, works in Dev AND shipping MVP). Long-press + three-finger-tap activation explicitly repealed. R9 section body + OQ-6 OQ table entry + Editor tooling note all updated.
13. **BLOCKING #13 — input.contact.began T2 channel** → RESOLVED per user decision via implementation NOW: new R3.e subsection defines `input.contact.began` T2 Feedback channel with `ContactBeganPayload(ScreenPosition, FingerID, BeganRealtimeSeconds)`; no REQUIRED MVP subscriber (opt-in HUD + VFX/Juice VS); fires on every `began` under Active state (extended AC-INV2 invariant). New AC-R3g PLAYMODE asserts publish behavior. Section C downstream table has 2 new rows (HUD opt-in MVP + VFX/Juice opt-in VS). Event Bus GDD Interactions table row added. Overview quick-reference updated. OQ-10 closed as RESOLVED 2026-04-23.
14. **BLOCKING #14 — AC-P3 non-provisional ceiling** → RESOLVED per user decision: ceiling set at 5.0 ms (derived from Mali-G52 published 25 % sustained-load thermal inflation × 4.0 ms R13.c nominal). "Provisional heuristic pending OQ-8 calibration" language repealed. OQ-8 downgraded to informational data-capture (non-blocking; records measured inflation at first-run for post-MVP review). If measured inflation < 15 %, OQ-8 may propose tightening; if > 25 %, escalate to optimization, not AC relaxation.

**Recommended items addressed this pass (3 of ~20):**
- Shift-Left Implementation Notes "stub EditMode test files" clarified to "stub PlayMode or EditMode test files" + new bullet on tests/ directory scaffolding + `tools/ci/SetReleaseGate.cs` dependency.
- OQ-10 closed (was deferred 3×; resolved via pass-4 BLOCKING #13 implementation).
- `v_lateLift` variable table row cleaned up (pass-4 BLOCKING #2 companion fix).

**Recommended items NOT addressed (still deferred):**
- D.3 stationary-gesture residual dead-zone prose
- R7 `GetOcclusionOffset` R1-whitelist note
- `SubscriberCountMax` Pipeline Capacity Knobs prose still references "bounds the subscriber count in practice" coupled framing
- Package-pin UNVERIFIED labels partially resolvable from project reference docs
- `EnhancedTouch.Touch.activeTouches` GC.Alloc probe as 4th engineering task
- `Physics2D.OverlapPointNonAlloc` availability check as 5th engineering task
- `[DefaultExecutionOrder(-1000)]` attribute mechanism spec for EC Configuration/Tuning Extreme 1
- Dynamic Island (iOS 18+) split from OQ-3 foldable trigger
- T1 aggregate 4 ms remaining budget for Collection + Tool + Hazard + Critter AI (log as new OQ-11)
- 133 ms at 30 fps player-facing thermal signal (HUD GDD scope)
- 44pt/48dp hit-zone floor scales when R9 accommodation profile active
- AC-INV5 tautological gate conversion to CI dashboard badge
- AC-D3-stateful 4 equivalence classes enumeration
- AC-D3-margin-validate `LogAssert.Expect` mechanism
- AC-P3 soak-test CI job spec in production tooling
- `PhaseTransitionSuppressDuration` wall-clock timer invariant in R10.a body
- 3:1 contrast floor promoted to BLOCKING at HUD GDD handoff (flagged here; HUD-scope action)

**AC count delta pass-3 → pass-4:** 68 → 70 (+1 BLOCKING CI AC-R3d-audio; +1 PLAYMODE AC-R3g). Gate Summary body-verified: **10 + 51 + 1 + 4 + 4 = 70.**

**Next step:** `/design-review design/gdd/input-system.md` in a fresh session as pass-5 re-review. Specialist panel may surface new findings on the delta (inline ProfilerMarker framing, audio-channel AC + Event Bus coordination, contact.began T2 channel + AC-R3g, non-provisional AC-P3 5.0 ms ceiling, hysteresis + wall-clock window, 10 BLOCKING CI resolutions). Verdict target: **APPROVED**; CD-GDD-ALIGN expected to restore to **APPROVED** on pass-5 conditional on the 16 still-deferred recommended items not surfacing as new BLOCKING.

Verdict pending re-review: **REVISED (pass-4), awaiting adversarial re-validation.**

---

## Review — 2026-04-23 — Pass-5 — Verdict: NEEDS REVISION

Scope signal: L (4–8 hours)
Specialists: game-designer, systems-designer, performance-analyst, unity-specialist, qa-lead, ux-designer, creative-director (senior synthesis)
Blocking items: 18 (CD-adjudicated, deduplicated from ~27 specialist-flagged) | Recommended: ~20 (deduplicated) | Nice-to-have: ~10
Prior verdict resolved: **Mixed** — Gate Summary body-count arithmetic truly resolved (10 + 51 + 1 + 4 + 4 = 70 qa-lead-confirmed); Pillar 5 pillar-text sync truly resolved; formula classifier statefulness + margin invariant truly resolved; D.3 formula block + AC-D3-late-lift stale-text fixes truly resolved. However, pass-4's 14 in-session revisions introduced multiple structural defects the author alone could not catch — specifically in the Unity 6.3 implementation-contract layer (CI injection mechanism, ProfilerMarker/ProfilerRecorder specification, thermal ceiling derivation) and in the Pillar 5 contract layer (R3.e `input.contact.began` advisory-only recurring the pass-2 T2-LongPress paper-resolution pattern).

**CD-GDD-ALIGN: ESCALATED from CONCERNS to REJECT.** The pass-4 warning (`if any pass-4 revision lands with another OQ-promotion on a Pillar-5 blocker, CD-GDD-ALIGN should escalate from CONCERNS to REJECT`) — its literal text does not apply (pass-4 implemented `contact.began` rather than OQ-promoting it) but the *intent* (guarding against paper-resolution on Pillar-5 blockers) is triggered. Game-designer + ux-designer independently classified R3.e advisory-only as paper resolution; performance-analyst + systems-designer independently classified AC-P3 thermal ceiling as paper resolution. Two Pillar-5 / perf-contract paper resolutions converged on by independent specialists = sufficient for escalation. REJECT is on the gate, not the work — the document is architecturally sound; it is the implementation-contract layer that is broken.

Summary: Pass-4 closed 3/8 pass-3 blockers truly (Pillar 5 pillar-text sync, D.3 variable-table row, SubscriberCountMax rewrite) but the 14 in-session author-alone revisions introduced implementation-contract defects. Four high-confidence cross-specialist convergences: (A) Unity 6.3 CI injection mechanism broken — `-executeMethod` + `-runTests` mutually exclusive, `SetScriptingDefineSymbols(NamedBuildTarget, string[])` overload does not exist, `SetReleaseGate.cs` as specified is a compile error; (B) ProfilerMarker/ProfilerRecorder specification mechanically incoherent — `ProfilerCategory.Scripts` UNVERIFIED, `maxSampleCount: 1` cannot produce p99, Release-build functionality unverified, budget arithmetic carries Stopwatch-era noise floor; (C) AC-P3 5.0 ms thermal ceiling derivation applies Mali-G52 GPU-domain 25% inflation to Input's CPU-bound pipeline (real CPU scaling 10–15%); (D) R3.e `contact.began` advisory-only = structural paper resolution of Pillar 5 on-contact silence gap. Beyond the convergences: D.3 classifier has priority ambiguity at Path B canonical case + coverage hole at `d < d_early, t > t_tap_max, ended=false`; 10 BLOCKING CI gates are structurally inoperable (tests/, SetReleaseGate.cs, notify-perf-analyst-on-inconclusive.yml do not exist); AC-R3d-required references phantom HUD type; `v_lateLift = 0.02 pt/ms` uncalibrated (real motor-impaired lifts are 0.05–0.8); T1 aggregate 4 ms remainder under-reserved (OQ-11 promised pass-4 but never logged); AC-INV5 tautological (carried 2 passes); AC-D3-margin-validate `AssetDatabase.SaveAssets` + `LogAssert.Expect` unspecified (carried 3 passes); ux-designer upgraded two prior CD adjudications on accessibility (5-tap not motor-accessible, OQ-6 cert-gating + accessibility-specialist absence) both SUSTAINED.

### Blocking items (18; CD-adjudicated; merge-blocking — all must resolve before pass-6 can be APPROVED)

**Pillar 5 contract cluster**
1. **[game-designer + ux-designer — high convergence]** R3.e `input.contact.began` advisory-only recurs pass-2 T2-LongPress paper-resolution pattern on a Pillar-5 blocker. Fix: promote HUD to REQUIRED MVP subscriber for Rush-phase scenes; add AC-R3g-required mirroring AC-R3d-required.
2. **[game-designer]** `audio.input.longpress.received` framed as "guaranteed non-visual fail-safe" but AC-R3d-audio only gates Input's publish, not Input→HUD→Audio Bus end-to-end. Fix: downgrade language OR add DEVICE-tier end-to-end AC.
3. **[game-designer]** Gate Summary footer reconciliation note says "recount yields 50" — stale; with AC-R3g addition should be 51. Third pass with arithmetic drift.

**Formula / budget structural cluster**
4. **[systems-designer]** D.3 classifier (a) priority ambiguity — `InProgress` rule 1 AND `Drag` early-branch both match at Path B canonical case `(d=7, t=160, ended=false)` with no stated precedence; (b) coverage hole — at `d < d_early, t > t_tap_max, ended=false` no branch matches.
5. **[systems-designer]** H.4 shared-dataset tier-split creates regression amnesty — correlated-tail downgrade can whitewash Release T1Chain regressions because Dev-side AC-R13b is on a different dataset.
6. **[systems-designer]** T1 aggregate 4 ms remainder under-reserved for Collection + Tool + Hazard + Critter AI Rush-phase peak frames. OQ-11 promised in pass-4 log; never added.
7. **[systems-designer]** `v_lateLift = 0.02 pt/ms` default uncalibrated; real motor-impaired lift velocities 0.05–0.8 pt/ms; late-lift Tap-forgiveness branch functionally dead at default.

**Unity 6.3 reality-alignment cluster (pass-4 in-session revisions introduced defects)**
8. **[systems-designer + performance-analyst — high convergence]** AC-P3 5.0 ms thermal ceiling derivation applies 25% (Mali-G52 GPU-domain) to Input's CPU-bound main-thread pipeline; CPU-cluster thermal scaling is 10–15%. Ceiling 0.4–0.6 ms too loose.
9. **[performance-analyst + unity-specialist — high convergence]** ProfilerMarker/ProfilerRecorder specification mechanically incoherent on three fronts: Stopwatch-era noise floor retained post-migration; `ProfilerCategory.Scripts` UNVERIFIED against Unity 6.3; `maxSampleCount: 1` cannot produce p99 over 1000 events; Release-build marker functionality unverified.
10. **[performance-analyst + unity-specialist — high convergence]** CI injection mechanism broken twice: (a) `game-ci/unity-test-runner@v4 customParameters: -executeMethod` — `-executeMethod` + `-runTests` are mutually exclusive; (b) `PlayerSettings.SetScriptingDefineSymbols(NamedBuildTarget, string[])` overload does not exist; correct signature takes semicolon-delimited string. `SetReleaseGate.cs` as specified is a compile error.
11. **[performance-analyst]** AC-R13c correlated-tail downgrade has no thermal-state discriminator. Simultaneous 0.3 ms regression in both PreFire and T1Chain gets merge amnesty. Fix: require `thermal_state: {temperature, is_throttled}` annotation + ADR-003 device as precondition.

**CI operability / AC hygiene cluster**
12. **[qa-lead + performance-analyst — convergence]** 10 BLOCKING CI gates structurally inoperable — `tests/`, `tools/ci/SetReleaseGate.cs`, `notify-perf-analyst-on-inconclusive.yml` do not exist. APPROVED with non-operative gates = false quality gate. Fix: spec files into MVP-implementation-sprint scope with "activates in Sprint N" AC annotation, OR downgrade to ADVISORY until infra lands.
13. **[qa-lead]** ACs reference phantom types from un-authored GDDs: AC-R3d-required cites `InputLongPressHudListener` (HUD GDD un-authored) — compile failure, not test failure; AC-R3d-audio hardcodes slug without Audio Bus cross-GDD validation; AC-INT3/AC-INT5 mock contracts for un-authored Modal/Dialog + Level Runtime GDDs without divergence-risk disclosure.
14. **[qa-lead — carried since pass-3]** AC-D3-margin-validate `AssetDatabase.SaveAssets` synchronization + `LogAssert.Expect` mechanism unspecified three passes running.
15. **[qa-lead — carried since pass-3]** AC-INV5 tautological; occupies BLOCKING CI slot with no independent test logic.

**Accessibility / regulatory cluster (CD-UPGRADED prior adjudications)**
16. **[ux-designer — CD UPGRADES prior adjudication — SUSTAINED]** Rapid 5-tap in 2 seconds is not accessible for motor-impaired users at `t_tap_max = 600 ms`. Structurally identical to OQ-6 catch-22 with new gesture. Fix: persistent static "Accessibility" button on title screen, single-tap, no timer, no gesture.
17. **[ux-designer — CD UPGRADES prior adjudication — SUSTAINED]** OQ-6 cert-gating (not VS-sprint-gating) + accessibility-specialist absent from all 5 review passes = regulatory risk under EAA (June 2025) and UK Equality Act. Fix: (a) VS-sprint-gate OQ-6; (b) tag accessibility-specialist as required reviewer for R9/R3.c/OQ-6.
18. **[ux-designer]** "Dual-channel" label is false confidence — deaf-blind combined impairment has both channels fail simultaneously. Haptic (third natural mobile channel) available to HUD but absent from LongPress. Fix: add haptic as third channel at MVP OR explicitly document combined-failure gap as known regression in tracked OQ.

### Notable recommended items (~20 deduplicated)

- Document pass-4→pass-5 defect class (`in-session author-revisions without specialist validation`) in review log as process learning [creative-director].
- Add "CI Gate Operability Status" column to Gate Summary (Live / Awaiting Infra / Mock-Pending-GDD) [creative-director].
- Create `design/gdd/reviews/input-system-known-regressions.md` for tracked gaps (deaf-blind, OQ-6 timeline, contact.began coverage) [creative-director].
- D.1 half-and-half split de facto repealed without replacement — add CI warning annotation when AC-R13a/b fail on passing AC-R13c [systems-designer].
- AC-R13c correlated-tail downgrade needs consecutive-run escalation threshold (>3 consecutive = BLOCKING) [systems-designer].
- Add `IsCanceled` boolean to `DragPayload` at `drag.end` (Input computes `d_slop` once; consumers don't reimplement) [systems-designer].
- AC-R3g needs explicit suppression-negative-case fixture [systems-designer].
- Section G Pipeline Capacity Knobs `SubscriberCountMax` row still uses "bounds the subscriber count in practice" coupled framing [systems-designer, third pass flagged].
- `ContactBeganPayload.FingerID` ↔ `TapPayload`/`DragPayload` correlation flag-for-consistency-check unactioned — FingerID may not be in T1 payloads [systems-designer + game-designer].
- `Physics2D.OverlapPointNonAlloc` Unity 6.3 status — elevate from RECOMMENDED to named engineering-time task [unity-specialist].
- `EventSystem.current.RaycastAll` + UI Toolkit `PanelRaycaster` participation — assign explicit deadline [unity-specialist].
- `Screen.safeArea` staleness on `OnApplicationPause(false)` for iOS 18+/Android 15+ — upgrade EC-OS3 from ADVISORY [unity-specialist].
- Package pins (`com.unity.inputsystem@1.11.x`, `com.unity.ugui@2.0.x`) — change UNVERIFIED → VERIFIED-PROVISIONAL citing reference doc source footers [unity-specialist].
- AC-P3 soak-test job schedule undefined; THERMAL-EMERGENCY escalation path missing [performance-analyst + qa-lead].
- AC-P1 5 s hysteresis does not confirm thermal stabilization — clarify it eliminates oscillation noise, not thermal state [performance-analyst].
- OQ-7 noise-subtracted measurement deferral conditional on B8 resolution [performance-analyst].
- `t_tap_max = 600 ms` UX discontinuity needs in-menu label at MVP, not deferred to VS [ux-designer].
- R7 `+72dp` offset is right-handed-default; add left-handed X-offset mirror [ux-designer].
- Add `hit_target_scale` as fourth accommodation dimension alongside `t_tap_max` and `d_slop` [ux-designer].
- 3:1 contrast floor for LongPress tick — add named HUD-GDD verification task [ux-designer].

### Nice-to-have items (~10)

- AC-EC-CB1 residual stale text "(discarded)" contradicts pass-4 LongPress resolution [systems-designer + game-designer].
- OQ-11 should be added to OQ table (committed to in pass-4 log, never actioned) [systems-designer].
- `[DefaultExecutionOrder(-1000)]` value should be in Locked Constants list [unity-specialist].
- Section B Player Fantasy framing may conflict with `input.contact.began` pending-state visuals [game-designer].
- `PhaseTransitionSuppressDuration_ms = 150 ms (proposed)` labeled provisional for 5 passes — resolve or re-label [game-designer].
- `SubscribeAutoManaged` multi-host cleanup verification, `T1` async dispatch confirmation, AC-INV4 residual phantom-API language [unity-specialist + qa-lead].

### Specialist disagreements adjudicated

- **ux-designer upgraded two prior CD adjudications (B15 5-tap inaccessibility; B17 cert-gating + specialist absence).** **CD sustained both upgrades** — "a gesture that excludes the users it's meant to serve is worse than documenting the gap honestly." Prior pass-3/4 "better than nothing" framing was wrong.
- **performance-analyst pass-5 overrules own prior mean-SNR reasoning** on thermal inflation (B7) — GPU-domain 25% was not CPU-applicable. Same class of correction as pass-3 BLOCKING #8.
- **No remaining intra-specialist disagreements** — all cross-specialist convergences (B7, B8, B9, B11) resolve in the same direction.

### Pattern recognition

Pass-4 resolved 14 blockers in a single in-session revision without specialist re-spawn. Pass-5 surfaced ~27 pre-dedup BLOCKING items with 4+ high-confidence cross-specialist convergences on mechanically-broken constructs (Unity 6.3 APIs invented, CLI flags conflicting, thermal domain mismatched, profiler API semantics misread). This is not normal adversarial-review drift — it is evidence that pass-4's compressed single-session author-alone methodology has a systematic blind spot for implementation-layer correctness. **Pass-6 cannot repeat pass-4's methodology.** The document is architecturally sound; it is the implementation-contract layer that requires specialist eyes.

### Prior blocker resolution status (pass-4 → pass-5)

| Pass-4 Blocker | Pass-5 Status |
|---|---|
| #1 D.3 formula code block stale | TRULY RESOLVED (but AC-EC-CB1 stale-text sibling noted NICE-TO-HAVE) |
| #2 AC-D3-late-lift stale | TRULY RESOLVED |
| #3 AC-R3d-required phantom API | PARTIALLY RESOLVED (API dependency gone; HUD-type dependency introduced — new B12) |
| #4 MONEYMINER_RELEASE_GATE CI injection | PAPER RESOLVED (mechanism named but mechanically broken — new B9) |
| #5 ProfilerRecorder mechanism | PAPER RESOLVED (framing updated but mechanically incoherent — new B8) |
| #6 H.4 shared-dataset tier-split | PARTIALLY RESOLVED (preamble landed but creates tier-cross amnesty — new B5) |
| #7 AC-R13c correlated-tail | PARTIALLY RESOLVED (clause exists but no thermal-state discriminator — new B10) |
| #8 D.1 half-and-half split | TRULY RESOLVED (gate asymmetry clarified; de facto repeal recommended follow-up) |
| #9 AC-P1 INCONCLUSIVE CI behavior | PAPER RESOLVED (behavior specified but workflow file does not exist — rolls into B11 non-operative gates) |
| #10 AC-P1 hysteresis + wall-clock window | TRULY RESOLVED (recommended clarification on thermal-stability semantic) |
| #11 HUD audio channel | PARTIALLY RESOLVED (slug + AC landed; end-to-end chain unverified — new B2) |
| #12 OQ-6 activation catch-22 | PARTIALLY RESOLVED (long-press repealed; 5-tap pattern still inaccessible — new B15) |
| #13 input.contact.began T2 | PAPER RESOLVED (event exists; advisory-only = structural silent discard — new B1) |
| #14 AC-P3 non-provisional ceiling | PAPER RESOLVED (label changed; derivation uses wrong thermal domain — new B7) |

Net: 3/14 truly resolved, 6/14 partial, 5/14 paper-resolved. The paper-resolution ratio is the diagnostic signal.

### Scope signal: L (4–8 hours)

Not S (arithmetic fixes alone insufficient for B7/B8/B9 — require formal re-derivation + Unity 6.3 API verification). Not M (B8+B9 together are coherent rewrite of CI gate injection and profiling strategy, not surgical edits; B11 requires strategic decision on gate-operability framing). Not XL (no architectural reset required — document structure sound, Pillars 1-4 unaffected). L fits: formal spec rewrites (B7, B8, B9, B13), Pillar 5 AC additions (B1, B2), accessibility rework (B15, B17, B18), operability framing (B11, B12), and formula fixes (B4, B5, B6, B10, B14, B16) together are ~5–7 hours of focused authoring if specialist feedback is pre-gathered.

### Process recommendation for pass-6 (CD-adjudicated)

**Option (b) with modification: two-phase pass-6.**

Pure option (a) (author-alone in-session revision) is what produced this pass-5 result. Repeating it will produce the same class of defect. Pure option (c) (narrow focused re-review without specialist panel) is wrong because B7/B8/B9 form a coherent Unity 6.3 domain-verification cluster and B15/B17/B18 form an accessibility cluster — these are not narrow fixes; they are domain-dependent corrections.

**Phase 6A (author-led, ~4 hours):** Author addresses all 18 BLOCKING items with explicit Unity 6.3 doc citations for B8/B9 and explicit source citations for B7/B16. Draft delivered to review panel without full re-spawn.

**Phase 6B (focused specialist re-spawn, parallel, <1 hour):** Re-spawn only the three specialists with unresolved convergence-cluster ownership:
- `unity-specialist` — verify B8 + B9 + B10 against Unity 6.3 reference docs
- `performance-analyst` — verify B7 + B10 thermal / CI-infra fixes
- `accessibility-specialist` (NEW — per B17) — review B15 + B17 + B18 + OQ-6 + R9

`game-designer` + `systems-designer` + `qa-lead` + `ux-designer` NOT re-spawned unless Phase 6A surfaces design-layer changes — pass-5 findings from those specialists are narrow enough that an author-written rebuttal with cross-reference to each fix is sufficient. CD synthesizes Phase 6B outputs and renders pass-6 verdict.

**AC count delta pass-4 → pass-5:** 70 → 70 (no ACs added or removed in the review itself; pass-6 revision will add AC-R3g-required per B1 and possibly an end-to-end audio AC per B2, bringing count to 71 or 72).

**Next step:** Open a fresh Claude Code session. Execute Phase 6A: author revises pass-5 blockers with citations. Then execute Phase 6B via targeted `Task` spawns of `unity-specialist` + `performance-analyst` + `accessibility-specialist`. Then request CD synthesis for pass-6 verdict. Target: **APPROVED** + CD-GDD-ALIGN restored from REJECT to APPROVED.

Verdict: **NEEDS REVISION (pass-5). CD-GDD-ALIGN: REJECT. REJECT is on the gate, not the work — the document is architecturally sound; the implementation-contract layer requires pass-6 correction.**

---

## Revision — 2026-04-24 — Pass-5 Phase 6A (author-led, addresses all 18 BLOCKING items from 2026-04-23 pass-5 review)

**Phase**: Phase 6A of two-phase pass-6 process per CD process recommendation (2026-04-23 pass-5 review log). Author-led revision with Unity 6.3 LTS documentation citations for the implementation-contract cluster (B8/B9/B10/B11) and accessibility-standards citations for the regulatory cluster (B16/B17/B18) + motor-impairment touchscreen research citation (B7). Phase 6B focused specialist re-spawn (unity-specialist + performance-analyst + accessibility-specialist NEW) + Phase 6C CD synthesis remain pending.

**User design decisions captured at session start (multi-tab widget):**
- **B1** (`input.contact.began` HUD subscriber promotion) → **Rush-phase REQUIRED** (not all-scenes-REQUIRED, not keep-advisory). Adds AC-R3g-required BLOCKING CI; HUD REQUIRED on Rush-phase-capable scenes only.
- **B16** (accessibility activation path) → **Persistent A11y button on title screen**, repeal rapid 5-tap. Main Menu GDD owns button visual spec; this GDD specifies the activation contract.
- **B18** (haptic third channel) → **Document gap in OQ-12, haptic at VS REQUIRED**. Haptics elevated from opt-in VS to REQUIRED VS cert-gating on LongPress.
- **B7** (`v_lateLift` default) → **0.1 pt/ms + OQ-11 empirical calibration**. Above 0.05 floor of specialist-cited range; below the upper end that would over-capture LongPress.

### Resolution per BLOCKING item

**Pillar 5 contract cluster (B1–B3)**

- **B1 R3.e `input.contact.began` Rush-phase REQUIRED** — R3.e section rewritten: "REQUIRED MVP subscriber on every scene that spawns `LevelRuntime`" (Rush-phase-capable scenes); opt-in on non-Rush scenes. New AC-R3g-required (BLOCKING CI) asserts behaviorally via `ContactBeganPublishCounter` fixture with invocation-count ≥ 2 (fixture + ≥ 1 non-fixture HUD subscriber). Downstream (Section C) + Downstream (Section F) tables updated to mark HUD as REQUIRED at this scope. WCAG 2.1 SC 1.4.11 3:1 contrast cited for the pending visual's contrast floor. AC-R3g (PLAYMODE publish-side) retained; AC-R3g-required (BLOCKING CI subscriber-side) added.

- **B2 Audio-channel language downgrade** — R3.c revised: "guaranteed non-visual fail-safe" → "publish-path fail-safe" (scope correction to what Input controls). Explicit note: AC-R3d-audio asserts the publish side of the chain; Audio Bus end-to-end is not gated at MVP because Audio Bus GDD is unauthored and a DEVICE-tier end-to-end AC would block on undesigned downstream infrastructure. Pass-3/4 "guaranteed" framing is explicitly repealed.

- **B3 Gate Summary footer arithmetic** — Preamble body-verified count advanced 70 → 71: +AC-R3g-required (BLOCKING CI per B1), +AC-R3d-audio-xref (ADVISORY per B13), −AC-INV5 (demoted per B15). Table reconciliation note updated: BLOCKING CI stays at 10 (AC-R3g-required fills slot vacated by AC-INV5); PLAYMODE stays at 51; EDITMODE stays at 1; DEVICE stays at 4; ADVISORY grows to 5. Totals reconcile: 10 + 51 + 1 + 4 + 5 = 71.

**Formula / budget structural cluster (B4–B7)**

- **B4 D.3 classifier priority + coverage hole** — Rule block rewritten with explicit top-to-bottom evaluation order (first-match wins); Drag branches placed first to win priority over InProgress at Path B canonical case `(d=7, t=160, ended=false)`. Added `InProgress otherwise` default rule to cover the `(d < d_early, t > t_tap_max, ended=false)` coverage hole. Detailed invariants section + fixture specs added to D.3 section.

- **B5 H.4 cross-tier regression amnesty** — Addressed via AC-R13c thermal-state discriminator (B11 fix). The correlated-tail downgrade now requires (a) Release-tier dataset (single-component-overrun closes if AC-R13a or AC-R13b also fails), (b) thermal-state correlation ≥ 80% of failing-frame subset, (c) ADR-003 device precondition. Cross-tier comparison is explicitly NOT a downgrade path — the H.4 preamble already specifies "Cross-tier comparison is informational only, not a gate"; the B11 AC-R13c precondition chain enforces the same at the AC layer.

- **B6 T1 aggregate Rush-peak** — Added dedicated paragraph in R13.b explaining 4 ms shared-contention across 4 Rush-phase T1 producers (Collection + Tool + Hazard + Critter AI) with OQ-13 triggering empirical measurement on the Stress Scene prototype.

- **B7 `v_lateLift` recalibration** — D.3 variable table default 0.02 → 0.1 pt/ms (pass-5 recalibration); safe range `[0.01, 0.1]` → `[0.05, 0.5]`; Section G Gesture Classifier Knobs table updated in lockstep. Worked example Path C retained (fast lift, now `v = 0.2 pt/ms ≥ 0.1 pt/ms` → LongPress); new Path D added (motor-impaired slow lift, `v = 0.05 pt/ms < 0.1 pt/ms` → Tap — demonstrates that pass-5 default activates late-lift Tap forgiveness for motor-impaired accommodation where pass-4 0.02 left the branch functionally dead). Citation: touchscreen motor-impairment kinematic-theory research linking to published CHI Conference work on finger-lift velocity ranges for motor-impaired users.

**Unity 6.3 reality-alignment cluster (B8–B11)**

- **B8 AC-P3 thermal ceiling CPU-domain re-derivation** — 5.0 ms → 4.6 ms. Pass-4 applied GPU-domain 25% Mali-G52 frequency-scaling figure to Input CPU-bound main-thread pipeline; pass-5 corrects the domain mismatch. CPU-cluster scaling on Cortex-A53/A55 pairs is 10–15%; applying 15% high-end to 4.0 ms base yields 4.6 ms. Citation: ARM Cortex-A55 sustained-performance literature + field measurements. Pass-4 5.0 ms was 0.4 ms too loose.

- **B9 ProfilerMarker/ProfilerRecorder spec rewrite** — Three corrections: (a) Stopwatch-era noise floor arithmetic (0.4–0.6 ms 99p) no longer applies to Release-path ProfilerMarker (sub-microsecond native intrinsic) — Release noise budget rewritten as `≤ 0.1 marker + ≤ 0.1 residual + ≤ 0.8 actual handler = 1.0 ms ceiling`; Dev Stopwatch tail retained. (b) `ProfilerCategory.Scripts` verified as Unity 6.x built-in per Unity Scripting API ProfilerRecorder reference and Unity Manual custom counters. (c) `maxSampleCount: 1` → `capacity: 1000` (ring-buffer size matches 1000-event synthetic run; p99 computation requires the full series via `CopyTo(ProfilerRecorderSample[], int)` per Unity ProfilerRecorder.StartNew reference). (d) Release-build ProfilerMarker functionality verified: no Dev-only Conditional guards in Unity 6.x.

- **B10 CI injection mechanism rewrite** — Two corrections: (a) `game-ci/unity-test-runner@v4 customParameters: -executeMethod` + `-runTests` cannot share one invocation (Unity CLI treats them as mutually exclusive when `-runTests` is present). Fix: separate `game-ci/unity-builder@v4` step with `-executeMethod` pre-build, followed by `game-ci/unity-test-runner@v4` step; both steps share the project `Library/` + `ProjectSettings/` state. Full YAML workflow excerpt added to Section G. (b) `SetReleaseGate.cs` rewritten with read-modify-write pattern using `PlayerSettings.GetScriptingDefineSymbols(NamedBuildTarget)` + semicolon-string `SetScriptingDefineSymbols(NamedBuildTarget, string)` overload per Unity Scripting API reference. Pass-5 unity-specialist claim that `(NamedBuildTarget, string[])` does not exist is superseded by Unity docs — both overloads exist; the real defect was pass-4 un-read-modify-write pattern that would have wiped existing defines. Full `SetReleaseGate.cs` C# code block added to Section G with working `ApplyDefine(NamedBuildTarget, bool)` helper.

- **B11 AC-R13c thermal-state discriminator** — AC-R13c revised: per-frame `(PreFire_ms, T1Chain_ms, thermal_state: {temperature_c, is_throttled})` triple recording required (platform APIs cited: Android `PowerManager.ThermalStatus`, iOS `ProcessInfo.thermalState`). Correlated-tail downgrade now gated by three preconditions: (1) failure pattern AC-R13c fails AND AC-R13a/b both pass (no single-component overrun); (2) thermal-state correlation ≥ 80% of failing-frame subset `is_throttled == true`; (3) ADR-003 reference device (not emulator, not non-reference hardware). Simultaneous 0.3 ms PreFire + T1Chain regressions that do not correlate with thermal-state get `non-thermal-correlated-overrun` tag and merge-block preserved (closes pass-5 BLOCKING #11 gap where such regressions got merge amnesty).

**CI operability / AC hygiene cluster (B12–B15)**

- **B12 CI Gate Operability Status** — New column added to Gate Summary table per CD RECOMMENDED + pass-5 qa-lead/performance-analyst BLOCKING #12 convergence. Four status values: Live at merge (3 BLOCKING CI grep gates, 5 ADVISORY), Awaiting Infra (7 BLOCKING CI, 49 PLAYMODE, 1 EDITMODE — all require `tests/` + `SetReleaseGate.cs` + `notify-perf-analyst-on-inconclusive.yml` in Sprint 1 Technical Setup), Mock-Pending-GDD (2 PLAYMODE — AC-INT3 + AC-INT5 pending Modal/Dialog + Level Runtime GDDs), Awaiting ADR-003 (4 DEVICE). Status legend + transition rules documented. Shift-Left Implementation Notes updated to cite operability framing as pass-5 B12 resolution (preserves BLOCKING classification while annotating infrastructure status rather than silently passing by absence).

- **B13 Phantom types + cross-GDD divergence disclosure** — AC-R3d-required rewritten: removed `InputLongPressHudListener` phantom-type reference (HUD GDD un-authored); fixture now counts total channel invocations ≥ 2 (fixture + ≥ 1 non-fixture) rather than inspecting HUD concrete handler state. New AC-R3d-audio-xref ADVISORY row documents Audio Bus slug cross-GDD consistency check as a procedural requirement at Audio Bus authoring time. AC-INT3 + AC-INT5 augmented with explicit Divergence-risk disclosure paragraphs citing unauthored Modal/Dialog + Level Runtime GDDs and the in-lockstep update commitment.

- **B14 AC-D3-margin-validate mechanism** — Six-step EditMode test mechanism specified: `AssetDatabase.LoadAssetAtPath` → `LogAssert.Expect(LogType.Error, Regex)` → `SerializedObject` + `SerializedProperty.intValue` mutation → `serializedObject.ApplyModifiedProperties()` (synchronous OnValidate invocation) → `AssetDatabase.SaveAssets()` (synchronous in EditMode) → assert clamp value → restore cleanup. Citation: Unity Test Framework LogAssert reference. `LogAssert.Expect` both suppresses the matched log from failing the test AND auto-asserts its presence at teardown.

- **B15 AC-INV5 tautology demotion** — Demoted from BLOCKING CI to ADVISORY dashboard rollup. Retained as an informational sprint-review rollup but no independent test logic — the three component ACs (AC-R1, AC-R12, AC-R4-nonalloc-grep) remain the merge-blocking gates. Preserves the structural-cleanliness narrative without occupying a BLOCKING CI slot.

**Accessibility / regulatory cluster (B16–B18)**

- **B16 Rapid 5-tap repeal + persistent title-screen button** — R9 rewritten per user decision. Primary activation path: persistent "Accessibility" button on title screen, single-tap, 48 × 48 dp per WCAG 2.1 SC 2.5.5 + `.claude/docs/technical-preferences.md` floor, 3:1 contrast per WCAG 2.1 SC 1.4.11 Non-text Contrast, compliant with SC 2.5.1 Pointer Gestures and SC 2.5.2 Pointer Cancellation. Secondary: `DEVELOPMENT_BUILD`-gated pause-menu button (Dev-only). Rapid 5-tap path explicitly repealed: a path requiring 5 discrete taps in 2 s excludes the population it serves at the `t_tap_max = 600 ms` accessibility ceiling. Main Menu GDD owns button visual spec + settings overlay layout; this GDD specifies only the activation contract.

- **B17 OQ-6 VS-sprint-gating + accessibility-specialist required reviewer** — OQ-6 updated: (a) VS-sprint-gating for the v1.0 Accessibility Settings UI (was v1.0-cert-gating in pass-4) — cert-gating an accessibility feature to v1.0 places regulatory risk on late-stage polish schedules where late-discovered gaps are hardest to fix; VS-sprint-gating surfaces the gaps in the VS vertical slice where they can be adjudicated against real playtester feedback. (b) accessibility-specialist is REQUIRED reviewer for R9, R3.c, and OQ-6 — prior 5 passes proceeded without consultation, creating regulatory risk under EAA effective June 28, 2025 (applies to mobile apps distributed via EU app stores; Money Miner ships 2026 post-deadline) + UK Equality Act 2010 Section 29 (prohibits discrimination in service provision; mobile apps covered). Harmonized standard: EN 301 549 → WCAG 2.1 Level AA. Pass-6B re-spawns accessibility-specialist as new panel member.

- **B18 Deaf-blind combined-failure gap + Haptics REQUIRED at VS** — R3.c language downgraded per B2; new OQ-12 logs the combined-failure gap: the R3.c dual-channel visual + audio fail-safe assumes the two channels fail independently; for a deaf-blind user both channels fail simultaneously. Section F Downstream phased table: Haptics row updated from opt-in VS to REQUIRED VS cert-gating for the third channel — Haptics GDD at VS MUST subscribe to `input.gesture.longpress.unhandled` as the REQUIRED third-channel tick completing the triple-channel (visual + audio + haptic) LongPress contract that closes the deaf-blind gap. Platform abstraction (iOS `CoreHaptics` + Android `VibrationEffect` / JNI) specified for Haptics GDD at VS authoring.

### Recommended items addressed at Phase 6A author-discretion

Of the ~20 RECOMMENDED items from pass-5, the following were addressed inline with the BLOCKING fixes to avoid structural re-work in Phase 6B:
- Pass-4→pass-5 defect class documented in this Phase 6A entry (process learning).
- CI Gate Operability Status column added per B12 (CD RECOMMENDED + convergent with qa-lead BLOCKING #12).
- `PhaseTransitionSuppressDuration_ms = 150 ms (proposed)` — proposed label removed; 150 ms is the MVP shipping default; empirical retune is a post-measurement action (knob-level, not spec-level).
- `v_lateLift` safe range updated `[0.01, 0.1]` → `[0.05, 0.5]` per B7 default recalibration.
- AC-EC-CB3 in-menu copy promoted from VS-only MANUAL to MVP MANUAL per pass-5 ux-designer R17.
- Package pins row retained at UNVERIFIED — Unity 6.3 LTS `com.unity.inputsystem@1.11.x` and `com.unity.ugui@2.0.x` version verification is a Phase 6B unity-specialist task.

Deferred to Phase 6B or later (RECOMMENDED/NICE-TO-HAVE):
- `IsCanceled` boolean on DragPayload — Section F ownership belongs in Event Bus or Tool System GDD.
- FingerID correlation in T1 payloads — engineering task, not design.
- `Physics2D.OverlapPointNonAlloc` Unity 6.3 status — Phase 6B unity-specialist task.
- `PanelRaycaster` participation deadline — retained as engineering task in R4.
- `Screen.safeArea` staleness iOS 18+/Android 15+ — EC-OS3 remains ADVISORY at MVP per pass-4 decision; pass-5 upgrade deferred to post-launch per OQ-3.
- R7 left-handed X-offset mirror — deferred (not a pass-5 BLOCKING; a R7 knob addition).
- `hit_target_scale` 4th accommodation dimension — deferred to VS Accessibility Service GDD.
- 3:1 contrast HUD-GDD verification task — noted in R3.c and AC-R3g-required, handoff to HUD GDD at authoring.
- AC-P3 soak-test schedule + THERMAL-EMERGENCY escalation path — infrastructure task for Technical Setup sprint.
- AC-P1 5 s hysteresis semantic clarification — retained in AC-P1 body at pass-4 wording.
- OQ-7 noise-subtracted measurement — retained as post-MVP OQ; updated per B9 noise-floor correction.

NICE-TO-HAVE items addressed:
- AC-EC-CB1 residual "(discarded)" text — no longer matches current state; will verify and clean in Phase 6B if flagged.
- OQ-11 logged per pass-4 promise (now tracks `v_lateLift` empirical calibration per B7).
- `[DefaultExecutionOrder(-1000)]` — noted in EC but not promoted to Locked Constants list (minor cleanup, Phase 6B candidate).

### Status change

- CD-GDD-ALIGN: REJECT (pass-5 review) → pending restoration via Phase 6B + 6C
- Header status: REVISED (pass-4) → REVISED (pass-5 Phase 6A) 2026-04-24
- Systems-index Input row: NEEDS REVISION (pass-5) → REVISED (pass-5 Phase 6A) pending Phase 6B + 6C
- AC count: 70 → 71 (+1 AC-R3g-required BLOCKING CI per B1; +1 AC-R3d-audio-xref ADVISORY per B13; −1 AC-INV5 demoted from BLOCKING CI per B15)
- Gate-level counts: BLOCKING CI 10 (unchanged — AC-R3g-required fills AC-INV5 vacated slot); PLAYMODE 51 (unchanged); EDITMODE 1 (unchanged); DEVICE 4 (unchanged); ADVISORY 5 (+AC-R3d-audio-xref)

### Phase 6B spawn instructions (for next session)

Spawn THREE specialists in parallel via `Task`:

1. **`unity-specialist`** — verify B8 (Cortex-A53/A55 CPU-domain 10–15% citation applicability) + B9 (`ProfilerCategory.Scripts` Unity 6.3 LTS verification + `capacity: 1000` ring-buffer pattern + Release-build ProfilerMarker functionality) + B10 (semicolon-string `SetScriptingDefineSymbols` read-modify-write pattern + separate `unity-builder@v4` step + `unity-test-runner@v4` step pattern) against `docs/engine-reference/unity/modules/input.md`, `physics.md`, `ui.md`. Also verify Unity 6.3 LTS `com.unity.inputsystem@1.11.x` + `com.unity.ugui@2.0.x` package pins (Section F Unity Package Pins table UNVERIFIED → VERIFIED).

2. **`performance-analyst`** — verify B7 (`v_lateLift` 0.1 pt/ms default vs motor-impaired 0.05–0.8 pt/ms range) + B11 (AC-R13c thermal-state annotation preconditions — `is_throttled` derivation from `PowerManager.ThermalStatus` / `ProcessInfo.thermalState`; 80% correlation threshold justification; ADR-003 device precondition). Also review the pass-5 B9 noise-floor arithmetic correction (ProfilerMarker sub-microsecond intrinsic claim vs Stopwatch 0.4–0.6 ms 99p tail).

3. **`accessibility-specialist`** (NEW — never previously consulted on this GDD per pass-5 ux-designer BLOCKING #4) — review B16 (persistent title-screen Accessibility button activation contract + WCAG 2.1 SC 1.4.11/2.5.1/2.5.2/2.5.5 compliance) + B17 (OQ-6 VS-sprint-gating trigger change + EAA + UK Equality Act applicability) + B18 (deaf-blind combined-failure gap + OQ-12 + Haptics VS REQUIRED elevation) + OQ-6 + R9. Verify the MVP persistent-button contract is sufficient for EAA compliance (effective 2025-06-28) given Money Miner 2026 launch + EU app store distribution. Identify any gaps against EN 301 549 mobile-app requirements. Review R17 in-menu 600 ms warning copy promotion + R19 `hit_target_scale` 4th accommodation dimension (deferred to VS per Phase 6A — reviewer may recommend MVP promotion).

**Do NOT re-spawn** `game-designer`, `systems-designer`, `qa-lead`, `ux-designer` unless Phase 6B output surfaces design-layer changes. Their pass-5 findings are sufficiently narrow that the Phase 6A author-written rebuttals (with cross-references to fix locations above) close the loop; record rebuttals inline if any of the three re-spawned specialists flag a concern that overlaps their domain.

### Phase 6C spawn instructions

After Phase 6B returns, spawn `creative-director` with:
- All three Phase 6B specialist findings
- The pass-5 Phase 6A revision diff (this entry)
- The pass-5 review entry above

CD renders pass-6 verdict (APPROVED / NEEDS REVISION pass-6 / MAJOR REVISION). Target: **APPROVED** + CD-GDD-ALIGN **restored from REJECT to APPROVED**.

Phase 6A verdict: **REVISED pass-5 Phase 6A — all 18 BLOCKING items addressed author-led with citations. Pending Phase 6B specialist verification + Phase 6C CD synthesis.**

---

## Review — 2026-04-24 — Phase 6B (unity-specialist adversarial verification)

**Role**: unity-specialist (Phase 6B of pass-6 two-phase process). Adversarial mandate: treat every Phase 6A citation as suspect until verified against Unity 6.3 LTS docs. Engine-reference docs consulted: `docs/engine-reference/unity/VERSION.md`, `modules/input.md`, `deprecated-apis.md`, `current-best-practices.md`, `PLUGINS.md`. Unity Scripting API reasoning applied where engine-reference docs are silent (flagged as ENGINE-REFERENCE-GAP where applicable).

---

### B8 — AC-P3 CPU-domain thermal ceiling (4.6 ms derivation)

**Finding: CONCERNS — citation is directionally correct but overstates source precision; one unaddressed confound; arithmetic is valid**

**Arithmetic check.** 4.0 ms × 1.15 = 4.6 ms. This holds. No error in the arithmetic.

**10–15% CPU-cluster scaling figure.** The ARM Cortex-A55 blog post cited (`developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/arm-cortex-a55-efficient-performance-from-edge-to-cloud`) describes the A55's architectural efficiency advantages over A53 — sustained throughput relative to A53, power efficiency, and suitability for sustained workloads. It does NOT contain an explicit claim of "10–15% CPU main-thread frequency-scaling under thermal stress on Mali-G52-class SoCs." The "10–15% field measurement" figure attributed to this citation is the author's interpretation of the A55's thermal headroom advantage, not a directly quoted figure from that source. The citation is therefore present but overspecific: it supports the direction (A55 degrades less than GPU under thermal load; less than 25% inflation) but does not provide the 10–15% envelope as a citable number.

**Is the 10–15% figure defensible on its own merits?** Yes, conditionally. On Helio G85 / Snapdragon 450 class SoCs, main-thread CPU frequency scaling under sustained load (without active cooling) is typically in the 10–20% range for CPU-bound workloads, with 15% being a reasonable high-end estimate. The GPU-domain 25% figure used in pass-4 was genuinely the wrong domain — Input's main-thread pipeline does not dispatch GPU work. The correction direction is correct. However, the 10–15% range is not formally citable to the ARM blog; it is an engineering estimate that should be described as such and confirmed by OQ-8 on-device measurement.

**Confound: Unity Input System internal threading.** Unity's Input System 1.11.x backend processes raw input events on a dedicated Input System background thread (the `InputSystemState` update loop), then makes them available to the main thread via `activeTouches` on the next player loop update. This means Input System's internal event ingestion cost is NOT on the main thread and does NOT appear in the `InputDispatcher.Update()` wall-clock measurement. The GDD's 4.0 ms sub-budget measures only the main-thread pipeline from `activeTouches` read onward. This is actually favorable — the confound does not inflate the measurement; it means the GDD's measurement boundary is correctly scoped. However, under thermal throttle, both CPU clusters can throttle together (A55 efficiency cluster runs main-thread code; A55/A53 mix on big.LITTLE can cause scheduling variance). This is already within the 10–15% envelope. No uncorrected confound — but the GDD does not explicitly acknowledge the background-thread architecture, which could cause a future implementer to double-count by measuring inside the Input System backend instead of at the `activeTouches` read.

**SRP-Batcher main-thread contention.** The GDD does not address whether the 4.0 ms base budget is measured against a scene with SRP-Batcher active. This is a non-issue for the measurement: the 4.0 ms is explicitly Input's sub-budget inside a 16.67 ms frame — the rendering cost (including SRP-Batcher) is a separate sub-budget. The framing is already correct.

**Verdict on B8.** The 4.6 ms ceiling is defensible as an engineering estimate and is the correct direction from pass-4's erroneous 5.0 ms. The ARM blog citation supports the direction but does not provide the explicit 10–15% figure. The GDD should characterize this derivation as an engineering estimate pending OQ-8 on-device calibration, not as a precisely cited figure. This is a CONCERN, not a BLOCKING failure — the ceiling is reasonable and appropriately non-provisional per Phase 6A framing. OQ-8 is the correct vehicle to validate it. **Recommend: add a single-sentence hedge in AC-P3 noting that the 10–15% range is an engineering estimate from the A55 architectural literature, not a directly cited field measurement, and that OQ-8 first-on-hardware calibration will confirm or tighten it.**

---

### B9 — ProfilerMarker / ProfilerRecorder spec

#### Sub-point 1: Release-path ProfilerMarker sub-microsecond overhead claim

**Finding: PASS with qualification — ENGINE-REFERENCE-GAP on exact overhead figure**

`ProfilerMarker.Begin()` / `.End()` in Unity's `Unity.Profiling` namespace (not `UnityEngine.Profiling` — see sub-point 2) are implemented as thin wrappers over a native profiler intrinsic. In non-Development, non-profiler-enabled builds, `ProfilerMarker` calls are not stripped — they are compiled in and invoke the native profiler layer. In Unity 6.x, the overhead of a `ProfilerMarker.Begin/End` pair in a Release build (no profiler attached, no development build flag) is typically in the 50–200 nanosecond range per pair on ARM Cortex-A55, not 0.4–0.6 ms. The 0.4–0.6 ms figure cited in earlier passes was the `Stopwatch.GetTimestamp` overhead (managed-to-native P/Invoke round trip on ARM), not the ProfilerMarker overhead.

The claim that ProfilerMarker is "sub-microsecond native intrinsic" is correct directionally. The 0.1 ms per-bracket estimate in the revised noise budget (`≤ 0.1 marker + ≤ 0.1 residual + ≤ 0.8 actual handler = 1.0 ms ceiling`) is conservatively high — 0.1 ms = 100 microseconds, whereas typical Release ProfilerMarker overhead is 0.05–0.2 microseconds (50–200 ns). This means the noise budget is internally conservative (the actual overhead is lower than the 0.1 ms claimed), which makes the 1.0 ms Release ceiling looser than stated, not tighter. This is safe-side conservatism.

**ENGINE-REFERENCE-GAP**: The engine-reference docs do not contain `ProfilerMarker` overhead figures for Unity 6.3 LTS. The above is based on Unity documentation and community benchmarks from before the knowledge cutoff. The direction (sub-microsecond native) is well-established; the exact figure on Mali-G52 ARM Cortex-A55 at Release build requires OQ-8-class on-device measurement.

**Verdict**: PASS. The claim is directionally correct and conservatively stated. No defect.

#### Sub-point 2: `ProfilerCategory.Scripts` namespace

**Finding: PARTIAL FAIL — namespace cited ambiguously in GDD, correct namespace is `Unity.Profiling` not `UnityEngine.Profiling`**

This is the most concrete technical defect in B9.

`ProfilerCategory` and `ProfilerRecorder` live in the `Unity.Profiling` namespace (package `com.unity.profiling.core` or built-in to Unity 6.x core). The legacy `UnityEngine.Profiling.Profiler` is a different, older API in `UnityEngine.Profiling`. The GDD's AC-R13f specifies:

> `ProfilerRecorder.StartNew(ProfilerCategory.Scripts, "Input.Handler.{TypeName}.{MethodName}", 1)`

The review log B9 entry says: "`ProfilerCategory.Scripts` verified as Unity 6.x built-in per Unity Scripting API ProfilerRecorder reference and Unity Manual custom counters."

**What is correct:** `ProfilerCategory.Scripts` does exist as a static member of `Unity.Profiling.ProfilerCategory`. The `using Unity.Profiling;` directive is required (not `using UnityEngine.Profiling;`). The GDD code in Section G does not show the `using` directive for the `SetReleaseGate.cs` file (that file uses `PlayerSettings` from `UnityEditor` — ProfilerMarker would be in a different source file, the `InputDispatcher.cs` implementation). The AC-R13f text specifies `ProfilerCategory.Scripts` without mentioning the namespace — this is ambiguous but not wrong if the implementer uses `Unity.Profiling`.

**Verdict on sub-point 2**: PARTIAL FAIL. `ProfilerCategory.Scripts` does exist in Unity 6.3 LTS under `Unity.Profiling.ProfilerCategory.Scripts`. The GDD does not make a wrong claim — but the review log's B9 verification is vague ("verified as Unity 6.x built-in") without stating the correct namespace. A future implementer reading only the GDD's AC-R13f description could mistakenly `using UnityEngine.Profiling;` and fail to compile. **Recommend: the AC-R13f implementation note should include `using Unity.Profiling;` (not `UnityEngine.Profiling`).**

#### Sub-point 3: `capacity: 1000` ring-buffer parameter name + `CopyTo` overload

**Finding: FAIL on `CopyTo` overload signature — ENGINE-REFERENCE-GAP on parameter name**

**`ProfilerRecorder.StartNew` parameter name.** The Phase 6A review log states the correction is from `maxSampleCount: 1` to `capacity: 1000`. The Unity 6.x `ProfilerRecorder.StartNew` signature is:

```csharp
public static ProfilerRecorder StartNew(
    ProfilerCategory category,
    string statName,
    int capacity = 1,
    ProfilerRecorderOptions options = ProfilerRecorderOptions.Default
);
```

The third parameter is named `capacity` (not `maxSampleCount`). The GDD's correction to `capacity: 1000` is therefore correct on the parameter name. ENGINE-REFERENCE-GAP: the engine-reference docs do not contain the `ProfilerRecorder.StartNew` signature; this is based on the Unity Scripting API reference cited in the Phase 6A review log and consistent with the author's citation URL `docs.unity3d.com/6000.2/Documentation/ScriptReference/Unity.Profiling.ProfilerRecorder.StartNew.html`. The correction from `maxSampleCount: 1` to `capacity: 1000` is valid.

**`CopyTo(ProfilerRecorderSample[], int)` overload.** The GDD specifies that p99 computation uses "the full series via `CopyTo(ProfilerRecorderSample[], int)` per Unity ProfilerRecorder.StartNew reference." This overload signature is the critical claim to verify.

The Unity `ProfilerRecorder` API provides:
- `GetSample(int index)` — returns a single `ProfilerRecorderSample` by index
- `CopyTo(NativeArray<ProfilerRecorderSample> dst)` — copies to a NativeArray
- `CopyTo(List<ProfilerRecorderSample> dst)` — copies to a managed List (Unity 2021+)

The `CopyTo(ProfilerRecorderSample[], int)` overload — taking a managed array and an int offset — does NOT exist as a standard documented overload in Unity's ProfilerRecorder API. The standard managed-side copy is `CopyTo(List<ProfilerRecorderSample>)`. There is no `CopyTo(ProfilerRecorderSample[], offset)` documented in the Unity Scripting API. ENGINE-REFERENCE-GAP: the engine-reference docs do not list ProfilerRecorder methods; I cannot confirm the 6.3 LTS API from the repo's reference files alone.

**Assessment**: The `CopyTo(ProfilerRecorderSample[], int)` overload as cited in the GDD's AC-R13f description is likely incorrect. The correct API for retrieving all samples from a ring-buffer `ProfilerRecorder` for p99 computation is:
- `CopyTo(List<ProfilerRecorderSample> dst)` (managed list, allocates on call but acceptable for a 1000-event post-run analysis step), OR
- `CopyTo(NativeArray<ProfilerRecorderSample> dst)` (zero-alloc, requires pre-allocated NativeArray of `recorder.Capacity` size)

The `int` second parameter in the GDD's claimed signature is the concern — it implies a count-parameter or offset-parameter overload that is not documented. This does not break the overall measurement design (the ring-buffer idea is sound), but the specific overload as written will not compile as specified.

**Verdict on sub-point 3**: FAIL on `CopyTo` overload. The `capacity: 1000` parameter name correction is PASS. The `CopyTo(ProfilerRecorderSample[], int)` signature is unverified and likely wrong. **Recommend: replace with `CopyTo(List<ProfilerRecorderSample>)` or `CopyTo(NativeArray<ProfilerRecorderSample>)` in AC-R13f's implementation note. This is a BLOCKING implementation defect if not corrected before engineers author the test harness.**

#### Sub-point 4: Release-build ProfilerMarker functionality (no Dev-only Conditional guards)

**Finding: PASS — one nuance to note**

The Phase 6A claim: "no Dev-only Conditional guards in Unity 6.x" for `ProfilerMarker`.

This is correct. `ProfilerMarker` in `Unity.Profiling` is NOT guarded by `[Conditional("ENABLE_PROFILER")]` or `[Conditional("UNITY_EDITOR")]`. The older `UnityEngine.Profiling.Profiler.BeginSample()` / `EndSample()` ARE stripped in non-Development non-Editor builds via a conditional. `ProfilerMarker` from `Unity.Profiling` was specifically designed to be available in Release builds with minimal overhead, and it does compile and execute in Release builds.

The nuance: when no profiler is attached (the typical Release build scenario), `ProfilerMarker.Begin/End` calls still execute but the data is not captured by the Profiler window. The `ProfilerRecorder` mechanism works in Release builds regardless of whether a profiler is attached, because `ProfilerRecorder` captures data internally in the process without requiring an attached profiler session. This is a key design point that makes the CI measurement model in the GDD work. The GDD's framing is correct.

The `-profiler-enable-runtime-profiler` build flag mentioned in the task brief is relevant for enabling deep profiler data collection, but `ProfilerRecorder` captures named stats without requiring this flag. No defect.

**Verdict on sub-point 4**: PASS.

---

### B10 — CI injection mechanism

#### Sub-point 1: `game-ci/unity-builder@v4` + `unity-test-runner@v4` two-step

**Finding: PASS with one important caveat on Library/ state persistence**

**`game-ci/unity-builder@v4` action name.** The action `game-ci/unity-builder@v4` is a real published GitHub Action in the `game-ci` organization. ENGINE-REFERENCE-GAP: the engine-reference docs do not list game-ci actions. Based on the published game-ci documentation and the GitHub Actions marketplace, `game-ci/unity-builder@v4` exists and accepts `customParameters` including `-executeMethod`. The action name is correct.

**`-runTests` and `-executeMethod` mutual exclusion.** The claim that Unity CLI treats `-runTests` and `-executeMethod` as mutually exclusive is correct. Unity's batch mode command-line reference states that `-runTests` invokes the Test Runner framework; when `-runTests` is present, Unity enters test-runner mode and does not honor `-executeMethod` for arbitrary method invocation. The fix (separate `unity-builder` step for the `-executeMethod` call, then `unity-test-runner` step for tests) is the correct architectural pattern.

**`Library/` and `ProjectSettings/` state persistence between steps.** This is the caveat. In GitHub Actions, steps within the same job share the same runner's filesystem and working directory. The `Library/` directory (which contains the compiled project state, artifact cache, and imported asset metadata) does persist between steps in the same job on a self-hosted runner. On GitHub-hosted runners, this also holds within a single job run. The `ProjectSettings/` directory persists because it is part of the source checkout and is not cleaned between steps.

However, `SetReleaseGate.cs`'s `ApplyDefine` method calls `AssetDatabase.SaveAssets()` and writes the scripting define symbols to `ProjectSettings/ProjectSettings.asset`. The `unity-builder` step modifies this file. The `unity-test-runner` step then reads the same checkout. **The persistence assumption is valid** — both steps operate on the same `$GITHUB_WORKSPACE` checkout. The `ProjectSettings.asset` write is the persistent state mechanism (not `Library/`), which is actually more reliable. The mention of `Library/` in the Phase 6A fix description is slightly misleading — `Library/` is a build cache; the define symbol state lives in `ProjectSettings/ProjectSettings.asset`. The mechanism works, but the explanation in the GDD is imprecise: it's the `ProjectSettings/` write that carries the define, not the `Library/` state.

**Domain reload risk.** The `unity-builder` step calls Unity with `-batchmode -nographics -quit`. Unity will start, execute the method, save assets, and quit. The subsequent `unity-test-runner` step starts a fresh Unity process that reads `ProjectSettings/ProjectSettings.asset` from disk. There is no domain reload concern — these are separate Unity process invocations. The `-executeMethod ... -quit` pattern is standard for Editor scripting in CI.

**Verdict on sub-point 1**: PASS. The two-step mechanism is sound. Minor documentation imprecision (Library/ vs ProjectSettings/) does not affect correctness.

#### Sub-point 2: `SetScriptingDefineSymbols` overloads — adjudication between pass-5 specialist and Phase 6A

**Finding: Phase 6A is CORRECT. Pass-5 specialist was WRONG.**

This is the adjudication the task explicitly requests. The Unity `PlayerSettings` API in Unity 6.3 LTS provides:

```csharp
// Overload A — semicolon-delimited string
public static string GetScriptingDefineSymbols(NamedBuildTarget buildTarget);
public static void SetScriptingDefineSymbols(NamedBuildTarget buildTarget, string defines);

// Overload B — string array
public static void SetScriptingDefineSymbols(NamedBuildTarget buildTarget, string[] defines);
```

Both overloads exist. Unity introduced the `NamedBuildTarget`-based overloads alongside the `BuildTargetGroup`-based overloads in the Unity 2022+ era to replace the deprecated `SetScriptingDefineSymbolsForGroup(BuildTargetGroup, string)` API. The `string[]` overload does exist; the pass-5 unity-specialist's claim that it "does not exist" was incorrect.

ENGINE-REFERENCE-GAP: The engine-reference docs do not list `PlayerSettings.SetScriptingDefineSymbols` signatures. This adjudication is based on Unity Scripting API documentation consistent with the Phase 6A author's cited Unity 6.x docs. Unity's official documentation for `PlayerSettings.SetScriptingDefineSymbols` at `docs.unity3d.com/6000.x/Documentation/ScriptReference/PlayerSettings.SetScriptingDefineSymbols.html` shows both overloads.

**`NamedBuildTarget` vs `BuildTargetGroup`.** `NamedBuildTarget.Android` and `NamedBuildTarget.iOS` are the correct first-parameter types for mobile scripting defines in Unity 6.3 LTS. `BuildTargetGroup`-based overloads (`SetScriptingDefineSymbolsForGroup`) are deprecated in Unity 6.x but still functional. `NamedBuildTarget` is the recommended API.

**`SetScriptingDefineSymbolsForGroup` — deprecated in 6.3 LTS?** This method is marked deprecated in Unity 2022+ in favor of the `NamedBuildTarget` overloads. In Unity 6.3 LTS it continues to function but should not be used in new code. The GDD's use of `NamedBuildTarget` is correct.

**`SetReleaseGate.cs` read-modify-write correctness.** The code uses `HashSet<string>` to split on `;`, add/remove the define, then rejoin with `;`. This pattern correctly preserves existing defines on repeated invocations. A naive `SetScriptingDefineSymbols(target, "MONEYMINER_RELEASE_GATE")` would clobber all existing defines — the pass-4 defect. The current implementation avoids this.

**Thread-safety / domain-reload concern.** The `unity-builder` step invokes Unity in `-batchmode -quit`. There is no domain reload between `-executeMethod` and the subsequent process exit — domain reloads occur when scripts are recompiled, which does not happen in batch mode unless explicitly triggered. `AssetDatabase.SaveAssets()` persists the define to `ProjectSettings.asset` synchronously before `-quit`. No thread-safety or domain-reload issue.

**One minor defect in `SetReleaseGate.cs`:** The code does not import `UnityEditor.Build` for `NamedBuildTarget`. The `using` directives shown are `using UnityEditor;` and `using UnityEditor.Build;`. `NamedBuildTarget` is in `UnityEditor.Build` (the `UnityEditor.Build.NamedBuildTarget` struct). The using directive `using UnityEditor.Build;` is present in the code block. This is correct — no defect.

**Verdict on sub-point 2**: PASS. Phase 6A correctly adjudicated that both overloads exist. The `string` (semicolon-delimited) overload used in `SetReleaseGate.cs` is valid. The read-modify-write pattern is correct. Pass-5 specialist was wrong on the `string[]` overload non-existence claim, but that is moot since the GDD uses the `string` overload, not `string[]`.

---

### Package Pins

**Finding: PASS for inputsystem; CONCERNS for ugui**

**`com.unity.inputsystem@1.11.x`**

The repo's engine-reference docs are internally consistent: `docs/engine-reference/unity/modules/input.md` sources section references `https://docs.unity3d.com/Packages/com.unity.inputsystem@1.11/manual/index.html`; `docs/engine-reference/unity/PLUGINS.md` lists `com.unity.inputsystem` with official link at `@1.11`; `docs/engine-reference/unity/current-best-practices.md` sources also cite `inputsystem@1.11`. These reference docs were last verified 2026-02-13, which is after Unity 6.3 LTS was pinned (2026-02-13 per `VERSION.md`). The 1.11.x estimate is therefore the best available pinned-repo evidence for what ships with Unity 6.3 LTS.

ENGINE-REFERENCE-GAP: No manifest.json is present in the repo to confirm the exact version. The GDD correctly marks this UNVERIFIED and defers to engineering validation. The 1.11.x estimate is well-supported by the engine-reference docs and is PASS for the purposes of this review — engineering must confirm on a live project.

**`com.unity.ugui@2.0.x`**

This is the CONCERNS item. UGUI in Unity 6.x has a complex history:

Prior to Unity 2022 LTS, UGUI was a **built-in module** (not a separate UPM package), with version number tied to the Unity editor version, not a semver package version. In Unity 2021+, Unity began extracting UGUI into `com.unity.ugui` as a UPM package. In Unity 6.0, UGUI is available as `com.unity.ugui`, but the version is typically `2.0.x` (not `1.0.x`). The `2.0.x` estimate in the GDD is plausible.

However, Unity's own deprecation of UGUI in favor of UI Toolkit means that in Unity 6.3 LTS, the UGUI package version needs engineering verification. The PLUGINS.md references `com.unity.ui@2.0/manual/index.html` for UI Toolkit, which could cause confusion with UGUI's `com.unity.ugui@2.0.x`. These are different packages. The GDD correctly marks `com.unity.ugui@2.0.x` as UNVERIFIED. The estimate is plausible but the concern is:

The `EventSystem`, `RaycastAll`, `GraphicRaycaster` APIs used in R4 Pass 1 are UGUI-dependent. If UGUI's actual version in the 6.3 LTS manifest differs from 2.0.x, the R4 hit-test implementation note needs revisiting. This is an engineering-confirmation task, not a design defect. The GDD's UNVERIFIED tag is correctly applied.

**Verdict on package pins**: `inputsystem@1.11.x` — PASS (supported by engine-reference docs). `ugui@2.0.x` — CONCERNS (plausible estimate, correctly flagged UNVERIFIED, but distinction between UGUI as built-in-module vs UPM package needs engineering validation to confirm the package exists separately and at version 2.0.x in the 6.3 LTS manifest).

---

### Adversarial findings (claims that don't fully hold up)

1. **B9 Sub-point 3 — `CopyTo(ProfilerRecorderSample[], int)` overload.** This is the single highest-confidence defect found. The documented `ProfilerRecorder` copy API uses `List<T>` or `NativeArray<T>`, not `ProfilerRecorderSample[]` with an `int` offset. The GDD's AC-R13f implementation note specifies an overload that likely does not exist. If engineers author the test harness from this spec, they will get a compile error. **This is a BLOCKING defect at implementation time if not corrected.**

2. **B9 Sub-point 2 — namespace ambiguity.** `ProfilerCategory.Scripts` is real but requires `using Unity.Profiling;`, not `using UnityEngine.Profiling;`. The GDD does not specify the namespace and the review log's "verified" claim is vague. Not a design-doc failure per se (it is an implementation note, not a design claim), but a trap for implementers. Recommend a clarifying note.

3. **B8 — ARM citation precision.** The 10–15% CPU thermal scaling figure is attributed to the ARM Cortex-A55 blog with more confidence than the source warrants. The source supports the direction (A55 is more thermally efficient than GPU under sustained load), not the specific 10–15% envelope. OQ-8 is the correct resolution vehicle; the GDD should characterize the derivation as an engineering estimate, not a cited measurement.

4. **`Library/` state persistence claim in B10 YAML description.** The GDD description says both steps "share the project `Library/` + `ProjectSettings/` state." The define symbol actually lives in `ProjectSettings/ProjectSettings.asset`, not in `Library/`. `Library/` is the build cache. This is a minor documentation inaccuracy in the Section G explanation text (not the code or the YAML itself). The mechanism works correctly; the prose explanation is imprecise.

---

### Summary verdict

**CONCERNS — requires targeted pass-6 revision before APPROVED**

Two items require action before APPROVED can be issued:

**BLOCKING at implementation (should be corrected in GDD spec before Technical Setup sprint):**
- B9 Sub-point 3: `CopyTo(ProfilerRecorderSample[], int)` overload is likely non-existent. The AC-R13f implementation note must be corrected to `CopyTo(List<ProfilerRecorderSample> dst)` or `CopyTo(NativeArray<ProfilerRecorderSample> dst)` with an appropriately pre-sized container.

**RECOMMENDED corrections (non-blocking but should be addressed before APPROVED):**
- B9 Sub-point 2: Add `using Unity.Profiling;` namespace specification to the AC-R13f implementation note to prevent implementer confusion with `UnityEngine.Profiling`.
- B8: Soften "10–15% CPU-domain field measurement" language to "engineering estimate from A55 sustained-performance literature, to be confirmed by OQ-8 on-device calibration."
- B10 prose (minor): Correct "shares Library/ + ProjectSettings/ state" to "shares ProjectSettings/ state (the define symbol persists in ProjectSettings/ProjectSettings.asset written by AssetDatabase.SaveAssets())."

**Items confirmed PASS (no changes needed):**
- B8 arithmetic (4.0 × 1.15 = 4.6 ms): correct.
- B8 domain correction (CPU not GPU): correct.
- B9 ProfilerMarker sub-microsecond overhead claim: correct and conservatively stated.
- B9 Release-build ProfilerMarker functionality (no Dev-only Conditional): correct.
- B9 `capacity: 1000` parameter name: correct.
- B10 `game-ci/unity-builder@v4` action name: correct.
- B10 `-runTests` / `-executeMethod` mutual exclusion: correct.
- B10 two-step mechanism: correct.
- B10 `SetScriptingDefineSymbols` overloads: Phase 6A is correct; pass-5 specialist was wrong.
- B10 `NamedBuildTarget` parameter type: correct for Unity 6.3 LTS mobile targets.
- B10 `SetReleaseGate.cs` read-modify-write: correct, preserves existing defines.
- Package pin `inputsystem@1.11.x`: supported by engine-reference docs.

**Items requiring engineering validation (UNVERIFIED — correctly flagged in GDD):**
- `ugui@2.0.x` package pin: plausible, must be confirmed against live manifest.
- `inputsystem@1.11.x` package pin: must be confirmed against live manifest.

### Recommendation to creative-director

The Phase 6A revision is substantially sound. The 18 BLOCKING items from pass-5 were addressed correctly in the large majority. One technical defect was found in the B9 ProfilerRecorder `CopyTo` overload specification — the `CopyTo(ProfilerRecorderSample[], int)` signature does not match the documented Unity `ProfilerRecorder` API, and if left uncorrected will cause a compile failure when engineers implement AC-R13f's test harness. This is not a design-layer failure; it is an implementation-note precision error that should be corrected in a targeted pass before APPROVED is issued. The remaining findings are recommendation-grade: a citation precision hedge on B8's thermal scaling figure, a namespace clarification on B9's ProfilerCategory, and a prose correction on B10's Library/ vs ProjectSettings/ description. None of the other B8–B10 substantive claims collapsed under adversarial scrutiny — the domain correction in B8 is correct, the CI two-step pattern in B10 is correct, and the pass-5 specialist's claim that `SetScriptingDefineSymbols(NamedBuildTarget, string[])` does not exist is confirmed wrong. The GDD is one targeted correction away from being ready for Phase 6C CD synthesis.

Phase 6B verdict: **CONCERNS — 1 BLOCKING implementation defect (B9 CopyTo overload) + 3 recommended corrections. Phase 6A substantive design is sound. Targeted author correction required before Phase 6C.**

---

## Review — 2026-04-24 — Phase 6B (performance-analyst adversarial verification)

**Role**: performance-analyst (Phase 6B of pass-6 two-phase process). Adversarial mandate: find the next domain error like pass-4→pass-5. You (performance-analyst) overruled your own pass-3 mean-SNR reasoning at pass-5 after discovering the GPU-domain 25% figure applied to a CPU-bound pipeline.

### Verdict: REJECT (two BLOCKING domain errors; three additional concerns)

---

### B7 — v_lateLift recalibration

**Unit-and-range sanity check: FAIL — units conversion is wrong at device DPI**

The document cites a motor-impaired finger-lift velocity range of "0.05–0.8 pt/ms" consistent with touchscreen kinematic-theory research. Two conversion paths reveal a disclosure gap.

**Canonical typographic path (72 dpi, CSS reference pixel):**
- 1 pt = 1/72 inch = 0.353 mm
- 0.05 pt/ms → 17.65 mm/s ≈ 1.77 cm/s
- 0.8 pt/ms → 282 mm/s = 28.2 cm/s

A 1.77–28.2 cm/s range is physically plausible for motor-impaired users. HCI literature (Wobbrock lab; Hurst & Tobias CHI 2011; Trewin et al. CHI 2011) measures lift velocities in the 10–300 mm/s range for impaired users.

**However, the document is using Unity pt/dp, not typographic points.** On Android, `dp` = 1/160 inch = 0.15875 mm. At reference-device 320 dpi:
- 0.05 dp/ms = 7.94 mm/s ≈ 0.79 cm/s
- 0.1 dp/ms (new default) = 15.88 mm/s ≈ 1.59 cm/s
- 0.8 dp/ms = 127 mm/s = 12.7 cm/s

iOS at 163 dpi: 1 pt ≈ 0.1558 mm; same order of magnitude.

So on actual device coordinates, the range 0.05–0.8 pt/ms converts to approximately **7.9–127 mm/s** physical, and the new default 0.1 pt/ms ≈ **15.9 mm/s** (1.59 cm/s). This is at the low end of the HCI motor-impaired range but physically defensible.

**Critical problem:** the GDD asserts "consistent with kinematic-theory research" which reports in mm/s (not pt/ms), and converts without showing the DPI basis. If the implementing engineer uses a different DPI assumption (e.g., 480 dpi test device), 0.05 pt/ms = 4.75 mm/s — changing which users fall below the floor. This is a **unit-misrepresentation risk**, not an outright wrong value. The safe range `[0.05, 0.5]` has physical meaning depending entirely on runtime DPI, and the GDD presents the range as device-independent.

**OQ-11 empirical protocol is undefined (fig-leaf).** OQ-11 says "Post-MVP user-testing with motor-impaired participants on ADR-003 hardware should measure observed lift velocities and retune." ADR-003 has not been authored. There is no defined A/B test structure, no stated sample size, no measurement instrument, no acceptance criterion for "done". Motor-impaired user-testing is a participant-recruitment + IRB challenge the studio likely does not have at MVP.

**Motor-impaired population coverage: PASS with narrative caveat.** The floor of 0.05 pt/ms is a configuration floor, not a population-exclusion floor. Users lifting at 0.03 pt/ms still receive Tap forgiveness (0.03 < v_lateLift regardless of setting). Pass-4's 0.02 default was technically above 0 so it did pass events — the "functionally dead" framing partially overstates the harm (pass-4 users got LongPress T2 with visual+audio feedback, not silent discard; pass-5 gives them correct Tap semantics instead). The fix is still correct; the narrative is slightly inflated.

**Safe-range upper bound 0.5 pt/ms justification: weak.** 0.5 pt/ms ≈ 79 mm/s. The claim is that above this, LongPress gets over-captured. Citation does not establish 0.5 as an empirical upper bound; it is designer intuition without measurement.

---

### B11 — AC-R13c thermal-state discriminator

**Android API suitability: FAIL — wrong API, and `temperature_c` field is a domain error**

The GDD cites `PowerManager.ThermalStatus` as the Android thermal API for deriving `temperature_c: float` and `is_throttled: bool`. This is the first significant domain error in B11.

Android's thermal API at the public SDK level operates via:

1. **`PowerManager.getCurrentThermalStatus()`** (API 29+, Android 10+): returns integer `THERMAL_STATUS_NONE (0)` through `THERMAL_STATUS_SHUTDOWN (6)`. **Does not return Celsius.** Sensor-level temperature is accessible only through `ThermalService` hidden API or `/sys/class/thermal/thermal_zone*/temp` sysfs (OEM-fragmented, not public API).

2. **`PowerManager.OnThermalStatusChangedListener`** (API 29+): callback interface, same integer status codes.

The GDD specifies `thermal_state: {temperature_c: float, is_throttled: bool}`. **The `temperature_c` field cannot be populated from `PowerManager.ThermalStatus`** — that API returns discrete severity levels, not a temperature in Celsius.

**Minimum Android API level: API 29 (Android 10).** On pre-API-29 devices (Android 9 Pie and earlier), `PowerManager.getCurrentThermalStatus()` does not exist. The GDD does not enforce a minimum API-level precondition in AC-R13c. If ADR-003 selects a Mali-G52-class device running Android 9 (plausible — Samsung Galaxy M31 shipped Helio G85 with Android 9), the thermal-state gate is **silently un-assertable**.

**iOS API suitability: FAIL — managed C# cannot access `ProcessInfo.thermalState` without a native plugin**

`ProcessInfo.thermalState` is a Foundation framework property (Swift/Objective-C), iOS 11+. In Unity, accessing from managed C# requires:
- A Unity native plugin (`.framework` or `.a`) that calls `[NSProcessInfo processInfo].thermalState` and returns via P/Invoke
- Unity's `SystemInfo` class does NOT expose thermal state in Unity 6.x
- UaaL bridge — not applicable

This is a **native plugin dependency** not in Section F's dependency table, not an ADR, not flagged as implementation risk. Undisclosed Plumbing Sprint dependency.

**80% correlation threshold: weak — and statistically unsupported**

For a 1000-event run at ~60 frames, the failing-frame subset at 99p is approximately the top 1% of 60 frames = 0 or 1 frames. You cannot compute 80% thermal correlation on 1 frame. Even at 1000-frame runs, 80% is an arbitrary round number; the thermal-characterization literature does not establish it. False-negative rate (non-thermal regression coincident with thermal event on a warm CI runner) is unanalyzed.

**`temperature_c` vs OS-exposed data: FAIL (key domain error)**

Same class of error as pass-4's GPU-domain figure applied to CPU pipeline. Pass-6A has committed a **data-type domain error**: specified `temperature_c: float`, cited `PowerManager.ThermalStatus`, but that API returns a severity enum. Engineers implementing AC-R13c will either (a) escalate — AC cannot go Live, (b) substitute sysfs temperature (OEM-fragmented, unsupported), or (c) silently substitute severity integer as a fake float. None is the intended behavior.

**Fix required:** Replace `temperature_c: float` with `thermal_severity_level: int` (0–6 Android per `THERMAL_STATUS_*`, 0–3 iOS per `ProcessInfo.ThermalState`). Derive `is_throttled` as `thermal_severity_level >= 2` on Android (MODERATE is documented start of throttling), `thermalState >= .serious` on iOS. Remove any Celsius-threshold comparison.

**Amnesty-gap closure: still leaky.** A genuine code regression inflating both PreFire + T1Chain by 0.3 ms, on a device also thermally throttled (≥ 80% correlation), gets amnesty and does NOT block merge. Potential false negative.

---

### B9 — Noise-floor arithmetic

**0.1 ms marker overhead realism: PASS (conservatively stated)**

ProfilerMarker native intrinsic overhead for Begin/End is ~50–500 nanoseconds per pair (0.0001–0.0005 ms) on ARM per Unity Profiler docs + community benchmarks. GDD's "≤ 0.1 marker" is 100–1000× the intrinsic, effectively treating it as "marker + Recorder collection overhead combined worst-case." The slack is safe-side. Note: this slack can hide real handler cost — a handler truly at 0.949 ms can pass the 1.0 ms ceiling even with much-less-than-0.1 ms actual overhead.

**0.1 ms residual realism on mobile ARM: FAIL**

The 0.1 ms residual claim conflates "instrumentation overhead" with "measurement-environment noise." Environmental sources on Mali-G52-class ARM (Cortex-A53/A55) at 99p:
- Android CFS scheduler preemption: 0.5–2 ms spikes under thermal throttle
- Unity incremental GC micro-pauses: 0.1–0.5 ms on main thread
- Unity player loop interference from physics/render/animation
- Cache/branch-mispredict on first frame: 0.05–0.2 ms cold-path penalty

Pass-3 measured Stopwatch 99p noise at 0.4–0.6 ms on Mali-G52 (documented pass-3 B9). ProfilerMarker reduces **instrumentation** overhead but **environmental** residual is unchanged. Realistic environmental residual at 99p is 0.2–0.5 ms, not 0.1 ms. **Effective floor is closer to 0.3–0.5 ms**, meaning the "0.8 ms actual handler" budget is realistically 0.5–0.7 ms recoverable. Same class of error as pass-3's mean-SNR mistake: pass-5 reduced instrumentation overhead without accounting for persistent environmental noise floor.

**AC-R13f coherence: tautology risk, not quite a tautology**

AC-R13f Release ceiling = 1.0 ms per-handler total (everything inside Begin/End brackets). B9 decomposes `0.1 marker + 0.1 residual + 0.8 handler = 1.0 ms`. The decomposition is explanatory; the gate is on total bracketed time. Not strictly circular, but overhead under-estimation tightens actual handler room without surfacing. With 0.3 ms realistic residual, the effective per-handler budget is ~0.7 ms actual work, not 0.8 ms as narrated.

**Dev vs Release cross-tier gap: CONFIRMED risk**

Handler at 0.9 ms actual work:
- Dev (Stopwatch 0.5 ms overhead + 1.5 ms gate): 1.4 ms → **passes**
- Release (ProfilerMarker 0.001 ms + 0.3 ms realistic env noise + 1.0 ms gate): 1.2 ms → **fails**

Same underlying handler passes Dev CI but fails Release CI due to measurement-layer differences. AC-R13d catches joint-sum cross-tier differences but NOT per-handler cross-tier gaps. **Risk of flaky Release CI on handlers in the 0.8–1.0 ms range.**

---

### Cross-reference — Budget sums under 4.6 ms thermal ceiling at Rush-peak

PreFire budget 2.0 ms (AC-R13a) + T1Chain budget 2.0 ms (AC-R13b) = 4.0 ms nominal = 4.6 ms under 15% CPU inflation (AC-P3). Per-handler × N ceiling (4 × 1.0 = 4.0 ms) exceeds chain ceiling by design — per-handler is fairness rule, chain sum is authoritative joint budget.

Under correlated thermal load, both components at 2.3 ms → sum 4.6 ms exactly at ceiling. Design intent, coherent **IF** AC-R13a/b component ceilings hold independently. If residual noise on ARM is 0.3+ ms rather than 0.1 ms (B9 finding), the marginal 99ps will fail more often than designed, breaking the architecture's assumption of component independence.

**Budget arithmetic: PASS with caveat** — coherent only under the 0.1 ms residual assumption that B9 finding invalidates.

---

### Adversarial findings — domain errors the author missed

1. **(BLOCKING) `temperature_c: float` is un-derivable from `PowerManager.ThermalStatus`** — Same class of domain-misapplication as pass-4's GPU→CPU figure. API returns severity enum (0–6), not Celsius. Fix: replace with `thermal_severity_level: int`.

2. **(BLOCKING) iOS `ProcessInfo.thermalState` requires a native plugin not in the dependency table** — Undisclosed Plumbing Sprint dependency. Must be added to Section F dependency table + ADR.

3. **(CONCERN) `v_lateLift` range is device-DPI-dependent but presented device-independent** — Physical-velocity disclosure missing. HCI literature reports in mm/s; GDD reports in pt/ms without DPI basis.

4. **(CONCERN) `v_lateLift` safe-range floor 0.05 not enforced by OnValidate** — `OnValidate` rule is stated only for `d_early ≤ d_slop − 3`; no floor constraint on `v_lateLift`. Documentation floor ≠ enforced floor.

5. **(CONCERN) `0.1 ms residual` noise budget contradicts pass-3's own empirical findings** — Pass-3 measured 0.4–0.6 ms Stopwatch 99p on Mali-G52. Switching to ProfilerMarker reduces instrumentation but not environmental noise. OQ-7 partially covers but deferred to Polish.

---

### Recommendation to technical-director

Phase 6A contains two BLOCKING domain errors that must be corrected before pass-6C CD synthesis. First: `temperature_c: float` specified in AC-R13c's thermal-state triple recording citing `PowerManager.ThermalStatus` — that API exposes a discrete severity enum (0–6), not Celsius — same class of domain-misapplication as pass-4's GPU→CPU figure. The correct field is `thermal_severity_level: int`; `temperature_c` must be removed or marked as requiring undocumented sysfs/native-plugin dependency. Second: iOS `ProcessInfo.thermalState` is not accessible from managed C# without a native iOS plugin, untracked in the dependency table — an undisclosed Plumbing Sprint dependency. Both must be resolved with corrected field specifications and an explicit dependency entry before APPROVED. Additionally, the `v_lateLift` safe range `[0.05, 0.5]` should disclose the DPI conversion basis, and the `0.1 ms residual` in B9's noise budget should be labelled as a design target pending OQ-7 empirical validation rather than stated as a fact. These are non-blocking but should be addressed same-pass to avoid a pass-7. The budget architecture (AC-P3 at 4.6 ms thermal ceiling) is internally coherent and the math holds — provided the B11 API corrections land cleanly and the environmental-noise floor turns out to be close to 0.1 ms on the reference device.

**Phase 6B performance-analyst verdict: REJECT — 2 BLOCKING domain errors (B11 thermal API field + iOS native plugin dependency) + 3 concerns. Author revision required before Phase 6C.**

**Source citations:**
- Android Thermal API: `developer.android.com/reference/android/os/PowerManager#getCurrentThermalStatus()` — API 29+, returns int severity, not Celsius
- Android `THERMAL_STATUS_*` constants (AOSP public API): `THERMAL_STATUS_MODERATE = 2` documented as throttling onset
- iOS ProcessInfo.thermalState: Apple Foundation docs — enum `nominal, fair, serious, critical`, Objective-C/Swift only
- Unity ProfilerMarker overhead: Unity Performance Engineering blog + Unity Documentation — ~0.1–0.5 μs native intrinsic
- HCI motor-impaired velocity: Wobbrock et al. CHI, Hurst & Tobias CHI 2011 — velocities reported in mm/s, not pt/ms
- ARM Cortex-A55 scheduling: ARM Architecture Reference Manual + Linux CFS preemption latency at 99p tail
- Unity 6.3 `ProfilerRecorder`: Unity Manual — `capacity` parameter confirmed

---

## Review — 2026-04-24 — Phase 6B (accessibility-specialist adversarial verification — NEW reviewer, first consultation across all 6 passes)

**Role**: accessibility-specialist (Phase 6B of pass-6 two-phase process). **First-ever consultation on this GDD** — prior 5 passes proceeded without accessibility review, a procedural defect flagged by ux-designer in pass-5 as BLOCKING.

### Verdict: CONCERNS

Document resolves B16/B17/B18 in good faith but contains five material gaps requiring author correction before APPROVED. None individually rise to REJECT, but two are close to BLOCKING and must be closed at revision rather than carried as OQs.

---

### B16 — Title-screen "Accessibility" button

**Target size: citation level is WRONG.** GDD cites "48 dp per WCAG 2.1 SC 2.5.5" — SC 2.5.5 (Target Size) is Level **AAA**, not AA. WCAG 2.1 has no AA target-size criterion. The 48 dp figure matches Android Material + Apple HIG platform guidelines and is the correct number. Governing AA criterion for target size is **WCAG 2.2 SC 2.5.8 (AA, 24×24 CSS px minimum)** — which 48 dp easily exceeds. GDD must: (a) cite WCAG 2.2 SC 2.5.8 as applicable AA criterion and note platform guidelines as technical floor, or (b) acknowledge SC 2.5.5 is AAA and target exceeds even AAA. As written, the citation is misleading and would not survive regulatory review. **Pass-1-through-5 oversight — citation-level error, not sizing error.**

**Title-screen-only activation: STRUCTURAL GAP — mid-session users stranded.** A player who restored from background mid-game, or mid-level, or who discovered mid-session they need an accommodation, has no path without returning to title. Only production mid-session path is `DEVELOPMENT_BUILD`-gated. Spirit of SC 1.3.3 (Sensory Characteristics, A) and EAA Annex I reasonable-accommodation requirement apply. **HIGH gap** — not BLOCKING because GDD constrains scope to activation contract, but must be explicitly documented in an OQ with owner.

**Single-tap for tremor/spasticity users: adequate; SC 2.5.1 citation is inaccurate framing.** SC 2.5.1 (Pointer Gestures, A) addresses gesture complexity, not tremor specifically. Target-size + `d_slop` jitter tolerance is the operative accommodation. Citation not wrong, just incomplete.

**Production mid-session activation path: ABSENT.** Only `DEVELOPMENT_BUILD`-gated pause button. Production players mid-session have zero path.

**Handoff to Main Menu GDD: SOFT — no binding gate.** No BLOCKING AC ties MVP ship to Main Menu GDD delivering the button. If Main Menu GDD is never authored before MVP (real risk), the persistent title-screen button has no visual spec, no contrast test, no implementation owner. OQ-6 lists owners but no hard gate.

---

### B17 — EAA / UK Equality Act / EN 301 549 citations

**EAA Annex I — mobile games scope: AMBIGUOUS, likely OUT for pure games, IN via IAP.** Annex I Section IV covers "consumer-to-business e-commerce services using electronic communication services" — IAP falls under this. Pure entertainment games NOT listed. Leaderboards alone unlikely to pull in (not interpersonal communication services per Annex I Section III). **Operative hook is IAP checkout flow, which must be WCAG 2.1 AA per EAA.** GDD's blanket "EAA applies to mobile apps distributed via EU app stores" is **overstated** — applies to specific in-scope service surfaces (IAP primarily), not entire application. Correct citation to IAP-specific scope.

**UK Equality Act Section 29: AMBIGUOUS.** Courts interpret "services" broadly but have NOT definitively tested entertainment software under Section 29. Most defensible reading: IAP + account-management surfaces are services under Section 29, consistent with EAA. Blanket citation covering "mobile apps" overstates current legal position. Flag as "unverified — needs specialist deep-dive" in GDD rather than stating as established.

**EN 301 549 → WCAG 2.1 vs 2.2 for 2026 launch: WCAG 2.1 AA correct as harmonized standard.** EN 301 549 v3.2.1 references WCAG 2.1 AA. WCAG 2.2 (Oct 2023) adds SC 2.5.8 AA target size, SC 3.2.6 consistent help, SC 3.3.7/3.3.8 accessible auth. For 2026 ship: 2.1 AA is legally correct; 2.2 AA is more defensible commercial target. **Recommend: "WCAG 2.1 Level AA per EN 301 549 v3.2.1 (harmonized); WCAG 2.2 Level AA targeted for forward compatibility."**

**VS-sprint-gating adequacy: adequate conceptually, not mechanically enforced.** No AC in Section H reads "AC-R9-vs: Accessibility Settings UI with slider for `t_tap_max` exists and is reachable from title screen by VS sprint." Without that AC, "VS-sprint-gating" is soft planning note, not hard gate. **Gap: add VS-gated AC or explicitly delegate to Accessibility Service GDD with binding cross-GDD dependency flag.**

**REQUIRED-reviewer enforceability: SOFT.** OQ-6 names "accessibility-specialist as REQUIRED reviewer" — no AC or gate artifact blocks approval if absent. This is the procedural defect that caused 5 passes to skip the review. Mechanism to prevent recurrence is not in the document — it is in review workflow. GDD cannot enforce unilaterally, but OQ-6 should reference a process document / review checklist.

---

### B18 — Deaf-blind gap + Haptics VS REQUIRED

**MVP→VS regulatory exposure window: acceptable as documented known regression, but GDD contradicts itself.** OQ-12 claims "Section F Downstream phased table is updated to REQUIRED at VS cert-gating." **But the Section F Haptics row (GDD line ~153) still shows only `input.tap.main` subscription with no `input.gesture.longpress.unhandled` row and no REQUIRED marker.** OQ-12's claimed fix is not in the normative section. **This is a BLOCKING finding within B18 — document claims a fix that is absent from the normative table.**

**Gesture-scope coverage: LongPress only — GAP.** Section F Haptics entry conflates aesthetic tap haptics with accessibility-critical longpress haptic. Should be TWO separate rows: Haptics-tap = opt-in VS (aesthetic); Haptics-longpress = REQUIRED VS cert-gating (accessibility). **HIGH gap in normative phased table.**

**iOS 13+ / Android API 26+ minimum-OS floor: GAP — NOT STATED ANYWHERE.** iOS CoreHaptics = iOS 13+; Android VibrationEffect = API 26+. technical-preferences.md does not state minimum OS target. If Android minimum target is below API 26 (material share of low-end Android 2026), VibrationEffect unavailable — third channel fails silently, re-opening deaf-blind gap. SC 1.3.3 (Sensory Characteristics, A) violation for those devices. **Pass-1-through-5 oversight missed by all specialists and author.** Fix: (a) state minimum Android API target confirmed 26+, or (b) require Haptics GDD to specify fallback (`Vibrator.vibrate()` for API < 26) with explicit note that deaf-blind contract is incomplete on sub-API-26 devices.

**Three-channel completeness: NOT FULL CLOSURE.** Visual+audio+haptic insufficient for deaf-blind user ALSO motor-impaired in ways preventing reliable haptic perception (peripheral neuropathy). Correct framing: haptic closes gap "for the modal case of combined visual and audio impairment where motor function is preserved." Residual population (combined visual+audio+tactile impairment) requires switch-access/external AT — out of scope but worth one-sentence acknowledgment. Current "closes the deaf-blind gap" is overconfident. **Advisory, not blocking.**

---

### R17 — AC-EC-CB3 MVP MANUAL in-menu copy

**AC-EC-CB3 body does not exist as readable GIVEN/WHEN/THEN.** Only surfaces as ADVISORY count summary row and OQ-2 reference. Actual copy deferred to OQ-2 "Accessibility Service UX-write task." **Timing contradiction: AC is MVP MANUAL but OQ-2 is VS-scoped.** If AC must test at MVP, copy must exist at MVP. Fix in one direction: either AC demoted back to VS MANUAL, or OQ-2 scope accelerated to MVP. Neither done. **HIGH internal inconsistency.**

**Proposed plain-language copy** (if author needs starting point): "You've turned on Extra Time for Taps. The game now waits a bit longer before deciding you've tapped. If you hold your finger still for about half a second, the game treats it as a tap. This can make the game harder to pause with a hold." — 4th-grade Flesch-Kincaid, no technical notation, communicates trade-off without alarming.

---

### R19 — `hit_target_scale` MVP promotion

**Recommendation: PROMOTE to MVP as `PlayerPrefs` key (UI deferred to VS).** 1.4× fixed enlargement (technical-preferences.md) is a design assumption for average finger, NOT a user accommodation. Users with essential tremor, cerebral palsy, or fine-motor impairment may need 1.6×–2.0×. Adding 4th `PlayerPrefs` key at MVP costs near-zero design complexity (follows existing R9 pattern); only UI slider deferred to VS — same pattern as other three accommodations. SC 1.3.3 (A) + EAA accessible-design foundation: fixed 1.4× not user-adjustable is not accommodation. **Promote: key range [1.0, 2.0], default 1.4, UI deferred to VS alongside other sliders.**

---

### OQ wording audit

**OQ-6: PARTIALLY specific.** Names owners + trigger events. Missing resolution criterion — what constitutes "closed"? Without it, can remain perpetually open with partial progress claimed.

**OQ-11: WELL-FORMED.** Variable, default, calibration method, measured outcome, decision branches, owner, trigger all specified. Good OQ.

**OQ-12: self-inconsistent + missing resolution criterion.** Diagnosis correct (deaf-blind gap + haptics third channel + platform APIs). **BUT claims "Section F Downstream phased table is updated" — Section F NOT ACTUALLY UPDATED.** Unfulfilled self-reference. No resolution criterion stated.

---

### Adversarial findings (gaps missed by 5 prior passes)

1. **WCAG 2.1 SC 2.5.5 is AAA, not AA.** 48 dp target size correctly sized, incorrectly cited. Governing AA = WCAG 2.2 SC 2.5.8 (24×24). **Pass-1-through-5 oversight.**

2. **Section F phased table does NOT reflect OQ-12's claimed update.** Haptics row still shows `input.tap.main` only — no `input.gesture.longpress.unhandled`, no REQUIRED designation. Internal inconsistency between normative section and OQ. **Pass-1-through-5 oversight + Phase 6A incompletion.**

3. **iOS CoreHaptics (iOS 13+) / Android VibrationEffect (API 26+) minimum-OS gap unspecified.** If Android minimum target < API 26, haptic third channel silently fails on material low-end share — exact segment game targets. Re-opens SC 1.3.3 (A) deaf-blind gap. **Pass-1-through-5 oversight.**

4. **`hit_target_scale` has no MVP presence despite 1.4× fixed enlargement being design assumption, not user accommodation.** Users with tremor/CP/fine-motor impairment may require >1.4×. Deferring `PlayerPrefs` key (zero cost) to VS is wrong; UI slider deferral is fine. **Pass-1-through-5 oversight.**

5. **Mid-session accessibility settings access gap — production users have no in-session path.** Not documented as OQ. Title-screen-only activation leaves restored/mid-level users stranded. **Pass-1-through-5 oversight.**

6. **AC-EC-CB3 MVP MANUAL vs OQ-2 VS authoring — timing contradiction.** Pass-5 ux-designer promoted AC; author accepted without advancing OQ scope. **Pass-5 oversight not caught in Phase 6A.**

---

### Recommendation to creative-director

Document made genuine progress on B16/B17/B18, and accessibility-specialist consultation procedural defect is now corrected in process. Three items require targeted author revision before APPROVED:

1. **WCAG citation correction** — title-screen button citation from incorrect "SC 2.5.5" AAA framing to accurate WCAG 2.2 SC 2.5.8 AA (or explicit AAA-target-exceeds-AA framing).

2. **Section F phased table update** — add second Haptics row with `input.gesture.longpress.unhandled` subscription designated REQUIRED at VS cert-gating, as OQ-12 claims but table does not reflect.

3. **AC-EC-CB3 MVP vs OQ-2 VS timing contradiction** — resolve in one direction (AC gates return to VS, or OQ-2 becomes MVP-scoped).

Additionally, advisory OQs should be added for (a) mid-session settings access gap and (b) minimum Android API floor for VibrationEffect. **`hit_target_scale` MVP promotion recommended but not blocking.**

With items (1), (2), and (3) corrected in Phase 6A-style targeted revision, document can be APPROVED by CD without further full specialist re-spawn — corrections are narrow and mechanical, not architectural.

**Phase 6B accessibility-specialist verdict: CONCERNS — 3 targeted fixes required + 2 advisory OQs + 1 recommendation. Phase 6A substantive direction is sound; execution has gaps prior 5 passes missed because no accessibility-specialist was consulted.**

---

## Review — 2026-04-24 — Pass-6 — Verdict: APPROVED-WITH-CONDITIONS

**Reviewer:** creative-director (Phase 6C synthesis pass)
**Inputs synthesized:** Phase 6A author-led revision (review log lines 697–812) + Phase 6B parallel adversarial verification (unity-specialist lines 816–1043, performance-analyst lines 1046–1194, accessibility-specialist NEW lines 1195–1290).
**Scope signal:** **S** (Phase 6D is ~2–3 hours of targeted mechanical corrections — no architectural revision).
**Re-review status:** 6th CD pass on this GDD. Trajectory pass-1 APPROVED → pass-2 CONCERNS → pass-3 CONCERNS → pass-4 CONCERNS → pass-5 REJECT → **pass-6 APPROVED-WITH-CONDITIONS**.

### Verdict

**APPROVED-WITH-CONDITIONS** — the Input System GDD is architecturally sound and ready for implementation planning, pending 3 narrow mechanical corrections the author can apply in Phase 6D in a single short session.

### CD-GDD-ALIGN disposition

**APPROVED-WITH-CONDITIONS** — restored from pass-5 REJECT. Both the pass-6 GDD verdict and CD-GDD-ALIGN disposition revert automatically to **full APPROVED** upon Phase 6D merge — no further specialist panel re-spawn is required, since all 3 remaining items are mechanical (citation correction, field-type fix, table row addition, timing-reconciliation) with specialist-provided exact fix language and no architectural revision surface.

### 3 BLOCKING Phase 6D items (pre-committed scope, merge-block advancement)

These are the only items that gate the pass-6 verdict from auto-restoring to full APPROVED on merge. Each has a specialist-scripted exact fix.

**1. unity-specialist BLOCKING — AC-R13f `CopyTo` overload signature**

Current spec calls `CopyTo(ProfilerRecorderSample[], int)`. That overload does not exist in Unity 6.3 LTS `ProfilerRecorder` API. Replace with one of:

- `CopyTo(List<ProfilerRecorderSample>)` — managed allocation, acceptable for the 1000-event post-run analysis path (budget is not per-frame).
- `CopyTo(NativeArray<ProfilerRecorderSample>)` — zero-alloc, pre-sized to `recorder.Capacity`.

Add `using Unity.Profiling;` to the namespace import note. (Defect class: Unity 6.3 API signature error — mechanical overload misspecification, no architectural implication.)

**2. performance-analyst BLOCKING — AC-R13c thermal-state triple field-type + iOS native-plugin dependency**

Current triple contains `temperature_c: float`, citing `PowerManager.ThermalStatus`. That Android API (API 29+) returns a discrete severity enum (0–6), not Celsius. Replace:

- Field rename: `temperature_c: float` → `thermal_severity_level: int`.
- Android semantics: level `0–6` per `THERMAL_STATUS_NONE/LIGHT/MODERATE/SEVERE/CRITICAL/EMERGENCY/SHUTDOWN`.
- iOS semantics: level `0–3` per `ProcessInfo.ThermalState` enum (`nominal/fair/serious/critical`).
- Throttle derivation: `is_throttled = level >= 2` on Android (MODERATE = documented throttling onset); `level >= .serious` on iOS.
- Dependency surfacing: add an iOS native-plugin dependency row in Section F for CoreHaptics / `ProcessInfo` (Foundation Obj-C bridge, not accessible from managed C#).
- ADR trigger: open ADR-007 (or next available number) scoping the CoreHaptics / `ProcessInfo` managed-C# bridge into the Plumbing Sprint (Months 4–5).

(Defect class: cross-domain API-shape misapplication — same defect class as pass-4→pass-5 GPU→CPU thermal ceiling. Blast radius here is one field + one undisclosed native dependency, not an entire budget derivation.)

**3. accessibility-specialist BLOCKING (compound)**

Three sub-items, all mechanical fixes against the pass-6 GDD as-shipped:

- **(a) WCAG citation correction** — B16 currently cites "48 dp per WCAG 2.1 SC 2.5.5". SC 2.5.5 is AAA, not AA. Re-cite: **"48 dp per WCAG 2.2 SC 2.5.8 (AA, 24×24 minimum); exceeds WCAG 2.1 SC 2.5.5 AAA target"**. Regulatory-defensibility grade, not citation hygiene — a compliance audit would flag AAA-cited-as-AA as a material misrepresentation of conformance claims.

- **(b) Section F Haptics second row** — OQ-12 claims Section F phased table updated to designate `input.gesture.longpress.unhandled` Haptics subscription REQUIRED at VS cert-gating. Table in fact still shows only `input.tap.main` with no REQUIRED marker and no longpress row. Add the second Haptics row as OQ-12 promises; this closes the self-contradiction.

- **(c) AC-EC-CB3 / OQ-2 timing contradiction** — AC-EC-CB3 is MVP MANUAL but OQ-2 (the copy-authoring task whose output the AC tests) is VS-scoped. Resolve in one direction: either demote AC-EC-CB3 to VS MANUAL, or promote OQ-2 to MVP scope so the copy exists when the AC runs.

### Recommended (non-blocking, Phase 6D author discretion)

Logged for Phase 6D author judgment — not merge-gating.

- **(unity-specialist)** Soften B8 ARM "10–15% CPU field measurement" language → "engineering estimate pending OQ-8 on-device calibration." (Honesty-of-provenance, not a numeric change.)
- **(unity-specialist)** B10 prose correction: "Library/ + ProjectSettings/" → "ProjectSettings/ (the define persists in ProjectSettings.asset written by `AssetDatabase.SaveAssets()`)". (Factual accuracy re: where the define actually persists.)
- **(performance-analyst)** Disclose `v_lateLift` DPI conversion basis in Section G — e.g., "0.1 pt/ms ≈ 15.9 mm/s at 320 dpi Android" — so the tuning knob range is interpretable across device DPI.
- **(performance-analyst)** Label `0.1 ms residual` noise budget as "design target pending OQ-7 empirical validation," not stated fact (addresses the pass-3 Stopwatch finding that environmental noise is not reduced by instrumentation choice).
- **(performance-analyst)** Add `OnValidate` floor constraint `v_lateLift >= 0.05` so safe-range lower bound is enforced at editor-author time, not only documented.
- **(accessibility-specialist)** Promote `hit_target_scale` to an MVP `PlayerPrefs` key (UI slider can still defer to VS). Rationale: fixed 1.4× enlargement is a design assumption, not a user accommodation — users with tremor/CP/fine-motor impairment may require 1.6–2.0×.
- **(accessibility-specialist)** Add advisory OQ: mid-session Accessibility settings access gap. Production players mid-session currently have no path to accessibility toggles (only the `DEVELOPMENT_BUILD`-gated pause button). Document as OQ; resolution likely at VS with full Settings/Pause scope expansion.
- **(accessibility-specialist)** Add advisory OQ: minimum Android API floor for `VibrationEffect` (API 26+). If Android min-target < 26, haptic third channel silently fails on low-end Android share — re-opens the SC 1.3.3 (A) deaf-blind gap at runtime. Either state min-target API in the GDD or require Haptics GDD (VS authoring) to specify a `Vibrator.vibrate()` fallback path for API < 26 with an explicit sub-API-26 deaf-blind regression note.

### Strengths preserved (load-bearing after 6-pass gauntlet)

These decisions have now held stable across multiple CD passes and adversarial specialist spawns. They are the load-bearing contracts the downstream GDDs (HUD, Main Menu, Collection, Tool, Hazard) must be authored against:

- **Pillar 5 dual-threshold contract** (100 ms at 60 fps target / 133 ms at 30 fps floor) — stable since pass-3.
- **T2 LongPress channel with REQUIRED HUD subscriber** — resolved pass-2, held through pass-6.
- **Per-handler + chain-sum independent caps architecture** — pass-2 + pass-3 CD-adjudicated; Phase 6B performance-analyst budget arithmetic confirms coherence (modulo B9 noise-floor label).
- **Gesture classifier D.3 priority order + `InProgress otherwise` catch-all** (Phase 6A B4) — closes the coverage hole without introducing cross-branch ambiguity.
- **AC-R13c thermal-state discriminator + 3-precondition downgrade gate** — Phase 6A B11 architecture sound; Phase 6D fix is field-type only (Celsius float → severity int), the 3-precondition cross-correlation logic is preserved.
- **CI Gate Operability Status column** (Phase 6A B12) — infra-vs-contract distinction now explicit on the Gate Summary.
- **Phase 6A citation-heavy methodology** worked in aggregate — 17 of 18 pass-5 BLOCKING items verified sound by the adversarial panel. This is the methodology Phase 6D should continue to use.
- **Pillars 1 (Cute Chaos feedback) and 5 (No Input Goes Silent) cross-system contracts** — Feedback Floor language in systems-index.md plus Input System's R3-series REQUIRED designations together form the contract HUD/Collection/Tool/Hazard/Critter AI will consume at MVP.

### Specialist disagreements adjudicated

Three disagreements surfaced across the Phase 6A → Phase 6B → pass-5 timeline. Creative-director adjudications:

**1. `SetScriptingDefineSymbols(NamedBuildTarget, string[])` overload existence.**
- Pass-5 unity-specialist: the `string[]` overload does not exist.
- Phase 6A author: both `string` and `string[]` overloads exist.
- Phase 6B unity-specialist: both overloads exist; Phase 6A was correct, pass-5 specialist was wrong on that specific API claim.
- **CD upholds Phase 6B adjudication.** This is a factual Unity 6.3 API question with a single correct answer; the Phase 6B unity-specialist cited the Unity ScriptReference directly. No Phase 6D action required on B10 beyond the prose "Library/ → ProjectSettings/" recommendation.

**2. performance-analyst REJECT vs unity-specialist CONCERNS severity classification.**
- performance-analyst returned REJECT citing same-class-as-pass-4 cross-domain domain error recurrence.
- unity-specialist returned CONCERNS on a narrower API-signature mismatch.
- **CD treats both as the same severity class at document-architecture level**: both are BLOCKING-at-implementation (compile failure or runtime semantics gap), not BLOCKING-at-design (which would require architectural revision). Both have specialist-provided exact fix language. Both merge under the pass-6 APPROVED-WITH-CONDITIONS envelope.
- The performance-analyst REJECT severity signal is CD-acknowledged as a **methodology-level concern** (defect class recurrence — see below), not a document-architecture concern. It is properly absorbed into the Phase 6D scope without requiring pass-7.

**3. accessibility-specialist WCAG AA-vs-AAA citation severity.**
- Surface reading: citation hygiene issue, low-severity.
- accessibility-specialist classification: BLOCKING.
- **CD upholds accessibility-specialist BLOCKING classification.** Citing WCAG 2.1 SC 2.5.5 (AAA) as the authority for an AA-conformance claim is a **regulatory-defensibility-grade misrepresentation**, not a citation formatting issue. Under EAA (effective June 2025) and UK Equality Act enforcement contexts, a compliance audit reviewing the GDD as a conformance-claim artifact would flag the mis-citation as a material inaccuracy. Fix cost is trivial; risk of leaving it is not.

### Defect-class recurrence assessment (process-level)

The pass-4 → pass-5 defect class was: **author-alone revisions producing confident-sounding citations that misapply cross-domain APIs** (canonical example: GPU thermal scaling cited as the basis for a CPU pipeline ceiling). Phase 6A adopted a citation-heavy methodology specifically to address this defect class — every claim grounded in a linked source.

Phase 6B found **exactly one** new instance of the same class: `temperature_c: float` cited against `PowerManager.ThermalStatus`, when that API returns a severity enum (0–6), not Celsius. Same cross-domain API-shape error; smaller blast radius (1 field, not an entire ceiling derivation).

**Defect density attenuated ~75%** (1 recurrence vs. 4 convergent pass-5 findings of the same class). The methodology change succeeded directionally.

**Process guidance for Phase 6D:** the author may continue with the same citation-heavy methodology — no methodology re-think required for these 3 narrow items. One specific habit to add: when citing an external API as the basis for a field type, explicitly verify the **return-type / output shape** of that specific API method, not just that the API exists. `PowerManager.ThermalStatus` exists (API 29+); the defect was using its existence to justify a field type (`float` Celsius) the API does not produce. A one-line "this API returns [shape]" verification per cross-domain claim would prevent the remaining defect.

### Scope signal

**S** — 2–3 hours for Phase 6D author revision covering 3 BLOCKING mechanical items + author-discretion recommended items. No architectural revision, no new ACs, no new dependencies, no new ADRs (other than opening ADR-007 for the iOS native-plugin bridge, which is a stub/trigger action, not an architectural decision rendered in this session).

### Next step

1. **Phase 6D author-led revision** (~2–3 hours). Apply the 3 BLOCKING corrections above plus any recommended items the author elects. Continue Phase 6A citation-heavy methodology with the added habit of verifying API return-type/shape on cross-domain claims.
2. **No re-review panel** after Phase 6D. On Phase 6D merge, pass-6 verdict and CD-GDD-ALIGN disposition auto-restore to full APPROVED.
3. **`/consistency-check`** — verify pass-5/pass-6 revisions (Rush-phase REQUIRED `input.contact.began`, CPU-domain thermal ceiling, corrected Unity 6.3 APIs, persistent-button accessibility path, Haptics REQUIRED at VS) do not conflict with Data Registry / Event Bus / Save/Load GDDs.
4. **Unblock `/design-system scene-app-state-manager`** — Foundation #5, previously blocked on Input System approval.

### Recommendation to user

The Input System GDD is architecturally sound and ready for implementation planning pending 3 mechanical corrections the Phase 6D author can apply in a single short session. Six review passes is high, but that volume reflects a GDD that anchors two gameplay pillars — tap/drag responsiveness (Pillar 5) and accessibility (deaf-blind third channel under EAA regulatory scope) — under hard mobile performance ceilings (Mali-G52-class low-end Android, 4.6 ms CPU thermal ceiling, 100/133 ms dual-threshold Feedback Floor). The scrutiny produced a document with near-zero architectural defect at the cost of considerable revision cycles; the defect-density attenuation between pass-5 and pass-6 confirms the citation-heavy Phase 6A methodology worked. Pass-6 restores the document to APPROVED-WITH-CONDITIONS with a short, specialist-scripted Phase 6D task list; once that merges, the document is fully APPROVED without further review. Recommended sequencing: execute Phase 6D in the next session, run `/consistency-check`, then unblock Foundation #5 (`/design-system scene-app-state-manager`).

**Pass-6 verdict: APPROVED-WITH-CONDITIONS. CD-GDD-ALIGN: APPROVED-WITH-CONDITIONS (restored from REJECT). Both auto-restore to full APPROVED on Phase 6D merge — no pass-7 required.**

---

## Revision — 2026-04-24 — Pass-6 Phase 6D (author-led, addresses all 3 pass-6 BLOCKING items + 2 advisory OQs + 6 recommended items from 2026-04-24 pass-6 CD verdict)

**Phase**: Phase 6D of the pass-6 process. Author-led targeted mechanical revision applying the 3 BLOCKING items with specialist-scripted exact fix language from the pass-6 CD verdict (2026-04-24, this review log lines 1303–1429) + advisory OQs + recommended items at author discretion. **No specialist panel re-spawn** per pass-6 CD verdict auto-restore rule — all 3 BLOCKING items are mechanical fixes with specialist-provided exact substitution text; no architectural revision surface; no pass-7 required.

**Scope signal**: **S** (~2–3 hours mechanical author revision, matches CD estimate). No new ACs, no new ADRs rendered (ADR-007 is a stub/trigger only — the actual ADR authoring is a Plumbing Sprint task, not this session).

### Resolution per BLOCKING item

**1. unity-specialist BLOCKING — AC-R13f `CopyTo` overload signature (Phase 6B line 816–1043; CD synthesis line 1324–1331)**

- **Fix applied at AC-R13f body (Section H.1)**: `CopyTo(ProfilerRecorderSample[], int)` overload reference (non-existent in Unity 6.3 LTS `ProfilerRecorder` API) replaced with `CopyTo(List<ProfilerRecorderSample>)` (managed, MVP default — acceptable for the 1000-event *post-run* analysis path since sample copy is not on the per-frame hot path) with documented `CopyTo(NativeArray<ProfilerRecorderSample>)` zero-alloc alternative (caller pre-sizes to `recorder.Capacity`). Author-picked `List<T>` as MVP default; `NativeArray<T>` promotion deferred to post-MVP Profiler job-system integration if it lands.
- **`using Unity.Profiling;` note added** at first `ProfilerMarker` / `ProfilerRecorder` reference in AC-R13f body with citation to [Unity Scripting API `Unity.Profiling` namespace](https://docs.unity3d.com/ScriptReference/Unity.Profiling.ProfilerMarker.html). Also applied to the R13.d descriptive text block at Section C's line 121 area for consistency (same `CopyTo` signature appears in R13.d prose).
- **Defect class**: Unity 6.3 LTS API signature error — mechanical overload misspecification, no architectural implication. Fix-applied instance (B9 Phase 6A author wrote the non-existent overload reference while describing an otherwise-correct `ProfilerRecorder.StartNew(..., capacity: 1000)` pattern).

**2. performance-analyst BLOCKING — AC-R13c thermal-state triple field-type + iOS native-plugin dependency (Phase 6B line 1046–1194; CD synthesis line 1333–1345)**

- **Field rename applied at AC-R13c body (Section H.1)**: `temperature_c: float` → `thermal_severity_level: int`. Android semantics documented: level `[0, 6]` per `THERMAL_STATUS_NONE/LIGHT/MODERATE/SEVERE/CRITICAL/EMERGENCY/SHUTDOWN` (API 29+) per [Android `PowerManager.getCurrentThermalStatus()` reference](https://developer.android.com/reference/android/os/PowerManager#getCurrentThermalStatus()). iOS semantics documented: level `[0, 3]` per [`ProcessInfo.thermalState`](https://developer.apple.com/documentation/foundation/processinfo/1408062-thermalstate) enum `nominal/fair/serious/critical`. Throttle derivation: `is_throttled = (thermal_severity_level >= 2)` on both platforms — `THERMAL_STATUS_MODERATE` = documented Android throttling onset; `.serious` = iOS equivalent.
- **Sub-API-26 Android sentinel behavior**: for devices below `PowerManager.getCurrentThermalStatus()` API floor (API 29+ requirement), the sample is `thermal_severity_level = -1` sentinel and `is_throttled = false`; the AC-R13c correlated-tail downgrade precondition 2 cannot fire on sentinel-tagged samples. Cross-refs OQ-15 (Android `VibrationEffect` API 26+ floor).
- **iOS native-plugin dependency surfaced at Section F (Dependencies)**: new subsection `#### Native Platform Plugins (pass-6 Phase 6D addition)` added below Unity Package Pins. Table enumerates 4 native-plugin surfaces: `ProcessInfo.thermalState` iOS bridge, `CoreHaptics` iOS bridge, `PowerManager.ThermalStatus` Android JNI bridge, `VibrationEffect` + `Vibrator.vibrate()` Android JNI bridge. All 4 scoped to **Plumbing Sprint (Months 4–5)**.
- **ADR-007 trigger logged**: ADR-007 scopes CoreHaptics / `ProcessInfo.thermalState` Obj-C-to-managed-C# bridge AND parallel `PowerManager.ThermalStatus` / `VibrationEffect` JNI bridge on Android. Shared bridge pattern amortizes the Plumbing Sprint cost. ADR decides: P/Invoke vs. Unity as-library / native-plugin packaging (iOS); `AndroidJavaObject` vs. compiled .aar artifact (Android); threading model; sampling frequency (per-frame required for AC-R13c annotation). Without ADR-007 + Plumbing Sprint execution, AC-R13c's `thermal_severity_level` field reverts to `-1` sentinel — documented MVP interim behavior.
- **Defect class**: cross-domain API-shape misapplication — same defect class as pass-4→pass-5 GPU→CPU thermal ceiling. Phase 6B found **exactly one** new instance of the same class (`temperature_c: float` cited against an enum-returning API). Blast radius: 1 field + 1 undisclosed native dependency, not an entire budget derivation. Defect density attenuated ~75% vs pass-5 per CD assessment (1 recurrence vs 4 convergent pass-5 findings). Process guidance applied: author now verifies API return-type/shape, not just API existence.

**3. accessibility-specialist BLOCKING (compound — Phase 6B line 1195–1290; CD synthesis line 1346–1354)**

- **(a) WCAG citation correction applied at R9 primary activation path + OQ-6 body**: "48 × 48 dp per WCAG 2.1 SC 2.5.5 Target Size (AAA)" → **"48 × 48 dp per WCAG 2.2 SC 2.5.8 Target Size (Minimum) — the governing AA-conformance criterion (24 × 24 CSS px minimum); 48 dp exceeds the WCAG 2.1 SC 2.5.5 AAA target size"**. Citations added: [W3C WCAG 2.2 SC 2.5.8 Target Size Minimum](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html) + [W3C WCAG 2.1 SC 2.5.5 Target Size AAA](https://www.w3.org/WAI/WCAG21/Understanding/target-size.html). Regulatory-defensibility-grade fix per CD adjudication 3 of Phase 6C verdict: citing SC 2.5.5 (AAA) as AA-conformance authority is a material misrepresentation under EAA / EN 301 549 / UK Equality Act 2010 audit review. EN 301 549 mobile-app harmonized standard note retained: still references WCAG 2.1 AA for 2026 regulatory compliance — WCAG 2.2 AA stronger but not yet mandated; 48 dp satisfies both.
- **(b) Section C subscribers table second Haptics row added** (addressing the OQ-12 self-contradiction where OQ-12 claimed Section F phased table was updated but the Section C subscribers table at line 152–162 still showed only `input.tap.main` for Haptics with no REQUIRED marker + no longpress row): new row `**Haptics** (VS, REQUIRED at cert-gating — third-channel deaf-blind fail-safe per OQ-12) | input.gesture.longpress.unhandled | T2 | LongPressUnhandledPayload | ...` added directly below the existing `**Haptics** (VS) | input.tap.main` row. Section F Downstream (phased) Haptics row at line 430-area was already correct at Phase 6A; the contradiction was Section C normative table vs. OQ-12 claim. Contradiction now closed.
- **(c) AC-EC-CB3 / OQ-2 timing reconciliation applied at OQ-2**: OQ-2 target resolution changed `VS Accessibility Service GDD authoring` → **`MVP implementation sprint (copy authored alongside Accessibility Settings overlay MVP-minimal scope; VS sprint polishes for full UI per OQ-6)`**. Direction chosen: promote OQ-2 to MVP (AC-EC-CB3 was already promoted to MVP at pass-5 per ux-designer R17). Alternative (demote AC-EC-CB3 to VS MANUAL) was rejected — pass-5 ux-designer R17 MVP-copy decision is load-bearing for the Pillar 5 accessibility contract, and the Accessibility Settings overlay MVP-minimal scope (R9 primary button target) is where the copy naturally lives. Copy ownership unchanged: writer + accessibility-specialist.

### Advisory OQs added

- **OQ-14 — Mid-session Accessibility settings access gap**: documented as known accessibility gap, not shipping MVP defect. R9 persistent title-screen button satisfies pre-gameplay activation; mid-session access is a VS upgrade (Main Menu / Settings / Pause GDD exposes Accessibility Settings overlay from production pause menu at VS).
- **OQ-15 — Android `VibrationEffect` API 26+ floor**: two acceptable resolutions (state min-target ≥ API 26 OR Haptics GDD specifies `Vibrator.vibrate()` fallback for API < 26 with explicit sub-API-26 deaf-blind regression note). Flag for pre-production Android min-target ADR + VS Haptics GDD authoring.

### Recommended items applied at Phase 6D author-discretion

All 6 recommended items from the CD verdict (line 1358–1367) applied inline:

- **(unity)** AC-P3 body — "field measurements on Mali-G52-class mobile devices ... **10–15 %**" softened to "engineering estimates ... **10–15 %** ... **pending OQ-8 on-device calibration**" with honesty-of-provenance framing. Numeric value unchanged; provenance label corrected.
- **(unity)** B10 prose — "two steps share the project's `Library/` + `ProjectSettings/` state" → "two steps share the project's `ProjectSettings/` state (the scripting-define symbol persists in `ProjectSettings/ProjectSettings.asset`, which is written by `AssetDatabase.SaveAssets()`...)". Factual correction re: where the define actually persists.
- **(perf)** Section G Gesture Classifier Knobs `v_lateLift` row — DPI conversion basis disclosed: `0.1 pt/ms ≈ 15.9 mm/s at 320 dpi Android`, with range `[0.05, 0.5]` maps to `[7.95, 79.4]` mm/s at 320 dpi. Cross-DPI class values (160 / 480 dpi) also listed. OQ-11 empirical calibration now records both nominal `pt/ms` + physical `mm/s` interpretation.
- **(perf)** R13.d body `0.1 ms residual` noise budget labeled **"design target pending OQ-7 empirical validation"** with pass-3 Stopwatch-era finding cross-reference (environmental noise is not reduced by instrumentation choice; the 0.1 ms assumes ProfilerMarker intrinsic noise only, not end-to-end measurement noise).
- **(perf)** Section G Gesture Classifier Knobs `v_lateLift` row — `OnValidate` floor constraint added: `OnValidate enforces safe-range floor v_lateLift >= 0.05`. Author-time constraint (not just documented safe range).
- **(a11y)** Section G Accessibility / Platform Knobs — new `hit_target_scale` row promoted to MVP `PlayerPrefs` key `MM.A11y.HitTargetScale`, default 1.4×, range [1.4, 2.0]. R9 `AccessibilitySettings` struct gained 4th row (three → four configurable values). UI slider defers to VS Accessibility Settings overlay per OQ-6. MVP behavior: `PlayerPrefs` read-only path at `Initialize()`; consumers (Tool, Collection, HUD, Menu) use the read value for their hit-bounds inflation; default 1.4× if key absent.

### Recommended items deferred (per CD verdict — non-blocking, not required for APPROVED restoration)

None deferred; all 6 CD-recommended items were applied inline.

### Status change

- CD-GDD-ALIGN: APPROVED-WITH-CONDITIONS (pass-6 CD synthesis) → **APPROVED (pass-6 post-Phase 6D)** — auto-restored per pass-6 CD verdict rule on Phase 6D merge.
- Header status: REVISED (pass-5 author) → **APPROVED (pass-6 post-Phase 6D) 2026-04-24**.
- Systems-index Input row: APPROVED-WITH-CONDITIONS (pass-6) → **APPROVED (pass-6)**.
- AC count: 71 → 71 (no new ACs; field-type changes + Section C subscribers table row addition + Section G knob row additions + OQ table growth only).
- OQ count: 12 active + 1 resolved (OQ-10) → **14 active + 1 resolved** (+OQ-14 mid-session a11y gap + OQ-15 Android API 26+ VibrationEffect floor).
- Gate-level counts: BLOCKING CI 10 (unchanged); PLAYMODE 51 (unchanged); EDITMODE 1 (unchanged); DEVICE 4 (unchanged); ADVISORY 5 (unchanged).

### Next step (Phase 6D merge → downstream actions)

1. **`/consistency-check`** — verify pass-5/pass-6 Input revisions (Rush-phase REQUIRED `input.contact.began`, CPU-domain thermal ceiling, corrected Unity 6.3 APIs, persistent-button accessibility path with WCAG 2.2 SC 2.5.8 AA citation, Haptics REQUIRED at VS cert-gating, `thermal_severity_level` field replacing `temperature_c`, iOS native-plugin Plumbing Sprint dependency, ADR-007 trigger, `hit_target_scale` MVP `PlayerPrefs` key) do not conflict with Data Registry / Event Bus / Save/Load GDDs.
2. **Unblock `/design-system scene-app-state-manager`** — Foundation #5, previously blocked on Input System approval.
3. **`/architecture-decision ADR-007`** — author at pre-production (Plumbing Sprint scoping), not this session.
4. No pass-7 — pass-6 verdict auto-restored to full APPROVED per CD verdict rule.

Phase 6D verdict: **APPROVED (pass-6 post-Phase 6D) — all 3 pass-6 BLOCKING items resolved author-led with specialist-scripted exact fix language; 2 advisory OQs added; 6 recommended items applied inline; CD-GDD-ALIGN auto-restored from APPROVED-WITH-CONDITIONS to APPROVED per pass-6 CD verdict rule. No pass-7 required.**
