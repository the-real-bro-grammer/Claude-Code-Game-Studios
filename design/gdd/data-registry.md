# Data Registry

> **Status**: Approved (pending independent `/design-review` in a fresh session)
> **Author**: Robert Michael Watson + game-designer / systems-designer / unity-specialist / creative-director / qa-lead
> **Last Updated**: 2026-04-20
> **Implements Pillar**: Foundation for all five pillars (no direct player-facing fantasy — infrastructure that enables every other system to deliver its pillar)
>
> **Creative Director Review (CD-GDD-ALIGN)**: CONCERNS → RESOLVED 2026-04-20 (3 fixes applied: E_norm objectivity locked in R8.a and D.2; Balance Audit window UX acceptance criteria added as AC-R8c / AC-R8d; OQ-1 hard rule captured in R8.b)

## Summary

The Data Registry is the centralized authoring layer for all designer-editable configuration in Money Miner — tool stats, hazard timings, loot tables, level quotas, biome rules, character definitions, shop products, and currencies all live here as Unity ScriptableObject assets. It is read by every gameplay, economy, and progression system but writes from none, establishing a single source of truth per data type. No runtime mutation is permitted; all values are authored in-editor and shipped as immutable assets.

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: `None`

## Overview

Money Miner requires rapid iteration on dozens of tunable values — loot-table weights, bomb fuse durations, gopher rhythm cycles, quota curves, shop prices, star thresholds — all of which must be designer-editable without engineer involvement. The Data Registry provides a Unity ScriptableObject-based pipeline that makes this possible. Each data category (tools, hazards, loot tables, levels, biomes, skins, shop products, currencies) has a typed ScriptableObject schema; authored assets live under `Assets/Data/[category]/` and are referenced by consuming systems via serialized fields or typed catalog lookups.

The registry enforces four rules: (1) **no runtime mutation** of ScriptableObject fields — all values are immutable after load; (2) **no `Resources.Load`** — all access is via direct serialized reference or a typed catalog wrapper; (3) **every data category has an editor-time validation rule set** that catches authoring errors (null references, out-of-range values, duplicate IDs) before they ship; (4) **every ScriptableObject carries a stable GUID-based ID** for cross-referencing, surviving asset renames. Together these rules protect the bottleneck role — 13+ downstream systems read Data Registry assets, so a corruption here corrupts everything.

This is the enabling layer for every pillar. Data-driven balance (Pillar 3) requires designer-accessible values. Level pacing (Pillar 4) depends on hand-authored level data read from the registry. Feedback event schemas (Pillar 5 via Event Bus) originate here. No runtime system may hardcode configuration: if a designer cannot change it in the Unity editor, it is not a value — it is a constant that belongs in code.

## Player Fantasy

Data Registry has no player fantasy of its own — and that's the point. The player never opens a registry, never reads a config, never even knows the system exists. They tap gopher holes, route treasure across a toy-farm diorama, and feel clever. Everything the Data Registry does happens three layers beneath that.

What it protects is the illusion of a tight, hand-tuned little heist. **Cute Chaos** depends on every tool, gopher, and upgrade feeling deliberately placed by a designer who cares — inconsistent stats or mismatched data would make the chaos feel sloppy instead of slapstick. **Mastery Through Economy** lives or dies on numbers the player can trust: if a shop upgrade's cost or effect silently drifts between levels, the whole "clever brain of the operation" fantasy collapses into "the game is cheating me." **Prep + React** needs the prep side to be legible and stable — the same tool must cost the same, do the same, and read the same every single run, or planning becomes guesswork.

Data Registry succeeds by being invisible. Its fantasy is the one the player never articulates: *"this game was made by someone who knew exactly what they were doing."* When players stop noticing the seams, the heist feels handcrafted. That's the whole job.

## Detailed Design

### Core Rules

**R1. Immutability at Runtime.** No system mutates a ScriptableObject field after load. All `DataRegistryEntry` fields are get-only accessors exposing private `[SerializeField]` backing fields. Runtime state (current stock count, cooldown remaining) lives in consumer system components, never in the Data Registry asset itself.

**R2. No `Resources.Load`.** All access is via (a) direct serialized inspector reference on a MonoBehaviour or ScriptableObject, or (b) a typed Catalog SO's `GetById(string)` / `All` API. Any PR introducing `Resources.Load` fails the CI gate.

**R3. Individual SO Assets + Typed Catalog Wrappers.** Each data entry is its own `.asset` file under `Assets/Data/[category]/`. One `[Category]Catalog.asset` per category aggregates all entries into a serialized list. Consumer systems hold a serialized reference to the Catalog, never to individual entries or to file paths.

**R4. GUID-Based Stable IDs.** Every `DataRegistryEntry` subclass has a serialized private `_id` string populated once via `AssetDatabase.AssetPathToGUID` in `OnValidate()`. IDs survive asset renames, moves, and build. Duplicated assets must be caught at the `AssetPostprocessor` layer because the serialized `_id` copies verbatim on duplicate.

**R5. Layered Editor-Time Validation.** Three layers:
- **Layer 1 — `OnValidate()`** per SO class: catches out-of-range values, null required references, empty display names. Logs via `Debug.LogWarning(this)` — never throws.
- **Layer 2 — `AssetPostprocessor.OnPostprocessAllAssets`**: catches duplicate IDs (cross-asset), broken cross-catalog references, missing sub-objects.
- **Layer 3 — EditMode CI tests**: batch validation via `AssetDatabase.FindAssets`. Blocks merges on duplicate IDs, null required fields, range violations, loot-table weight sums.

**R6. Direct Sprite/Prefab References.** ScriptableObjects serialize `Sprite` / `GameObject` references directly. No string-ID sprite lookups. Atlas discipline (art bible S8.4) is enforced by an `AssetPostprocessor` warning when a referenced sprite is not in any `SpriteAtlas`.

**R7. Hierarchical Event Taxonomy for Feedback Floor.** Audio and Visual event IDs use dot-notation string keys (`audio.tool.place.ramp`, `vfx.collect.coin.splash`) stored in typed `AudioEventTaxonomy` and `VisualEventTaxonomy` SOs. Format regex: `^(audio|vfx)\.[a-z_]+\.[a-z_]+\.[a-z_]+$`. All emitting systems reference event IDs by string constants cross-checked against the taxonomy SO at editor time.

