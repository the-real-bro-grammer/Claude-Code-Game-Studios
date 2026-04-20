# Art Bible: Money Miner

*Created: 2026-04-20*
*Status: Approved — all 9 sections complete, AD-ART-BIBLE CONCERNS resolved*

> **Art Director Sign-Off (AD-ART-BIBLE)**: APPROVED (CONCERNS resolved) — 2026-04-20
> 5 issues raised and fixed: S4.1 shared-hue footnote; S4.2 Canopy Slate saturation-ceiling exemption; S5.1 gopher body saturation rule; S8.3/S5.6 Shop resolution policy collapsed to one decision; S7.3 gem-icon CVD check added.

---

## Section 1: Visual Identity Statement

### 1.1 The One-Line Rule

**"Everything in Money Miner is a toy on a table — solid, chunky, lit from above, with nothing too small to read at arm's length."**

The anchor line ("a playset in a shoebox") correctly names the diorama concept but leaves "shoebox" ambiguous — is the camera inside the box looking up, or above the box looking in? The refined rule collapses that ambiguity: the camera is always *above the table*, the toys are always *solid and chunky*, and legibility at arm's length (portrait phone, one-handed play) is baked into the law. Every subsequent visual decision — character scale, outline weight, particle size, UI padding — can be tested against this sentence.

### 1.2 Supporting Visual Principles

**Principle 1: Chunky Geometry, Black Borders, No Noise**

*Definition*: Every gameplay element — gophers, tools, treasure, cats, bombs — is built from rounded geometric primitives (cylinders, capsules, wedge-ramps, spheres). All fills are flat on gameplay objects. All gameplay objects carry a black outline at minimum 3px at 1080p portrait; outlines scale proportionally on lower-density screens. Surface detail (fur texture, wood grain, scuff marks) is confined to the background diorama floor and non-interactive props only.

*Design test*: When an artist asks whether the gopher's fur should have a hatched texture, this principle says no — flat fill on the body, black outline, detail only if the gopher reads as a prop rather than a gameplay actor. If a new tool type is being designed and a gradient is proposed for its fill, reject it; gradients are background-only.

*Pillar anchor*: **Pillar 2 — Readable at a Glance.** Flat fills eliminate the mid-frequency visual noise that forces the eye to "resolve" a surface before categorizing an object. At max board density, the player's visual system categorizes objects by shape + outline + color simultaneously; noise on fills adds a resolution step that breaks the <1-second parse window.

---

**Principle 2: Color Owns Function, Not Mood**

*Definition*: The seven gameplay-function colors (ramps=blue, nets=orange, guard posts=green, dogs=red, treasure=yellow, bombs=pulsing red-white cycle, cats=cool purple-grey) are immutable. These hues are never reused decoratively. Background mid-green and atmospheric gradients are drawn from a separate, desaturated palette that cannot contain these seven hues. New objects that enter the game must be assigned a function color from the reserved set or given a new reserved hue — never a borrowed one.

*Design test*: When a new seasonal skin proposes orange pumpkins for the Halloween event, this principle says orange is a reserved function color (nets). The pumpkin skin must use a different hue (e.g., deep red-brown) or be placed only on non-interactive background props. When a UI button needs a confirmation accent, it may not use yellow (treasure) as its positive color.

*Pillar anchor*: **Pillar 2 — Readable at a Glance** and **Pillar 5 — No Input Goes Silent.** Color-as-function means the player reads tool type before reading anything else. When chaos peaks (5 holes, airborne treasure, two bomb timers), color alone carries the load. VFX feedback (Pillar 5) can use flash and scale freely, but it must not introduce new hues into the reserved set.

---

**Principle 3: Slapstick Scale, Never Realistic Scale**

*Definition*: Characters are 1.5–2× the head-to-body ratio of a realistic counterpart. Treasure items are oversized relative to the holes they exit. Failed states — a bomb detonating, a cat escaping with loot, a dog tripping — are exaggerated in physics scale and animation timing (squash/stretch overshoot, brief hold on the widest deformation). No environmental element may dwarf a gameplay actor in a way that makes the actor hard to tap with a thumb.

*Design test*: When a new hole animation is proposed and the gopher's body is smaller than 40px at a standard 375pt portrait width, this principle says scale it up. When a bomb explosion VFX is proposed at realistic smoke volume, reduce the opaque fill and push the cartoonish pop: big flash, fast dissipation, no lingering cloud that covers adjacent objects.

*Pillar anchor*: **Pillar 1 — Cute Chaos.** Slapstick only reads as slapstick — not as punishment — when the failure state is physically exaggerated and the scale of the character communicates "toy," not "creature." A gopher the size of a lentil dying silently to a bomb reads as loss. A rotund gopher launching skyward with starbursts reads as Looney Tunes.

### 1.3 Supporting Narrative

Money Miner sits at the intersection of three visual traditions that have never shared a screen: the high-saturation flat geometry of 1980s plastic farm toys, the exaggerated slapstick staging of golden-era cartoon shorts, and the rigorous readability engineering of top-tier mobile puzzle games. Each tradition alone produces something familiar. Together — chunky toy forms, slapstick timing, and Candy Crush-level color discipline — they produce a game that looks like no existing title on the App Store. The diorama camera angle (15-degree overhead) is the compositional key: it is the angle at which a child places their face to look at a toy farm on a table, which immediately codes everything the player sees as something to be *played with* rather than merely observed.

The visual identity is load-bearing for the design, not decorative. Because the skill ceiling lives partly in tool preparation (Pillar 3), the board must communicate tool placement state with zero ambiguity before the chaos phase begins. Because every input must feel answered within 100ms (Pillar 5), feedback VFX must be loud and immediate — but if that feedback obscures the board state, Pillar 2 overrides it. These tensions are pre-resolved by the three principles: chunky geometry keeps object identity legible under VFX overlay, reserved function colors survive any flash or particle burst, and slapstick scale keeps failure moments quick-reading and emotionally correct. No other mobile casual game has encoded that specific hierarchy of constraints into its visual law.

---

## Section 2: Mood & Atmosphere

### 2.1 State-by-State Mood Definitions

| # | Game State | Primary Emotion Target | Lighting Character | Atmospheric Descriptors | Energy Level | Mood-Carrying Visual Element |
|---|---|---|---|---|---|---|
| 1 | **Prep Phase** | The player feels like a clever strategist — calm, unhurried, quietly confident that their setup is about to pay off. | Golden hour, low sun from screen-right; warm amber-yellow cast (3200K); low contrast; soft top-fill with long raking shadows from tools cast leftward across the diorama floor. | Sun-warmed, drowsy, tactile, pastoral, expectant | Measured | Tool drag-shadow: each tool casts a warm amber drop-shadow that stretches and contracts on the floor as the player holds and places it — a physical "weight on the table" cue that reinforces calm deliberateness. |
| 2 | **Rush Phase** | The player feels exhilarated and pleasantly overwhelmed — the productive panic of spinning too many plates, with a grin. | High-noon, directly overhead; neutral-warm (5000K); high contrast; tight top-fill, minimal shadow bleed, shadows compressed directly under objects. | Bright, crisp, urgent, buzzing, electric | Frenetic | The ambient light temperature snaps +400K hotter and the diorama floor vignette compresses inward by 15% at Rush start — a subliminal "everything got tighter" signal delivered in 0.3 seconds, never noticed consciously. |
| 3 | **Level Victory** | The player feels like they just pulled off a heist — cool, satisfied, a little smug; the star rating is the reveal. | Golden-hour callback, but theatrical: warm key light from above-left, cool-blue rim light from below-right, mid-high contrast. Spotlit. | Triumphant, gleaming, theatrical, suave, warm | Celebratory | A brief desaturated-background "spotlight lock" on the farm at the moment of quota completion — the board dims and coin particles arc upward in a tight fan, referencing the casino chip toss from a heist-film win scene. Lasts 0.8 seconds before the star-count UI expands. |
| 4 | **Level Defeat** | The player feels the butt of a cartoon joke, not a casualty — it happened fast, it was funny, and the retry button is already visible. | Overcast flat light, neutral (5500K), low contrast. Shadows eliminated. The "flat toy in bad light" look — dull but never dark or oppressive. | Sheepish, flat, briefly silly, matter-of-fact, light | Warm-idle | A single large sweat-drop particle arc from the gopher character's head (Looney Tunes timing: drop rises, peaks, falls, splats on the floor) exactly as the defeat banner appears. The background palette desaturates 20% but never goes cool — defeat cannot feel cold. |
| 5 | **Biome Map / Meta** | The player feels a sense of ownership and mild anticipation — surveying their territory, seeing progress, picking the next adventure. | Overcast-but-warm, diffuse fill (4000K); low contrast; soft global illumination from above. The map is always daytime, never dramatic. | Storybook, pastoral, breadcrumb, cozy, wide | Contemplative | The map camera rests 5-10 degrees higher than the in-level camera (wider overhead angle), shrinking all nodes to toy-scale — reinforcing that each level is "a level in the box," not a world the player inhabits. Completed biome nodes glow with a faint warm amber pulse. |
| 6 | **Shop** | The player feels the anticipation of deliberate spending — like browsing a well-stocked hardware store before a big project; considered, not rushed. | Interior artificial light (warm incandescent, 2800K); low-mid contrast; soft overhead fill, slight side-kick from screen-left casting light across item surfaces. No sun. | Inviting, boutique-warm, focused, still, slightly intimate | Measured | All non-selected shop items sit slightly desaturated and in shadow; when hovered/selected, a warm "showcase spotlight" snaps on — a subtle scale-up (1.05×) plus the light pool appears beneath the item, like a display case. Reinforces deliberate attention without urgency. |
| 7 | **Main Menu / Launch** | The player feels immediate delight and mild anticipation — this game looks fun, this world is charming, and I want to press Play right now. | Golden-hour, warm amber (3000K), low contrast, soft — the same palette as Prep Phase but slightly more saturated, as if the farm is "in its best light" for a photograph. | Inviting, photographic, golden, warm-still, toy-shop-window | Warm-idle | The farm diorama is visible as the background, idle-animating: one gopher peeks out of a hole and retreats, one cat wanders across and sits. The scene is Beatrix Potter still-life energy — something is quietly alive, but patient. The Play button is the only high-contrast element on screen. |

### 2.2 Prep-to-Rush Transition — Tonal Differentiation Protocol

Prep and Rush share the same farm geometry. The tonal shift must be unambiguous without repainting any gameplay-function object. The following changes fire simultaneously at Rush Phase start:

1. **Color temperature** steps from 3200K (amber-golden) to 5000K (neutral-bright) over 0.3 seconds.
2. **Vignette** compresses: the soft dark edge on the diorama border tightens inward by 15%, visually shrinking the playfield — "now we're focused."
3. **Shadow compression**: long raking shadows from Prep snap to tight compressed shadows directly beneath objects, eliminating the calm "long afternoon shadow" read.
4. **Ambient audio shift** (art director note for audio lead): Prep uses soft ambient (wind, birds); Rush uses escalating mechanical percussion. Visual and audio mood changes must land within the same 0.3s window.

These four changes together signal the gear-shift without touching a single gameplay-function color.

### 2.3 Mood Palette Constraints

All mood expression (lighting color temperature, vignette tint, background saturation, particle color) must draw from the desaturated background palette defined in Section 4. The seven reserved gameplay-function colors (blue / orange / green / red / yellow / pulsing red-white / cool purple-grey) are invariant across all seven states. A lighting rig that makes the net (orange) look red in Rush Phase lighting is a failed lighting rig — adjust the rig, not the object color.

Defeat state specifically: background desaturates 20% maximum. It may never shift toward blue or cool-grey, as a cool palette reads as grim or melancholy — incompatible with Slapstick Scale (Principle 3).

---

## Section 3: Shape Language

### 3.1 Character Silhouette Philosophy

**Thumbnail Rule.** Every character must be identifiable at 24×24px using silhouette alone — no color, no outline detail. This is the primary constraint. If a character fails the thumbnail test, its proportions are wrong before any other review begins. (Pillar 2; Principle 1)

**Per-Character Distinguishing Trait**

- **Gopher**: A fat hemisphere body (width 1.4× height) with two large protruding rounded ears sitting proud of the silhouette crown. At 24px the ear bumps read as the "friendly dome." Body-to-head ratio: 1 head tall, 1.2 heads wide — effectively one round lump with ears. This is where Slapstick Scale (Principle 3) is most extreme: gophers are 2 head-units tall total, with the head constituting 50% of total height. They are almost spheres with limbs.

