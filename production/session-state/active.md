# Active Session State

*Last Updated: 2026-04-21 (pass-3 fix session)*

## Current Phase

**Systems Design** — 3 of 20 MVP GDDs complete: Data Registry (revised pass 3, pending targeted pass-4 verification of Tier-1 bundles) + Event Bus (pending first review) + Save/Load (NEEDS REVISION per prior review). Next action: /clear, then targeted pass-4 of Data Registry in the fresh session (verify Tier-1 bundles only per CD recommendation — not a full re-review).

<!-- STATUS -->
Epic: Systems Design
Feature: Data Registry GDD
Task: /design-review pass 3 fix pass (2026-04-21, fresh session per CD recommendation) — 21 pass-3 blocking items resolved (6 Tier-1 convergent bundles + 2 Tier-2 unified fixes + 13 surgical edits). User adjudicated Mali-G52 scaling disagreement via Option Y (hard-deadline split: AC-P2a Editor sprint ADVISORY + AC-P2b Mali-G52 on-device VS BLOCKING). Tier-1: D.3 V_i excludes gems until Currency & Economy GDD lands (with OnValidate warning path); StarThreshold denominator anchored to LevelData.quota (not Ceiling); ProgressionNode unlockConditionType enum closed to {LEVEL_COMPLETE, STAR_COUNT, COIN_TOTAL, TOOL_OWNED} gated by Monetization ADR; ToolConfig upgrade cost monotonicity + new AC-R5d BLOCKING CI; AC-R8e rewritten as two gates (60% 1-star floor + 40pp 3-star spread ceiling, both registered as tuning knobs); AC-NHV wc-l guard refusing silent-pass on empty allowlist. Tier-2: new R4.a Asset Import Protocol subsection (StartAssetEditing/StopAssetEditing bracketing + empty-GUID sentinel guard + delayCall drain in tests); Gate Summary recounted to 42 AC identifiers / 26 scaffolding units. Surgical (13): EC-R2 Domain Reload enabled DoD; Debug.LogWarning(message, this) two-arg; AC-R4 callbackOrder=100 + BuildReport path assertion + stale-id fixture; new AC-R7a event-ID constant↔taxonomy drift test; AC-P1 stddev clamp (<20% of median, extend to 30 iters); AC-P4 per-invocation with delta-only check; AC-R5b OneTimeSetUp fixture precheck; AC-EC-A2 split to Part 1 MANUAL evidence + Part 2 BLOCKING CI; ProgressionNode rewardAmount bounded (COINS [1,10_000], GEMS [1,500]); Tool E_norm denominator params strictly positive; CurrencyDefinition field renamed to earnRateBaselineCoinsPerCompletedLevel; AC-R8d strict-> pins at 0.30f and 31/100. Status header and review log updated; awaiting targeted pass-4 verification (Tier-1 bundles only, not full re-review).
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
- [ ] **`/design-review design/gdd/data-registry.md` (targeted pass-4)** — run in a FRESH session after /clear; verify ONLY the 6 Tier-1 bundles + 2 Tier-2 unified fixes landed correctly (NOT a full re-review, per CD pass-3 recommendation)
- [ ] **`/design-review design/gdd/event-bus.md`** — run in a FRESH session for independent validation
- [ ] **`/design-review design/gdd/save-load.md`** — run in a FRESH session for independent validation
- [ ] **`/consistency-check`** — recommended before designing the next system (verify save-load values don't conflict with existing GDDs)
- [ ] **Patch Event Bus GDD** — add 7 proposed `gameplay.*` events from Save/Load OQ-4 (currency.changed, tool.purchased, tool.upgraded, cosmetic.purchased, progression.node_unlocked, biome.unlocked, session.level_selected)
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