**R8. Pillar 3 Balance Audit Hook.** `ToolConfig` exposes a computed `EfficiencyRatio` (effect magnitude / base cost). A custom editor window aggregates all `ToolConfig` entries and renders a sortable table. A `/balance-export` editor command dumps to `design/balance/tool-balance-[date].csv`. Unit tests assert no tool's ratio exceeds 2× the category mean unless a `_dominanceJustified` flag is set by the designer.

**R8.a (locked from CD-GDD-ALIGN Concern 1): `E_norm` is objective, not subjective.** `E_norm` is *derived* from authored ToolConfig parameters — it is never a separate designer-editable float. Per-tool-type derivation rules are normative (see D.2). A designer cannot silently tune `E_norm` to hide a dominance problem; they can only adjust the underlying objective parameters (angle range, catch area, coverage radius), which have their own gameplay consequences.

**R8.b (locked from CD-GDD-ALIGN Concern 3): `_dominanceJustified` has two hard gates.**
- (i) Setting `_dominanceJustified = true` requires a non-empty `_justificationNote` string (≥ 30 characters, authored by a designer). `OnValidate` rejects the flag with an empty note.
- (ii) The Balance Audit window renders a **blocking modal** (not an advisory banner) if > 30% of any tool category is justified — blocking modal means the designer cannot close the audit window until lead sign-off is captured. This prevents the `[ALL JUSTIFIED]` silent escape that CD-GDD-ALIGN flagged as Pillar 3 theater.

### MVP Data Categories (11)

| # | Category Class | Holds | Owner System | Key Validation |
|---|---|---|---|---|
| 1 | `ToolConfig` | Type, cost, effect params, upgrade tier chain | Tool System | Cost > 0; upgrade chain acyclic; all tier refs resolve |
| 2 | `HazardConfig` | Type, spawn timing, threat params, counter-tool ref, penalty | Hazard System | Fuse duration > 0; counter-tool ref resolves; penalty ≥ 0 |
| 3 | `GopherConfig` | Variant ID, rhythm pattern, loot table ref, anim state set | Gopher Spawn | Rhythm has ≥ 1 launch event; loot ref resolves |
| 4 | `LootTable` | Entry list (item, weight, qty range), guaranteed-drop flags | Collection System | Weights sum to 1.0 ± 0.001; no weight ≤ 0; qty min ≤ max |
| 5 | `LevelData` | Biome ref, quota, time budget, hole layout, hazard density, allowed tool mask | Level Runtime | Quota > 0; time ∈ [30,120]s; **quota achievability simulation** (error if un-completable) |
| 6 | `BiomeData` | Display name, unlock node ref, allowed hazards, max hole count, ambient audio ID, bg asset | Biome Map | Max holes ∈ [1,6]; ambient audio ID resolves; ≥ 1 hazard listed |
| 7 | `CurrencyDefinition` | Currency ID (coin/gem), name, icon, earn-rate baseline, IAP product ID (stub at MVP) | Currency & Economy | Earn-rate > 0; icon non-null; no duplicate IDs |
| 8 | `StarThresholdConfig` | Level ref, 1/2/3-star quota floors | Level Scoring | 0 < 1-star < 2-star < 3-star ≤ 1.0 |
| 9 | `ProgressionNode` | Node ID, unlock condition type + threshold, reward type + ref | Progression | Reward ref resolves; no circular deps; threshold > 0 |
| 10 | `AudioEventTaxonomy` | Hierarchical event ID → AudioClip / mixer group map | Audio Bus (Feedback Floor contract) | No duplicate keys; format regex matches; non-null clips at non-ambient tiers |
| 11 | `VisualEventTaxonomy` | Hierarchical event ID → VFX prefab / particle reference | VFX / Juice (Feedback Floor contract) | No duplicate keys; format regex matches; non-null VFX prefabs at MVP for `vfx.collect.*` + `vfx.hazard.*` |

**Vertical Slice additions**: `CosmeticConfig`, `ShopProductConfig`
**Shippable v1.0 additions**: `AnalyticsEventTaxonomy`, `LocalizationStringTable`

### States and Transitions

Data Registry has no runtime state machine. Asset lifecycle:

| State | Entry Condition | Exit Condition | Behavior |
|-------|----------------|----------------|----------|
| Authored | Designer creates `.asset` via `[CreateAssetMenu]` | Passes `OnValidate` | `_id` populated; fields editable in inspector |
| Validated (editor) | `OnValidate` + `AssetPostprocessor` both pass | Build or git commit | Asset is a valid authoring candidate |
| Catalog-Registered | Asset added to the category's Catalog serialized list | Designer removes it | Asset is discoverable via `Catalog.GetById(id)` |
| Loaded | Catalog SO's `OnEnable` runs → `Dictionary<string, T>` built | Domain reload / scene unload | O(1) lookup available to consumers |
| Consumed | Consumer system reads fields from the loaded SO | Scene transition | Fields treated as immutable by all consumers |

No transitions occur at runtime beyond `Loaded ↔ Consumed`. Designer-driven transitions (`Authored → Validated → Catalog-Registered`) happen only in the Unity editor.

### Interactions with Other Systems