- **Cat**: Tall and narrow relative to gopher. Two pointed ear triangles break the top silhouette. Long tail curls around the right side, giving an asymmetric hook shape that reads as predatory even at thumbnail. Body-to-head ratio: 3 head-units tall, 0.9 heads wide — distinctly vertical, which contrasts with gopher's horizontal dome. Slapstick Scale applies via the oversized head (40% of total height), but the verticality signals threat.

- **Guard Dog**: Stocky rectangle — wider than the cat, shorter than the cat, with a blunt square muzzle that protrudes left. Ears are folded flat (no ear peaks), producing a smooth top silhouette that reads immediately as "not the cat." The protruding muzzle is the single identifying trait at 24px; it gives the dog a direction cue that gophers and cats lack.

**Friend vs. Foe Silhouette Grammar**

- Friends (gophers, dogs): silhouettes are convex and low-to-ground. No sharp protrusions at the crown. The eye is invited inward.
- Foes (cats): silhouettes carry upward-pointing triangular breaks (ear peaks) and an asymmetric tail hook. The eye is deflected outward. This is a direct application of Gestalt figure-ground: convex forms read as figure/friendly, concave or spiked outlines read as disruption.

**Numerical proportions by character**

| Character | Head/Body ratio | Total height (head-units) | Dominant silhouette shape |
|---|---|---|---|
| Gopher | 1:1 | 2 | Horizontal dome |
| Cat | 1:2 | 3 | Vertical spike-top |
| Dog | 1:1.5 | 2.5 | Blunt front-heavy rectangle |

### 3.2 Environment Geometry

**Dominant vocabulary: soft geometric, not organic.** The farm floor, fences, barns, and carts are built from the same rounded primitive family as characters — rectangles with heavily radiused corners (corner radius ≥ 25% of shorter edge), cylinders for posts, wedge-trapezoids for rooftops. Organic forms (irregular rock shapes, natural tree silhouettes) are permitted only as decoration at the diorama edge, never in the playfield. Reason: organic forms compete with character silhouettes and add the visual noise that Principle 1 prohibits.

**Environment vs. Character contrast.** Characters are rounder and shorter than their environment counterparts — a fence post is taller and more rectangular than any character. This size-and-angularity contrast creates a figure-ground separation that lets characters pop without requiring heavier outlines. Environment geometry uses the same 3px black outline rule, but fill is always a muted, desaturated hue from the background palette — never a function color.

**Biome Shape Vocabulary Rule.** The core primitive set (rounded rectangles, cylinders, wedges, spheres) is invariant across all biomes. What changes per biome is the aspect ratio and texture of the surface detail on non-interactive props:

- **Meadow Farm**: short, wide, low-profile (the baseline, corner radius 25%)
- **Orchard**: medium height, cylindrical trunks dominate, rounder canopy spheres added (corner radius 25%)
- **Mines**: taller rectangles, less radius on corners (**corner radius floor: 15%** — more angular), introducing the game's only intentionally harsher geometry as environmental storytelling
- **Arctic Farm**: Meadow Farm proportions, but surface props (snow piles, icicles on barn eaves) add small downward-pointing triangles — the only biome where triangular shapes appear in the environment

The rule: corner radius is the single dial that shifts across biomes. It never goes below 15% (Mines). This keeps the toy-aesthetic intact while giving each biome a distinct geometric "temperature."

### 3.3 UI Shape Grammar

**UI echoes the toy-diorama aesthetic, with a distinct HUD register.** Buttons and panels use the same rounded-rectangle primitive as the environment, but with a specific stroke rule: UI panels carry a 2px warm-white inner stroke in addition to the 3px black outer stroke, creating a "plastic casing edge" effect. This differentiates UI surfaces from in-world surfaces (which have only the black outer stroke) without breaking the toy-aesthetic continuity.

**Primitive rules by UI element type**

- **Buttons**: Rounded rectangle, corner radius = 50% of height (pill shape at small sizes, stadium at wide). Minimum touch target: 44×44pt per Apple HIG, enforced. Black outer stroke 3px, warm-white inner stroke 2px.
- **Panels / modals**: Rounded rectangle, corner radius = 12-16pt fixed (not percentage-based, so large panels do not over-round). Max panel width at 375pt = 343pt (16pt margin each side). Stack vertically; no horizontal scrolling.
- **HUD readouts** (coin counter, timer, quota): Pill-shaped background chip, height 32pt, left-icon + right-numeral layout. Chips sit in the top bar, never overlapping the playfield.
- **Cards** (shop, tool selector): Rounded rectangle, corner radius 10pt, 16pt internal padding. Cards at 375pt width: 2-column grid with 8pt gutter.

**Readability against Rush Phase background.** During Rush Phase the playfield is busy. UI elements maintain legibility through three shape rules, not color: (1) all UI panels carry a drop shadow (4pt blur, 60% opacity black) that lifts them off the playfield regardless of background state; (2) the 2px warm-white inner stroke creates a consistent light-source edge that reads under any background color temperature shift; (3) HUD chips are fixed at the top 56pt of screen — they never overlap the playfield geometry.

### 3.4 Hero Shapes vs. Supporting Shapes

**Hero shape family: circle and dome.** Coins, gophers, gems, and bomb hazards are all primary spheroids or discs. The player must notice these first. Circles carry the highest pre-attentive salience of any primitive — they resolve faster than polygons at peripheral glance. This directly serves Pillar 2 (Readable at a Glance): the objects the player must respond to first are all round, so the visual system builds a rapid rule — "round = act now." (Principle 1: these hero shapes are the canonical chunky geometric primitives.)

**Bomb salience is intentional and confirmed.** Bombs carry double-signal: circle/hero shape + pulsing red function color. This makes bombs the highest pre-attentive salience object on the board. Gophers also use the circle family but do not pulse — color discipline (Principle 2) preserves the hierarchy so gophers read as friendly-hero while bombs read as danger-hero.

**Supporting shape family: rectangle and wedge.** Ramps, nets, guard posts, carts, fences, and barns are all rectangular or wedge-based. They do not compete with the hero circle family for pre-attentive attention. Rectangles recede into figure-ground because the eye has already been trained by Pillar 2's color discipline to categorize rectangles as "infrastructure, not target."

**Hierarchy rule for new object design.** Any new gameplay object that requires player reaction (tap, route, avoid) must be assigned a circle or dome as its primary silhouette shape, or must carry a prominent circular sub-element (e.g., a bomb with a spherical body even if it has a rectangular fuse). Any new static or passive object (tool slot, zone marker, decoration) must use a rectangle or wedge as its primary shape. A new object that violates this assignment must be escalated for review before entering the pipeline — shape hierarchy is treated as a functional constraint, not an aesthetic preference.

---

## Section 4: Color System

### 4.1 Primary Function Palette

These seven colors are load-bearing. No hue may be borrowed, shifted per biome, or reassigned decoratively. Treat as immutable constants.

| Object | Hex | Semantic Role | Appears On | Never Appears On |
|---|---|---|---|---|
| Ramps | `#2D7FE8` | "Path is redirected here" | Ramp body, directional arrow overlay | Sky backgrounds, water props, any non-CTA UI accent |
| Nets | `#F47C20` | "Collector — gather to here" | Net frame, collection zone indicator | Harvest VFX, sunset gradients, any warning tint |
| Guard Posts | `#3DB55A` | "Safe zone / defender present" | Post body, coverage radius ghost | Farm floor, foliage props, any positive UI feedback |
| Dogs | `#D93B2B` *(static — see note †)* | "Hazard — mobile threat, avoidable" | Dog body, patrol zone tint | Any negative UI feedback, bomb body (static), blood-heat VFX |
| Treasure | `#F5C800` | "Reward — collect this" | Coins, gems, loot particles, quota fill bar | Stars on victory screen (if star-shape), warning states |
| Bombs | `#D93B2B` / `#FFFFFF` *(pulsing — see note †)* | "Danger — imminent destruction" | Bomb body only — see Bomb Pulse Spec below | Any static red use (dogs use this hue; bombs PULSE it) |
| Cats | `#8B7FA8` | "Threat — theft, passive larceny" | Cat body, shadow tint | Purple UI elements, any neutral surface |

† **Shared-hue rule**: Dogs and Bombs both use `#D93B2B`, distinguished solely by animation state. **Dogs are static red; Bombs pulse red-white per the Bomb Pulse Spec below. Any static use of `#D93B2B` on a non-Dog object is a violation.** Backup distinguishing cues: bomb is spherical (hero shape, S3.4); dog is rectangular blunt-muzzle silhouette. See also S4.6 CVD safety for dog-vs-bomb pair analysis.

**Bomb Pulse Specification.**
The bomb alternates between peak-danger and off-danger states on a 0.8-second cycle:

- **Peak (0.0s – 0.2s):** Body color `#D93B2B` at 100% saturation. White highlight ring `#FFFFFF` at 90% opacity, 4px outer glow.
- **Transition (0.2s – 0.4s):** Eases to off-state (ease-in-out cubic).
- **Off (0.4s – 0.6s):** Body color `#E8A09A` (desaturated red-pink, ~40% saturation). White ring at 20% opacity.
- **Transition (0.6s – 0.8s):** Eases back to peak.
- Pulse cycle accelerates from 0.8s to 0.4s linearly over the last 3 seconds of fuse. At 0.4s cycle the white ring becomes a full-body flash.
- Dogs are static `#D93B2B` — the pulse is the sole distinguishing signal between dog (static red) and bomb (pulsing red). Backup cue: bomb is spherical (hero shape); dog is rectangular blunt-muzzle silhouette.

### 4.2 Background / Mood Palette

No hue in this palette may fall within 30° of any function color's hue angle on the HSL wheel. All entries are intentionally low-saturation (≤40%) so that function colors read as high-contrast against them.

| Name | Hex | Role | Saturation |
|---|---|---|---|
| Floor Tan | `#C8B89A` | Diorama floor surface, primary ground plane | 22% |
| Barn Cream | `#EDE0C8` | Wall surfaces, fence rails, non-interactive prop fills | 28% |
| Canopy Slate | `#8FA89A` | Overcast sky tint, flat-light atmospheric fill | 18% |
| Dusk Umber | `#A08060` | Prep Phase shadow tone, long-shadow color, shadow gradient source | 30% |
| Night-Map Indigo | `#3A3850` | Meta/map screen only — dark field under node icons | 15% |
| Defeat Ash | `#B4AAAA` | Defeat state background desaturation target (never cooler than this) | 10% |

Hue-gap confirmation: Floor Tan, Barn Cream, and Dusk Umber occupy the 30°-60° arc (warm tan); Defeat Ash is near-neutral (minimal hue vector). Night-Map Indigo is the 240° muted indigo (map screen only). **Canopy Slate `#8FA89A` is the only palette entry whose hue (~155°) falls within 30° of a function color** — specifically Guard Post Green (~135°). It is **exempted under the saturation-ceiling rule**: at ≤18% saturation, Canopy Slate cannot read as a function color even when placed adjacent to Guard Post green. The general rule: any background palette hue within 30° of a function color must be ≤20% saturation. No other background entry approaches the ramp blue (210°-230°), net orange (25°-35°), dog/bomb red (0°-10°), treasure yellow (50°-60°), or cat purple (270°-290°) arcs.

### 4.3 Semantic Color Vocabulary

**What each function color communicates — the full grammar:**

| Color | Signal | Player Reads It As |
|---|---|---|
| Blue (`#2D7FE8`) | "Direction change forced here" | Infrastructure, routing tool |
| Orange (`#F47C20`) | "Collection point — funnel to this" | Infrastructure, collection tool |
| Green (`#3DB55A`) | "Protected zone, ally stationed" | Safety, defender |
| Red (static, `#D93B2B`) | "Mobile danger — avoid or block" | Threat, dog patrol |
| Yellow (`#F5C800`) | "Reward — this is worth collecting" | Loot, quota progress |
| Red-White pulse | "Countdown danger — imminent loss" | Bomb, urgency |
| Purple-Grey (`#8B7FA8`) | "Thief — passive threat" | Cat, loss-by-inaction |

**Feedback mapping rules** — when a new UI or VFX state needs a color:

- **Positive outcome** (level pass, bonus collected): Barn Cream `#EDE0C8` surface with Treasure Yellow `#F5C800` particles. Never use Guard Post Green — green is not "success," it is "safe zone."
- **Negative outcome** (quota missed, tool limit hit): Defeat Ash `#B4AAAA` surface. Never use Dog Red or Bomb pulse — both are reserved for in-world hazards.
- **Neutral / informational**: Floor Tan `#C8B89A` or Barn Cream `#EDE0C8`. No function color.
- **Warning** (approaching quota deadline): UI timer tints to Dusk Umber `#A08060`, not red — red is reserved for in-world hazard objects.
- **Any new VFX** that needs a color: must use background palette colors. If the VFX intensity requires a saturated hue, escalate — do not borrow function colors.

