# Game Concept: Money Miner

*Created: 2026-04-20*
*Status: Draft — pending /design-review*

> **Creative Director Review (CD-PILLARS)**: CONCERNS (accepted with revisions to Pillars 1 and 5) — 2026-04-20
> **Art Director Review (AD-CONCEPT-VISUAL)**: APPROVE (Direction B — Cartoon Heist Diorama selected) — 2026-04-20
> **Technical Director Review (TD-FEASIBILITY)**: CONCERNS (accepted — 4 pre-production action items logged) — 2026-04-20
> **Producer Review (PR-SCOPE)**: CONCERNS (accepted — re-baselined to 9 months, soft cuts pre-committed) — 2026-04-20

---

## Elevator Pitch

> A cute mobile routing-puzzle where gophers fling gold, diamonds, and trinkets out of their farm holes — plan your tool loadout, place ramps and nets in the calm prep phase, then react in real time with guard dogs and quick catches as cats try to steal treasure mid-flight and bombs threaten to destroy your holes. Bank enough loot before the timer runs out, or the farm gets overrun.

---

## Core Identity

| Aspect | Detail |
| ---- | ---- |
| **Genre** | Casual mobile real-time routing / collection puzzle |
| **Platform** | Mobile (iOS + Android), portrait orientation |
| **Target Audience** | Mid-core mobile players ages 18-40 — "Mastery Completionists" (see Player Profile) |
| **Player Count** | Single-player with async social (leaderboards, gift-sending) |
| **Session Length** | 15-30 minutes, 5-10 levels per session |
| **Monetization** | F2P with IAP (consumables, premium currency, no-ads), no pay-to-win |
| **Estimated Scope** | Medium (9 months, solo developer) |
| **Comparable Titles** | Candy Crush Saga (meta-progression + cute casual), Plants vs. Zombies (tool placement + two-phase levels), Overcooked (prep-and-react session rhythm) |

---

## Core Fantasy

A clever little farm heist — the player is the brain of the operation, routing treasure out of 1-5+ cute gopher holes across a toy-farm diorama. The fantasy is NOT twitch reflex mastery; it is the smug satisfaction of a plan coming together. You pre-position your ramps, deploy your guard dog at exactly the right moment, and watch your carts fill with gold while cats slink away empty-pawed.

The core emotion: **"I'm too clever for them."**

This fantasy earns its differentiation from the tap-to-catch genre through the prep phase. Every level is a small puzzle followed by a small performance — you are both the architect and the improviser.

---

## Unique Hook

**Like Candy Crush + Plants vs. Zombies, AND ALSO a two-phase level structure (calm prep, reactive rush) where shop mastery and tool loadouts matter as much as reaction speed.**

The hook is the two-phase level rhythm. Mobile casual is dominated by single-phase loops (tap, match, swipe). Money Miner's prep-then-react cadence means the player always has both a "clever" moment (planning) and a "reflex" moment (execution) in every 60-90 second level — and mastery lives in both.

---

## Player Experience Analysis (MDA Framework)

### Target Aesthetics

| Aesthetic | Priority | How We Deliver It |
| ---- | ---- | ---- |
| **Sensation** (sensory pleasure) | 3 | Juicy collection feedback (coin ping, dust puffs, satisfying impacts), cartoon-heist visual language, Pillar 5 audiovisual polish |
| **Fantasy** (make-believe, role-playing) | 6 | Light — cute gopher farm theme, but not character-driven |
| **Narrative** (drama, story arc) | N/A | Explicitly excluded by anti-pillar (no cutscenes, no dialogue trees) |
| **Challenge** (obstacle course, mastery) | 1 | Prep-phase loadout strategy, rush-phase execution, star ratings, leaderboards, tool balance |
| **Fellowship** (social connection) | 7 | Async only — leaderboards, gift-sending; no live chat or real-time PvP |
| **Discovery** (exploration, secrets) | 2 | New biomes, new tools, new hazards unlocking as you progress; shop upgrades reveal new strategies |
| **Expression** (self-expression, creativity) | 5 | Loadout choice is expressive; cosmetic shop (gopher hats, farm decor) is the secondary expression layer |
| **Submission** (relaxation, comfort zone) | 4 | Early biomes are low-stakes; daily login rewards; comforting return to the farm |

### Key Dynamics (Emergent player behaviors)