| Downstream System | Consumes | Interface | Ownership |
|---|---|---|---|
| **Level Runtime** | `LevelData` + `BiomeData` catalogs | Serialized `LevelCatalog` / `BiomeCatalog` refs; queries `GetById(id)` | Data Registry owns schema; Level Runtime owns playback |
| **Level Scoring** | `StarThresholdConfig` catalog | Serialized `StarThresholdCatalog` ref | Data Registry owns threshold data; Scoring owns evaluation |
| **Gopher Spawn & Launch** | `GopherConfig` + `LootTable` catalogs | Serialized `GopherCatalog` + `LootTableCatalog` refs | Data Registry owns configs; Gopher Spawn owns rhythm + instantiation |
| **Tool System** | `ToolConfig` catalog | Serialized `ToolCatalog` ref | Data Registry owns stats; Tool System owns placement + deployment behavior |
| **Critter AI** | Indirect — reads `ToolConfig` (dog configs) and `HazardConfig` (cat configs) via Tool + Hazard systems | No direct Data Registry access | Data Registry owns tuning; Critter AI owns behavior execution |
| **Hazard System** | `HazardConfig` catalog | Serialized `HazardCatalog` ref | Data Registry owns hazard stats; Hazard System owns spawn + resolution |
| **Collection System** | `LootTable` catalog (shared with Gopher Spawn) | Serialized `LootTableCatalog` ref | Data Registry owns drop rules; Collection System owns banking math |
| **Currency & Economy** | `CurrencyDefinition` catalog | Serialized `CurrencyCatalog` ref | Data Registry owns currency metadata; Economy owns ledger |
| **Shop & IAP** (VS) | `ShopProductConfig` catalog | Serialized `ShopProductCatalog` ref | Data Registry owns product metadata; Shop owns transaction |
| **Progression** | `ProgressionNode` catalog | Serialized `ProgressionCatalog` ref | Data Registry owns unlock tree; Progression owns state tracking |
| **Cosmetic / Skin System** (VS) | `CosmeticConfig` catalog | Serialized `CosmeticCatalog` ref | Data Registry owns skin metadata; Cosmetic owns application |
| **HUD** | Icon refs from multiple catalogs (coin icon from `CurrencyDefinition`, tool icons from `ToolConfig`) | Indirect — HUD receives SO refs from consumer systems, not from Data Registry directly | UI observes; does not query registry |
| **Audio Bus** (Feedback Floor contract) | `AudioEventTaxonomy` | Serialized taxonomy SO ref; emitting systems declare event ID string constants cross-checked at editor time | Data Registry owns taxonomy; systems own emission |
| **VFX / Juice** (VS) (Feedback Floor contract) | `VisualEventTaxonomy` | Same pattern as Audio | Data Registry owns taxonomy; VFX owns execution |
| **Analytics** (v1.0) | `AnalyticsEventTaxonomy` | Same taxonomy pattern | Data Registry owns schema; Analytics owns dispatch |
| **Localization** (v1.0) | `LocalizationStringTable` catalogs (per-locale) | Serialized catalog refs | Data Registry owns string data; Localization owns lookup |

## Formulas

Data Registry's formulas are validation and audit formulas, not gameplay balance formulas. Gameplay balance formulas live in the consuming systems' GDDs (Tool System, Level Scoring, etc.).

### D.1 NormalizedProbability (LootTable runtime)

Raw weights are designer-authored with no sum-to-1 constraint. Runtime normalizes; the editor inspector surfaces computed probabilities so designers see what they actually authored.

`P_i = W_i / W_total`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Raw weight of entry i | W_i | float | (0, ∞) | Designer-authored weight for a single LootTable entry; must be strictly positive |
| Sum of all entry weights | W_total | float | (0, ∞) | Sum of all `W_i` values in the same LootTable |
| Normalized probability of entry i | P_i | float | (0, 1] | Runtime draw probability for entry i; all `P_i` in a table sum to exactly 1.0 |

**Output Range:** (0, 1] per entry; full set sums to 1.0 exactly. Single-entry tables yield `P_i = 1.0`.

**Example:** Three entries — Coin (W=10), Gem (W=3), Nothing (W=2). `W_total` = 15. `P_coin` = 0.667, `P_gem` = 0.200, `P_nothing` = 0.133. Sum = 1.0.

**Validation (R5):** `OnValidate` rejects `W_i ≤ 0` or `W_total == 0`. CI test asserts both conditions across all LootTable assets. No sum-to-1 check is enforced (runtime normalizes).

### D.2 EfficiencyRatio (Pillar 3 audit)

A tool's within-category value-per-coin, computed at editor time and surfaced in the balance audit window.

