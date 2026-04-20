# Active Session State

*Last Updated: 2026-04-20*

## Current Phase

**Systems Design** — 2 of 20 MVP GDDs complete: Data Registry + Event Bus (both pending independent `/design-review`). Next: Save/Load GDD (Foundation #3).

<!-- STATUS -->
Epic: Systems Design
Feature: Event Bus GDD
Task: Complete — CD-GDD-ALIGN CONCERNS resolved (AC-F6 added, R8 bootstrap buffer, R9 Analytics codegen); entity registry updated with 4 constants; systems-index reflects progress. Awaiting `/design-review event-bus` in fresh session.
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
- [ ] **`/design-review design/gdd/data-registry.md`** — run in a FRESH session for independent validation
- [ ] **`/design-review design/gdd/event-bus.md`** — run in a FRESH session for independent validation
- [ ] **`/design-system save-load`** — third Foundation GDD
- [ ] Continue authoring MVP Foundation GDDs in order: Save/Load, Input, Scene/App State Manager, Audio Bus, 2D Physics & Arc Trajectory
- [ ] Then MVP Core: Level Runtime, Level Scoring, Gopher Spawn, Tool System, Critter AI, Hazard System, Collection System
- [ ] Then MVP Feature: Currency & Economy, Progression
- [ ] Then MVP Presentation: HUD, Biome Map, Main Menu/Settings/Pause (minimal), Modal/Dialog+Level End (minimal)
- [ ] Run `/review-all-gdds` when all MVP GDDs are complete
- [ ] Run `/gate-check systems-design` before advancing to Architecture phase

## Active Files

- `design/gdd/game-concept.md` — concept doc (approved)
- `design/art/art-bible.md` — art bible (approved)
- `design/gdd/systems-index.md` — systems decomposition (approved)
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
