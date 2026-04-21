# Systems Index: Money Miner

> **Status**: Approved — all four review gates applied (enumeration, TD-SYSTEM-BOUNDARY CONCERNS resolved, PR-SCOPE CONCERNS resolved, CD-SYSTEMS CONCERNS resolved)
> **Created**: 2026-04-20
> **Last Updated**: 2026-04-20
> **Source Concept**: `design/gdd/game-concept.md`
> **Source Art Bible**: `design/art/art-bible.md`

---

## Overview

Money Miner is a mobile casual routing/collection game (Unity 6.3 LTS, URP 2D Renderer, mobile portrait). The core loop is a two-phase level structure: **Prep Phase** (drag static tools onto the farm) → **Rush Phase** (1-5+ gopher holes fling treasure; player reacts with consumables; cats steal; bombs threaten) → **Scoring** (quota met → 1-3 stars). Pillar 4 (Prep+React) requires Level Runtime + Scoring + Tool + Hazard + Collection + Critter AI to all function together at MVP — these form the non-negotiable Pillar-4 Six.

The decomposition produces **30 systems across 5 layers** (Foundation / Core / Feature / Presentation / Polish), with 20 targeted for MVP (fun-test in month 2-3), 4 added at Vertical Slice (month 4), 4 at Shippable v1.0 (month 9), and 2 deferred post-launch. The Technical Director identified 4 required splits to prevent God Object drift (Level Runtime, Tool System, Hazard System, Accessibility/Tutorial) and added an Event Bus to Foundation for decoupled event broadcasting. The Producer flagged 3 MVP→VS slips (Accessibility Service, Shop UI, Cat Theft AI) and 2 MVP scope trims (Main Menu/Settings/Pause, Modal/Dialog) to bring the 8-week MVP window from ~2× over to recoverable.

---

## Systems Enumeration