### 4.4 Per-Biome Color Temperature Rules

The 7 function colors do not shift. Only background palette temperature and saturation shift.

| Biome | Ambient Temp | Background Shift | Atmospheric Tint | Floor Hue Shift |
|---|---|---|---|---|
| **Meadow Farm** (MVP) | 4500K, neutral-warm | Baseline — all palette values as defined in 4.2 | None | Floor Tan `#C8B89A` unchanged |
| **Orchard** | 3800K, golden-warm | +8% saturation on Floor Tan, Barn Cream shifts to `#F0E0C0` | Faint amber atmospheric haze: `#F5DCA0` at 10% overlay | Floor warms to `#C8A878` |
| **Mines** | 2500K, deep amber-orange | Floor Tan darkens to `#7A6448`; Barn Cream to `#B09070` | Torch-orange vignette `#804020` at 15% overlay on playfield edges | Floor darkens sharply — highest contrast shift |
| **Arctic Farm** | 6500K, cool-daylight | Floor Tan shifts to `#C0C8D0` (slight blue-grey); Barn Cream to `#E8EEF0` | No tint overlay — clarity is the biome's atmosphere | Snow-white ground `#E8EEF0` on non-path tiles |
| **Volcano** | 2000K, lava-deep | Floor darkens to `#4A3020`; atmospheric heat shimmer (VFX) | Deep red-brown vignette `#602010` at 20% overlay | Ash-black floor `#302820` between lava tiles |

**Arctic Outline Exception (locked).** Arctic is the only biome that risks function-color confusion: the background cools to blue-grey territory. Ramp blue (`#2D7FE8`) must be tested against Arctic floor `#C0C8D0` at 10px and 24px scales. If contrast ratio falls below 3:1, **ramp outline weight increases from 3px to 4px in Arctic biome only** — the single permitted per-biome outline exception.

### 4.5 UI Palette

**Decision: UI palette is a DERIVED SUBSET of the world palette, not a separate system.** Rationale: Mobile casual games that run a fully separate UI palette create a "widget on a world" seam that reads as cheap and low-craft. Because the diorama aesthetic is the game's primary identity differentiator, the UI must feel continuous with the world — plastic HUD chips, not glass-morphism panels.

| Role | Hex | Derivation |
|---|---|---|
| UI Surface (primary) | `#EDE0C8` | Barn Cream — matches non-interactive prop fills |
| UI Surface (secondary) | `#C8B89A` | Floor Tan — recessed panels, card backgrounds |
| UI Text — Headline | `#1A1410` | Near-black warm (not pure black — matches outline tone) |
| UI Text — Body | `#3D3020` | Dark warm brown — 60% of headline contrast |
| UI Text — Disabled | `#6B6055` | Dark warm grey — passes WCAG AA 4.5:1 against Barn Cream; required for outdoor sunlight readability per S7 ux-designer check |
| UI Text — Error | `#A02018` | Darkened Dog Red — never same hex as in-world `#D93B2B` |
| UI Accent — Primary CTA | `#2D7FE8` | Ramp Blue — used ONLY for the primary CTA button (Play, Confirm) |
| UI Accent — Secondary | `#F5C800` | Treasure Yellow — used ONLY for quota fill bar, coin counter icon |
| UI Positive | `#EDE0C8` surface + `#F5C800` particle | **No solid green** — reason: green is reserved for Guard Post |
| UI Warning | `#A08060` (Dusk Umber) | Not red — red is reserved for in-world hazard |
| UI Error / Destructive | `#A02018` | Same as UI Text Error — distinct from `#D93B2B` |

**Ramp Blue (`#2D7FE8`) in UI — locked.** Permitted for the single primary CTA per screen only. When Ramp Blue appears on a button, it reads "the action that moves the board forward" — same semantic as in-world (ramps redirect flow). This is intentional semantic reinforcement, not a violation of Principle 2. Any secondary button must not use blue.

### 4.6 Colorblind Safety Analysis

**Test requirement:** All critical pairs must pass Coblis v2 simulation or equivalent at 24px game-board scale before any biome ships.

**Deuteranopia (red-green, most common, ~6% of males)**

| Pair | Risk | Backup Cue |
|---|---|---|
| Dogs red `#D93B2B` vs. Guard Posts green `#3DB55A` | HIGH — both shift toward brownish-yellow | Shape: dog is rectangular blunt-muzzle; post is cylindrical vertical. Motion: dog patrols, post is static. |
| Treasure yellow `#F5C800` vs. Guard Posts green `#3DB55A` | MEDIUM — yellow holds under deuteranopia; green shifts tan | Shape: treasure is disc/sphere (hero shape); post is rectangle (supporting shape). |
| Nets orange `#F47C20` vs. Dogs red `#D93B2B` | HIGH — both shift toward similar yellow-brown | Shape: net is rectangular infrastructure; dog is character with muzzle. Motion: dog moves. |

**Protanopia (red-blind, ~1% of males)**

| Pair | Risk | Backup Cue |
|---|---|---|
| Dogs red `#D93B2B` vs. Ramps blue `#2D7FE8` | LOW — blue remains distinct | None required. |
| Bombs pulse `#D93B2B` vs. Guard Posts green `#3DB55A` | HIGH — red and green both flatten | Bomb is spherical + animated pulse (white flash). Post is static cylinder. Sound: bomb has distinct ticking audio. |

**Tritanopia (blue-yellow, rare, ~0.01%)**

| Pair | Risk | Backup Cue |
|---|---|---|
| Ramps blue `#2D7FE8` vs. Cats purple-grey `#8B7FA8` | MEDIUM — blue and purple-grey may conflate | Shape: ramp is wedge (supporting); cat is vertical silhouette with pointed ears (character). Motion: cat moves, ramp is static. |
| Treasure yellow `#F5C800` vs. Nets orange `#F47C20` | MEDIUM — blue channel loss shifts both toward similar tone | Shape: treasure is sphere/disc; net is rectangular frame. Size: nets are large infrastructure; coins are small hero items. |

**Monochromacy (complete color blindness, very rare)**

All seven function colors collapse to greyscale. Full monochromacy is a non-blocking accessibility target (not WCAG-mandated for games) but requires that shape + motion alone carry all critical gameplay information. Confirmation: every hazard (bomb, dog, cat) has a unique silhouette and motion pattern per Section 3. Ramps and nets, as static infrastructure, must carry icon overlays (directional arrow on ramp, net-weave pattern on net) so they remain distinguishable without color. These icon overlays are required in all biomes.

**Dogs red vs. Bombs pulsing red — primary same-hue pair.** Static vs. animated is the primary cue. Backup: bomb is spherical (hero shape, 3.4), dog is rectangular-muzzle (supporting shape register). Secondary backup: bomb has a visible fuse element (short cylinder on top) that dogs never have. Tertiary: audio (ticking). These three non-color cues make the pair safe under all colorblindness types.

---

## Section 5: Character Design Direction

### 5.1 Gopher Design Bible

**Visual Archetype — The Platonic Gopher**

The canonical gopher is a horizontal dome with stubby arms, paddle-like paws, a permanent wide grin, and two oversized rounded ears sitting proud of the crown (S3: 2 head-units tall, head = 50% of height, body width = 1.4× height). Eyes are large dark discs — 30% of face width — with a single white specular dot. Nose: a small dark oval centered low on the face. There are no visible legs at game camera; the body terminates in a flat base that sits flush with the ground plane, reinforcing the "toy on a table" read. *Pillar 1: Cute Chaos; Pillar 2: Readable at a Glance.*

**Across-Biome Variation Rules**

Base body color is drawn from the biome's background palette (never a function color). Meadow gopher: warm mid-tan `#C09A6A` (~35% saturation — close in hue to Net Orange but exempted under S4.2's saturation-ceiling rule; skin variants MAY NOT exceed 40% saturation on body fill). Mines gopher: charcoal-grey `#6A5A48`. Arctic gopher: cool off-white `#D8E0E8`. Volcano gopher: deep brown-red `#7A4030`.

Biome identity is carried by one costume accent — never more than one — on the head or torso only: a miner's helmet (Mines), a knitted ear-flap hat (Arctic), a bandana (Meadow, default), a fireproof collar (Volcano). The accent must not alter the ear silhouette or the dome crown. The platonic silhouette must survive removal of the accent. *Pillar 2.*

**Cosmetic Variant Rules**

Hats and outfits may add a single element to the silhouette above the crown (max 12px height at game camera). They may never: (a) recolor the ears or face (ears are the primary silhouette identifier), (b) introduce any of the seven function colors, (c) reduce ear visibility to below 20% of their default height. Accessories on the body (bow tie, sash) are permitted if they stay within the existing body bounding box. *Pillars 1 and 2.*

**Expression and Pose Philosophy (4-pose system — locked)**

Gophers hold four posed states (no frame-by-frame animation on body): Idle (upright, ears perked), Alert (body tilted 10° forward, ears flattened), Tossing (arm raised, leaning back 15°), and Defeated (both arms down, body slumped 20°, eyes half-closed). Face overlay animates separately: 2 eye states (open / squint) × 2 mouth states (grin / straight) = 4 face combos covering all four poses. Happy/celebrating moments are delivered via face overlay + positional tween (brief scale pulse, star-burst particles), not a distinct body pose. *Solo-dev budget: 4 body sprites + 4 face overlays = 8 assets per gopher variant.* *Pillar 1; Pillar 5.*

**LOD at Game Camera (60-80px)**

At 60-80px the following details are legible and must be designed: ear shape, eye size, costume accent silhouette, and arm position. The following are not legible and must not be designed-for: individual fur strokes, specular highlights below 4px, nostril detail, claw detail on paws. Design rule: no detail smaller than 6px at game camera. *Pillar 2.*

### 5.2 Cat Design Bible

**Visual Archetype — The Platonic Cat**

Vertical spike-top silhouette (S3: 3 head-units tall, 0.9 heads wide). Two sharply triangular ears break the crown — these are the primary foe signal from Section 3's friend/foe grammar and must never be rounded. Tail hooks rightward and upward in a tight curl, creating the asymmetric silhouette. Eyes are narrow horizontal ellipses (sly, not large-and-friendly), 20% of face width. Body color: cat purple-grey `#8B7FA8` (invariant function color, S4). The tail tip carries a slightly lighter tone `#A89DC0` — the only internal value variation permitted on cat fills. *Pillar 2; Principle 2.*

**Across-Biome Variation Rules**

Because cat body color is a locked function color, biome variation is carried by: (a) a secondary costume element on the tail tip only (Arctic: tail tip becomes white-tipped; Mines: tail tip is singed brown), or (b) a small accessory below the face that does not interrupt the ear silhouette (Meadow: no accessory, baseline; Arctic: a scarf at neck level; Mines: dust goggles pushed up to forehead — never covering eyes). Function color of body never shifts. *Principle 2; Pillar 2.*

**Pose Philosophy**

Four poses: Sneaking (body low, 30° forward lean, ears flat), Pouncing (mid-air arc, legs extended, ears pinned), Retreating (body tilted 20° backward, tail extended behind), and "Looting" (sitting upright, one paw holding a coin with exaggerated pride, tail curled smugly). The Looting pose is the visual reward for the cat — designed to read as funny, not threatening. *Pillar 1: the enemy's "victory" is presented as a comedy beat, not a punishment.*

**Cute-But-Foe Design Rule**

The cat is an adorable troublemaker, not a villain. The line is maintained by three rules: (1) eyes are narrow but never angry-angled (no inner-brow furrow), (2) the "Looting" pose always shows the cat's face in a satisfied grin, never a sneer, (3) slapstick defeat — when the dog catches the cat, the cat's expression is wide-eyed surprise (a cartoony "!!" read), not pain. Cats are mischievous; dogs interrupt their mischief; both are toys. *Pillar 1; Principle 3: Slapstick Scale.*

### 5.3 Dog Design Bible

**Visual Archetype — The Platonic Guard Dog**

Stocky rectangle with protruding blunt muzzle (S3: 2.5 head-units, muzzle protrudes left = direction indicator). Ears are flat and floppy — smooth top silhouette, zero ear peaks. Dog body color: `#D93B2B` dog red (invariant). Muzzle is `#E8B89A` (warm cream, background-safe). Large dot eyes, 25% of face width — rounder than cat eyes, friendlier register. *Pillar 2: friend/foe grammar (convex, low, no crown spikes).*

