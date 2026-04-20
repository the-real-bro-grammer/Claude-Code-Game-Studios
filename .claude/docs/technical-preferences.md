# Technical Preferences

<!-- Populated by /setup-engine. Updated as the user makes decisions throughout development. -->
<!-- All agents reference this file for project-specific standards and conventions. -->

## Engine & Language

- **Engine**: Unity 6.3 LTS (6000.3.x)
- **Language**: C# (.NET 8+, Unity-flavored)
- **Rendering**: Universal Render Pipeline (URP) — 2D Renderer with mobile-tuned settings
- **Physics**: Unity 2D Physics (Box2D-based; non-deterministic — see `docs/engine-reference/unity/modules/physics.md`)

## Input & Platform

<!-- Written by /setup-engine. Read by /ux-design, /ux-review, /test-setup, /team-ui, and /dev-story -->
<!-- to scope interaction specs, test helpers, and implementation to the correct input methods. -->

- **Target Platforms**: Mobile (iOS + Android), portrait orientation
- **Input Methods**: Touch
- **Primary Input**: Touch
- **Gamepad Support**: None
- **Touch Support**: Full
- **Platform Notes**: Finger occlusion must be accounted for in all UI and gameplay interactions (tool spawn offset above finger, ~1.4× hitbox enlargement relative to visual). Safe-area API handling across aspect ratios 19.5:9 to 21:9. Performance target is low-end Android (Mali-G52-class GPU). Touch hit zones must respect a minimum ~44pt (iOS) / ~48dp (Android) accessibility floor.

## Naming Conventions

- **Classes**: PascalCase (e.g., `PlayerController`)
- **Public fields / properties**: PascalCase (e.g., `MoveSpeed`, `JumpVelocity`)
- **Private / protected fields**: `_camelCase` (e.g., `_moveSpeed`, `_currentHealth`)
- **Methods**: PascalCase (e.g., `TakeDamage()`, `GetCurrentHealth()`)
- **Events / delegates**: PascalCase, present-tense for actions, past-tense for notifications (e.g., `OnHealthChanged`, `HealthChanged`)
- **Files**: PascalCase matching the primary class (e.g., `PlayerController.cs`)
- **Scenes / prefabs**: PascalCase matching their purpose (e.g., `MainMenu.unity`, `GopherHole.prefab`)
- **ScriptableObjects**: PascalCase with purpose suffix (e.g., `RampToolConfig`, `MeadowBiomeData`)
- **Constants**: UPPER_SNAKE_CASE for `const` (e.g., `MAX_HEALTH`); PascalCase for `static readonly` (e.g., `DefaultMoveSpeed`)
- **Namespaces**: `MoneyMiner.<Module>` (e.g., `MoneyMiner.Gameplay`, `MoneyMiner.UI`)

## Performance Budgets

- **Target Framerate**: 60fps on mid-tier devices; 30fps floor on low-end Android (Mali-G52 class). Soft-lock at 30fps on devices that cannot sustain 60 after detection.
- **Frame Budget**: 16.67ms (60fps) / 33.33ms (30fps)
- **Draw Calls**: ≤200 per frame on low-end Android; target ≤120 average. SRP Batcher + sprite atlas discipline mandatory.
- **Memory Ceiling**: 3GB process budget (hard ceiling); 2GB target (with headroom for OS / background apps / surface compositor)
- **Physics Bodies**: ≤40 active Rigidbody2D at peak density (object pooling mandatory)
- **Battery / Thermal**: Target ≤10% battery per 30-minute session on a 3-year-old device; no sustained thermal throttling in standard play.

## Testing

- **Framework**: Unity Test Framework (NUnit-based). PlayMode tests for integration, EditMode tests for pure logic.
- **Minimum Coverage**: TBD — to be set in architecture phase (`/test-setup`). Candidate baseline: 80% for balance/economy/physics-independent logic; playtest-and-evidence for feel/UX.
- **Required Tests**: Balance formulas, gameplay systems (tool behavior, hazard interactions, economy), scoring math, save/load integrity, shop transaction logic.

## Forbidden Patterns

<!-- Add patterns that should never appear in this project's codebase -->
- [None configured yet — add as architectural decisions are made]

## Allowed Libraries / Addons

<!-- Add approved third-party dependencies here. Do NOT pre-populate speculative dependencies. -->
<!-- Add each library only when it is actively being integrated in the current session. -->
- [None configured yet — add as dependencies are approved]

## Architecture Decisions Log

<!-- Quick reference linking to full ADRs in docs/architecture/ -->
- [No ADRs yet — use /architecture-decision to create one]

### Pre-Production ADRs Required (from TD-FEASIBILITY gate)

The following ADRs must be created before MVP implementation begins:

1. **Leaderboard Model** — commit to "treasure banked count" (non-deterministic Unity 2D physics acceptable; leaderboards do not compare physics replay scores)
2. **URP Mobile Settings** — 2D Renderer only, no HDR, no MSAA, disabled 2D shadows, single Forward renderer, SRP Batcher enabled
3. **Save Model** — local save format (JSON via Unity JsonUtility or ScriptableObject serialization); cloud save integration deferred to plumbing sprint
4. **Monetization Boundary** — IAP product catalog, ad placement rules, no-ads product; enforces anti-pillar "no pay-to-win"

## Engine Specialists

<!-- Written by /setup-engine when engine is configured. -->
<!-- Read by /code-review, /architecture-decision, /architecture-review, and team skills -->
<!-- to know which specialist to spawn for engine-specific validation. -->

- **Primary**: unity-specialist
- **Language/Code Specialist**: unity-specialist (C# review — primary covers it)
- **Shader Specialist**: unity-shader-specialist (Shader Graph, HLSL, URP/HDRP materials)
- **UI Specialist**: unity-ui-specialist (UI Toolkit UXML/USS, UGUI Canvas, runtime UI)
- **Additional Specialists**: unity-dots-specialist (ECS, Jobs system, Burst compiler), unity-addressables-specialist (asset loading, memory management, content catalogs)
- **Routing Notes**: Invoke primary for architecture and general C# code review. Invoke DOTS specialist for any ECS/Jobs/Burst code. Invoke shader specialist for rendering and visual effects. Invoke UI specialist for all interface implementation. Invoke Addressables specialist for asset management systems.

### File Extension Routing

<!-- Skills use this table to select the right specialist per file type. -->
<!-- If a row says [TO BE CONFIGURED], fall back to Primary for that file type. -->

| File Extension / Type | Specialist to Spawn |
|-----------------------|---------------------|
| Game code (.cs files) | unity-specialist |
| Shader / material files (.shader, .shadergraph, .mat) | unity-shader-specialist |
| UI / screen files (.uxml, .uss, Canvas prefabs) | unity-ui-specialist |
| Scene / prefab / level files (.unity, .prefab) | unity-specialist |
| Native extension / plugin files (.dll, native plugins) | unity-specialist |
| General architecture review | unity-specialist |