- Players will **optimize loadouts before each level**, comparing tool costs against hazard types in the level brief.
- Players will **develop muscle memory for routing patterns** — "I always put a ramp on hole 2 and a guard post between holes 3 and 4."
- Players will **ration consumables strategically** across a session rather than burning them on every level.
- Players will **discuss and share tool combos** in external communities (TikTok, Discord) — the prep phase creates shareable strategy content.
- Players will **speedrun / optimize** late-game levels for star ratings once mastered.

### Core Mechanics (Systems we build)

1. **Gopher Hole Treasure Flinging** — Each hole has a rhythm and a loot table; treasure arcs into the playfield with predictable but non-trivial physics.
2. **Two-Phase Level Structure** — Prep phase (10-20s): drag static tools onto the farm. Rush phase (45-90s): real-time treasure flies, player deploys consumables and collects loot.
3. **Tool System** — Static tools (ramps, nets, guard posts) placed during prep; consumables (guard dogs, emergency nets) deployed during rush. Each tool has upgrade paths from the shop.
4. **Hazard System** — Bombs (destroy a hole if not avoided), cats (steal treasure mid-flight, countered by dogs); more hazards unlock in later biomes.
5. **Shop + Economy** — Coins earned per level; spent on permanent tool upgrades, consumable stocks, and cosmetics. Premium currency (gems) via IAP for convenience and cosmetics only.

---

## Player Motivation Profile

### Primary Psychological Needs Served

| Need | How This Game Satisfies It | Strength |
| ---- | ---- | ---- |
| **Autonomy** (freedom, meaningful choice) | Loadout selection, tool placement, purchase prioritization, biome exploration order | Core |
| **Competence** (mastery, skill growth) | Star ratings per level, visible tool upgrades, leaderboards, increasing hazard complexity | Core |
| **Relatedness** (connection, belonging) | Async leaderboards, gift-sending, shareable strategy clips | Supporting |

### Player Type Appeal (Bartle Taxonomy)

- [x] **Achievers** (goal completion, collection, progression) — How: 3-star every level, max every tool, unlock every biome, every cosmetic collectable.
- [x] **Explorers** (discovery, understanding systems, finding secrets) — How: biome unlocks, new tool reveals, hidden combo synergies, shop upgrade paths reveal new strategies.
- [ ] **Socializers** — Minimal. Async social only. Explicitly anti-pillar.
- [x] **Killers/Competitors** (domination, PvP, leaderboards) — How: async weekly/monthly leaderboards, high-score runs, star-rating competition.

### Flow State Design

- **Onboarding curve**: First 3 levels introduce only the core verb (place 1 ramp, collect from 1 hole). Level 4 adds the rush-phase timer. Level 7 introduces the first hazard (cat). Level 12 introduces bombs. Tool count grows linearly with biome progression.
- **Difficulty scaling**: Hole count scales from 1 → 2 → 3 → 4+ as biomes unlock. Quota tightens relative to available tools, forcing better loadouts. Hazard density scales to keep the prep-phase decision meaningful.
- **Feedback clarity**: Star rating on level end (1-3 stars based on % quota met), coin earn display, progress bar on the biome map, visible unlocks.
- **Recovery from failure**: Fast retry (< 2 seconds to restart a failed level). No hard punishment — energy cost is low, no loot loss. Failure is educational ("the cats got through — try a dog or a guard post").

---

## Core Loop

### Moment-to-Moment (30 seconds)

During **prep phase**: drag tools from the tray onto the farm. Tap-hold a tool in the tray, drag it onto a tile, release to place. Tap a placed tool to remove/reposition.

During **rush phase**: treasure arcs out of gopher holes; player taps falling treasure to collect, swipes to redirect, taps deployed tools (ramps redirect, nets catch, guard dogs chase cats). Cats appear at farm edges and sneak toward holes; dogs counter them. Bombs arc randomly and explode on contact unless caught by a net.

Every tap and drag receives audiovisual feedback within 100ms (juice, coin pings, dust puffs), subject to the readability clause — feedback never obscures board state.

### Short-Term (5-15 minutes, per level)

1. Level brief (5s): quota, timer, hazards present, unlocked tools
2. Prep phase (10-20s): place static tools, select loadout from unlocked inventory
3. Rush phase (45-90s): real-time treasure flight + hazards + consumables
4. Scoring beat (5s): banked treasure vs quota → 1/2/3 stars, coin reward, rare gem chance
5. Level map returns; next level teased; "continue" or "shop" choice

### Session-Level (15-30 minutes)

- 5-10 levels cleared per session
- Energy / lives system (5 retries per hour refill) OR ad-gated retries
- Natural stop: out of energy, or boss/milestone level cleared
- Off-app hook: push notification when energy refills, daily login streak reward, weekly leaderboard reset