`EfficiencyRatio = E_norm / C_base`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Normalized effect magnitude | E_norm | float | [0, 1] | **Objective, derived — never a designer-editable float** (R8.a). Per tool type: **Ramps** = `redirectAngleRangeDegrees / 180`; **Nets** = `catchZoneAreaTiles / maxCatchZoneAreaTiles` (max = biome max hole count × 2 tiles); **Guard Posts** = `coverageRadiusTiles / maxCoverageRadiusTiles` (max = half the biome's max hole count). Additional tool types must specify their objective derivation rule in their ToolConfig schema before shipping. |
| Base cost in coins | C_base | int | [1, 999] | Authored purchase cost from `ToolConfig`; R5 enforces > 0 |
| Efficiency ratio for tool i | EfficiencyRatio_i | float | [0, ∞) | Relative value-per-coin within a category; higher = more efficient |

**Output Range:** [0, ∞). Dominance threshold creates a soft ceiling.

**Example:** Ramp tool, `C_base` = 30 coins, `E_norm` = 0.7 → `EfficiencyRatio` = 0.0233.

**Within-Category Mean:**

`CategoryMean_k = (1 / N_k) * Σ(EfficiencyRatio_i) for all i in category k`

Arithmetic mean (not median). Small categories (3-8 tools at MVP) have unstable standard deviation; arithmetic mean is the conservative choice — dominant outliers raise the bar for themselves.

**Dominance Rule (R8):** Tool flagged if `EfficiencyRatio_i > 2.0 * CategoryMean_k` AND `_dominanceJustified == false`.

**`_dominanceJustified` treatment:** Justified tools REMAIN in `CategoryMean` calculation (excluding them would deflate the mean and make other tools appear dominant). Conservative: the outlier raises the bar for everyone.

**Full audit example:** Category "Redirection" with 3 tools — Ramp (0.0233), Curved Wall (0.015), Mega Ramp (0.045). Mean = 0.0278. Threshold = 0.0556. All three pass. Counter-example: if Mega Ramp `E_norm` = 0.9 at cost 10, ratio = 0.09 > 0.0556 → flagged unless `_dominanceJustified = true`.

### D.3 QuotaAchievabilityCeiling (LevelData author-time simulation)

Validates that a `LevelData` quota is mathematically reachable under worst-case conditions.

`Ceiling = H * EV_loot * L_budget * C_rate`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Hole count | H | int | [1, 6] | Number of active gopher holes in this `LevelData` (`BiomeData.maxHoles` bounds) |
| Expected loot value per arc | EV_loot | float | [0.5, 50.0] coins | `Σ (P_i * V_i)` across referenced LootTable entries, where `V_i` is the mid-point coin value of entry i's qty range |
| Launches per time budget | L_budget | float | [1, 60] | Worst-case launches per hole: `T_level / T_arc_max` where `T_level` ∈ [30, 120] s and `T_arc_max` is the slowest gopher arc from the referenced `GopherConfig` rhythm |
| Collection rate | C_rate | float | [0.0, 1.0] | **Worst-case 0.6.** Corresponds to worst allowed tool loadout + slowest gopher rhythm; not a skill estimate |
| Quota achievability ceiling | Ceiling | float | [0, ∞) | Maximum coins collectible under worst-case over the full level duration |

**Output Range:** [0, ~10,800] coins at extreme settings (6 holes × 50 EV × 60 launches × 0.6).

**Worst-case assumptions:** (1) worst allowed tool loadout from the authored allowed-tool mask, (2) slowest gopher rhythm (`T_arc_max` not mean), (3) `C_rate` fixed at 0.6 floor. Does NOT simulate worst player skill below 0.6 — this is a mathematical bound, not a playtest estimate.

**Pass / Warning / Error bands:**

| Condition | Verdict | CI effect |
|---|---|---|
| `Ceiling >= Quota / 0.95` | **PASS** (comfortable headroom) | No action |
| `Quota <= Ceiling < Quota / 0.95` | **WARNING** (95-100% of ceiling — brutal but legal) | Logged advisory; author sees computed ceiling number |
| `Ceiling < Quota` | **ERROR** (mathematically unreachable) | Blocks merge |

**Tolerance justification:** 95% threshold chosen over 90% (too permissive — forces near-perfect play even with skilled player) and 99% (too conservative — rejects well-designed tight levels). The 0.6 C_rate worst case already bakes in ~40% slack vs. a skilled player, so a level at 97% of worst-case ceiling is still comfortable for an average player.

**Example:** 3 holes, EV_loot = 8 coins, T_level = 60s, T_arc_max = 4s → L_budget = 15. C_rate = 0.6. `Ceiling` = 3 × 8 × 15 × 0.6 = 216 coins. Quota 180 = 83% → PASS. Quota 210 = 97% → WARNING. Quota 230 = 107% → ERROR.

## Edge Cases

### Authoring-Time

- **If a designer duplicates an asset (Ctrl+D or git merge conflict resolved by accepting both branches)**: the new asset copies `_id` verbatim from the source. `OnValidate` on the duplicate does not re-run `AssetPathToGUID` unless explicitly triggered. `AssetPostprocessor.OnPostprocessAllAssets` catches this on the next import event and logs `Debug.LogError` naming both asset paths. Duplicate `_id` blocks the CI EditMode test. Resolution: designer deletes the duplicate or clicks "Reset ID" on the custom inspector.

- **If a `LevelData` is authored before its referenced `BiomeData` exists**: `OnValidate` evaluates biome ref as null, logs a WARNING, and reports the quota achievability simulation as SKIPPED (not ERROR). The CI EditMode test treats SKIPPED as an ERROR to block merges. The WARNING state exists only to allow mid-session authoring without blocking the editor.

- **If a designer sets `_dominanceJustified = true` on every tool in a category**: D.2 still includes all tools in `CategoryMean_k`. CI passes (no unflagged dominators). The balance audit window renders an `[ALL JUSTIFIED]` banner requiring lead review. Not automatically blocked — consistent with `_dominanceJustified` as a designer escape valve.

- **If a `StarThresholdConfig` references a `LevelData` that was removed from the catalog**: Unity serializes the reference as a missing-object sentinel (not null, but compares to null as `false`). `OnValidate` logs an error. CI build step iterating all `StarThresholdConfig` assets catches it as null-equivalent. Resolution: designer re-assigns or deletes the orphan.

- **If an event ID passes the taxonomy regex but has no `AudioClip` assigned**: format validation passes. Non-null clip enforcement fires only for non-ambient tiers. Designer assigning a valid-format key with a null clip at a `vfx.collect.*` tier passes `OnValidate` silently; the CI EditMode test asserts non-null clips for all `vfx.collect.*` + `vfx.hazard.*` entries, blocking the merge.

- **If a `LootTable` has a single entry with `W_i = 0.0001`**: `OnValidate` only rejects `W_i ≤ 0`; epsilon weights are accepted. At runtime `P_i = 1.0` (single entry normalizes regardless of weight). For multi-entry tables with one epsilon weight, the entry is practically dead. Mitigation: CI test adds an advisory (not error) if `W_i < 0.01` for any entry in a multi-entry table.

### Build-Time

- **If a scene holds a direct serialized reference to a `ToolConfig` that is deleted before build**: Unity serializes this as a missing-object reference, not null. Build does not fail automatically. The CI build step must include an explicit `AssetDatabase` sweep for missing-object references in all scenes in build settings. This sweep is a BLOCKING CI gate.

- **If a GUID is manually hex-edited in a `.meta` file (manual conflict resolution)**: `AssetDatabase.AssetPathToGUID` returns the new GUID; the `_id` field stored in the `.asset` file still holds the old GUID. `Catalog.GetById(old_id)` returns null at runtime for any consumer holding the old ID as a string constant. CI EditMode test cross-checks every asset's `_id` against `AssetPathToGUID` for that asset path, logging error on mismatch. Resolution: "Reset ID" button re-runs population.

- **If two catalogs of the same category exist (e.g., two `ToolCatalog.asset` files)**: R3 requires one per category, but Unity does not enforce singleton catalog assets. A consumer referencing the wrong catalog silently operates on a subset. CI build step asserts `AssetDatabase.FindAssets("t:ToolCatalog")` returns exactly one result per category type. ERROR gate.

### Runtime

- **If consumer code attempts mutation of an SO field via reflection at runtime (bypassing get-only accessors)**: Unity does not prevent this. Field write succeeds; corrupts shared in-memory SO state for all consumers for the rest of the session. Mitigation: CI static-analysis rule greps for `SetValue` or `FieldInfo.SetValue` on `DataRegistryEntry` subtypes outside test assemblies, failing the build if found.

- **If `Catalog.GetById(id)` is called before `OnEnable` completes (race during domain reload)**: the internal `Dictionary` is not yet built; `GetById` returns null. Consumers must check null on every `GetById` call and treat null as a configuration error, not a gameplay state. Systems calling `GetById` in `Awake` must use `[DefaultExecutionOrder]` on the catalog or delay to `Start`.

- **If `AudioEventTaxonomy.GetClip(id)` is called with a valid-format key that has no entry in the taxonomy**: returns null. Audio Bus's null-clip path must play silence and log `Debug.LogWarning(id)`, never throw. Preserves audio-less gameplay instead of crashing; warning surfaces the unregistered ID to the designer.

### Formula Boundaries

- **If `QuotaAchievabilityCeiling` exactly equals `Quota` (ratio = 100%)**: falls in the WARNING band, not PASS. Because `Ceiling >= Quota / 0.95` requires `Ceiling >= Quota * 1.053`, exact equality fails the PASS condition but passes the non-error condition. Author must reduce quota by at least 1 coin or raise one of `H`, `EV_loot`, `T_level` to achieve 5% margin. Intentional: 100% of worst-case ceiling is unreachable for any player below perfect.

- **If an `EfficiencyRatio` category contains exactly one tool (N_k = 1)**: `CategoryMean_k` equals that tool's own ratio. The threshold is `2.0 * own_ratio`. The tool trivially passes. Balance audit window renders `[SINGLE ENTRY — DOMINANCE CHECK SKIPPED]` label instead of a misleading green result.

- **If all `LootTable` weights are equal (e.g., all `W_i = 1.0`)**: `P_i = 1/N` for all entries. Uniform distribution. This is a valid designer intent; `OnValidate` renders a uniform-distribution preview. Documented here because "why does my 10/10/10 table give equal odds?" is a common designer question.

- **If `EV_loot = 0` because a LootTable has only zero-value entries (e.g., a decoy gopher's pure-nothing table)**: `Ceiling = 0` regardless of other inputs → ERROR per D.3. But a pure-nothing gopher is a valid design intent. Resolution: designer sets `_excludeFromQuotaSimulation = true` on the LootTable; `QuotaAchievabilityCeiling` treats missing EV contribution as a conscious opt-out, not a misconfiguration.

## Dependencies

### Upstream

**None.** Data Registry is a Foundation system with zero dependencies on other Money Miner systems. It depends only on the Unity 6.3 engine (ScriptableObject lifecycle, `AssetDatabase`, `AssetPostprocessor`) and the project's namespace / naming conventions (from `.claude/docs/technical-preferences.md`).

### Downstream (hard — cannot function without Data Registry)

| System | Priority | Nature |
|---|---|---|
| Level Runtime | MVP | Data dependency (`LevelData`, `BiomeData`) |
| Level Scoring | MVP | Data dependency (`StarThresholdConfig`) |
| Gopher Spawn & Launch | MVP | Data dependency (`GopherConfig`, `LootTable`) |
| Tool System | MVP | Data dependency (`ToolConfig`) |
| Hazard System | MVP | Data dependency (`HazardConfig`) |
| Collection System | MVP | Data dependency (`LootTable`) |
| Currency & Economy | MVP | Data dependency (`CurrencyDefinition`) |
| Progression / Unlock Gating | MVP | Data dependency (`ProgressionNode`) |
| Biome Map / Level Select | MVP | Data dependency (`BiomeData`, `ProgressionNode`) |
| Audio Bus | MVP | Data dependency (`AudioEventTaxonomy`) |
| HUD | MVP | Data dependency (indirect — via consumer systems' SO refs) |

### Downstream (phased — dependency activates at system's delivery tier)

| System | Delivery Tier | Nature |
|---|---|---|
| VFX / Juice | Vertical Slice | Data dependency (`VisualEventTaxonomy`) |
| Shop & IAP | Vertical Slice (full) | Data dependency (`ShopProductConfig`, reads `CurrencyDefinition`) |
| Cosmetic / Skin System | Shippable v1.0 | Data dependency (`CosmeticConfig`) |
| Analytics / Telemetry | Shippable v1.0 | Data dependency (`AnalyticsEventTaxonomy`) |
| Localization | Shippable v1.0 | Data dependency (`LocalizationStringTable`) |

### Downstream (soft — reads values but system functions if Data Registry lookups return null)

| System | Nature |
|---|---|
| Critter AI | Reads tuning via Tool System + Hazard System; no direct Data Registry access |
| Save / Load | Reads save schema version from `DataRegistryEntry.Id`-keyed lookups; can fall back to default schema if catalog is missing |

### Interfaces

See **Section C → Interactions with Other Systems** for the interface contract per downstream system (serialized reference pattern, ownership boundary, query API shape). This section avoids duplicating that table.

### Cross-System Ownership Rule

Data Registry owns **schema and immutable data values**. Downstream systems own:
- **Runtime state** (current stock, cooldown remaining, active instances)
- **Game behavior** (when to spawn, how to respond, what to render)
- **Save/Load state** (player progression, owned items)

If a value would need to change at runtime, it does not belong in Data Registry. If a schema change is required, Data Registry owns the update; consumers must adapt.

## Tuning Knobs

Data Registry's own tuning knobs are validation thresholds. Gameplay balance values (tool stats, quota targets, loot drop rates) live in the registry assets themselves (ToolConfig, LevelData, LootTable, etc.), not in the Data Registry system.

| Parameter | Current Value | Safe Range | Effect of Increase | Effect of Decrease |
|---|---|---|---|---|
| `EfficiencyRatio` dominance multiplier (D.2) | 2.0× category mean | [1.5×, 3.0×] | More permissive — fewer tools flagged as dominant; risk: real outliers slip through audit | Stricter — more tools flagged; risk: false positives cause designer churn on legitimate tier variations |
| `QuotaAchievabilityCeiling` WARNING band ratio (D.3) | 0.95 (95% of ceiling) | [0.90, 0.99] | More permissive WARNING band — fewer levels flagged brutal; allows tighter authored quotas | Stricter — more levels flagged brutal; better player experience on difficulty curves |
| `QuotaAchievabilityCeiling` worst-case `C_rate` (D.3) | 0.6 | [0.5, 0.75] | More optimistic ceiling estimate — allows tighter authored quotas; risk: un-completable levels ship | More pessimistic — more quotas rejected as ERROR; safer for new designers but frustrating for experienced tuners |
| LootTable min-weight advisory threshold | 0.01 | [0.001, 0.05] | Looser — fewer epsilon-weight advisories | Stricter — more entries flagged as dead; helps catch designer typos |
| CI validation severity (duplicate IDs) | ERROR (blocks merge) | ERROR only | N/A | Changing to WARNING is strictly wrong — duplicate IDs corrupt runtime lookups |
| CI validation severity (null required refs) | ERROR | ERROR only | N/A | Changing to WARNING lets broken assets ship; strictly wrong |
| CI validation severity (missing sprite atlas membership) | WARNING | [WARNING, ERROR] | N/A | Upgrading to ERROR when atlas discipline is established (art bible S8.4) makes this a ship-gate |
| CI validation severity (epsilon loot weights) | ADVISORY (logged only) | [ADVISORY, WARNING, ERROR] | N/A | Upgrade to WARNING once v1.0 design is stable; never ERROR (valid designer intent) |
| Event taxonomy format regex | `^(audio\|vfx)\.[a-z_]+\.[a-z_]+\.[a-z_]+$` | **Locked** | N/A | Format change requires migration of every emitting system's event ID string — treat as architecture change, not tuning |

### Interaction Between Tuning Knobs

- **Independent**: Dominance multiplier (D.2) and `C_rate` floor (D.3) — no cross-effect.
- **Coupled**: Quota WARNING band ratio (D.3) and `C_rate` floor (D.3). Tightening one without the other produces either over-permissive or over-punitive validation. Change these together when adjusting difficulty calibration.

## Visual/Audio Requirements

**Not applicable.** Data Registry has no runtime visual or audio presence. It produces data consumed by other systems that themselves have V/A requirements (e.g., VFX/Juice consumes `VisualEventTaxonomy`; Audio Bus consumes `AudioEventTaxonomy`). The V/A requirements for *those emissions* live in the emitting systems' GDDs, not here.

If a future author considers adding runtime visual/audio to Data Registry (e.g., a runtime-visible "asset loaded" indicator), treat it as a scope violation and route the concern to VFX/Juice or HUD instead.

## UI Requirements

**No player-facing UI.** Data Registry is entirely editor-side; the player never sees any Data Registry surface. Consumer systems (HUD, Shop UI, Biome Map) render values that originated in Data Registry, but their UI requirements live in their own GDDs.

**Editor tooling UX (not a GDD-level concern; noted for scope clarity):**
- Custom inspectors for each `DataRegistryEntry` subclass (inline upgrade chain previews, clip preview buttons, regex validation feedback)
- Balance Audit window (surfaces `EfficiencyRatio` per tool category with sortable columns)
- `/balance-export` editor menu command
- Validation Console integration (channels all R5 validation output into the Unity Console with clickable asset links)

Editor tooling is implemented alongside the Data Registry code and is scoped in the Technical Setup / Production phase — not designed here.

## Cross-References

| This Document References | Target GDD | Specific Element | Nature |
|---|---|---|---|
| Event Bus decouples VFX/Audio/HUD/Modal from Core system internals | `design/gdd/event-bus.md` (pending) | Event Bus publish/subscribe API | Ownership handoff — Data Registry declares the event taxonomy; Event Bus owns the dispatch mechanism |
| Save/Load must handle schema versioning from day one | `design/gdd/save-load.md` (pending) | Schema migration rules | Rule dependency — Data Registry asset schemas must be versioned via `_schemaVersion` on each DataRegistryEntry subclass |
| Visual/audio event emission contract from CD-SYSTEMS Feedback Floor | `design/gdd/input-system.md` / `collection-system.md` / `tool-system.md` / `hazard-system.md` / `critter-ai.md` (all pending) | Event IDs per Feedback Floor contract | Data dependency — emitters must use event IDs registered in `AudioEventTaxonomy` / `VisualEventTaxonomy` |
| Atlas discipline for sprite references | `design/art/art-bible.md` | S8.4 Sprite Atlas Strategy | Rule dependency — Data Registry enforces atlas membership via AssetPostprocessor warning |
| Loot table probabilities feed Collection System banking math | `design/gdd/collection-system.md` (pending) | `NormalizedProbability` output | Data dependency — Collection System rolls loot via the normalized distribution Data Registry computes |
| Tool EfficiencyRatio is the audit lever for Pillar 3 | `design/gdd/tool-system.md` (pending) | `EfficiencyRatio` per-tool constant | Data dependency — Tool System reads `E_norm` from ToolConfig and exposes it to the balance audit window |
| Quota achievability simulation validates `LevelData` quotas | `design/gdd/level-runtime.md` + `level-scoring.md` (pending) | `QuotaAchievabilityCeiling` value | Data dependency — Level Runtime pass/warning/error classification shapes merge eligibility for any LevelData asset |

## Acceptance Criteria

31 criteria across 6 categories. Gate classifications: **BLOCKING CI** (must pass for merge) / **ADVISORY** (logged, non-blocking) / **PLAYMODE** (requires Unity PlayMode test runner) / **MANUAL** (designer-visible check).

### H.1 Rule Compliance

- **AC-R1 — Runtime Immutability** — GIVEN a ScriptableObject asset is loaded into a running game session, WHEN any game system attempts to set a field on that SO at runtime, THEN the operation must either fail to compile (readonly field enforcement) or be caught by a CI static-analysis grep for assignment patterns on SO types, with zero violations permitted. **BLOCKING CI**

- **AC-R2 — No `Resources.Load`** — GIVEN the full game codebase, WHEN a CI grep job scans for `Resources.Load`, THEN zero matches appear in any file under `src/` or `tests/`. **BLOCKING CI**

- **AC-R3 — Typed Catalog Structure** — GIVEN the 11 MVP data categories, WHEN the asset database is inspected, THEN exactly one Catalog asset of the correct typed wrapper class exists per category. A CI EditMode test enumerates Catalog instances and asserts count-per-type equals 1. **BLOCKING CI**

- **AC-R4 — GUID Stability** — GIVEN any SO asset in the registry, WHEN inspected in the Editor or via EditMode test, THEN its `_id` field equals `AssetDatabase.AssetPathToGUID(assetPath)`. **BLOCKING CI**

- **AC-R5 — Three-Layer Validation Present** — GIVEN the Data Registry implementation, WHEN a CI job runs, THEN all three validation layers execute (OnValidate on SO save, AssetPostprocessor on import, EditMode CI tests). All three must report zero errors for merge. **BLOCKING CI**

- **AC-R6 — Atlas Discipline Warning** — GIVEN any SO with a Sprite or Prefab field, WHEN the AssetPostprocessor runs on import, THEN SOs referencing sprites not in any `SpriteAtlas` emit a console warning labeled `[DataRegistry] Atlas discipline:`. Logged but non-blocking at MVP. **ADVISORY**

- **AC-R7 — Event Taxonomy Format** — GIVEN an AudioEventTaxonomy or VisualEventTaxonomy SO, WHEN its `eventKey` is set to any string, THEN `OnValidate` evaluates `^(audio|vfx)\.[a-z_]+\.[a-z_]+\.[a-z_]+$`; non-matching strings cause a Unity validation error in Inspector and a blocking EditMode test failure in CI. **BLOCKING CI**

- **AC-R8 — EfficiencyRatio Balance Audit** — GIVEN all ToolConfig SOs, WHEN the balance audit hook runs (on asset save or CI), THEN any asset with EfficiencyRatio > 2× category mean is flagged with a warning; CI reports the count; zero flagged assets is not required but count must be reported. **ADVISORY**

- **AC-R8a — E_norm is Derived, Not Authored** — GIVEN a ToolConfig with an `E_norm` field, WHEN the CI EditMode test inspects the field, THEN it verifies `E_norm` is a computed property (not a serialized `[SerializeField]`) and that its value matches the tool-type-specific derivation rule from D.2 (Ramps: angle/180°; Nets: area/max area; Guard Posts: radius/max radius). Any drift between `E_norm` and the derivation is a blocking failure. **BLOCKING CI**

- **AC-R8b — `_dominanceJustified` Requires Justification Note** — GIVEN a ToolConfig with `_dominanceJustified = true`, WHEN `OnValidate` runs, THEN the asset's `_justificationNote` string must be non-empty and ≥ 30 characters; empty or too-short notes cause OnValidate error. CI EditMode test covers both boundary (29 chars = fail, 30 chars = pass). **BLOCKING CI**

- **AC-R8c — Balance Audit Window UX** — GIVEN the Balance Audit window is opened in Unity Editor, WHEN the lead designer inspects it, THEN it must (1) be sortable by ratio descending, (2) render flagged rows in a visually distinct manner (e.g., red row background or warning icon), (3) support one-click jump from audit row to the offending asset in the Project view, (4) expose a CSV export menu command. If any of these four capabilities is missing, the window fails its Pillar 3 enforcement purpose. **MANUAL** (lead designer sign-off at end of Data Registry implementation sprint)

- **AC-R8d — Category-Wide Justification Modal** — GIVEN a tool category where > 30% of tools have `_dominanceJustified = true`, WHEN the Balance Audit window is opened, THEN it renders a blocking modal (not a dismissible banner) that prevents closing the window until a lead designer sign-off is recorded via a lead-designer-only editor pref. CI EditMode test asserts the modal fires on the 31% boundary with a synthetic category. **BLOCKING CI** (test firing) + **MANUAL** (sign-off mechanism)

### H.2 Formula Correctness

- **AC-D1 — NormalizedProbability** — GIVEN a LootTable with weights `[1, 3, 6]`, WHEN `NormalizedProbability` is computed, THEN results are `[0.1, 0.3, 0.6]` (sum = 1.0, ±0.0001). A second case with weights `[0, 0, 1]` yields `[0, 0, 1]`. EditMode unit test. **BLOCKING CI**

- **AC-D2 — EfficiencyRatio Dominance Boundary** — GIVEN a ToolConfig with `E_norm = 1.0, C_base = 0.5` (ratio = 2.0) and category mean = 1.0, WHEN audit runs, THEN the asset is flagged. A second case at 1.99× mean must NOT be flagged. EditMode unit test covers both boundary values. **BLOCKING CI**

- **AC-D3 — QuotaAchievabilityCeiling Band Boundaries** — GIVEN Ceiling = 0.94× Quota, THEN ERROR. GIVEN Ceiling = 0.97× Quota, THEN WARNING. GIVEN Ceiling = 1.01× Quota, THEN PASS. GIVEN Ceiling = 1.00× Quota exactly, THEN WARNING (not PASS, per EC-F1). All four covered by a single parameterized EditMode unit test. **BLOCKING CI**

### H.3 Edge Case Handling

- **AC-EC-A1 — Duplicate Asset GUID Collision** — GIVEN an SO asset is duplicated via Ctrl+D, WHEN the AssetPostprocessor runs on the duplicate's import, THEN a console error `[DataRegistry] Duplicate _id detected:` names both asset paths; CI EditMode test blocks merge on any collision. **BLOCKING CI**

- **AC-EC-B2 — Hex-Edited GUID Mismatch** — GIVEN an SO asset whose `.meta` file GUID was modified externally (simulated by test fixture), WHEN the CI EditMode GUID-consistency test runs, THEN `_id` ≠ `AssetDatabase.AssetPathToGUID(path)` detection fires and blocks merge. **BLOCKING CI**

- **AC-EC-R1 — Reflection Mutation of SO** — GIVEN the full codebase, WHEN a CI static-analysis grep scans for reflection patterns targeting SO field writes (e.g., `FieldInfo.SetValue` on `DataRegistryEntry` subtypes), THEN zero matches under `src/`; any match is a blocking CI failure. **BLOCKING CI**

- **AC-EC-F1 — Ceiling Equals Quota Exactly** — GIVEN D.3 inputs where `H × EV × L × 0.6 = Quota` exactly (ratio = 1.0), WHEN evaluated, THEN result band is WARNING, not PASS. Covered in AC-D3's parameterized test. **BLOCKING CI**

- **AC-EC-A2 — Asset Rename Preserves `_id`** — GIVEN an SO asset with a known `_id`, WHEN renamed in the Editor, THEN `OnValidate` re-runs, `_id` updates to the new GUID, Inspector refresh confirms; CI GUID-consistency test then passes. **MANUAL** (rename) + **BLOCKING CI** (post-rename consistency check)

### H.4 Performance

- **AC-P1 — Catalog Dictionary Build Time** — GIVEN any single-category Catalog with up to 500 SO assets, WHEN the `Dictionary<string,T>` is built at Editor load or domain reload, THEN build time ≤ 50ms (measured via `System.Diagnostics.Stopwatch` in EditMode perf test). **ADVISORY**

- **AC-P2 — Catalog Load Time at Scene Entry** — GIVEN all 11 Catalogs pre-warmed, WHEN a scene is entered in PlayMode and all Catalogs are accessed for the first time in that session, THEN total time from scene load to all Catalog lookups returning valid results ≤ 100ms on minimum-spec Mali-G52 Android hardware. **PLAYMODE**

- **AC-P3 — Memory Footprint** — GIVEN all 11 Catalogs loaded with MVP asset sets, WHEN a Unity Memory Profiler snapshot is taken immediately after Catalog initialization, THEN combined heap allocation attributable to Data Registry ≤ 32MB. **ADVISORY**

### H.5 Consumer Integration (PLAYMODE per system)

- **AC-C1 — Level Runtime** — GIVEN a LevelData SO with defined quota, biome, and tool references, WHEN Level Runtime requests level config at scene load, THEN it receives a populated `LevelData` instance with all fields non-null and non-default; no `Resources.Load` in the call stack. **PLAYMODE**

- **AC-C2 — Tool System** — GIVEN a ToolConfig SO with EfficiencyRatio and Sprite reference, WHEN the Tool System requests config by GUID, THEN Catalog lookup returns the correct `ToolConfig` within a single dictionary access; Sprite ref is non-null. **PLAYMODE**

- **AC-C3 — Hazard System** — GIVEN a HazardConfig SO with defined damage and trigger params, WHEN the Hazard System requests config by GUID, THEN returned `HazardConfig` matches authored values; lookup completes without warnings. **PLAYMODE**

- **AC-C4 — Critter AI (via Tool/Hazard)** — GIVEN GopherConfig and HazardConfig SOs referenced by the same LevelData, WHEN Critter AI initializes its behavior tree, THEN it reads tool-avoidance weights from ToolConfig and hazard-response thresholds from HazardConfig via Catalog lookups; no null-reference exceptions. **PLAYMODE**

- **AC-C5 — Gopher Spawn System** — GIVEN a GopherConfig SO with spawn weight and max-count fields, WHEN the spawn system initializes for a level, THEN it reads `GopherConfig` from the Catalog; spawned gopher count respects `maxCount`. **PLAYMODE**

- **AC-C6 — Collection System** — GIVEN a LootTable SO with ≥ 3 entries, WHEN Collection System requests loot resolution, THEN it receives a normalized probability distribution (sum = 1.0) from `NormalizedProbability` and selects exactly one entry without error. **PLAYMODE**

- **AC-C7 — Currency & Economy** — GIVEN a CurrencyDefinition SO with exchange rate and display icon, WHEN Economy requests currency metadata by GUID, THEN it receives the correct definition; icon Sprite ref is non-null. **PLAYMODE**

- **AC-C8 — Progression System** — GIVEN a ProgressionNode SO with prerequisite and unlock conditions, WHEN Progression evaluates unlock eligibility, THEN it reads all referenced ProgressionNode SOs from the Catalog with no missing-ref errors; correctly evaluates at least one PASS and one FAIL in an integration test fixture. **PLAYMODE**

- **AC-C9 — Biome Map** — GIVEN a BiomeData SO with terrain, hazard, and audio-zone references, WHEN Biome Map initializes a level region, THEN it reads `BiomeData` from the Catalog; all nested reference fields resolve to non-null assets. **PLAYMODE**

- **AC-C10 — HUD** — GIVEN StarThresholdConfig and CurrencyDefinition SOs relevant to the current level, WHEN the HUD initializes its score display and currency counter, THEN it reads both configs from their Catalogs within a single scene-load frame and displays authored threshold values without fallback defaults. **PLAYMODE**

- **AC-C11 — Audio Bus** — GIVEN an AudioEventTaxonomy SO with at least one valid event key, WHEN the Audio Bus requests the event config by key string, THEN the event config is returned within one dictionary access; the key passes regex validation at lookup time. **PLAYMODE**

### H.6 No Hardcoded Values

- **AC-NHV — Zero Hardcoded Gameplay Values in Source** — GIVEN the full `src/` directory, WHEN a CI grep job scans for numeric literals and string constants matching known gameplay value patterns (damage values, probability weights, currency rates, quota numbers, star thresholds), THEN zero matches appear outside Data Registry SO assets and their schema definitions. Blocking failure produces a list of offending file paths and line numbers. **BLOCKING CI**

### Gate Summary

| Gate Level | Count | Systems / Files |
|---|---|---|
| **BLOCKING CI** | 16 | All R1-R8 (except R6+R8 advisory), D.1-D.3, EC-A1, EC-B2, EC-R1, EC-F1, EC-A2 (post-rename), AC-NHV |
| **ADVISORY** | 4 | R6 atlas warning, R8 balance audit count, P1 build time, P3 memory |
| **PLAYMODE** | 11 | All H.5 consumer integration + P2 load time |
| **MANUAL** | 1 | EC-A2 rename observation (paired with BLOCKING CI follow-up) |

**Shift-left note for implementation**: the 16 BLOCKING CI criteria should have stub EditMode test files created at story start, before any SO schema is authored. The PLAYMODE integration criteria require a test scene fixture with pre-populated Catalog assets — this fixture is scaffolded during the same sprint as Catalog implementation, not deferred to QA hand-off.

## Open Questions

| # | Question | Owner | Target Resolution |
|---|---|---|---|
| ~~OQ-1~~ **RESOLVED 2026-04-20 via CD-GDD-ALIGN Concern 3**: `_dominanceJustified` requires non-empty `_justificationNote` (≥ 30 chars); Balance Audit window renders blocking modal if > 30% of a category is justified, requiring lead designer sign-off. See R8.b. |
| OQ-2 | Should the custom ToolConfig inspector show the full upgrade-tier chain inline, or link out to each tier SO? Inline = better authoring ergonomics but larger inspector render; linked = cleaner UI but more clicks for comparisons. | Tools Programmer + game-designer | During Tool System GDD authoring |
| OQ-3 | When Localization ships at v1.0, does Data Registry extend to hold per-locale strings (`LocalizationStringTable` as an extension of the same pattern), or does Localization get its own registry mirror? | Localization lead + technical-director | v1.0 architecture phase |
| OQ-4 | Event ID string constants for consumers: generate from taxonomy at build-time (C# source generator) vs. hand-written `public const string EventIdRampPlaced = "audio.tool.place.ramp"` per emitter? Source generator is cleaner but adds toolchain risk; hand-written drifts from taxonomy without taxonomy-source-of-truth enforcement. | Lead Programmer | Architecture phase (`/architecture-decision`) |
| OQ-5 | At Full Vision (5 biomes, 100+ levels, cosmetic catalog expansion), should Data Registry adopt Addressables? TD recommendation says yes at post-launch if memory pressure warrants. Trigger condition: Catalog heap > 32MB (AC-P3) on 3GB Android devices. | Technical Director | Post-launch memory profiling |
| OQ-6 | `_excludeFromQuotaSimulation` flag on LootTable (EC-F4): should this flag be visible in the inspector to all designers, or gated behind a "show advanced" toggle? Visible = may encourage misuse; gated = friction on valid decoy-gopher designs. | Lead Designer | During first decoy-gopher level design |