| # | System Name | Category | Layer | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|-------|----------|--------|------------|------------|
| 1 | Input System | Core | Foundation | MVP | Not Started | — | (none) |
| 2 | Scene / App State Manager | Core | Foundation | MVP | Not Started | — | (none) |
| 3 | Data Registry (inferred) | Core | Foundation | MVP | Revised (pending re-review — see [review log](reviews/data-registry-review-log.md)) | [design/gdd/data-registry.md](data-registry.md) | (none) |
| 4 | Save / Load (inferred) | Persistence | Foundation | MVP | **NEEDS REVISION** (see [review log](reviews/save-load-review-log.md)) | [design/gdd/save-load.md](save-load.md) | Data Registry |
| 5 | 2D Physics & Arc Trajectory | Gameplay | Foundation | MVP | Not Started | — | (none) |
| 6 | Audio Bus (inferred) | Audio | Foundation | MVP | Not Started | — | (none) |
| 7 | Event Bus (inferred, TD-added) | Core | Foundation | MVP | Designed (pending `/design-review`) | [design/gdd/event-bus.md](event-bus.md) | (none) |
| 8 | Level Runtime | Gameplay | Core | MVP | Not Started | — | Scene Manager, Data Registry, Save/Load, Input, Audio, Event Bus |
| 9 | Level Scoring (TD-split from Level Runtime) | Gameplay | Core | MVP | Not Started | — | Level Runtime, Data Registry, Event Bus |
| 10 | Gopher Spawn & Launch | Gameplay | Core | MVP | Not Started | — | Data Registry, 2D Physics, Level Runtime, Event Bus |
| 11 | Tool System | Gameplay | Core | MVP | Not Started | — | Data Registry, Input, 2D Physics, Level Runtime, Event Bus |
| 12 | Critter AI (TD-split; shared dog+cat substrate) | Gameplay | Core | MVP | Not Started | — | Data Registry, Event Bus, Tool System |
| 13 | Hazard System | Gameplay | Core | MVP | Not Started | — | Data Registry, 2D Physics, Level Runtime, Tool System (contract), Critter AI |
| 14 | Collection System | Gameplay | Core | MVP | Not Started | — | Input, 2D Physics, Level Runtime, Audio, Event Bus |
| 15 | Currency & Economy | Economy | Feature | MVP | Not Started | — | Save/Load, Data Registry, Collection System |
| 16 | Progression / Unlock Gating | Progression | Feature | MVP | Not Started | — | Save/Load, Data Registry, Level Scoring |
| 17 | HUD | UI | Presentation | MVP | Not Started | — | Level Runtime, Level Scoring, Currency, Tool System, Input, Event Bus |
| 18 | Biome Map / Level Select | UI | Presentation | MVP | Not Started | — | Progression, Save/Load, Input |
| 19 | Main Menu / Settings / Pause | UI | Presentation | MVP (minimal) | Not Started | — | Scene Manager, Save/Load, Audio, Input |
| 20 | Modal / Dialog + Level End | UI | Presentation | MVP (minimal) | Not Started | — | Scene Manager, Event Bus |
| 21 | VFX / Juice | Gameplay | Core | Vertical Slice | Not Started | — | Event Bus, Audio Bus |
| 22 | Accessibility Service (TD-split from Polish combo) | Meta | Core | Vertical Slice | Not Started | — | Save/Load (interface hooks authored in MVP GDDs for Level Runtime / VFX / HUD) |
| 23 | Tutorial / Onboarding (TD-split from Polish combo) | Meta | Polish | Vertical Slice | Not Started | — | Level Runtime, HUD, all teachable Core systems |
| 24 | Shop & IAP | Economy | Feature | Vertical Slice (MVP stub) | Not Started | — | Data Registry, Currency, Save/Load, Cosmetic (contract) |
| 25 | Cosmetic / Skin System | Economy | Feature | Shippable v1.0 | Not Started | — | Data Registry, Save/Load, Shop (contract) |
| 26 | Analytics / Telemetry | Meta | Polish | Shippable v1.0 | Not Started | — | Event Bus, all event-producing systems |
| 27 | Ads SDK | Meta | Polish | Shippable v1.0 | Not Started | — | Scene Manager, Shop |
| 28 | Localization | Meta | Polish | Shippable v1.0 | Not Started | — | all UI systems (string table consumers) |
| 29 | Cloud Save | Persistence | Polish | Post-launch | Not Started | — | Save/Load |
| 30 | Leaderboard | Meta | Polish | Post-launch | Not Started | — | Level Scoring, Save/Load (cloud) |

---

## Categories

| Category | Description | Systems |
|----------|-------------|---------|
| **Core** | Foundation infrastructure | Input, Scene Manager, Data Registry, Save/Load, Event Bus, Level Runtime, Level Scoring |
| **Gameplay** | Core mechanical systems | 2D Physics, Gopher Spawn, Tool, Critter AI, Hazard, Collection, VFX/Juice |
| **Progression** | Player growth and unlocks | Progression / Unlock Gating |
| **Economy** | Resource creation and consumption | Currency & Economy, Shop & IAP, Cosmetic |
| **Persistence** | Save state and continuity | Save/Load, Cloud Save |
| **UI** | Player-facing information | HUD, Biome Map, Shop UI (within Shop&IAP), Main Menu/Settings/Pause, Modal/Dialog/Level End |
| **Audio** | Sound and music | Audio Bus |
| **Meta** | Outside the core loop | Accessibility Service, Tutorial, Analytics, Ads SDK, Localization, Leaderboard |

---

## Priority Tiers

| Tier | Definition | Target Milestone | Money Miner System Count |
|------|------------|------------------|--------------------------|
| **MVP** | Required for the core loop to function; fun-test gate | Month 2-3 | 20 (15 full + 5 trimmed-scope) |
| **Vertical Slice** | Required for one complete polished biome | Month 4 | +4 new, + scope expansions on 2 trimmed MVP systems |
| **Shippable v1.0** | Mobile launch requirements (IAP, analytics, ads, localization) | Month 9 | +4 |
| **Post-launch** | Content-complete and live-service systems | Post-launch live-ops | +2 |

### MVP Scope Notes (Producer-Recommended Trims)