### Long-Term Progression (days/weeks)

- **World progression**: Unlock new biomes (Meadow → Orchard → Mines → Arctic Farm → Volcano). Each biome introduces new hazards, new treasure types, new visual palette within the Cartoon Heist Diorama direction.
- **Tool progression**: Unlock new tools (ramps → magnets → conveyor belts) and new consumables (dogs → falcons → security drones).
- **Shop economy**: Earn coins → spend on tool upgrades (bigger net, faster dog, extra prep time, more consumable slots), cosmetics (gopher hats, farm decor), or energy refills.
- **IAP layer**: Premium currency (gems) for instant energy refills, exclusive cosmetics, no-ads option. Never for power-only items (anti-pillar).
- **Long-term goal**: 100% biome completion, max all tools, leaderboard climbing, cosmetic collection.

### Retention Hooks

- **Curiosity**: Next biome teased, next tool teased, "what happens when I unlock the falcon?"
- **Investment**: Tool upgrade progress, cosmetic collection, star rating completion
- **Social**: Weekly leaderboards with friends, gift-sending, shareable level clips
- **Mastery**: Star rating improvement, combo discovery, speedrun potential on late levels

---

## Game Pillars

### Pillar 1: Cute Chaos

The screen can be frantic, but it must always look adorable. Bombs, failure, stolen treasure — every "bad" event is comedic, not punishing.

*Design test*: If a feature is debated, pick the version where failure reads as slapstick within 1 second — not as punishment.

### Pillar 2: Readable at a Glance

Even at maximum board density (5+ holes, bombs, cats, treasure in the air), any player past level 10 must parse the full board state in under one second.

*Design test*: If late-game levels require squinting, redesign the visual hierarchy — do not reduce the chaos.

### Pillar 3: Mastery Through Economy, Not Just Reflex

The skill ceiling lives in *which tools you bring* and *how you sequence purchases*, as much as in reaction speed.

*Design test*: If two loadouts both win a level, good. If only one strategy ever works, rebalance tools — don't narrow the options.

### Pillar 4: Prep + React, Always Both

Every level has a calm planning beat AND a reactive chaos beat. Never a pure puzzle, never pure twitch.

*Design test*: If a level is all-prep or all-react, it's broken — add the missing beat.

### Pillar 5: No Input Goes Silent

Every input gets audiovisual feedback within the Feedback Floor — **100 ms at the 60 fps target / 133 ms at the 30 fps floor** (device-frame-rate-aware per `input-system.md` AC-P1; linear interpolation between). Sustained `fps < 30` is a thermal emergency outside this contract. When juice obscures board state, readability wins.

*Design test*: If feedback and legibility conflict, trim the feedback before sacrificing either. If the 60 fps target cannot be held under thermal load on the reference device class (Mali-G52), the 30 fps floor is the documented degradation path — not a waiver.

### Anti-Pillars (What This Game Is NOT)

- **NOT real-gambling in the shop**: No gacha with hidden odds, no "near-miss" manipulation. *Why*: violates Cute Chaos and long-term player trust.
- **NOT pay-to-win**: Shop sells cosmetics, convenience, soft-boosts, and energy; never hard power walls. *Why*: violates Mastery Through Economy; mastery must be earnable.
- **NOT a narrative game**: No cutscenes, dialogue trees, or character arcs. Cute flavor text only. *Why*: incompatible with 9-month timeline and dilutes the prep+react core.
- **NOT a real-time social game**: Social is async only (leaderboards, gift-sending). No live chat, no real-time PvP. *Why*: violates solo-dev scope and platform/moderation burden.

---

## Visual Identity Anchor

**Selected direction**: **Cartoon Heist Diorama** (Direction B from AD-CONCEPT-VISUAL)

**One-line visual rule**:
> Everything is a toy viewed from 15 degrees above — a playset in a shoebox.

**Supporting visual principles**:

1. **Geometric Primitives with Thick Outlines**
   - Shape language: cylinders, boxes, spheres, wedges — recognizable in silhouette. Holes are perfect dark circles. Ramps are flat wedges. Gophers are rounded cylindrical bodies.
   - *Design test*: If a shape requires a silhouette longer than a 2-second glance to identify, simplify it.

2. **Flat Fills on Gameplay, Gradients Only in Background**
   - Every interactive or stateful object uses flat fills with single-color accents. Backgrounds may use gradients and atmosphere.
   - *Design test*: If a treasure piece or tool uses a gradient, that gradient is a readability bug — flatten it.

