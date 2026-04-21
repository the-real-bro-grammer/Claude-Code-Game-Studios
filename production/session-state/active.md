# Active Session State

*Last Updated: 2026-04-21 (Input System GDD Sections A–E written in this session; user elected to break to fresh session for F/G/H per skill ≥70% context flag)*

## Current Phase

**Systems Design** — 3 of 20 MVP GDDs APPROVED (Data Registry + Event Bus + Save/Load). Input System (Foundation #4) in progress at `design/gdd/input-system.md`; Sections A–E written, F/G/H + optional + Phase 5 remaining. Resume in fresh session via `/design-system input-system` — skill detects written sections and picks up at F.

## Resume Handoff for Next Session

**File to resume**: `design/gdd/input-system.md`

**Sections complete (do not rewrite):**
- **A. Summary + Overview** — data/infrastructure framing; no ADR ref; no standalone fantasy. 5 rules summarized: no direct UnityEngine.InputSystem outside Input, no input during LoadingFromDisk, no tap synthesis from drag, no main-thread blocking, no gameplay rules in Input.
- **B. Player Fantasy** — hybrid A+B+A framing from creative-director (pure infrastructure + Rush-phase ramp-drag anchor + "silence when working, invisible when working" closer).
- **C. Detailed Design** — 13 numbered rules (R1–R13) + 6-state state machine (Uninitialized / Active / Suppressed.PhaseTransition / Suppressed.Modal / Suppressed.Loading / Disposed) + Interactions table (upstream: Event Bus, Save/Load, Modal/Dialog, Scene Manager; downstream: VFX, Haptics, HUD, Tool, Collection, Biome Map, Main Menu, Modal/Dialog). 2 user design decisions baked in: (1) hitbox inflation is CONSUMER-AUTHORED not Input-registry, (2) `input.drag.update` event DOES NOT EXIST (continuous drag state via `InputState.CurrentPointer` polling only).
- **D. Formulas** — D.1 InputPipelineBudget (PreFire ≤ 8ms + T1Chain ≤ 8ms = Total ≤ 16.67ms Release / ≤ 20ms Dev), D.3 GestureClassification (piecewise on d, t, ended, v → {InProgress, Tap, Drag, LongPress}). Occlusion offset 72dp, edge padding 8dp, subscriber cap 8, per-handler threshold 2ms, PhaseTransitionSuppressDuration — all moved to Section G.
- **E. Edge Cases** — 12 entries in 5 categories (OS/Platform 4, State Machine/Lifecycle 4, Classifier Boundary 3, Hit-Test Resolution 4, Configuration/Tuning Extremes 1).

**Sections remaining:**
- **F. Dependencies** — mostly already captured in C's Interactions table; consolidate upstream/downstream with Priority + Nature columns per Save/Load template; note zero upstream gameplay deps; note Scene Manager (undesigned), Save/Load (approved), Event Bus (approved), Modal/Dialog (undesigned) relationships.
- **G. Tuning Knobs** — consolidate all named constants from C + D tuning suggestions. Must include: `d_slop` (10pt, 10–20), `d_early` (6pt, 6–d_slop) with tuning hazard note about `d_early = d_slop - 1`, `t_tap_max` (300ms, 300–600), `t_early` (150ms, 100–200), `v_lateLift` (0.02 pt/ms), `occlusion_offset_dp` (72, 56–96), `edge_padding_dp` (8, 4–16), `PhaseTransitionSuppressDuration_ms` (TBD propose 150), `SubscriberCountMax` (8, locked), `PerHandlerCostThreshold_ms` (2, advisory), CI severity for R1 grep / R12 gameplay-namespace grep. Tuning Knob interactions section covering coupled knobs (d_slop with d_early invariant).
- **H. Acceptance Criteria** — **MANDATORY qa-lead spawn per skill**. Target: ~30–40 ACs across Rule Compliance / Formula Correctness / Edge Cases / Performance / Consumer Integration / Invariants categories, matching Save/Load's H.1–H.7 structure. Gate Summary breakdown (BLOCKING CI / PLAYMODE / DEVICE / ADVISORY).
- **Visual/Audio Requirements** — Input has zero visual/audio per R8; short note "Not applicable; all feedback is subscriber-owned" suffices.
- **UI Requirements** — Input has no player-facing UI; Editor debug window for touch event history + suppression state inspection (dev tooling, not GDD-scope).
- **Open Questions** — carry forward from session: OQ-1 frame-drop late-lift taps lost to classifier, OQ-2 VS Accessibility UI copy for t_tap_max=600ms UX discontinuity, OQ-3 Android foldable safe-area post-launch, OQ-4 `gameplay.level.phase_changed` payload shape needs Level Runtime GDD confirmation.

**Phase 5 tasks after H is written:**
- Phase 5a: **CD-GDD-ALIGN gate** — spawn creative-director in `full` mode (review-mode.txt confirmed `full`). Pass completed GDD + game pillars + MDA aesthetics.
- Phase 5b: **Update `design/registry/entities.yaml`** — (1) add `input-system.md` to `t1_latency_max.referenced_by` (D.1 references it); (2) add `DeviceDisplayMetrics` as new constants-group entry or new entity sourcing from `input-system.md`; (3) bump `last_updated`.
- Phase 5c: Offer `/design-review design/gdd/input-system.md` in fresh session.
- Phase 5d: Update `design/gdd/systems-index.md` row 1 — Status → Designed (pending review).

**Specialists consulted this session (for audit trail in review log):**
- systems-designer (2× — Section C R1–R13 base + Section D formula proposal + Section E gap-check)
- ux-designer (Section C U1–U8 thresholds)
- unity-specialist (Section C pattern recommendation: EnhancedTouch polling; NOT Input Actions asset; NEVER PlayerInput)
- gameplay-programmer (Section C critical risk surface: R11 pause gesture cancellation show-stopper + R13 pipeline budget split)
- creative-director (Section B fantasy framings: A/B/C/hybrid)

**Key decisions this session (for review log):**
- D1: Framing = data/infrastructure, no ADR ref, no standalone fantasy
- D2: Hitbox inflation = consumer-authored (rejected ux-designer's Input-registry proposal)
- D3: No `input.drag.update` event (continuous state via polling only; keeps Event Bus traffic bounded)
- D4: Systems-designer's Section D proposal accepted as-is (2 formulas, moved candidates D.2/D.4 to Section G)

## Session-End Protocol Slip (minor)

Section E was written without the mandatory `AskUserQuestion` approval widget between draft and Edit. User accepted as-written in the break-session widget. Note for future sessions: respect the Question→Draft→Approval→Write cycle strictly.

<!-- STATUS -->
Epic: Systems Design
Feature: Input System GDD
Task: Input System GDD paused at end of session for fresh-session handoff. Sections A–E WRITTEN + accepted (Overview, Player Fantasy, Detailed Design 13 rules + state table + interactions, Formulas D.1/D.3, Edge Cases 12 entries × 5 categories). Resume with `/design-system input-system` in fresh session — the skill's Phase 4 detects sections with content and resumes at F. See "Resume Handoff for Next Session" section above for the complete remaining-work checklist, open questions to carry forward, registry updates, and Phase 5 tasks. 5 specialists consulted (systems-designer 2×, ux-designer, unity-specialist, gameplay-programmer, creative-director); 4 key design decisions baked in.
<!-- /STATUS -->

## Completed This Session

- [x] `/start` — onboarding, review mode set to `full`
- [x] `/brainstorm` — game concept authored (`design/gdd/game-concept.md`)
  - CD-PILLARS: CONCERNS → APPLIED (Pillar 1 + Pillar 5 revisions)
  - AD-CONCEPT-VISUAL: APPROVE (Direction B — Cartoon Heist Diorama)
  - TD-FEASIBILITY: CONCERNS → APPLIED (4 pre-production action items logged)
  - PR-SCOPE: CONCERNS → APPLIED (re-baselined 9 months; soft cuts pre-committed)
- [x] `/setup-engine` — Unity 6.3 LTS + C# + URP 2D Renderer configured
  - CLAUDE.md updated; `.claude/docs/technical-preferences.md` populated
  - 5 Unity specialist agents given Version Awareness sections
  - Unity reference docs verified at `docs/engine-reference/unity/`
- [x] `/art-bible` — art bible authored (`design/art/art-bible.md`, 9 sections)
  - AD-ART-BIBLE: CONCERNS → APPROVED (5 clarification items resolved)
- [x] `/map-systems` — systems index authored (`design/gdd/systems-index.md`, 31 systems)
  - Enumeration: APPROVED
  - TD-SYSTEM-BOUNDARY: CONCERNS → APPLIED (4 splits + Event Bus)
  - PR-SCOPE: CONCERNS → APPLIED (3 MVP→VS slips, 2 MVP scope trims, cat theft deferred)
  - CD-SYSTEMS: CONCERNS → APPLIED (Feedback Floor contract, Pillar 3 VS-gate, multi-hole confirmed, playtest logger)

## Next Tasks

- [x] **`/design-system data-registry`** — ✅ Complete (pending review)
- [x] **`/design-system event-bus`** — ✅ Complete (pending review)
- [x] **`/design-system save-load`** — ✅ Complete, CD-GDD-ALIGN APPROVED (pending review)
- [x] **`/design-review design/gdd/data-registry.md`** — 2026-04-21 pass 1, MAJOR REVISION NEEDED → REVISED in-session (12 structural blockers). See review log pass-1 entry.
- [x] **`/design-review design/gdd/data-registry.md` (pass 2)** — 2026-04-21, NEEDS REVISION → REVISED in-session (12 precision blockers). See review log pass-2 entry.
- [x] **`/design-review design/gdd/data-registry.md` (pass 3)** — 2026-04-21 (fresh session), NEEDS REVISION → REVISED (pass-3 fix pass in fresh session per CD recommendation; 21 blocking items resolved across Tier 1 / Tier 2 / 13 surgical edits). See review log pass-3 entry + revision entry.
- [x] **`/design-review design/gdd/event-bus.md` (pass 2)** — 2026-04-21 (fresh session), NEEDS REVISION. 20 blockers + 1 structural amendment (Pillar 5 metric reformulation). Systems-index + review log updated. CD recommends fresh session for pass-3 revision. See review log pass-2 entry.
- [x] **`/design-review design/gdd/data-registry.md` (targeted pass-4)** — 2026-04-21, APPROVED (0 blockers, 2 advisory). Systems-index updated. Data Registry promoted to Approved status.
- [x] **Revise Event Bus GDD (pass-3 fix pass)** — 2026-04-21 (fresh session per CD recommendation), all 20 pass-2 blockers + 1 D.4 structural amendment resolved. 4 user design decisions adjudicated upfront. See review log pass-3 revision entry.
- [x] **Event Bus APPROVED** — user selected Option B (Accept revisions and mark Approved) after pass-3 fix pass completed, per CD pass-2 guidance. Systems-index promoted to Approved; review log final verdict entry appended. Targeted pass-4 verification skipped as acceptable risk trade for scope velocity. 2026-04-21.
- [x] **`/design-review design/gdd/save-load.md` (pass 2)** — 2026-04-21 (fresh session), MAJOR REVISION NEEDED. 15 blockers / 14 recommended / 5 nice-to-have. 0 of 10 pass-1 blockers resolved (GDD unchanged since authoring per git). 5 specialists + CD synthesis — unusually high convergence (4 of 5 flagged R10 false-premise, R12 KVPair<K,V> field-drop, R11 task.Wait() IL2CPP deadlock, R8.b missing version guard). New pass-2 findings: R12 banned list incomplete (HashSet/readonly/record/properties/interfaces), Gate Summary arithmetic wrong (34 claimed vs 43 actual), R3 fsync durability overstated for Android OEMs, R4 migrator-chain indexing ambiguity, R7 producer coherency race, SemaphoreSlim disposal missing. Systems-index + review log updated. CD 8-step resolution sequence: (1) re-ground R10 keystone, (2) fix R12, (3) add ConfigureAwait(false) AC, (4) re-derive D.1-D.4, (5) resolve OQ-1, (6) rewrite Fantasy, (7) regenerate AC table w/ R9a/R9b R11a/R11b R8a-R8e + 7 coverage-gap ACs, (8) patch OQ-4 into Event Bus. CD recommends FRESH session for revision.
- [x] **Revise Save/Load GDD (pass-3 fix pass)** — 2026-04-21 (same session as pass-2 per user selection, acknowledging fresh-session recommendation). All 10 pass-1 + 15 pass-2 blockers addressed across R10 keystone + R12 + R11 + R8.b + R3 + R7 + R4 + D.1/D.2/D.3. 3 user design decisions pre-answered. AC table regenerated 60 identifiers. OQ-1/OQ-4 closed. Event Bus GDD patched with 7 new events in the same session. See review log 2026-04-21 Revision entry.
- [x] **Patch Event Bus GDD** — 7 `gameplay.*` events added in this session (currency.changed, tool.purchased, tool.upgraded, cosmetic.purchased, progression.node_unlocked, biome.unlocked, session.level_selected) — all T2 tier, typed payloads specified, Save/Load listed as subscriber on each.
- [ ] **Decide Save/Load verification path** — either (a) `/design-review design/gdd/save-load.md` targeted pass-4 verification in a fresh session (spot-check applied fixes, NOT full adversarial), or (b) direct acceptance and promote to Approved (Event Bus precedent: Option B taken after pass-3 revision). Recommendation: given volume of pass-3 changes, targeted pass-4 is lower-risk.
- [ ] **`/consistency-check`** — recommended before designing the next system (verify save-load values don't conflict with existing GDDs)
- [ ] **`/design-system input-system`** — fourth Foundation GDD
- [ ] Continue authoring MVP Foundation GDDs in order: Input, Scene/App State Manager, Audio Bus, 2D Physics & Arc Trajectory
- [ ] Then MVP Core: Level Runtime, Level Scoring, Gopher Spawn, Tool System, Critter AI, Hazard System, Collection System
- [ ] Then MVP Feature: Currency & Economy, Progression
- [ ] Then MVP Presentation: HUD, Biome Map, Main Menu/Settings/Pause (minimal), Modal/Dialog+Level End (minimal)
- [ ] Run `/review-all-gdds` when all MVP GDDs are complete
- [ ] Run `/gate-check systems-design` before advancing to Architecture phase

## Active Files

- `design/gdd/game-concept.md` — concept doc (approved)
- `design/art/art-bible.md` — art bible (approved)
- `design/gdd/systems-index.md` — systems decomposition (approved; counts updated to 3/20)
- `design/gdd/data-registry.md` — Foundation #1 GDD (complete, pending review)
- `design/gdd/event-bus.md` — Foundation #2 GDD (complete, pending review)
- `design/gdd/save-load.md` — Foundation #3 GDD (complete, CD-GDD-ALIGN APPROVED, pending review)
- `design/registry/entities.yaml` — entity registry (updated with Save/Load referenced_by entries)
- `.claude/docs/technical-preferences.md` — engine + platform config
- `CLAUDE.md` — references Unity 6.3 LTS reference docs

## Key Decisions

- Engine: Unity 6.3 LTS (6000.3.x)
- Platform: Mobile iOS + Android, portrait, Mali-G52 floor
- Timeline: 9 months to Shippable v1.0; MVP Month 2-3; Vertical Slice Month 4
- Visual direction: Cartoon Heist Diorama (toy playset aesthetic, flat vector, 7 reserved function colors)
- Pillar-4 Six (non-negotiable MVP): Level Runtime + Scoring + Tool + Hazard + Collection + Critter AI
- Pillar 3 validation is a VS-gate (not MVP-gate) — documented in systems index
- 4 Pre-production ADRs required before first prototype: Leaderboard model (banked-count), URP mobile settings, Save schema versioning, Monetization boundary

## Pre-Production Action Items

1. Write Leaderboard Model ADR (banked-count, non-deterministic physics OK)
2. Write URP Mobile Settings ADR (no HDR, no MSAA, disabled 2D shadows, Forward renderer)
3. Write Save Schema Versioning ADR (JSON format, migration rules)
4. Write Monetization Boundary ADR (IAP catalog, ad placement rules, no pay-to-win)
5. Build "Stress Scene" in Week 2 of prototype (5 holes × 40 Rigidbody2D on target Android device)
6. Schedule Plumbing Sprint at Months 4-5 (IAP + ads + analytics + cloud save)

## Open Questions

- First biome choice for v1.0 second slot: Meadow + Orchard vs. Meadow + Mines (decision during `/art-bible` biome authoring pass, or during `/design-system` for Gopher Spawn)
- Energy system model: lives-regen vs. ad-gated retries vs. cooldown-free (deferred to soft-launch A/B test)
- Leaderboard cadence: daily/weekly/monthly/season (deferred to post-launch)