- **Main Menu / Settings / Pause**: MVP scope = tap-to-play screen + pause button + audio toggle. Full Settings (reduce-motion toggle, locale, gift-sending, etc.) at VS.
- **Modal / Dialog + Level End**: MVP scope = Level End screen + one confirmation modal. Full modal framework at VS.
- **Critter AI**: MVP scope = dog chase behavior only. Cat theft AI added at VS.
- **Hazard System**: MVP scope = bombs fully active; cats spawn and visually present but AI theft behavior deferred to VS (via Critter AI extension).
- **Audio Bus**: MVP scope = minimal channel routing for placeholder SFX. Full mix bus + music crossfades + Prep→Rush audio shift at VS.
- **Shop & IAP**: MVP scope = data model + coin-upgrade transactions + debug-panel UI. Full Shop UI (filter chips, cards, cosmetics) and IAP plumbing at VS.

### Pillar-4 Six (Non-negotiable MVP)

These six systems MUST all land at MVP for the Pillar 4 (Prep+React) fun-test to be valid:
- Level Runtime (state machine)
- Level Scoring (quota + stars)
- Tool System (3 static + 1 consumable)
- Hazard System (at minimum bombs)
- Collection System (tap/swipe/bank)
- Critter AI (at minimum dog chase)

If any of these cannot ship at MVP, the MVP fun-test hypothesis cannot be validated. Producer called this the "Pillar 4 firewall."

### CD-SYSTEMS Resolutions (Applied)

The Creative Director flagged that MVP fun-test validity requires three additional commitments. Applied as follows:

**Feedback Floor (MVP cross-system contract — Pillars 1 + 5)**

Every input in MVP must emit ONE audio event + ONE visual tick within 100ms, via Event Bus. This is not full VFX/Juice (which stays at VS) — it is the minimum feedback contract required to fun-test Pillar 1 (Cute Chaos) and Pillar 5 (No Input Goes Silent) at Month 2-3. Authored as explicit GDD sections in:
- **Input System** — defines the event taxonomy (tap, drag-start, drag-end, swipe)
- **Collection System** — fires `CollectionSuccess` / `CollectionMiss` events on catch outcomes
- **Tool System** — fires `ToolPlaced` / `ToolPickedUp` / `ConsumableDeployed` events
- **Hazard System** — fires `BombDetonated` / `CatStolen` / `CatCaught` events
- **Critter AI** — fires `DogDeployed` / `DogCaughtCat` / `DogReturned` events

MVP subscribers = placeholder audio (single SFX per event type) + placeholder visual (sprite-swap + scale-punch, no particles). Full VFX/Juice at VS consumes the same events without requiring changes to producer systems.

**Pillar 3 Validation — VS-Gate (not MVP-gate)**

Pillar 3 (Mastery Through Economy) is validated at Vertical Slice (Month 4), not at MVP (Month 2-3). MVP fun-test validates Pillars 2, 4 + now Pillars 1, 5 (via Feedback Floor). Rationale: Shop UI slipped to VS per producer scope adjustments; the "choose loadout → playtest → iterate" loop that IS Pillar 3 requires the Shop screen to be functional, not a debug panel. The MVP playtest brief must explicitly state this deferral so playtesters and dev do not attempt to validate Pillar 3 prematurely.

**Multi-Hole (1-5+) at MVP — Confirmed**

Gopher Spawn & Launch ships with support for 1-5+ gopher holes at MVP. The "routing" core verb collapses to "catching" at 1-2 holes — this is the difference between the fantasy delivered and the fantasy deferred. Level 1 may use 1 hole for onboarding; levels ramp to 3+ holes by mid-MVP (level 8+) to validate routing. System GDD for Gopher Spawn must specify the N-hole scalability at MVP authoring time.

**Playtest Logger (MVP, inline to Level Runtime)**

A minimal dev-only logger authored inside Level Runtime's GDD captures per-level MVP fun-test metrics: quota met / time elapsed / retry count / tool usage. Not tied to Analytics (which is at v1.0). Outputs to local JSON for manual review. Lightweight — not a separate system, but a GDD section within Level Runtime.