3. **Unambiguous Tool Color Coding**
   - Each tool category maps to one color: ramps = blue, nets = orange, guard posts = green, dogs = red. Treasure = flat yellow. Threats (cats, bombs) = cool purple-grey and pulsing desaturated red respectively.
   - *Design test*: A new player who has seen a tool once must recognize it on the next level without the label.

**Color philosophy summary**:
Saturated primaries with strong black outlines. Background is a flat mid-green with a subtle crosshatch texture. Treasure is a literal flat yellow. Tool colors are reserved and unambiguous. Gradients exist only in atmospheric background elements (sky, distant farm buildings). Danger states (bombs pulsing, cats alerted) use motion + color together, never color alone, to remain accessible.

This anchor is the foundation of the forthcoming art bible (`/art-bible`).

---

## Inspiration and References

| Reference | What We Take From It | What We Do Differently | Why It Matters |
| ---- | ---- | ---- | ---- |
| **Candy Crush Saga** | Mobile casual meta-progression architecture; level map; soft energy system; store patterns | Prep+react two-phase level structure vs. single-phase matching | Proves the audience and monetization model |
| **Plants vs. Zombies** | Tool placement on a grid; real-time tower defense rhythm; unlockable tool progression | 2D arc physics instead of lane-based collision; shorter levels; cute farm vs. undead horror | Proves prep-and-react design works in casual |
| **Overcooked** | Prep → chaos → score cadence; visual readability under pressure; juicy feedback | Solo play, mobile tap input, async social | Proves the two-phase rhythm sustains session-length fun |
| **Fruit Ninja** | Tap/swipe satisfaction; juice on every interaction | Strategic tool placement adds a layer twitch-only games lack | Reference for feedback density and tactile feel |
| **Archero** | Easy-start / deep-mastery mobile formula; tool/ability loadout expressions | 2D physics instead of twin-stick combat; casual collection theme | Validates the "casual but skill-expressive" positioning |

