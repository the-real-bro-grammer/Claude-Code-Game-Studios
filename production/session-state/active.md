# Active Session State

*Last Updated: 2026-04-21 (Save/Load pass-3 fix pass complete — Revised pending targeted pass-4 verification or direct acceptance)*

## Current Phase

**Systems Design** — 3 of 20 MVP GDDs authored; **2 APPROVED** (Data Registry + Event Bus); **Save/Load Revised pass 3** pending pass-4 verification or direct acceptance. Per CD pass-2 Event Bus policy precedent: either run targeted `/design-review` pass-4 (spot-check applied fixes, not full adversarial) or directly accept revision and promote to Approved. Event Bus GDD also patched with 7 new `gameplay.*` events in this session to close Save/Load OQ-4.

<!-- STATUS -->
Epic: Systems Design
Feature: Save/Load GDD
Task: Save/Load pass-3 fix pass complete (2026-04-21, same session as pass-2 after user chose "Skip review, revise now" acknowledging session-state warning about fresh-session recommendation). All 10 pass-1 + 15 pass-2 blockers addressed. 3 user design decisions pre-answered via single AskUserQuestion widget: (1) R10 threading = off-main serialize + I/O (snapshot build stays on main; JsonUtility.ToJson + file I/O both on Task.Run; POCO self-containment rule R10.a added with BLOCKING CI AC-R10c; ConfigureAwait(false) enforcement via AC-R10d); (2) Reference device = defer to ADR-003 with placeholder (ADR-003 scope expanded to include named Android + iOS reference devices); (3) OQ-4 = patch Event Bus GDD now in this session (7 new events added below gameplay.level.quota_progress row: currency.changed, tool.purchased, tool.upgraded, cosmetic.purchased, progression.node_unlocked, biome.unlocked, session.level_selected — all T2 tier with typed payloads and Save/Load among subscribers). Core revisions: R10 rewritten (keystone), R11 semaphore-only pause path (no task.Wait, eliminates IL2CPP deadlock), R12 banned-list expanded to exhaustive (HashSet/SortedSet/Queue/Stack, List<List>, readonly/init, properties, record, interface fields, DateTime/TimeSpan/Guid, nullable value types, UnityEngine.Object-derived), R8.b stale-tmp version-ordering guard + AC-R8b-reject branch, R3 Android OneUI 3.x fsync caveat with architectural mitigation, R7.a producer coherency rule + AC-R7a, R4 disambiguated to root-driven + R4.c invariant table + AC-R4c, R5 migrator signature aligned. Formulas: D.1 range fixed [30, 3000] default 250ms, D.2 upper bound widened to 3000, D.3 split into D.3a FileSizeEstimate + D.3b FileSizeBudget (SaveFileSizeBudget renamed throughout), D.4 unchanged (multi-key cost resolved by R10). Coalescing queue model pseudocode updated to hold POCOs not strings; dispatch uses WaitAsync + ConfigureAwait(false); pause-path uses _writeLock.Wait synchronous. Player Fantasy rewritten in career-progress framing aligning with Resume Policy (Biome Map landing explicit). OQ-1 closed (keep banked committed). OQ-4 closed (7 events patched into Event Bus). AC table regenerated to 60 body-verified identifiers: Gate Summary reconciles 21 BLOCKING CI + 33 PLAYMODE + 5 DEVICE (ADR-003) + 1 ADVISORY = 60. R9 split into R9a (BLOCKING CI banned-field-name grep) + R9b (ADVISORY semantic review). R11 split into R11a (PLAYMODE lock) + R11b (DEVICE timing). R8 expanded into outcome-specific ACs (R8a FirstRun, R8b RecoveredFromTmp, R8b-reject, R8c LoadedClean, R8d RecoveredFromBackup + toast, R8e ResetToDefault + modal). 7 coverage-gap ACs added: R13a Settings schema version, EC-Race3 SemaphoreSlim disposal, EC-Uninit uninitialized guard, EC-PlaytestIsolation playtest logger CI grep, plus R10c/R10d/R4c already listed. Tuning Knobs: safety_margin_factor range/default corrected, R8.d silent option removed (light toast required minimum), R8.e modal tone contract, new CI severity entries for R10c/R10d/R9a/PlaytestIsolation, reference device authority = ADR-003 locked. Event Bus GDD patched with 7 new rows; Save/Load Interactions/Dependencies tables updated from "PROPOSED" → "Exists in Event Bus GDD". Systems-index updated to "Revised pass 3". Review log appended with full pass-3 revision entry (structured: Applied Changes / AC regeneration / Tuning Knobs / Event Bus patch / Next Step). Per CD Event Bus precedent: next step is targeted pass-4 verification OR direct acceptance.
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