**Upgrade Tier Visuals**

Three tiers, all within Dog Red + background-safe accessories:

- **Basic Dog**: plain red body, cream muzzle, floppy ears, no accessories
- **Trained Dog**: small neckerchief in Barn Cream `#EDE0C8`, ears slightly more upright (less floppy, same flat-top silhouette), slightly wider chest (body 10% wider)
- **Elite Dog**: a simple badge on chest (circular, Barn Cream, no new function color), ears fully upright but still flat-topped, body 20% wider than Basic. A star detail on the badge is Floor Tan `#C8B89A` — background palette only.

Rule: no tier may introduce a new function color. Silhouette must remain identifiably dog at 24px after upgrade. *Pillar 3: Mastery Through Economy (upgrades are legible progression).*

**Behavior Visuals**

- **Deploy**: drops in from top of screen, lands with a squash-and-stretch bounce (Principle 3: Slapstick Scale)
- **Chase**: body tilts 30° forward, legs are implied by a rapid "running shimmy" (2-frame alternation: lean left / lean right)
- **Return/Dismissal**: body tilts upright, brief tail-wag (1 bounce cycle)
- **Celebration**: full-body jump (body raises 8px, squashes on land), small star-burst particles in background palette tones only

**Dog-Cat Conflict Visual (locked)**

When dog catches cat: a **pure-white cartoon collision cloud** (white oval with jagged edges, 48×48px, 0.4s hold) replaces both characters for 0.4 seconds. Cloud dissipates. Dog reappears standing upright, ears perked. Cat reappears mid-bounce-off-screen (retreating pose, angled 45° upward). No stars, no blood, no impact lines — only the cloud. **Pure white is reserved exclusively for this moment in gameplay** — the only time it appears, making the event instantly readable as a slapstick resolution. *Pillar 1; Principle 3: Slapstick, never punitive.*

### 5.4 Tool Design Language

**Ramps**

Blue wedge (`#2D7FE8`) per S4. Surface detail: a single directional arrow overlay (2px white line, arrow points along the ramp angle) is required in all biomes per S4.6 monochromacy rule. The ramp body is a solid wedge with a slightly raised "lip" at the top edge — a 4px ridge that reads as a physical launcher. Biome variation: Mines ramp has a rougher silhouette edge (straight cuts, 15% corner radius per S3.2 Mines rule). Arctic ramp has a snow cap prop on the flat face (background-safe white `#E8EEF0`). Core color and shape never change.

**Nets**

Orange rectangle frame (`#F47C20`) with a net-weave icon overlay (2px Barn Cream `#EDE0C8` crossing-line pattern, 4 cells × 4 cells) required per S4.6. The frame has a slight trapezoid taper (wider at base) to read as a catching shape. Biome variation follows the same one-prop rule as ramps: an icicle trim (Arctic) or rope-and-timber frame details (Mines). Core orange frame is invariant.

**Guard Posts**

Green cylinder (`#3DB55A`) with a flat circular cap. A small flag or banner on a 2px pole at cap-top is the signature element — flag color is Barn Cream (never a function color). Biome variation: flag design changes (skull-and-crossbones silhouette for Mines, snowflake silhouette for Arctic — both in Barn Cream). Core green cylinder never changes.

**Tool States**

| Tool Type | State | Visual Change |
|---|---|---|
| Static tool (ramp / net / guard post) | Unplaced (in prep tray) | 80% opacity, drop shadow off |
| Static tool | Placed (idle in Prep) | 100% opacity, warm Prep Phase drop shadow (Dusk Umber, 4px) |
| Static tool | Active (treasure arc on ramp / treasure caught by net / cat inside post radius) | A Treasure Yellow `#F5C800` particle arc emits from the tool — the only moment function color appears on a tool, and it is the treasure's own color, not the tool's |
| Static tool | Post-level summary (shown on scoring screen) | 50% opacity, 60% saturation, Defeat Ash `#B4AAAA` backdrop behind usage-count numeral |
| Consumable (guard dog / emergency net) | Unused (in rush-phase tray) | 100% opacity, stock-count badge in top-right corner |
| Consumable | Deployed (in rush-phase world) | Character/object appears on board; tray stock-count decrements |
| Consumable | Exhausted (stock = 0) | Tray icon desaturates to 40% saturation, crossed-out overlay in Defeat Ash |

*Pillar 4: Prep+React — state transitions give the player instant board-state reads between Rush Phase decisions.*

### 5.5 Expression and Personality Framework

**Expressiveness Calibration (locked at 4/10)**

Reference scale: Candy Crush Saga characters are at 7/10 (looping idle animations, responsive winks, full mouth animation). Money Miner targets **4/10 — the Crossy Road register**. Characters are endearing and readable, not conversational. They do not address the player. Their expressions are state flags, not performances.

**Required Expression Set (all characters)**

| Expression | Trigger | Visual Signal |
|---|---|---|
| Idle | Default, between events | Upright pose, open eyes, grin (gopher/dog) or neutral (cat) |
| Alert | Player interaction nearby / deploy | Forward lean, eyes wide |
| Happy | Successful action (treasure tossed, cat caught) | Brief scale pulse, wide grin, star particles — no dedicated body pose |
| Defeated/Surprised | Negative event (bomb, cat steals) | Wide eyes, arms up, sweat drop particle |

**Animation Budget Ceiling — Solo Dev Rule**

Maximum 3 animated states per character, each state maximum 2-frame alternation (no skeletal animation, no full walk cycles). All motion beyond 2-frame alternation is achieved via positional tweening (scale, rotation, translation) on a single sprite. VFX particles carry the remaining emotional weight. A character that needs a 6-frame walk cycle is out of scope — use a 2-frame shimmy + positional tween instead. *Supports Pillar 5: feedback must be immediate, not dependent on long animation resolution.*

### 5.6 LOD and Camera Distance Philosophy

**Camera Distance Reference**

At game camera (60-80px character height on a 375pt portrait screen), the player is seeing approximately 8-10× the thumbnail (24px). At this distance the following detail tiers are visible:

| Detail Tier | Visible at 24px | Visible at 60-80px | Design rule |
|---|---|---|---|
| Silhouette shape | Yes | Yes | Must be correct at 24px first |
| Ear / muzzle subshape | No | Yes | Design to read at 60px, test at 24px without it |
| Eye shape and size | No | Yes | Minimum eye disc diameter: 8px at game camera |
| Costume accent silhouette | No | Yes | Must read as distinct shape, not color |
| Outline weight | Implied | Clear | 3px black at game camera = visible and crisp |
| Surface detail (fur, texture) | No | No | Prohibited on gameplay objects (Principle 1) |
| Face overlay (expression) | No | Marginal | Eyes only — mouth detail below 4px is lost |

**The 6px Floor Rule**

No intentional detail smaller than 6px at game camera (60px character height). Any detail that only appears at 120px+ (e.g., badge text, fine accessory filigree) is non-functional decoration and is out of scope for the solo-dev timeline. If it cannot be read at 60px it does not exist in the design budget.

**Thumbnail Rule Primacy**

The 24px silhouette test (S3) is the baseline. The 60-80px game camera is the design target. **No separate Shop-only asset tier exists.** The Shop screen renders the same 128×128 gameplay sprite scaled to 120px via Canvas — a shared source, not a second asset (see S8.3). At 120px render, the extra detail tier (eye shape + mouth shape simultaneously readable) is achievable because the 128×128 source resolution already encodes that information for sharp gameplay display; the Shop scale-up surfaces it without additional authoring. If playtest shows Shop softening at 120px, the ONE permitted response is to upsize the gameplay source to 160×160 — used everywhere, never Shop-only. *Pillar 2; Principle 1.*

---

## Section 6: Environment Design Language

### 6.1 Farm Architectural Style and World Framing

**Cultural Reference: American Playset Americana, circa 1970-1985.**

The Money Miner farm reads as a Fischer-Price / Little People toy farm from the late analog-toy era — chunky simplified barns with exaggerated rooftop pitch (1:1 pitch ratio, wider than real barns), round-cornered fence rails, squat silos with a domed cap. No regional specificity beyond "generic Midwestern American farmyard" as filtered through mass-market toy design. This era is chosen deliberately: it is pre-digital, pre-ironic, and universally coded as "toy" to players across cultures. *Pillar 1: Cute Chaos — the toy register activates immediately on load.*

**Biome World Coherence Rule: Distinct Dioramas, Not a Continuous World.**

Each biome is a separate toy-box diorama on the same table — a different playset purchased separately, not a region of one farm. This matters for scope: biomes share no architectural continuity obligation. A Mines diorama and a Meadow diorama can coexist in the meta-map as "different toy boxes the player owns." This gives the solo developer full license to change architectural vocabulary per biome without visual contradiction. The meta-map (S2, State 5) renders all biome nodes as small toy-box icons to reinforce this read. *Pillar 3: Mastery Through Economy — each biome introduces a distinct environmental vocabulary the player learns to read.*

**Per-Biome Architectural Character:**

| Biome | Architectural Anchor | Distinguishing Shape |
|---|---|---|
| Meadow Farm | American toy barn: tall pitched roof (1:1 pitch), wide double door arc, red body, cream trim | Pitched wedge rooftop — biome's "signature shape" |
| Orchard | Low farmhouse + cylindrical trunk rows; no barn | Cylinder grid (tree trunks as repeating vertical element) |
| Mines | Timber-frame mine entrance arch (rectangular portal + two diagonal brace beams); no barn | Diagonal timber brace — the only diagonal structural line in the game |
| Arctic Farm | Meadow barn proportions + rounded snow-cap overhangs on every horizontal surface | Snow-cap overhang (downward-bulging white half-cylinder) |
| Volcano | Crumbled brick platform edges, no intact structures; broken arch remnants | Broken/interrupted geometry (arch with missing keystone) |

The Mines' diagonal timber brace is the sole exception to the "no diagonal structural lines" default. It is restricted to background props only, reinforcing the Mines' intentional angularity (S3.2: corner radius 15%). *Principle 1: structural diagonals on gameplay objects remain prohibited.*

### 6.2 Texture Philosophy

**Base approach: Flat-filled gameplay layer, single-gradient prop layer, no PBR.**

No physically-based rendering textures anywhere in the game. Mobile performance and the S1 visual law prohibit it.

**Rule: Gameplay objects = zero texture, zero gradient, zero pattern. Non-interactive props = one flat color with optional single-color crosshatch or dot-stamp detail.**

The "optional detail" on props is limited to a single background-palette tone applied as a 4px repeating stamp or 2px crosshatch — sufficient to read as "wood grain" or "stone" without introducing noise on the gameplay layer. The stamp/crosshatch is never on a gameplay-function surface. This is the precise boundary:

| Element Type | Texture Permitted | Example |
|---|---|---|
| Gophers, cats, dogs, tools, treasure | None — flat fill only | Gopher body: solid `#C09A6A` |
| Holes, zone markers, HUD chips | None | Hole: solid dark oval |
| Fence rails, barn walls, carts (bg) | Single-tone stamp, max 4px repeat | Barn wall: Barn Cream + 2px plank-line in Dusk Umber |
| Diorama floor | Gradient permitted (radial, center-light) | Floor Tan center lightens 15% toward center of playfield |
| Sky / diorama edge | Full gradient | Canopy Slate with top-fade to Barn Cream |

**Mobile draw call budget: 1 sprite atlas per biome, 2 atlases maximum per session.** The single-tone stamp approach allows all prop textures to bake into the atlas as flat sprites with no shader dependency. No runtime texture blending. *Supports Pillar 2: the GPU headroom saved here is the budget that buys responsive feedback VFX.*

### 6.3 Prop Density Rules

**Rule: Maximum 8 non-interactive background props in a single playfield view.**

| Biome | Density Tier | Prop Count (visible) | Rationale |
|---|---|---|---|
| Meadow Farm | Moderate | 5-8 | Baseline — enough to read "farmstead," not enough to crowd |
| Orchard | Moderate-Dense | 6-8 | Tree trunks define playfield lane implicitly; each trunk counts as 1 prop |
| Mines | Sparse | 3-5 | Dark floor + vignette already communicate biome; fewer props reduce confusion in low-key-light conditions |
| Arctic Farm | Moderate | 5-7 | Snow piles and icicle clusters count as one prop unit each |
| Volcano | Sparse | 3-5 | Lava tile VFX fills atmospheric space; prop silhouettes would compete |