**Non-game inspirations**:
- 1980s plastic farm toy sets (visual anchor for Cartoon Heist Diorama)
- Beatrix Potter illustrations (cute-creature tonal reference)
- Heist films (Ocean's Eleven tonal cue for "clever plan comes together")
- Golden-hour farm photography (background atmosphere cue)

---

## Target Player Profile

| Attribute | Detail |
| ---- | ---- |
| **Age range** | 18-40, with strong appeal in 25-35 |
| **Gaming experience** | Mid-core mobile; casual-adjacent with patience for depth |
| **Time availability** | 15-30 minute sessions, multiple per day; often short-session during commute/break |
| **Platform preference** | Mobile-primary; may also play PC casual games |
| **Current games they play** | Candy Crush, Coin Master, Archero, Royal Match, Gardenscapes |
| **What they're looking for** | Satisfying short sessions with meaningful decisions, not just matching. Visible progression. Skill expression without twitch stress. |
| **What would turn them away** | Pay-to-win walls, gacha pressure, narrative interruptions, missing progression at end-game, poor performance on their device |

---

## Technical Considerations

| Consideration | Assessment |
| ---- | ---- |
| **Recommended Engine** | **Unity 6 LTS (6000.0)** — mature mobile pipeline, C# familiarity, largest mobile ecosystem. (Developer preference confirmed; TD-FEASIBILITY approved.) |
| **Render Pipeline** | **URP 2D Renderer** — required mobile settings (no HDR, no MSAA, disabled 2D shadows, capped shadow atlas) to be codified in a dedicated ADR before first prototype. |
| **Key Technical Challenges** | (1) Stable 60fps on low-end Android (Mali-G52 class) with 40+ physics bodies at peak density. (2) Physics determinism — NOT pursued; leaderboards use banked-count, not physics-score (see Risks). (3) Touch precision with finger occlusion in a dense portrait playfield. |
| **Art Style** | 2D vector — flat fills, thick black outlines, geometric primitives. Cartoon Heist Diorama. |
| **Art Pipeline Complexity** | Low — vector-friendly style minimizes per-asset cost; achievable for solo dev in Illustrator / Affinity Designer and exported as SVG/PNG atlases. |
| **Audio Needs** | Moderate — SFX-heavy (juice across every interaction), short loopable ambient music per biome, no voice-over. SFX library + bespoke creation; music licensed or AI-assisted. |
| **Networking** | Minimal — async only. Cloud save (Unity Cloud Save or Firebase), async leaderboard backend (deferred to post-launch per scope cuts), no real-time multiplayer. |
| **Content Volume** | Shippable v1.0: 2 biomes, 40 levels, 5 tools, 3 consumables, 3 hazards. Post-launch live-ops extends to 5+ biomes, 100+ levels, additional tools/hazards. |
| **Procedural Systems** | None in core gameplay (levels are hand-designed per Pillar 4 requirement). Possible: procedural cosmetic combinations (randomized gopher hat colors). |

---

## Risks and Open Questions

### Design Risks

- **Prep phase may feel like friction on mobile.** Players expect instant action; 10-20 second prep could feel like a load screen to casual-first players. *Mitigation*: early biomes have minimal prep (1-2 tools), prep grows with player investment.
- **Pillar 3 (multiple viable loadouts) may be too expensive to balance for a solo dev across 40+ levels.** *Mitigation*: data-driven ScriptableObject configs so rebalancing doesn't require rebuilds; automated playtest tools to flag dominant strategies.
- **Pillar 4 (prep + react) may not land in MVP playtest.** If the two-phase rhythm isn't fun, 35 more levels won't save it. *Mitigation*: gate vertical slice on Pillar 4 fun-validation via `/playtest-report`.

### Technical Risks

- **Physics determinism across devices** — Unity Box2D is non-deterministic. *Decision (accepted)*: leaderboards compare "treasure banked count," not physics replay scores. Recorded as pre-production action item for ADR.
- **Peak-density performance on Mali-G52-class Android** — 5+ holes × multiple arcs + cats + dogs + VFX can tank frame rate. *Mitigation*: build a "stress scene" in week 2 of prototype (5 holes, 40 bodies, run on target device). Object pooling + sprite atlasing mandatory.
- **Mobile plumbing underestimated** — IAP + ads + analytics + store submission is consistently underestimated at 2-4 weeks; realistic is 6-8 weeks. *Mitigation*: dedicated plumbing sprint scheduled at months 4-5 in the roadmap.
- **Touch precision with finger occlusion** — solved pattern (tool-side spawn offset, enlarged hitboxes ~1.4× visual), noted for implementation.

### Market Risks

- **Saturated casual mobile market** — User acquisition cost is high. *Mitigation*: differentiated two-phase loop + unique visual direction (Cartoon Heist Diorama) create stronger creative-asset marketing angle than a me-too match-3.
- **Soft-launch may reveal fundamental retention issues** — Mobile casual depends on D1/D7/D30 retention curves that can't be predicted pre-launch. *Mitigation*: 4-8 week soft-launch window reserved (weeks 28-34) before global launch.

### Scope Risks

- **Level design workload for 40 levels** — each level needs both-phase design + tuning + playtest. Estimated ~2-3 hours per level = 80-120 hours alone. *Mitigation*: pre-committed cut (60→40 levels); design tools/templates to speed iteration.
- **Biome production cost** — art per biome is bounded by the flat vector style, but still meaningful. *Mitigation*: Shippable v1.0 scope is 2 biomes, not 3 (producer-recommended soft cut pre-committed).
- **Timeline re-baselined from 7-9 months to 9 months** per producer recommendation. Soft-launch eats 4-8 weeks; plumbing sprint is 6-8 weeks; if either overruns, the MVP cuts get pulled.

### Open Questions

- **Energy system model** — lives-regen (Candy Crush style) vs. ad-gated retries vs. cooldown-free. *Resolution*: soft-launch A/B test in Shippable v1.0 window.
- **Leaderboard cadence** — daily / weekly / monthly / season. *Resolution*: decide in `/design-system` for the Social system.
- **First biome theme** — Meadow Farm chosen as MVP biome. Alternative biomes for v1.0 second slot: Orchard vs. Mines. *Resolution*: decide during art-bible authoring (`/art-bible`) based on visual variety and hazard differentiation needs.
- **Cat/dog design direction** — how cute vs. how threatening? *Resolution*: early concept art pass during `/art-bible`.

---

## MVP Definition

**Core hypothesis**: *The prep + react two-phase level rhythm is intrinsically fun and sustains 15-30 minute sessions for the Mastery Completionist audience, even without meta-progression beyond a basic shop.*

**Required for MVP** (Month 2-3):
1. One playable biome (Meadow Farm) with 15 levels, hand-designed two-phase structure
2. Three static tools (ramp, net, guard post) with placement mechanic in prep phase
3. One consumable (guard dog) deployable during rush phase
4. Two hazards (bomb, cat) that threaten holes and steal treasure respectively
5. Timer + quota + 1/2/3 star scoring
6. Coin economy — earn from levels, spend on tool upgrades
7. Basic shop UI with functional upgrade paths
8. Fail/retry flow
9. Biome map UI with level selection and progression gating

**Explicitly NOT in MVP** (defer to later):
- IAP integration (wired in Shippable v1.0)
- Ad SDK
- Analytics SDK
- Cloud save (local save only in MVP)
- Leaderboards (deferred per scope cut)
- Cosmetics
- Additional biomes
- Additional tools beyond the 3 above
- Sound polish beyond placeholder SFX

### Scope Tiers

| Tier | Content | Features | Timeline |
| ---- | ---- | ---- | ---- |
| **MVP (Prototype)** | 1 biome, 15 levels, 3 tools, 1 consumable, 2 hazards | Core loop + coin economy + shop stub | Months 2-3 |
| **Vertical Slice** | 1 biome, 25 levels, 4 tools, 2 consumables, 2 hazards | Full shop, onboarding, local save, all Pillar 4 two-phase design proven | Month 4 |
| **Shippable v1.0** | 2 biomes (soft-cut from 3), 40 levels (soft-cut from 60), 5 tools, 3 consumables, 3 hazards | Full IAP, ads, analytics, cloud save, store submission, 4-8 week soft-launch | Month 9 |
| **Full Vision (Live-Ops)** | 5+ biomes, 100+ levels, 6 tools, 4 consumables, 4 hazards, battle pass, events, leaderboards, gift system | Full social + live-ops + cosmetics program | Post-launch |

**Pre-committed soft cuts under time pressure** (cascade order):
1. Leaderboard → defer to first post-launch patch
2. Level count 60 → 40 (committed)
3. Biome count 3 → 2 (committed)
4. Tool count 5 → 4 (last resort — damages Pillar 3)

---

## Pre-Production Action Items (from TD-FEASIBILITY)

These four items must be resolved during Pre-Production phase before coding begins on the MVP. Each will become an ADR via `/architecture-decision`:

1. **Leaderboard Model ADR** — commit to "treasure banked count" (non-deterministic physics is acceptable; leaderboards don't require physics replay).
2. **URP Mobile Settings ADR** — explicit pipeline configuration: 2D Renderer only, no HDR, no MSAA, disabled 2D shadows, single Forward renderer.
3. **Stress Scene in Week 2** — build a non-shipping test scene with 5 holes × 40 physics bodies running on target low-end Android device. This is the performance canary; re-run weekly.
4. **Plumbing Sprint Scheduled** — dedicate a single sprint at months 4-5 to IAP + ads mediator + GameAnalytics + cloud save. Use Unity IAP, single ad mediator (LevelPlay OR AdMob, not both), GameAnalytics over Firebase.

---

## Next Steps

- [ ] **`/setup-engine`** — configure Unity 6 LTS, populate `docs/engine-reference/unity/` with version-pinned reference
- [ ] **`/art-bible`** — define visual identity spec based on Cartoon Heist Diorama anchor (BEFORE writing GDDs; gates asset production)
- [ ] **`/design-review design/gdd/game-concept.md`** — validate concept completeness before going downstream
- [ ] **`/map-systems`** — decompose Money Miner into systems (level runtime, tool system, hazard system, economy, progression, shop, audio, UI, save), map dependencies, assign MVP priorities
- [ ] **`/design-system [first-system]`** — author per-system GDDs in dependency order
- [ ] **`/review-all-gdds`** — cross-system consistency check
- [ ] **`/gate-check`** — validate readiness before advancing to Architecture phase
- [ ] **`/create-architecture`** — produce the master architecture blueprint and Required ADR list
- [ ] **`/architecture-decision (×N)`** — record ADRs, starting with the 4 pre-production blockers above
- [ ] **`/create-control-manifest`** — compile decisions into actionable rules sheet for programmers
- [ ] **`/architecture-review`** — validate architecture coverage
- [ ] **`/ux-design`** — author UX specs for key screens (HUD, shop, biome map, level end)
- [ ] **`/prototype prep-react-core-loop`** — throwaway prototype to validate Pillar 4 fun hypothesis
- [ ] **`/playtest-report`** — document prototype playtest and validate core hypothesis
- [ ] **`/create-epics`** + **`/create-stories`** — decompose into implementable work
- [ ] **`/sprint-plan new`** — plan the first sprint
