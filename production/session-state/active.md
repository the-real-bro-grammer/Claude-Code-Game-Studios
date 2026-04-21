# Active Session State

*Last Updated: 2026-04-21*

## Current Phase

**Systems Design** — 3 of 20 MVP GDDs complete: Data Registry (revised, pending re-review) + Event Bus (pending review) + Save/Load (NEEDS REVISION per prior review). Next: re-review Data Registry in fresh session, then Input System GDD (Foundation #4).

<!-- STATUS -->
Epic: Systems Design
Feature: Data Registry GDD
Task: /design-review 2026-04-21 — verdict MAJOR REVISION NEEDED → REVISED (same session). 12 blocking items resolved + major recommended items applied. Edits touched ~30 sections: R4 GUID model rewritten, EC-R2 lazy-init, Edge Case labels (EC-A*/B*/R*/F*), C_rate conflict resolved, LootTable cleanup, Pillar 3 scope disclaim + AC-R8e VS playtest gate, MVP shop note, AC-D1/D2 fixtures in-range, R1/R5 split, R8c artifact-gated, R8d decision-logic only, NHV allowlist, C1-C11 DEFERRED, P1 tightened to 5ms, P4 added, 30% threshold promoted to tuning knob, gate summary recounted (38 ACs). Awaiting independent re-review in fresh session.
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
- [x] **`/design-review design/gdd/data-registry.md`** — 2026-04-21, MAJOR REVISION NEEDED → REVISED in-session. Awaiting independent re-review in fresh session. See `design/gdd/reviews/data-registry-review-log.md`.
- [ ] **`/design-review design/gdd/data-registry.md` (re-review)** — run in a FRESH session after /clear to validate the 12 blocking fixes
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