**Density Determination Rule:** Prop count is set by the complexity of the biome's atmospheric effects. Biomes with strong VFX (Volcano lava shimmer, Mines torch flicker) get lower prop counts to preserve Pillar 2. Biomes with flat atmosphere (Meadow, Arctic clear-sky) absorb moderate prop counts. *Pillar 2: Readable at a Glance — the player must parse the board state in under 1 second at maximum density.*

**Prop Placement Constraint:** No prop may be placed within the central 60% of the playfield width. All props are edge-adjacent (within 24pt of the diorama border) or anchored to the far background row. Zero props in the active zone where gopher holes, tools, and carts operate. *Pillar 4: Prep+React — cluttered placement zones destroy the calm deliberateness of prep.*

### 6.4 Environmental Storytelling

**Rule: Every biome story beat must resolve within the playfield frame. Zero off-screen story.**

Environmental storytelling operates via three layers: floor surface, prop silhouette, and atmospheric effect. All three are visible from game camera.

**Meadow Farm — "Peaceful, profitable farmstead":**
- Floor surface: warm Floor Tan with subtle radial lightening at center (sun on dirt)
- Props: a single red barn wall at frame edge, a fence line of 3-4 posts, scattered coin/clover floor decals (non-interactive, flat, background-palette)
- Atmospheric: birdsong particles (small white dot arcs rising from floor edge, 4px, 1.5s cycle) — the only prop-free storytelling element

**Mines — "Underground, treasure-rich, slightly dangerous":**
- Floor surface: darkened Floor Tan `#7A6448` with irregular stone-seam lines (2px Dusk Umber, no more than 4 seams per view) — reads as cave floor
- Props: 2-3 timber brace pairs at playfield edges, 1 overturned ore cart (background layer only), wall-mounted torch prop with flicker VFX
- Atmospheric: torch-orange vignette `#804020` at 15% opacity on playfield edges; faint dust-mote particles (Barn Cream `#EDE0C8` dots, 3px, random drift, max 6 on screen at once)

**Arctic Farm — "Cold, remote, treasure hidden under ice":**
- Floor surface: Snow-white ground `#E8EEF0`; ice-tile variants (2px Canopy Slate `#8FA89A` grid lines creating 48px tile illusion)
- Props: snow-pile clusters at frame edges, a barn wall with snow-cap overhang, icicle strings hanging from top frame edge (downward triangles, background-safe white)
- Atmospheric: occasional snowflake particle (3px cross shape, slow drift, max 8 on screen) — no wind blur; the stillness communicates "remote"

*Pillar 5: No Input Goes Silent — environmental storytelling particles (dust motes, snow, birds) are all ambient and never compete with feedback VFX. They are smaller than any feedback particle (always ≤4px), slower, and loop continuously rather than triggering on input.*

### 6.5 Playfield Zone Anatomy

**Zone types and their visual language:**

| Zone | Visual Marker | Shape | Color | Opacity | Legible in Rush? |
|---|---|---|---|---|---|
| Gopher hole zone | Dark oval depression in floor | Ellipse, 48×32px | `#3A2A18` (warm near-black) | 90% | Yes — high value contrast |
| Tool placement zone | Dashed rounded-rect outline | Rounded rect, corner radius per biome | Barn Cream `#EDE0C8` | 60% dash / 0% fill | Yes — dashes read even under VFX overlay |
| Collection zone (cart) | Solid floor patch below cart | Rounded rect, 80×48px | Net Orange `#F47C20` at 20% opacity | 20% | Yes — orange is function-reserved, reads at low opacity |
| Safe / no-placement zone | None by default; crosshatch stamp on drag-hover only | N/A at rest; 3px Defeat Ash `#B4AAAA` crosshatch on hover | Defeat Ash | 0% at rest, 70% on hover | Advisory (appears on player action only) |

**Rule: Zone markers may not use new hues.** The four zone types above exhaust the permitted zone palette. Any new zone type introduced by future mechanics must select from this set or escalate for art director review. *Principle 2: color owns function.*

**Rush Phase Legibility Guarantee:** The dashed tool-placement marker uses dashes rather than a solid line precisely because a solid low-opacity line reads as "floor texture" under Rush Phase VFX. Dashes carry rhythm — the eye registers a pattern, not a background. No zone marker changes opacity, shape, or color during Rush Phase. The visual language is identical in both states. *Pillar 4: Prep+React; Pillar 2.*

### 6.6 Diorama Edge Philosophy

**The diorama edge is a visible but soft physical frame — not a literal shoebox wall.**

The edge reads as "the table surface the playset sits on, fading into the room." It is not a hard crop, not a painted border, not a picture frame. The transition is a gradient vignette: the floor color fades (over 32pt) to a darkened rim tone (Floor Tan × 60% brightness), which then fades to a near-black `#1A1210` at the absolute edge, 8pt wide. This produces a "receding table edge" effect at the bottom and sides of the screen.

The top edge is handled differently: it fades to the atmospheric sky color of the biome (Canopy Slate `#8FA89A` for Meadow/Orchard, darkened slate for Mines, cooled slate for Arctic) over 48pt. No hard top crop.

**Organic forms at the diorama edge (S3.2 rule executed here):** Organic silhouettes — irregular bush shapes, root clusters, rock outcroppings — appear only within the 32pt vignette band. They are never flat; they are dark silhouettes (the near-black edge tone) stamped over the vignette gradient. They carry no outline — they are part of the edge atmosphere, not gameplay objects. *Pillar 1: the organic edge softens the diorama into "something alive at the border," reinforcing the toy-farm-on-a-table fantasy without adding visual noise to the active playfield.*

**Aspect Ratio Behavior (9:16 to 9:21):**

The playfield content area is fixed at a 9:16 safe zone centered in the screen. On taller devices (9:21), the extra vertical height extends the vignette band at the bottom edge — the playfield does not reflow, the diorama simply "has more table below it." This prevents any gameplay zone from shifting position. The extended bottom vignette uses the same gradient formula, simply scaled. Top extension, if any, adds sky band. No layout system changes required. *Supports solo-dev scope: one layout, one safe zone, passive edge extension.*

### 6.7 Static Scene Elements (Non-Interactive Props)

**Asset count budget: maximum 6 unique prop types per biome.**

| Biome | Prop Set (6 maximum) |
|---|---|
| Meadow Farm | Fence post, fence rail, barn wall section, hay bale, flower patch, wooden signpost |
| Orchard | Tree trunk, tree canopy sphere, low stone wall section, wooden crate, fruit basket, fence post (reuse) |
| Mines | Timber brace pair, ore cart, wall torch, stone wall section, rubble pile, mine rail segment |
| Arctic Farm | Snow pile, icicle cluster, barn wall + snow cap (counts as 1 reskin), frozen fence post (reuse + snow cap), ice tile group, sled prop |
| Volcano | Broken arch remnant, lava tile border (animated, 2-frame), ash pile, cracked floor slab, ember emitter prop, scorched fence post (reuse) |

**Reuse Strategy — the "Reskin + Cap" Rule:**

The fence post is the base reuse unit. Every biome carries a version: plain wood (Meadow), round stone (Orchard), timber-roughened (Mines), snow-capped (Arctic), scorched (Volcano). The base geometry is identical — a 16×48px rounded rectangle cylinder — with a biome-specific surface color from the biome's background palette and one optional cap element (snow hemisphere, scorch mark stamp, etc.). This single reuse chain delivers 5 biome variants from 1 geometry. The cap element adds max 1 unique sprite per biome. *This strategy reduces unique prop assets from a naive 30 (6 × 5) to approximately 14-18 total, supporting the solo dev 9-month timeline.* *Pillar 3: Mastery Through Economy extends to asset production economy.*

**Prop Naming Convention:**

Following the established convention: `env_[object]_[biome]_[variant].[ext]`

Examples: `env_fencepost_meadow_01.png`, `env_fencepost_arctic_snow.png`, `env_barn_meadow_wall.png`, `env_torch_mines_idle.png`

**The 6px Floor Rule (S5.6) applied to props:**

No prop detail smaller than 6px at game camera. Fence rail crosshatch: 2px lines minimum, spaced 8px. Barn plank lines: 2px minimum. Flower patches: individual flower units minimum 8×8px each or grouped into an undetailed mass. Any prop detail readable only at 2× zoom is out of scope. *Pillar 2: if a detail does not read at game camera, it consumes art budget with zero readability return.*

---

## Section 7: UI/HUD Visual Direction

> **Integration note**: This section reconciles art-director visual direction with ux-designer interaction check. Where they diverged, resolved decisions are marked **[locked]**. UX-flagged accessibility items are folded into each subsection where relevant.

### 7.1 HUD Approach: Screen-Space Chrome with Diegetic Flavor

**Decision [locked]: Screen-space chrome, not diegetic.**

Money Miner's Rush Phase demands sub-second information parsing across up to 5 simultaneous gopher events. Diegetic elements — floating wood-sign quota boards embedded in the diorama — introduce two failure modes: (a) they compete with gameplay-function objects for the player's pre-attentive field, and (b) they violate the 56pt fixed top-zone rule the moment the diorama camera shifts. Pillar 2 (Readable at a Glance) overrides the S1 aesthetic preference here. **This is the primary aesthetic-vs-readability trade-off in this section — Pillar 2 wins.**

The diegetic flavor is recovered through visual surface, not placement: HUD chips use Barn Cream (`#EDE0C8`), the 3px black outer stroke + 2px warm-white inner stroke plastic-casing-edge treatment from S3.3, and the same pill shape as in-world tool trays. The HUD reads "made of the same stuff as the toys" without inhabiting the playfield — the label on the side of the toy box, not a prop inside it.

**Aesthetic cost acknowledged:** A floating wood-sign quota board would reinforce S1 more completely. Deferred to a post-launch cosmetic option (e.g., a "Diorama HUD" skin).

### 7.2 Typography Direction

**Font family: Baloo 2** (Nunito as fallback). Rounded geometric sans-serif, wide ink traps, generous x-height, available on Google Fonts with Latin, Cyrillic, Devanagari, and CJK coverage via Baloo family variants. Both iOS and Android support it without embedding full font binaries. Rounded terminal shapes echo the toy-diorama geometry vocabulary (S3.1).

**Weight strategy: 2 weights maximum.** Regular (400) for body and captions; Bold (700) for headlines, button labels, HUD numerals. No Medium, no SemiBold — each weight is a memory load on low-end Android (S6.2 mobile budget).

**Type scale (pt, portrait mobile):**

| Role | Size | Weight | Notes |
|---|---|---|---|
| Headline | 22pt | Bold 700 | Screen titles, level-end star count |
| Body | 15pt | Regular 400 | Shop descriptions, settings labels |
| Caption | 12pt | Regular 400 | AA minimum for disabled text at `#6B6055` — use sparingly |
| Button label | 16pt | Bold 700 | All pill buttons; all-caps prohibited |
| HUD numeral | 20pt | Bold 700 | **Tabular figures (tnum) required** — see rule below |
| Coin counter | 20pt | Bold 700 | Same spec as HUD numeral; Treasure Yellow `#F5C800` icon left |

**Numeral stability rule [locked]:** All HUD numerals — quota fill count, timer, coin counter — must use tabular-figure (tnum) OpenType feature or a monospace numeral set. Baloo 2 supports tnum. Prevents layout shift as digits change during Rush Phase. A counter that reflows its width when going from "9" to "10" is a Pillar 2 violation.

**Dyslexia support (UX ADVISORY adopted):** No italic for any functional label. Tracking on HUD numerals stays at 0 (default), never tighter. Quota meter and timer use numerals rather than icon-only representations so screen readers have text to parse.

### 7.3 Iconography Style

**Style: Flat-filled with 2px black outline, toy-sign register.**

Icons feel like painted signs on a toy playset — flat color fill, same 2px black outline as UI strokes, no internal gradients, no drop shadows on the icon form itself. Corner radius on icon bounding shapes mirrors UI shape grammar: abstract icons (settings gear, sound toggle, back arrow) use slightly more angular geometry than characters (corner radius 4-6pt on icon primitives vs. 10-12pt on character heads). Abstract icons are "signage"; characters are "toys."

**Stroke weight:** 2px at all icon sizes. Icon bounding box minimum 24×24pt; touch target supplemented by the surrounding 44×44pt tap zone (S3.3).

**Abstract concept rule:** Settings gear = 6-tooth gear, simplified, 2px stroke, no fill gradient. Sound toggle = single speaker cone + wave arc (on) or cone + X (off). Back arrow = left-pointing chevron, not circular. Deliberately conventional — wayfinding icons must parse in under 200ms.

