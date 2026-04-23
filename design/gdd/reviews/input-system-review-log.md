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
