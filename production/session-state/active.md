# Active Session State

*Last Updated: 2026-04-21*

## Current Phase

**Systems Design** — 3 of 20 MVP GDDs complete: Data Registry (revised pass 2, pending pass 3 re-review) + Event Bus (pending first review) + Save/Load (NEEDS REVISION per prior review). Next action: /clear, then re-review Data Registry in the fresh session.

<!-- STATUS -->
Epic: Systems Design
Feature: Data Registry GDD
Task: /design-review pass 2 (2026-04-21) — verdict NEEDS REVISION → REVISED (same session). 12 new blocking items resolved after pass-1 same-session revision also resolved 12 different blockers. Pass-2 fixes: R3.a catalog population via AssetPostprocessor auto-scan; R4 split into Mechanism A (OnValidate, rename) + Mechanism B (AssetPostprocessor movedAssets[], folder-move); R4 Persistence pattern with EditorUtility.SetDirty + EditorApplication.delayCall + AssetDatabase.SaveAssetIfDirty; Unity 6.3 API currency (AssetDatabase.GUIDFromAssetPath(path).ToString() replaces deprecated AssetPathToGUID throughout R4/EC-A1/EC-A2/EC-B2/AC-R4/AC-EC-B2); AC-P1 rewritten to measure first-GetById (lazy-init path), not OnEnable; AC-P1/P2 arithmetic reconciled (per-Catalog 4ms × 11 = 44ms fits 50ms aggregate); AC-D1 fixture swapped to [1,9999,1] (float32-safe), normalization in double; AC-D2 full 3-tool synthetic category fixture table spelled out (T_A/T_B/T_C producing mean=0.05); AC-R1b rewritten as Roslyn analyzer DR0001 (grep can't resolve receiver types); AC-R5b fixture existence note with named path tests/editmode/data-registry/fixtures/layer2/; AC-EC-B2 uses on-disk fixture at tests/editmode/data-registry/fixtures/hex-edit/; gate summary recounted (39 identifiers: 19 BLOCKING CI + 6 ADVISORY + 11 DEFERRED + 3 MANUAL; EC-F1 folded as fixture row in AC-D3); ProgressionNode reward type enum closed to {COINS, GEMS, TOOL_UNLOCK, LEVEL_UNLOCK, BIOME_UNLOCK} gated by Monetization ADR. Awaiting pass-3 independent re-review in fresh session. 16 RECOMMENDED items deferred (see review log).
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
- [ ] **`/design-review design/gdd/data-registry.md` (pass 3)** — run in a FRESH session after /clear to independently validate pass-2 fixes
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