**Tool icons in the tray [locked]:** Tool icons match the in-world tool shape. Ramp icon IS the blue wedge with directional arrow overlay at 32×32pt. Net icon IS the orange frame with net-weave. Direct application of Principle 2's semantic color vocabulary — the icon and the world object share the same color and silhouette so Prep Phase drag operations feel like "picking up the real object." **No abstraction for tool icons.**

**Currency icon rule [locked, UX BLOCKING adopted]:** Soft currency (coins) and premium currency (gems) must be visually distinguished by **shape AND color**, never color alone. Coin icon = flat yellow disc with raised center-bevel silhouette. Gem icon = faceted blue-purple diamond shape — deliberately distinct from Ramp Blue (uses a darker gem-blue `#5A50A8` to avoid conflict with the function palette). Real-money IAP purchases must display local currency (e.g., "$2.99") as the primary label on the confirm button, never gem count alone.

**CVD check for gem color [advisory]:** Gem blue `#5A50A8` sits in the same hue family as Cat purple-grey `#8B7FA8`. Under tritanopia simulation (S4.6 MEDIUM risk for the Ramps vs. Cats pair) these two could conflate at small sizes. Before finalizing the gem icon, test `#5A50A8` against `#8B7FA8` at 32×32pt icon size using Coblis v2 or equivalent. If they conflate, shift gem toward a cooler, bluer tone — fallback: `#3A60C0` (farther from the purple axis). Shape disambiguation (diamond facets vs. cat silhouette) remains a strong backup regardless.

### 7.4 Button and Interactive Element Visual Language

**Primary vs. secondary hierarchy:**

| State | Fill | Stroke | Label Color |
|---|---|---|---|
| Primary CTA (Play, Confirm, Buy) | Ramp Blue `#2D7FE8` | 3px black outer + 2px warm-white inner | `#FFFFFF` Bold 700 |
| Secondary (Cancel, Back, Info) | Barn Cream `#EDE0C8` | 3px black outer + 2px warm-white inner | `#1A1410` Bold 700 |
| Disabled | Floor Tan `#C8B89A` | 2px `#6B6055` outer only (no inner stroke) | `#6B6055` Regular 400 |

One Ramp Blue button per screen maximum (S4.5 lock). If a screen requires two affirmative actions, one is primary (blue) and one is secondary (cream).

**Press state — mobile "click feel":** Scale compress to 96% over 80ms ease-in, release to 102% then settle to 100% over 120ms ease-out. Simultaneously, shadow compresses: 4pt blur drops to 2pt blur at peak press. No color shift on press — scale + shadow carry the full tactile read. Total animation: 200ms. References physical depression of a plastic button on a toy. 80ms threshold stays within Pillar 5's 100ms response budget.

**Destructive actions (reset save, delete, irreversible spend) [locked]:** Fill uses UI Error `#A02018` (never in-world Dog Red `#D93B2B`). Label is `#FFFFFF`. Outer stroke increases to 4px. Small warning icon (triangle with "!" in Barn Cream) appears left of the label text. **A confirmation modal is always required** — the button itself never executes a destructive action directly.

**Real-money purchase buttons [locked, UX BLOCKING adopted]:** Real-money IAP confirm buttons display the local-currency price as the primary label in Bold 16pt. Below the price, an 11pt caption shows the item name. This is distinct from in-game coin/gem spends, which display the currency amount + icon. Real-money buttons always trigger a confirmation modal with both the item and the price restated.

### 7.5 Panel and Modal Visual Treatment

**Entry / exit motion:** Modals slide up from the bottom edge over 220ms ease-out cubic. Exit is slide down, 180ms ease-in cubic. "From below" direction is consistent with the toy-on-a-table metaphor — the panel is pushed up from the table surface. No scale-bounce entry (bounce is reserved for in-game slapstick per Principle 3). No dissolve (dissolve reads as digital, not toy). Biome map transitions are the exception: map fades in over 300ms (deliberate "pulling back from the table" read per S2 State 5).

**Backdrop:** Semi-transparent Headline near-black `#1A1410` at 60% opacity, no blur. **Blur is explicitly prohibited** — GPU cost on low-end Android is unacceptable (S6.2 budget constraint), and blur reads as iOS glass-morphism which conflicts with the toy-diorama surface aesthetic. Flat dimming at 60% provides adequate contrast against Barn Cream modal surfaces. The gameplay world remains subtly visible behind the dim, reinforcing that the modal is "above the table," not replacing it.

**Modal surface:** Barn Cream `#EDE0C8` fill. Corner radius 16pt (S3.3 locked). 3px black outer stroke + 2px warm-white inner stroke (plastic-casing-edge). Drop shadow: 4pt blur, 60% opacity black (S3.3 locked). Max modal width at 375pt viewport = 343pt (16pt margin each side). Internal padding: 20pt all sides. Vertical stack layout only.

**Outdoor viewing note (UX CONCERN adopted):** The 3px black outer stroke carries all functional legibility under bright ambient. The 2px warm-white inner stroke is decorative enhancement — it will gracefully disappear in direct sunlight, but the black outline ensures panels remain legible. No panel variant may reduce outer outline weight below 2px under any circumstance.

### 7.6 In-Level HUD Layout Anatomy

**Top bar (fixed, 0-56pt from top) [revised per UX CONCERN]:**

- **Left:** Coin counter chip (pill, 32pt height). Treasure Yellow coin icon left, tabular numeral right.
- **Center:** Quota meter (pill, 32pt height, full-width minus margins). Fill color Treasure Yellow `#F5C800` from left; unfilled portion is Floor Tan `#C8B89A`. Quota count numeral (current / target) centered over bar in Headline near-black Bold 20pt. **This is the Pillar 3 and Pillar 4 critical element** — must read at a glance mid-chaos.
- **Right:** Timer chip (pill, 32pt height) + Pause icon chip (44×44pt, icon-only). Timer numeral in tabular Bold 20pt.

**Settings migrates to the Pause menu** — not in the top bar. Freeing 44pt of horizontal space for quota + timer density at 375pt portrait width (UX CONCERN adopted).

**Timer urgency signal [locked]:** At 10 seconds remaining, timer numeral tints to Dusk Umber `#A08060` — a warning tone drawn from the background palette. Never red (reserved for in-world hazard). Never yellow (reserved for treasure/quota — would confuse semantic vocabulary). Dusk Umber carries "approaching limit" signal without competing with active gameplay palette.

**Reduce-motion accessibility affordance [locked, UX BLOCKING adopted]:** When the "Reduce Motion" accessibility toggle is enabled:
- Bomb pulse cycle is suppressed. Bombs show a **static high-contrast ring** — solid `#D93B2B` body with a fixed 4px white outer ring at 90% opacity (no cycle, no flash). The fuse progression is instead communicated via a radial fill gauge on the bomb body: the red darkens progressively from 100% → 40% lightness over the fuse duration, with the final 3 seconds locking at 100% saturation. No 2.5Hz flash anywhere in the game.
- Timer urgency Dusk Umber transition at T-10s becomes instantaneous (no tween).
- UI enter/exit animations retain current durations (already under 300ms, below photosensitivity thresholds).

This is required for Google Play + App Store compliance on both accessibility and photosensitivity grounds.

**Prep Phase pause accommodation (UX CONCERN adopted):** A small Pause-Prep icon (32×32pt) sits at the far-left of the Prep tray during Prep Phase only. Tapping it freezes the Prep countdown timer. Enables motor-impaired players to take unlimited prep time. Hidden during Rush Phase. Visual: same icon language as main Pause button but with a small "P" badge overlay to disambiguate.

**Bottom tray — Prep Phase [revised per UX CONCERN]:**

Full-width pill-bottomed dock at 80pt height. Barn Cream `#EDE0C8` fill, 3px black outer stroke along top edge only. Tool icons arranged as 2-column card grid (S3.3 card spec) with stock count badge (Barn Cream chip, 16pt Bold, top-right of icon). **Critical tool slots center-aligned, not right-biased** (UX finger-occlusion check). Active selection: selected tool card receives Ramp Blue `#2D7FE8` outer stroke (2pt, replacing the black) as a selection ring. This is the single permitted secondary use of Ramp Blue in UI, justified by direct semantic link to tool deployment.

**Bottom tray — Rush Phase [locked, UX BLOCKING adopted]:**

Visually distinct from Prep tray — multiple signals, not one:
- Height: 56pt (narrower than 80pt Prep)
- Surface fill: Dusk Umber `#A08060` at 80% (instead of Barn Cream)
- **Mode badge**: a small "RUSH" text label in Bold 12pt Headline near-black appears center-top of the dock when the tray switches mode. Persists for the full Rush duration.
- **Consumable touch targets: 56×56pt minimum** (not 44pt). Visual chip can render at 40×40pt with invisible 56pt tap extension. Reduces missed activations under Rush chaos + finger occlusion.
- Mode-switch animation: 0.2s fade at the same moment as the S2.2 lighting shift (0.3s). Cut-style transition in spirit, with a brief dissolve so the mode change is unmistakable.
- Stock badge on each consumable, 16pt Bold. When stock reaches 0, chip desaturates to 40% and adds crossed-out overlay in Defeat Ash.

The darker surface + mode badge + size change combined ensure the player cannot muscle-memory drag when they should tap.

**Quota meter design note:** The meter is center-top, the largest single HUD element. It must be the first thing the player checks between chaos events. Fill direction (left-to-right, Treasure Yellow advancing) references the physical motion of coins traveling to the collection cart — a spatial metaphor grounded in the game's own vocabulary.

### 7.7 Animation Philosophy for UI

**Motion aesthetic: Snappy.** Justified by Pillar 5 (No Input Goes Silent) and Pillar 2. Bouncy motion (spring physics, overshoot) is reserved exclusively for in-world slapstick (S5.3 Dog Deploy, character expressions). Smooth (ease-in-out, no snap) reads as premium but slow — incompatible with reactive Rush pace. Deliberate (slow, weighty) is a tutorial/cutscene register, not core-loop. Snappy means: fast ease-out entries, fast ease-in exits, no linger.

**Easing rules:**

- **Enter** (panel, modal, tray): ease-out cubic, 180-220ms
- **Exit**: ease-in cubic, 150-180ms
- **State transform** (button press, quota fill): ease-in-out cubic, 80-200ms
- **Counter increment** (coin counter, quota numeral): linear, 60ms per digit tick

**Maximum UI animation duration: 300ms.** Any animation longer than 300ms is a cognitive interruption during Rush Phase. The level-end star reveal sequence is the sole exception — deliberate pause state, player has exited the board.

**Rush Phase coexistence rule:** During Rush Phase, VFX particles operate on the playfield layer. No more than 2 UI elements may animate simultaneously. Quota meter fill (as coins are collected) is the only continuous UI animation during Rush. Button presses and tray updates are instantaneous feedback events (within 100ms), not sustained animations. Visual hierarchy during Rush Phase: playfield VFX (loudest) > quota fill (sustained, ambient) > HUD numeral increments (fast, local) > nothing else moves. Keeps Pillar 2 intact at peak chaos.

**Reduce Motion compliance:** When the Reduce Motion toggle is enabled, all UI enter/exit animations stay under 180ms and use linear easing (no cubic curves). Press state scale compression remains (tactile, not motion) — but shadow compression is disabled. Counter increments become instantaneous.

### 7.8 Shop Screen Navigation (UX CONCERN adopted)

**Filter chip row (44pt) below shop header.** Category chips: "Tools / Consumables / Cosmetics / Offers." Active chip = Ramp Blue outer stroke treatment (mirror of tool tray selection language). Inactive chips = Barn Cream with 3px black outline. Filter is a single-select — one category visible at a time.

At 100+ items in Full Vision scope, a flat scroll list creates discovery failure. Filter chips provide the minimum discoverability infrastructure without a search bar (search is out of MVP scope). Shop items within a category use the S3.3 2-column card grid at 10pt corner radius, 16pt padding, 8pt gutter.

**Real-money boundary at the shop level:** Items requiring real-money IAP are grouped under the "Offers" filter chip. Card border for IAP items uses a 2pt-wider outer stroke (5px total) to visually signal "this is a different transaction." Price label is local currency, not gem count.

---

## Section 8: Asset Standards

> **Integration note**: This section merges art-director workflow preferences with technical-artist engine/performance constraints. Where they diverged, resolved decisions are marked **[locked]**. Engine target: Unity 6.3 LTS, URP 2D Renderer, Mali-G52-class Android as floor device.

### 8.1 File Formats

