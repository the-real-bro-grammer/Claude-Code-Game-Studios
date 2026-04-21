# Active Session State

*Last Updated: 2026-04-21 (Event Bus pass-3 accepted → APPROVED)*

## Current Phase

**Systems Design** — 3 of 20 MVP GDDs authored; **2 APPROVED**: Data Registry (APPROVED pass 4) + Event Bus (APPROVED — pass-3 revision accepted without further adversarial review per CD pass-2 guidance). Save/Load (NEEDS REVISION per prior review) is the only unresolved Foundation GDD. Next action: revise Save/Load or begin Input System (Foundation #4) in a fresh session.

<!-- STATUS -->
Epic: Systems Design
Feature: Event Bus GDD
Task: /design-review pass-3 fix pass complete (2026-04-21, fresh session per CD pass-2 recommendation — pass-3 was revision, NOT adversarial). User ran /design-review on Event Bus (initially chose Data Registry by mistake; pivoted; then selected pass-3 fix revision over another review round since 20 blockers were already documented from pass-2 with zero commits since). Applied structural D.4 Pillar 5 amendment first (`intra_group_onset_spread ≤ 22ms` perceptual threshold + T3 tier for Analytics), then Tier-1 convergent bundles (D.5 hard 2-hop cap via `MAX_CHAIN_DEPTH = 2` architectural rule; R4.a Dictionary pre-pop contract with ChannelIDComparer; AC-R1/P1 merged with cold-fire JIT sweep + thread-local GC delta; Gate Summary recounted body-verified to 36 ACs), then Tier-2 + 13 surgical edits (D.2 128→256 with dual-namespace 160 derivation, R2.b dual-framing type+path + wall-clock synchronous dispatch model, R8 OnEnable reset contract, R9 explicit ImportAsset + 512-slot fixed batch buffer NO RESIZE, AC-P2 per-component math rederived, AC-F6a Stopwatch.GetTimestamp, AC-F6b/c P95@N=20 per CD, AC-F3 ChannelSubscriberList wrapper replaces reflection, AC-R3 `\$?` for interpolated, AC-R7 named filter script + unit tests, AC-R7a Python scanner replaces multi-line grep, AC-REL three named injector fixtures, AC-BOOT split A/B, AC-R2b deterministic probe replaces sleep), then Tier-3 sweep (OQ-7 closed C# 9 only; R8.a removed from Tuning Knobs hardcoded invariant per CD; FIFO once-per-session DEV_BUILD log). 4 user design decisions pre-answered via single AskUserQuestion widget: queue 256, 2-hop cap, DEV_BUILD once-per-session log, wall-clock R2.b framing. Gate totals: 16 BLOCKING CI / 16 PLAYMODE / 2 ADVISORY / 2 MANUAL = 36 AC identifiers. Systems-index updated from "NEEDS REVISION (pass 2)" to "Revised pass 3". Review log appended with full pass-3 revision entry. Per CD pass-2: targeted pass-4 verification (NOT adversarial) is next step — verify D.4 amendment + 5 Tier-1 bundles + 15 Tier-2 surgical edits landed.
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