---

## Dependency Map

### Foundation Layer (no dependencies)

1. **Input System** — touch/drag/tap/swipe primitives. Everything interactive depends on it.
2. **Scene / App State Manager** — screen flow framework. Everything screen-bound depends on it.
3. **Data Registry** — ScriptableObject config pipeline. **Bottleneck** — every data-driven system reads from it.
4. **Save / Load** — local JSON serialization. **Bottleneck** — every persistent system depends on it.
5. **2D Physics & Arc Trajectory** — Box2D tuning + non-deterministic arc mechanics. Core to gameplay.
6. **Audio Bus** — SFX/music channels. Loosely coupled; most systems observe rather than depend.
7. **Event Bus** — pure C# typed channels. **Bottleneck** — decouples VFX/Audio/HUD/Modal from Core system internals.

### Core Layer (depends on Foundation)

8. **Level Runtime** — depends on: Scene Manager, Data Registry, Save/Load, Input, Audio, Event Bus. **Bottleneck** — all Core gameplay reports to it.
9. **Level Scoring** — depends on: Level Runtime, Data Registry, Event Bus.
10. **Gopher Spawn & Launch** — depends on: Data Registry, 2D Physics, Level Runtime, Event Bus.
11. **Tool System** — depends on: Data Registry, Input, 2D Physics, Level Runtime, Event Bus.
12. **Critter AI** — depends on: Data Registry, Event Bus, Tool System. Shared AI substrate for dogs (hero) and cats (hazard).
13. **Hazard System** — depends on: Data Registry, 2D Physics, Level Runtime, Tool System (via combatant contract), Critter AI.
14. **Collection System** — depends on: Input, 2D Physics, Level Runtime, Audio, Event Bus.
15. **VFX / Juice** — depends on: Event Bus, Audio Bus. Loose coupling via Event Bus subscriptions.
16. **Accessibility Service** — depends on: Save/Load. Pure query API consumed by Level Runtime, VFX, HUD, Physics (reduce-motion hooks authored at MVP; service ships at VS).

### Feature Layer (depends on Core)

17. **Currency & Economy** — depends on: Save/Load, Data Registry, Collection System.
18. **Progression / Unlock Gating** — depends on: Save/Load, Data Registry, Level Scoring.
19. **Shop & IAP** — depends on: Data Registry, Currency, Save/Load, Cosmetic System (contract).
20. **Cosmetic / Skin System** — depends on: Data Registry, Save/Load, Shop (via contract).

### Presentation Layer (depends on Features and Core)

21. **HUD** — depends on: Level Runtime, Level Scoring, Currency, Tool System, Input, Event Bus.
22. **Biome Map / Level Select** — depends on: Progression, Save/Load, Input.
23. **Shop UI** — depends on: Shop & IAP, Currency, Input.
24. **Main Menu / Settings / Pause** — depends on: Scene Manager, Save/Load, Audio, Input, Accessibility Service (at VS).
25. **Modal / Dialog + Level End** — depends on: Scene Manager, Event Bus. Level End is a consumer of the generic modal framework, not a special-case code path.

### Polish Layer (depends on everything)

26. **Tutorial / Onboarding** — depends on: Level Runtime, HUD, all teachable Core systems (Tool, Hazard, Collection).
27. **Analytics / Telemetry** — depends on: Event Bus, all event-producing systems.
28. **Ads SDK** — depends on: Scene Manager, Shop (for rewarded ads product catalog).
29. **Localization** — depends on: all UI systems (string table consumers).
30. **Cloud Save** — depends on: Save/Load (as adapter target).
31. **Leaderboard** — depends on: Level Scoring, Save/Load (cloud-backed).

---

## Recommended Design Order

Design GDDs in this order. Foundation first (unblocks everything), then MVP Core, then MVP Feature, then MVP Presentation, then VS additions, etc.