**Source (authoring) formats:**

| Category | Source Format | Rationale |
|---|---|---|
| Characters, tools, icons, UI elements | Affinity Designer `.afdesign` | Vector authoring preserves 3px outlines at any export resolution. No subscription cost. |
| Environment props, backgrounds | Affinity Designer `.afdesign` | Single tool for all vector + gradient work |
| Concept / mood boards | Figma (read-only archive) | Reference only — not in the engine pipeline |

**Delivery (game-ready) formats:**

| Category | Delivery Format | Notes |
|---|---|---|
| All sprites | PNG-24 with alpha | Lossless; transparent background mandatory; no JPEG anywhere |
| UI backgrounds (full-screen) | PNG-24 | Gradients must bake cleanly |
| VFX frames | PNG-24 with alpha, individual frames | Sequence-named; compositor/TA assembles atlas |
| Font | Baloo 2 .ttf source + Unity-imported TextMeshPro asset | 2-weight subset (Regular 400 / Bold 700) only |

Every asset has exactly two canonical file locations: the `.afdesign` master in `assets/source/` and the exported PNG in `assets/game-ready/`. No intermediate PSDs. If a game-ready file cannot be regenerated from its `.afdesign` master in a single export step, the master is incomplete.

### 8.2 Naming Convention [locked]

Base pattern: `[category]_[name]_[variant]_[state].[ext]`

| Token | Values | Examples |
|---|---|---|
| `category` | `char`, `tool`, `env`, `ui`, `vfx`, `icon`, `bg`, `audio` | — |
| `name` | lowercase, underscore-separated noun | `gopher`, `ramp`, `fence_post`, `btn_primary` |
| `variant` | biome slug OR skin ID OR `default` | `meadow`, `mines`, `skin01`, `default` |
| `state` | pose / frame / interaction state | `idle_01`, `active`, `used`, `hover`, `face_a` |

**Full examples:**
```
char_gopher_meadow_idle_01.png
char_gopher_meadow_face_happy.png
char_cat_mines_pouncing_01.png
tool_ramp_default_active.png
tool_ramp_mines_idle.png
env_fence_post_arctic_default.png
ui_btn_primary_default.png
icon_settings_default.png
vfx_coin_burst_01.png
bg_meadow_prep.png
bg_meadow_rush.png
```

**Version control:** No `_v2` suffixes in filenames. Git is the version history.

**State suffixes locked list:** `idle`, `active`, `used`, `hover`, `pressed`, `disabled`, `face_[expression]`, `walk`, `loop`, `burst`, `deploy`, `celebrate`. New states must be added to this list before use.

### 8.3 Texture Resolution Tiers [locked]

The game camera renders characters at 60-80px height (S3). All resolution targets derive from that anchor.

| Context | Character Height | Delivery Resolution |
|---|---|---|
| Gameplay (game camera) | 60-80px | 128×128px sprite sheet cell |
| **Shop / character select** | **~120px** | **Same 128×128 sprite scaled up via Canvas — no separate Shop texture** |
| Level-end star / reward | ~80-100px | 128×128px (gameplay tier) |
| Menu background (diorama) | Full screen | 1080×1920px per biome × mood state |
| Icons (HUD, tool bar) | 48-64px rendered | 64×64px |
| Map node thumbnails | ~40px | 64×64px (downsampled from 128 source) |

**Shop preview rule [locked]:** Shop characters use the gameplay 128×128 sprite scaled to 120px render via Canvas. No separate @2x Shop texture set. This saves ~1MB per character across all cosmetic variants. If softening at 120px is unacceptable during playtest, the gameplay source resolution is upsized to 160×160 and used in both contexts — never a dedicated Shop texture.

**Retina / HiDPI strategy:** Deliver at a single @1.5-equivalent resolution (128×128 source is oversized for the 60-80px render target, giving Mali-G52's bilinear filter enough source data to produce clean downsamples). Unity's sprite renderer scales down cleanly; scaling up from @1x introduces blur on 3px outlines. @3x is not warranted.

### 8.4 Sprite Atlas Strategy [locked]

**Atlas count: 2 max loaded simultaneously.** Matches S6.2.

| Atlas | Contents | Load scope |
|---|---|---|
| **Atlas A — Biome** | All environment tiles, non-interactive props, background stamps for the active biome (S6.7: 6 prop types max) | Loaded when biome is active; unloaded on biome switch |
| **Atlas B — Shared** | All gameplay characters (all variants), all HUD/UI sprites, all VFX sprite sheets | Always loaded |

**Atlas resolution: 2048×2048 hard cap.** Mali-G52 supports 4096 but runtime memory at 4096 (~32MB uncompressed) exceeds per-texture budget.

**At Full Vision (5 biomes), if Atlas B exceeds 2048×2048:** split into Atlas B-chars and Atlas B-UI. Atlas B must NEVER split by biome — characters and HUD are always on-screen regardless of biome, so biome-split of characters/UI would force multi-atlas loads mid-scene.

**Within an atlas:** sprites arranged alphabetically by filename. Unity's sprite packer or TA-chosen tool handles packing. Art team does not rely on atlas coordinate positions.

### 8.5 Texture Memory & Compression

**Per-platform compression [locked]:**

| Platform | Characters / UI (quality-sensitive) | Environment tiles / stamps |
|---|---|---|
| Android | ETC2 RGBA (hardware support on all Android 4.3+) | ETC2 RGBA (safe floor for Mali-G52) |
| iOS | ASTC 4×4 | ASTC 6×6 |

**Source imports: always PNG or PSD (lossless).** Never import a pre-compressed texture — Unity must recompress per platform from lossless source.

**Mipmaps disabled for all 2D sprites.** Reasons: (a) each atlas would grow ~33%, wasting memory; (b) mip lower-levels feather the 3px black outlines unacceptably. No exceptions.

**Memory budgets:**

| Scope | Ceiling |
|---|---|
| Single loaded level (total GPU texture) | ≤80MB target, 150MB hard cap |
| Atlas A (biome) | ~11MB ETC2 / ~7MB ASTC 6×6 compressed |
| Atlas B (shared) | ~3MB characters + ~3MB UI + ~1MB VFX compressed |
| Full-screen menu background (single biome × single mood state) | ~2-3MB |

**Compression validation:** Confirm 3px black outlines survive ETC2 compression at 2048×2048 during prototype. If banding appears on outlines, the character region of Atlas B escalates to ASTC 4×4 on Android (accepts ~2× size cost for quality).

### 8.6 Draw Call Budgets

**Total: ≤200 on low-end Android; ≤120 target average.**

| Layer | Allocation |
|---|---|
| Playfield (gameplay sprites, characters) | ≤40 |
| Background (biome tiles, props) | ≤20 |
| HUD / UI | ≤20 |
| VFX / particles | ≤15 |
| Miscellaneous / overdraw slack | ≤25 |
| **Target total** | **≤120** |

**SRP Batcher eligibility [locked]:** All gameplay sprites share one URP 2D Sprite-Unlit material variant. Do NOT create per-character materials. Forbidden on hot-path sprites:
- Per-instance `MaterialPropertyBlock` with non-SRP-Batcher-compatible properties
- Mixed atlas sources within the same camera sort layer without grouping
- Custom shaders that do not declare `CBUFFER_START(UnityPerMaterial)` / `CBUFFER_END`

**Overdraw ceiling:** In URP 2D, all sprites are alpha-blended transparent — overdraw is the primary Mali-G52 bottleneck. Maximum layer stack: **4 overlapping sprite layers at any screen position** (background + midground + character + VFX). No additional overlay sprites behind HUD.

### 8.7 Shaders and Materials

**Base shaders by category [locked]:**

| Category | Shader |
|---|---|
| Characters and gameplay sprites | `Universal Render Pipeline/2D/Sprite-Unlit-Default` |
| UI sprites | `Universal Render Pipeline/2D/Sprite-Unlit-Default` (or UI/Default for Canvas) |
| VFX particles | `Universal Render Pipeline/Particles/Unlit` |

**Shader Graph policy:**
- **Permitted**: VFX-only materials (coin sparkle, bomb pulse overlay, rush-phase glow stamp)
- **Forbidden**: core sprite materials (characters, environment, UI) — compile time and variant count overhead not justified for flat-fill
- **Never** use Shader Graph nodes that require HDR output (HDR is disabled)

**Shader variant budget: hard cap 32 variants.** No `#pragma multi_compile` keywords on sprite shaders without explicit audit.

**Material slot count: 1 material per SpriteRenderer.** URP 2D does not benefit from multi-pass materials; a second material doubles the draw call.

**Forbidden on Mali-G52:** geometry shaders, tessellation, compute shaders in the render path, screen-space reflections, any Volume post-process except color grading and FXAA. **No blur. No bloom. No glass-morphism.** Additive blending on particles permitted but counts toward overdraw.

### 8.8 Animation Constraints

**2-frame alternation (per S5.5) implementation:** Unity `Animator` with one `SpriteRenderer`. One Animation Clip per state (Idle / Active / Deploy = 3 max per character). Each clip = 2 keyframes swapping the sprite reference. Animator handles timing deterministically.

**Do NOT:** use runtime sprite array + `Update()` manual cycling; use `AnimationClip.SetCurve` with Sprite references at runtime; use Unity's `Animation` (legacy) system for positional tweens.

**Animator count per scene: ≤1 per character instance.** At peak density (40 active Rigidbody2D ceiling from technical-preferences), ≤40 active Animators. Animators with no active transitions: `animator.enabled = false` when off-screen or pooled.

**Positional tween library: DOTween** (free version or Pro). Established Unity community standard, zero-allocation pooled path, integrates cleanly with URP 2D. LeanTween is an acceptable alternative. Unity's built-in `Animation` is rejected for tweens (heavier than a tween library for simple transforms).

### 8.9 VFX and Particle Budgets

**Particle system: Unity `ParticleSystem` (not VFX Graph).** VFX Graph requires Compute Shader support; not guaranteed on Mali-G52-class Android devices. `ParticleSystem` with Burst jobs is the safe floor.

> Verify Mali-G52 compute support before any VFX Graph usage — `docs/engine-reference/unity/modules/rendering.md` confirms Unity 6.3 supports VFX Graph, but target hardware may not.

**Per-effect particle ceilings:**

| Effect | Max concurrent particles |
|---|---|
| Coin / gem pickup sparkle | 15 |
| Bomb flash / impact dust | 20 |
| Rush Phase screen effect | 30 |
| Idle ambient (background dust motes, snow, etc.) | 10 |

**Max concurrent effects on screen at Rush Phase peak: 4 simultaneous effects, ≤60 total particles.** Beyond 60 particles, Mali-G52 fill-rate from additive overdraw pushes the frame over 33.33ms.

**Forbidden particle features on mobile:** subemitters (doubles CPU cost), trails (mesh per particle, extreme fill-rate cost), texture sheet animation with >4 frames, soft particles (requires depth texture sampling).

### 8.10 Import Settings Presets [locked]

All assets must use one of 4 Unity Preset assets stored at `assets/import-presets/`. Committing an asset without a matching preset is a CI failure.

| Preset | Filter | Android Compression | iOS Compression | Max size | Mipmaps | sRGB |
|---|---|---|---|---|---|---|
| `preset-character.preset` | **Point** | ETC2 RGBA | ASTC 4×4 | 1024 | Off | On |
| `preset-environment.preset` | **Point** | ETC2 RGBA | ASTC 6×6 | 2048 | Off | On |
| `preset-ui.preset` | Bilinear | ETC2 RGBA | ASTC 4×4 | 1024 | Off | On |
| `preset-vfx.preset` | Bilinear | ETC2 RGBA | ASTC 6×6 | 512 | Off | On |

**Filter mode rationale:** Point filtering on characters + environment preserves hard 3px outlines (S1). Bilinear on UI + VFX accepts slight softening for cross-DPI scaling.

**Color space:** Linear project color space (Unity 6.3 default). All sprite textures marked sRGB=On.

**Anisotropic filtering:** disabled for all 2D sprite presets.

### 8.11 Addressables Strategy [locked — DO NOT adopt for v1.0]

**Recommendation:** Do NOT adopt Addressables for v1.0. Project scope (solo dev, 2 biomes, all local assets, no DLC, no remote content) does not justify the tooling overhead.

**Use instead:** Direct scene references for all gameplay assets (serialize the Atlas and character Prefabs into scenes/ScriptableObjects directly). **`Resources.Load` is legacy and forbidden** (from `docs/engine-reference/unity/current-best-practices.md`) — it prevents memory control and loads synchronously.

**Adoption trigger for Addressables:** If at Full Vision (5 biomes) the unloaded biome atlases cause memory pressure on 3GB-ceiling devices, adopt with a group structure of: `group-biome-[name]` (1 per biome, local), `group-ui` (always loaded), `group-characters-shared` (always loaded). Do not design for this now.

### 8.12 Authoring Templates and Canonical Assets

**Template files (solo-dev velocity rule):**
- `assets/source/_templates/_template_char.afdesign`
- `assets/source/_templates/_template_tool.afdesign`
- `assets/source/_templates/_template_env_prop.afdesign`
- `assets/source/_templates/_template_ui.afdesign`

Each template contains: artboard at delivery resolution, outline layer pre-set to 3px, color swatch panel with all 7 function colors + background palette.

**Canonical color swatches [locked]:**
- `assets/source/_templates/function-colors.afpalette` — single source of truth for the 7 function color hex codes.
- Every template file imports this palette.
- If a hex value must change, update `function-colors.afpalette` first; never hardcode a hex into an individual asset.

**Cosmetic / shop variants rule:** All shop skin variants must be authored from a copy of the base character template, NOT from a previous skin variant. Prevents incremental drift.

**Reference grid:** Template artboards include a non-printing 6px grid (S5.6 6px floor) and a 24×24px thumbnail-test frame (S3 thumbnail rule) centered on the artboard. Both visible at delivery.

### 8.13 Quality Control & Pipeline Automation

**Pre-delivery checklist (manual, solo dev):**

| Check | Method | Rule Source |
|---|---|---|
| 24px thumbnail test | Scale preview to 24px wide; all key shapes read distinctly | S3.1 |
| 6px detail floor | At 1:1 delivery resolution, no detail element smaller than 6px | S5.6 |
| Function-color audit | Eyedropper spot-check all fills; no reserved hue on a non-function object | S4.1 |
| Outline weight | At delivery resolution: 3px gameplay, 2px icons | S1 / S7.3 |
| Transparent background | Export preview shows no canvas color | 8.1 |
| Naming convention | Filename matches pattern in 8.2 exactly | 8.2 |
| Derives from template | Source file base layer matches `_template_[category]` artboard | 8.12 |

**Automated pre-build validation (Unity Editor CI script):**

A headless Unity Editor script (`-executeMethod ArtValidator.RunAll -batchmode -quit`) runs on every PR. **Blocking failures:**

1. Sprite dimensions not power-of-2 (required for ETC2/ASTC correctness)
2. Sprite missing a recognized Preset (8.10)
3. Atlas exceeds 2048×2048
4. Any texture has mipmaps enabled
5. Any texture uses `Resources/` folder path
6. Filter mode does not match preset category

**Advisory warnings (non-blocking at v1.0):**
- Naming convention violations (pattern in 8.2)
- Manual pre-delivery checklist items — solo dev honors on trust

The Unity Editor CI script itself is a pre-production action item (to be created during Technical Setup phase; tracked as an epic story).

---

## Section 9: Reference Direction

### Reference 1: Crossy Road (Hipster Whale, 2014)

**Identity**: A mobile arcade game built on voxel character cubes, celebrated for its instantly-readable toy-register silhouettes and non-punitive failure state.

**What to take**: The "expressiveness floor" — the precise amount of personality that reads as charming without requiring per-frame animation. Crossy Road's characters convey states through pose-and-scale alone: a single held lean, a sudden squash. They do not wink. They do not address the camera. This is the 4/10 expressiveness calibration locked in Section 5.5. Reference it during character review: if a proposed expression feels more emotive than a Crossy Road character, it has overshot.

**What to diverge from**: The voxel geometry and isometric camera. Money Miner is flat vector with a direct overhead-tilt camera. Importing voxel-style faceted shading or fake-3D depth planes would break the S1 toy-on-a-table law (flat fills on gameplay objects, no surface noise). Overindexing on Crossy Road produces blocky pseudo-3D — wrong material language for this project.

**Serves**: S5 (expressiveness calibration), S1 Principle 3 (Slapstick Scale without performance), S3 (silhouette-first character design).

### Reference 2: Mary Blair's Concept Art for "It's a Small World" (1964)

**Identity**: American mid-century illustrator whose theme-park work established a vocabulary of bold flat shapes, high-key color fields, and geometric folk-art motifs against warm neutral grounds — the ancestor of every mobile-casual "toy world" aesthetic.

**What to take**: The relationship between saturated figure and desaturated ground. Blair never let her background neutrals compete with the figures — warm creams and dusty tans push the vivid character shapes forward without a single shadow. This is the direct theoretical ancestor of Section 4's background palette rule (≤40% saturation, 30° hue-gap from any function color). When an environment artist asks why the floor must be so muted, Blair is the answer. Her work also demonstrates that only 4-6 hues are needed per scene — she proves the constraint works at large scale.

**What to diverge from**: Blair's compositions are static, symmetrical, and frontally staged for a horizontal aspect ratio (theme park viewing). Money Miner is portrait, asymmetric (off-center gopher holes, tool placement variety), and built for rapid eye-scan under chaos. Importing Blair's centered-and-symmetrical composition logic produces layouts that feel decorative rather than interactive. Her color discipline — yes. Her stage geometry — no.

**Serves**: S4 (background palette theory), S6 (biome color temperature), S2 (mood palette constraints).

### Reference 3: Plants vs. Zombies (PopCap Games, 2009)

**Identity**: A lane-defense mobile game whose art established one of the strongest functional-color vocabularies in the casual genre — every object category has a distinct silhouette register, and the art never lets the player mistake a tool for a threat.

**What to take**: The hard separation between "player infrastructure" silhouettes and "enemy" silhouettes, maintained without any UI labeling. In PvZ, no sunflower is ever confused with a zombie because the silhouette grammars are categorically different — upright organic vs. shambling hunched. Translate this directly: S3.4's hero/supporting shape grammar (circles = act now; rectangles = infrastructure) is the Money Miner equivalent. During review of any new object, run the PvZ silhouette test: could this be misread as belonging to the wrong category at a glance?

**What to diverge from**: PvZ's character expressiveness is at 8/10 — constant idle loops, mouth flaps, eyes that track. It also uses heavy outline-shading hybrids that give characters a semi-painterly surface. Both are out of scope: S5.5 locks expressiveness at 4/10 (solo-dev 2-frame animation budget), and S1 Principle 1 prohibits painterly surface detail on gameplay objects. PvZ is the right shape grammar reference, not the right execution budget reference.

**Serves**: S3.4 (hero vs. supporting shape hierarchy), S5 (character silhouette discipline), S4 (color-owns-function principle).

### Reference 4: Saul Bass Film Titles (1950s-1960s)

**Identity**: American graphic designer whose title sequences reduced complex narrative emotion to flat geometric shapes and bold two-color fields — the canonical proof that graphic minimalism and emotional impact are compatible.

**What to take**: The UI motion philosophy. Bass's sequences use sharp cuts and fast position slides — not dissolves, not fades, not easing curves. The shapes arrive and leave with conviction. This maps directly to S7.7's snap-style motion rule: ease-out entry, ease-in exit, 180-300ms maximum, no linger. When a UI animator proposes a gentle float-in for a modal, Bass is the counter-example: the panel should arrive with purpose, not drift. His title sequences are also a reminder that type and shape at large scale, high contrast, and plain fill communicate more authority than any textured or gradated alternative.

**What to diverge from**: Bass worked in a 1:1 aspect ratio film frame with no interactivity constraint and unlimited hold time. His compositions are designed to be read passively over 3-5 seconds. Importing his compositional pacing — slow reveals, sequential element entrances — into a mobile UI produces animations that outlast the player's patience during Rush Phase. Use his conviction of motion, not his tempo.

**Serves**: S7.7 (UI motion aesthetic, snap principle), S7.2 (type authority at 2 weights), S1 (flat geometry communicates fully without surface noise).

### Reference 5: Alto's Odyssey (Snowman, 2018) — Production Constraint Reference

**Identity**: An endless snowboarder mobile game built by a tiny team, praised for its visual coherence, atmospheric layered parallax, and the discipline to achieve a premium aesthetic on a minimal asset count.

**What to take**: The "few assets, high coherence" production method. Alto's team produced a game that reads as visually rich by establishing one strong atmosphere per biome and holding it relentlessly — no decorative exceptions, no one-off assets. Every asset earns its polygon budget. This directly models S6.7's 6-prop-max-per-biome rule and the fence-post reuse chain in S6.7. During production reviews, ask: "would an Alto's team greenlight this prop, or cut it?" If an asset is a decorative exception that serves one screen and cannot be reskinned, it should not exist.

**What to diverge from**: Alto's atmosphere is built on soft gradients, blur passes, and parallax depth — techniques that are explicitly forbidden in Money Miner (S8.7: no blur, no bloom; S6.2: flat-fill gameplay layer; S8.6: overdraw ceiling). Alto's palette is also low-saturation throughout, whereas Money Miner runs seven high-saturation function colors against a muted ground. The production discipline transfers; the technique and palette do not.

**Serves**: S6.7 (asset count budget, reuse strategy), S8 (solo-dev pipeline discipline), S8.11 (scope constraint philosophy).

### Reference Synthesis

These five references form a non-overlapping system. Blair provides the **color field theory** — why the background palette must step back hard. Crossy Road provides the **expressiveness floor** — what "charming but not performing" looks like in practice. Plants vs. Zombies provides the **shape-grammar proof** — that silhouette categories can carry gameplay-critical meaning at a glance. Saul Bass provides the **motion conviction** — that UI can communicate authority through geometric snap rather than organic ease. Alto's Odyssey provides the **production model** — that a small team can achieve visual richness by holding constraints ruthlessly rather than accumulating exceptions.

No two references overlap in their zone. Blair does not speak to animation. Bass does not speak to color discipline. PvZ does not speak to motion. Alto's does not speak to shape language. And the one reference that addresses production directly (Alto's) is the only one from the same medium (mobile game) with a comparable team size. That is the point: when an art team or outsource vendor asks "what does this game look like," they are asking five different questions simultaneously. These five references answer them, one each, without contradiction.

**The north star formed by layering all of them:**

> *A toy playset photographed by a mid-century illustrator, animated with cartoon economy, navigable as fast as a film title, made by two people in a garage.*

Every visual decision that serves all five references simultaneously is almost certainly correct.

### Reference Non-Examples

**Supercell's Clash of Clans / Brawl Stars family.** Tempting because these are mobile games with chunky cartoon characters, flat fills, and strong outline weight — the same surface descriptors as Money Miner. The failure mode is the Supercell *rendering* register: strong internal shading on character surfaces (a warm-to-cool lighting gradient baked into each sprite), ambient occlusion shadows on the ground plane, and painterly highlight strokes that imply 3D volume. Importing any of these breaks S1 Principle 1 (flat fills on gameplay objects) and would double the per-character art budget — the internal shading gradient alone adds a texture pass that the solo-dev pipeline cannot absorb. The shapes are superficially similar; the rendering language is categorically incompatible.

**Stardew Valley (ConcernedApe, 2016).** Tempting for two reasons: it is the definitive solo-dev success story with a farm setting, and its pixel art demonstrates what one person can produce with discipline and time. The failure mode is the pixel medium itself. Stardew's character detail (facial expressions, clothing, tool variety) is economical in pixel art but translates to hundreds of 1-pixel-detail decisions that do not survive ETC2 compression at 128×128 or Point-filter downscaling (S8.10, S8.5). More critically, Stardew's visual register is nostalgic-retro; Money Miner's register is toy-plastic-physical. Evoking Stardew would push the game toward RPG cosiness and away from casual-slapstick speed, conflicting with Pillar 1 (Cute Chaos) and Pillar 5 (No Input Goes Silent). It is the right solo-dev spirit, wrong visual medium and wrong emotional register.

**Pixar's feature-film character design (Toy Story, Finding Nemo, etc.).** Tempting because the game's premise is literally "toys come alive" and Pixar perfected the toy-character aesthetic. The failure mode is volumetric rendering expectation: Pixar characters are designed to read under 3D subsurface scattering, specular highlights, and rigged skeletal animation. Any attempt to reference Pixar character proportions or surface treatment sets an impossible benchmark for flat-vector 128×128 sprites. It also shifts the expressiveness register from 4/10 (Section 5.5, locked) to 9/10 — Pixar characters perform, Money Miner characters signal. Referencing Pixar implicitly invites scope creep toward idle animations, lip-sync, and per-character personality that belongs in a $30M production, not a 9-month solo build.