| Order | System | Priority | Layer | Agent(s) | Est. Effort |
|-------|--------|----------|-------|----------|-------------|
| 1 | Data Registry | MVP | Foundation | game-designer + unity-specialist | S |
| 2 | Event Bus | MVP | Foundation | game-designer + unity-specialist | S |
| 3 | Save / Load | MVP | Foundation | game-designer + unity-specialist | S |
| 4 | Input System | MVP | Foundation | game-designer + unity-specialist | S |
| 5 | Scene / App State Manager | MVP | Foundation | game-designer + unity-specialist | S |
| 6 | Audio Bus | MVP | Foundation | audio-director + unity-specialist | S |
| 7 | 2D Physics & Arc Trajectory | MVP | Foundation | systems-designer + unity-specialist | M |
| 8 | Level Runtime | MVP | Core | game-designer + systems-designer | M |
| 9 | Level Scoring | MVP | Core | systems-designer + economy-designer | S |
| 10 | Gopher Spawn & Launch | MVP | Core | game-designer + systems-designer | M |
| 11 | Tool System | MVP | Core | game-designer + systems-designer | M |
| 12 | Critter AI | MVP | Core | ai-programmer + game-designer | M |
| 13 | Hazard System | MVP | Core | game-designer + ai-programmer | M |
| 14 | Collection System | MVP | Core | game-designer + systems-designer | M |
| 15 | Currency & Economy | MVP | Feature | economy-designer | S |
| 16 | Progression / Unlock Gating | MVP | Feature | game-designer + economy-designer | M |
| 17 | HUD | MVP | Presentation | ux-designer + unity-ui-specialist | M |
| 18 | Biome Map / Level Select | MVP | Presentation | ux-designer + unity-ui-specialist | S |
| 19 | Main Menu / Settings / Pause (minimal) | MVP | Presentation | ux-designer + unity-ui-specialist | S |
| 20 | Modal / Dialog + Level End (minimal) | MVP | Presentation | ux-designer + unity-ui-specialist | S |
| 21 | VFX / Juice | VS | Core | technical-artist + unity-shader-specialist | M |
| 22 | Accessibility Service | VS | Core | accessibility-specialist + game-designer | M |
| 23 | Tutorial / Onboarding | VS | Polish | game-designer + ux-designer | M |
| 24 | Shop & IAP | VS | Feature | economy-designer + unity-specialist | L |
| 25 | Shop UI | VS | Presentation | ux-designer + unity-ui-specialist | M |
| 26 | Cosmetic / Skin System | Shippable v1.0 | Feature | economy-designer + art-director | M |
| 27 | Analytics / Telemetry | Shippable v1.0 | Polish | analytics-engineer | M |
| 28 | Ads SDK | Shippable v1.0 | Polish | unity-specialist + economy-designer | M |
| 29 | Localization | Shippable v1.0 | Polish | localization-lead | M |
| 30 | Cloud Save | Post-launch | Polish | unity-specialist | M |
| 31 | Leaderboard | Post-launch | Polish | economy-designer + network-programmer | M |

Effort: S = 1 design session (~2-4 hours); M = 2-3 sessions; L = 4+ sessions.

---

## Circular Dependencies

**None detected after TD-SYSTEM-BOUNDARY resolution.** Two potential circles were identified and resolved:

1. **Tool ↔ Hazard** — resolved by introducing the shared **Critter AI** system (both dogs and cats consume Critter AI; Tool defines dog config, Hazard defines cat config). Plus a "combatant contract" (collision layers + tags) that both systems reference without hard dependency. No circular dependency remains.

2. **Shop ↔ Cosmetic** — resolved via data-vs-transaction split: Cosmetic System defines the skin catalog data; Shop reads it and handles the transaction; Shop writes ownership via Save/Load. Neither system instantiates the other.

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| **Data Registry** | Architectural | Bottleneck — 6+ dependents; schema mistakes cascade. | Author FIRST; lock ScriptableObject authoring rules in GDD day 1; no runtime mutation. |
| **Save / Load** | Architectural | Bottleneck — 5+ dependents; migration risk at every v1.0→post-launch schema change. | Design schema versioning on day 1 (per TD). Include save-migration rules in GDD. Pre-commit to JSON (no binary) for readability. |
| **2D Physics & Arc Trajectory** | Technical | Non-deterministic physics across devices; performance on Mali-G52 under peak-density (40 bodies). | Build "stress scene" in Week 2 on target Android device. Object pooling mandatory. Leaderboards use banked-count not physics-score (ADR locked). |
| **Critter AI** | Technical + Design | New system per TD-SYSTEM-BOUNDARY; dog chase + cat theft behavior; shared AI substrate complexity. | Prototype dog chase behavior first; defer cat theft to VS. Pre-committed PR-SCOPE cut point. |
| **Tool System** | Design | Pillar 3 (Mastery Through Economy) requires multiple viable loadouts; solo-dev balance cost is high. | Data-driven ScriptableObject configs so tuning doesn't require rebuilds. Automated tests flag dominant strategies. |
| **Level Runtime** | Architectural | Bottleneck — all Core gameplay + HUD + Progression + Modal report to it. God Object risk. | TD-required split into Level Runtime (state machine) + Level Scoring (quota/stars) — done. Use MonoBehaviour-facade + pure-C#-core pattern for testability. |
| **Shop & IAP** | Scope + Platform | Plumbing sprint (IAP receipt validation, store submission, ad mediator) historically eats 6-8 weeks for solo devs. | Dedicate month 4-5 sprint to plumbing. Unity IAP + single ad mediator (LevelPlay OR AdMob, not both). |
| **VFX / Juice** | Scope | Pillar 5 (No Input Goes Silent) combined with Pillar 2 (Readability) requires precise balance; overdraw ceiling on Mali-G52. | Defer to VS so MVP fun-test validates the core loop without juice being a variable. Particle budget enforced (60 particles max at Rush Phase peak). |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 31 |
| Active (MVP/VS/v1.0) systems | 29 |
| Post-launch (deferred) systems | 2 |
| Design docs started | 3 |
| Design docs reviewed | 1 (Save/Load — MAJOR REVISION NEEDED) |
| Design docs approved | 0 |
| MVP systems designed | 3 / 20 (Data Registry + Event Bus + Save/Load; all pending independent review) |
| Vertical Slice systems designed | 0 / 5 (plus 2 MVP-minimal scope expansions) |
| Shippable v1.0 systems designed | 0 / 4 |

---

## Review Gate History

| Gate | Verdict | Date | Outcome |
|------|---------|------|---------|
| Enumeration review | APPROVED | 2026-04-20 | 22 systems approved before TD review |
| TD-SYSTEM-BOUNDARY | CONCERNS → APPLIED | 2026-04-20 | 4 splits + 1 new system (Event Bus) + Accessibility split from Tutorial; 5 issues resolved |
| PR-SCOPE | CONCERNS → APPLIED | 2026-04-20 | 3 MVP→VS slips (Accessibility Service, Shop UI, Cat theft AI) + 2 MVP scope trims (Main Menu/Settings/Pause, Modal/Dialog); Pillar-4 Six protected |
| CD-SYSTEMS | CONCERNS → APPLIED | 2026-04-20 | 5 resolutions applied: Feedback Floor contract (Input/Collection/Tool/Hazard/Critter AI emit audio + visual events at MVP), Pillar 3 documented as VS-gate, multi-hole 1-5+ confirmed at MVP, playtest logger inline to Level Runtime GDD |

---

## Next Steps

- [ ] **CD-SYSTEMS gate**: creative-director review of the full system set against pillars
- [ ] **/design-system [first-system]**: author GDDs in design order, starting with Data Registry
- [ ] **/map-systems next**: after each GDD completes, pick up the next undesigned system
- [ ] **/design-review `design/gdd/[system].md`**: run in a fresh session after each GDD is authored
- [ ] **/gate-check systems-design**: when all MVP GDDs are complete, validate readiness to advance
- [ ] **/prototype prep-react-core-loop**: prototype the Pillar-4 Six (Level Runtime + Scoring + Tool + Hazard + Collection + Critter AI) in Week 2 on target Android device as the "stress scene" per TD-FEASIBILITY pre-production action item
