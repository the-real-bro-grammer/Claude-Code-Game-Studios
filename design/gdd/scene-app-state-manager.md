# Scene / App State Manager

> **Status**: APPROVED (pass-6 author-led revision 2026-04-28 — pass-5 /design-review 2026-04-28 returned APPROVE-WITH-CONDITIONS; pass-6 applied all 5 BLOCKING + 16 RECOMMENDED items per pass-5 review log fix-list with mandatory unity-specialist Implementation-Contract Verification gate confirming AC-BOOT-FEPM bit-encoding correction landed CORRECTLY. Auto-promoted to APPROVED per CD pass-6 verdict rule on condition application.)
> **Author**: game-designer + unity-specialist + systems-designer + gameplay-programmer + engine-programmer + qa-lead (via /design-system); pass-2 revision via /design-review; pass-3 /design-review (5 specialists + creative-director); pass-4 author-led revision (no fresh adversarial spawn per pass-3 verification protocol); pass-5 /design-review (5 specialists + creative-director, verification-only); pass-6 author-led revision + targeted unity-specialist Implementation-Contract Verification (gate prototype per pass-5 CD strategic recommendation)
> **Last Updated**: 2026-04-28 (pass-6)
> **Last Verified**: 2026-04-28 (pass-5 /design-review + pass-6 unity-specialist targeted verification CORRECT on both AC-BOOT-FEPM claims)
> **Implements Pillar**: Protects Pillar 2 (Readable at a Glance — scene-context certainty, sortingOrder budget) + Enables Pillar 5 preconditions at scene boundaries (ensures Input is wired before first tap; feedback itself is owned by Input + HUD + VFX-Juice) + Implements Concept "Fast retry < 2 seconds" hard ceiling
> **Creative Director Review**: pass-1 CD-GDD-ALIGN APPROVED (post-condition-apply) 2026-04-24; pass-2 /design-review NEEDS REVISION → APPLIED 2026-04-27; pass-3 /design-review NEEDS REVISION 2026-04-27; pass-4 author-led revision applied 2026-04-27 — 12 BLOCKING + 4 RECOMMENDED; pass-5 /design-review APPROVE-WITH-CONDITIONS 2026-04-28 — 12/12 pass-3 BLOCKING verified landed; 5 NEW BLOCKING + 16 RECOMMENDED line-edits applied via pass-6 author-led revision 2026-04-28 (AC-BOOT-FEPM bit-encoding `2`→`1` + `[SetUp]` enforcement + 2 sub-cases per Domain/Scene Reload combinations; AC-R10c/R10d split into 6 tier-specific fixtures replacing stale `bg_session_expiry_seconds`; D.5 sums-of-ceilings 1000+500+1500+800=3800ms → 2000+500+1500+800=4800ms with re-derived 1.6× target invariant; AC-R9e ADVISORY → BLOCKING CI PLAYMODE per failure-severity asymmetry adjudication). Pass-6 mandatory unity-specialist Implementation-Contract Verification gate confirmed both AC-BOOT-FEPM claims CORRECT (bit-encoding + EditorSettings API surface). Auto-promoted to APPROVED per CD pass-6 verdict rule.

## Summary

Scene / App State Manager owns when scenes load, when scenes unload, and which high-level app state the game is in (Boot, Main Menu, Biome Map, Level, Shop, Settings overlay, Pause overlay). It coordinates the bootstrap order of the four Foundation siblings — Data Registry, Event Bus, Save/Load, Input — so downstream gameplay code always starts on a fully-initialized stack, and publishes transition events that let those siblings clean up (Event Bus `Disposed`, Input subscription teardown, Save/Load flush, Data Registry Catalog terminal) before the next scene's `Awake` runs. Without this system, every scene transition is a race condition waiting to happen.

> **Quick reference** — Layer: Foundation · Priority: MVP · Key deps: None upstream (Foundation sibling of Data Registry / Event Bus / Save/Load / Input)

## Overview

Money Miner runs as a small set of distinct screens — the boot splash, a main menu, a biome map for level selection, the level scene itself (which internally runs Prep Phase → Rush Phase → Scoring → level-end modal), an IAP/upgrade shop, and overlays for settings, pause, and modal dialogs. Something has to decide which screen is active right now, what screen comes next when the player taps "retry" or "level complete," and — critically — in what order the game's plumbing (Data Registry catalogs, Event Bus channels, Save/Load state, Input dispatcher) initializes before any scene's first frame runs. Scene / App State Manager is that decider. Players never interact with it directly; they experience it only as the absence of problems — the game doesn't freeze when they tap retry, the first tap after a loading screen isn't silently dropped, pausing mid-level doesn't destabilize in-flight physics or the save, and the Prep→Rush transition inside a level isn't a full scene reload (it's an in-scene phase flip handled by Level Runtime, not a Scene Manager event). At MVP the manager arbitrates six top-level app states (Boot, MainMenu, BiomeMap, Level, Shop, Paused) and the transitions between them; at Vertical Slice it absorbs new overlay states (Settings, Accessibility) without restructuring. It is a Foundation-layer system — it has no upstream gameplay dependencies, and five other GDDs (Input, Event Bus, Save/Load, Data Registry, plus every downstream screen) rely on it as the single source of truth for lifecycle and ordering.

### Provisional assumptions (to revisit during downstream GDD authoring)

- **Six-state MVP enumeration** (Boot, MainMenu, BiomeMap, Level, Shop, Paused) is assumed from the concept doc + systems index. Level Runtime (not yet designed) owns the Prep/Rush/Scoring sub-states *inside* the Level scene; Scene Manager does not reach inside the Level scene's internal phase machine. Shift the enumeration if Level Runtime design reveals a reason.
- **Prep→Rush is an in-scene phase flip, not a scene reload** — inferred from concept doc §Core Loop (single continuous level scene). This locks Scene Manager's scope *above* Level Runtime's internal state. Flag if Level Runtime design reveals a reason to split.

## Player Fantasy

Scene / App State Manager has no fantasy of its own. Players never choose to engage with it, never see its name, never interact with its state machine. What the section below describes is the **player experience it enables** — the emotional texture of the moments when Scene Manager is doing its job — and the matching texture of the moments it must prevent.

### Retry is a beat, not a load screen

*The highest-frequency Scene Manager touchpoint. The player meets this 5–10 times per session.*

The player loses a level — a bomb went off on hole 2, a cat slipped past the dog, the quota was 20 coins short at the buzzer. They tap **Retry**. Before the thought "okay, this time I'll buy the ramp first" finishes forming, the level is back in front of them: the shop is where they left it, their coins are correct, their loadout is preserved, and the prep-phase timer has not yet started counting down against their planning time. The heist failed. The gophers regroup before the player has finished sighing.

This is the load-bearing fantasy. The concept doc promises *"< 2 second fast retry"* — that is a **tone** target before it is a performance target. A three-second retry reads as blame. A 1.5-second retry reads as *let's go again*. Failure in Money Miner is slapstick (Pillar 1: Cute Chaos), and slapstick only lands if the recovery is instant. Scene Manager is the system that carries the momentum of the joke across the scene boundary. If that boundary adds friction, the joke dies.

### The game is already running when the player gets there

*The once-per-session touchpoint. Cold-start from a commute, a push notification, a lunch break.*

The player taps the app icon. Between the icon press and their thumb landing on the first interactable element, the app has booted, restored their save, loaded the biome map, and wired every input listener. Their first tap — the one choosing which level to replay — is heard, honored, and answered within the Feedback Floor (Pillar 5: 100 ms at 60 fps / 133 ms at 30 fps). Not a ghost tap registered during a loading transition. Not a double-tap coalesced into a wrong action because the first one was ignored. The first tap is the first interaction.

Mobile casual players have about eight seconds of patience before they task-switch to a different app. Scene Manager's job on boot is to make sure those eight seconds are spent playing, not waiting. The gophers are already waiting. They were ready before the player was.

### The seams are where the chaos isn't

*The rarest touchpoint, and the highest-stakes one. Interruption recovery.*

The player is mid-Rush. Cats are spawning. A phone call comes in and the app goes to background. Two minutes later the player returns. The game is paused — not reset, not expired, not mid-chaos with the cats having cleaned out the farm in the player's absence. They unpause. The board resumes exactly where it was: same timer, same arc trajectories in flight, same audio state. Their first post-unpause input is registered against the real game state, not against a stale frame Unity rendered before backgrounding.

This is the fantasy that protects all the others. Money Miner asks the player to commit to chaotic, reactive decisions in the Rush phase (Pillar 4: Prep + React). That commitment is only possible when the player trusts the container. The chaos lives inside the level; the transitions around it are calm. The vault is chaos; the approach to the vault is not. If the player is ever *uncertain which scene they are in*, or whether their input just went to the wrong state, Pillar 2 (Readable at a Glance) collapses — they are no longer reading the board, they are asking whether they are reading the right screen. Scene Manager is the part of the game the player never has to worry about, which is what lets them spend all their worry on bombs, cats, and escape routes.

### What this fantasy excludes

- **Not a source of juice or delight.** Scene Manager does not own loading animations, transitions, or any visual celebration of its own operation. Those belong to Main Menu / Modal / HUD / VFX-Juice GDDs. Scene Manager ships the frame-zero state; other systems decorate it.
- **Not a player-facing choice.** The player never picks an app state, never chooses a load mode, never sees a "switching scenes…" string. Every exposed decision (retry, pause, continue, back to map) is owned by the UI system that presents the button.
- **Not mastery.** No skill ceiling lives in this system. Pillar 3 (Mastery Through Economy) is unaffected by *how well* Scene Manager works, only by *whether* it works at all.

## Detailed Design

### Core Rules

**R1 — Async-Only Scene Loading Primitive.** All scene loads use `SceneManager.LoadSceneAsync(sceneName, LoadSceneMode)`. Main thread never blocks. Every call sets `asyncOp.allowSceneActivation = false` immediately after the call returns; activation is deferred until R7 (pre-unload hooks complete) and R8 (activation handshake with Modal/Dialog) both pass. The returned `AsyncOperation` is held in a private field on `AppStateManager`; concurrent requests are handled per R12.

**R2 — BootLoader as Single Foundation Construction Authority.** The game's first scene (Build Settings index 0) is `Boot.unity`, containing only a `BootLoader` MonoBehaviour. `BootLoader.Awake()` synchronously initializes Foundation siblings in strict sequence: (1) `DataRegistry.Initialize()`, (2) `EventBus.Initialize()`, (3) `SaveManager.Initialize()`, (4) `InputDispatcher.Initialize()`. Each sibling exposes `Task<bool> Ready`; `BootLoader` awaits each `Ready` before moving to the next. Only after all four complete does `BootLoader` call `BeginTransitionAsync("MainMenu")` (R6). No Foundation sibling's public API may be called before its predecessor's `Ready` has fired.

**R2.a — Foundation `Reset()` Discipline (Fast Enter Play Mode safety).** Every Foundation sibling exposes a `Reset()` method that clears all static fields, unsubscribes PlayerLoop injections, and returns the sibling to pre-init state. `BootLoader.Awake()` calls `Reset()` on every Foundation sibling unconditionally BEFORE the R2 init chain begins. This guarantees correctness when Unity's Fast Enter Play Mode is used with domain reload disabled (static state would otherwise persist across Play/Stop cycles and double-initialize). CI includes a "play twice without domain reload" smoke test (AC-BOOT-FEPM — see Section H).

**R2.b — `allowSceneActivation` Awake Restriction (Unity 6 API constraint).** Per Unity 6 documentation: "You cannot set and use `AsyncOperation.allowSceneActivation` within `Awake`, because this function is not a coroutine." `BootLoader.Awake()` and any code path originating synchronously from a MonoBehaviour `Awake` MUST NOT set `asyncOp.allowSceneActivation`. Scene transitions are initiated via `BeginTransitionAsync` (R6), which is an `async Task` method invoked AFTER `Awake` returns — this is safe. **Real failure mode if violated (pass-3 BLOCKING #9 rationale rewrite):** the `AsyncOperation` returned by `LoadSceneAsync` cannot be safely held in a static field across Fast Enter Play Mode cycles when domain reload is disabled — the second FEPM cycle reuses the stale AsyncOperation reference whose underlying native object has been invalidated, triggering `MissingReferenceException` or undefined activation behavior. The pass-2 "silently ignored on second FEPM cycle" claim was incorrect (the API is not silent — it produces undefined results); the prescription (defer all `allowSceneActivation` writes to post-Awake `async Task` paths) remains correct because `BeginTransitionAsync` always allocates a fresh `AsyncOperation` per call rather than reusing one across cycles. The `Reset()` discipline (R2.a) backstops this by clearing any static-field references during `BootLoader.Awake()` before R2 runs. Reference: [`AsyncOperation-allowSceneActivation`](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/AsyncOperation-allowSceneActivation.html).

**R3 — `RuntimeInitializeOnLoadMethod` Usage Restriction.** `[RuntimeInitializeOnLoadMethod]` in any Foundation sibling is restricted to: (a) `SubsystemRegistration` tier — static field reset only, must be idempotent; (b) `AfterAssembliesLoaded` tier — PlayerLoop injection only (Event Bus R2.b uses this); (c) no other tier. RIOLM must NOT instantiate MonoBehaviours, load scenes, construct singletons, or touch any state `BootLoader` also initializes. Violation is a code-review BLOCKING issue. This guarantees that `BootLoader.Awake()` is the single place Foundation state comes into existence.

**R4 — App-State Machine Ownership.** `AppStateManager` (persistent `DontDestroyOnLoad` MonoBehaviour, single-instance guarded, instantiated by `BootLoader`) is the single source of truth for `AppState`. MVP enum: `Boot`, `Loading`, `MainMenu`, `BiomeMap`, `Level`, `Shop`, `Disposed`. `Loading` is a transient sub-state entered the moment `BeginTransitionAsync` starts and exited the moment the activation handshake (R8) completes. `Paused` is NOT a separate state; it is a boolean flag `AppStateManager.IsPaused` layered on top of the current scene-state (e.g., `AppState = Level` + `IsPaused = true`). `AppStateManager` is the sole writer of both `AppState` and `IsPaused`; all other systems read.

**R5 — Transition Event Taxonomy (Three Channels).** Event Bus, Gameplay namespace, T2 tier:

- **`gameplay.scene.will_unload`** — fires immediately when `BeginTransitionAsync` is called, BEFORE any unload work. Payload: `{ Scene fromScene, Scene toScene, float timestamp_realtime }`. Subscribers: Foundation siblings via `IPreUnloadHook` (R7); analytics.
- **`gameplay.scene.load_ready`** — fires when new scene's `AsyncOperation.progress >= 0.9f` (NOT exact equality — Unity progress can jump 0.85→0.95 in one frame; see Unity 6 [`allowSceneActivation` docs](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/AsyncOperation-allowSceneActivation.html)), BEFORE `allowSceneActivation = true`. Payload: `{ Scene loadingScene, AppState targetState, Action<bool> activationCallback }`. **Single-subscriber enforcement via Event Bus `ExclusiveSubscribe` facet** (Phase 5 Event Bus bidirectional amendment): subscribers register via `EventBus.ExclusiveSubscribe("gameplay.scene.load_ready", handler)` which throws `InvalidOperationException` on second registration in the same Event Bus dispatch context. Modal/Dialog GDD owns the single production subscriber; test scenes that need to override call `EventBus.UnsubscribeAll(channel)` before re-subscribing. If zero subscribers ack within 500 ms, Scene Manager proceeds with activation anyway (test-scene fallback per D.4).
- **`gameplay.app.state_changed`** — fires AFTER `sceneLoaded` confirms the new scene is active AND `AppState` has been updated. Payload: `{ AppState previous, AppState next, bool isPaused, float timestamp_realtime }`. Subscribers: HUD, Audio Bus, analytics — any system reacting to "transition complete." This is the public post-transition notification; it never fires during the transition window.

**R6 — `BeginTransitionAsync` Is the Only Legal Scene-Change Entry Point.** No gameplay or UI code may call `SceneManager.LoadSceneAsync` / `UnloadSceneAsync` directly. The single public method `AppStateManager.BeginTransitionAsync(string targetScene, LoadSceneMode mode)` performs the full sequence: fire `will_unload`, await `Task.WhenAll` of `IPreUnloadHook` handlers (R7), call `LoadSceneAsync` with `allowSceneActivation = false`, await progress until `>= 0.9f` (NOT exact equality — IEEE 754 float; Unity progress can step 0.85→0.95 in one frame), fire `load_ready`, await activation ack (R8), set `allowSceneActivation = true`, await `sceneLoaded`, update `AppState`, fire `app.state_changed`.

**Asset reclamation ordering (pass-3 BLOCKING #8 — explicit `await` + ordering):** Unity 6's auto-reclaim contract is mode-dependent:

- **`LoadSceneMode.Single`** (e.g., MainMenu → Level transition): Unity automatically calls `Resources.UnloadUnusedAssets()` AFTER the new scene's `Awake` AND BEFORE `sceneLoaded` fires. Scene Manager MUST NOT add a redundant call here — doubling it adds 50–200 ms to the critical path with zero benefit. Confirmed in Unity 6 [`SceneManager.LoadSceneAsync` docs](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/SceneManagement.SceneManager.LoadSceneAsync.html): "When using LoadSceneMode.Single, Unity unloads the previous scene and unused assets are released."
- **`LoadSceneMode.Additive`** unload (the only additive case at MVP is `PauseOverlay` — see R9): Unity does NOT auto-reclaim. Scene Manager MUST `await Resources.UnloadUnusedAssets()` AFTER `await SceneManager.UnloadSceneAsync(...)` completes. Skipping this leaks the overlay's textures/materials/audio clips on every pause/unpause cycle. R9 owns this call site (see R9 step 3 of `RequestUnpause`).

The reclaim call appears in **R9** (additive overlay unload), NOT in `BeginTransitionAsync` itself (which is Single-mode for all MVP top-level transitions). The `Resources.UnloadUnusedAssets()` call MUST be awaited (`await`-on-the-`AsyncOperation`-it-returns), not fire-and-forget — Unity 6 returns an `AsyncOperation` and unloading completes asynchronously. Pass-2 R6 prose erroneously placed `Resources.UnloadUnusedAssets()` inside `BeginTransitionAsync`'s sequence; pass-3 BLOCKING #8 corrects this to additive-only.

Enforcement: CI source-grep rule fails build if any direct `SceneManager.LoadSceneAsync` call exists outside `AppStateManager.cs`.

**R7 — Pre-Unload Hook Contract (Typed Interface).** Foundation siblings needing ordered teardown implement:

```csharp
public interface IPreUnloadHook {
    Task OnBeforeUnloadAsync(Scene fromScene, CancellationToken ct);
    int Priority { get; }  // 0..100, lower runs first
}
```

Handlers register at boot via `AppStateManager.RegisterPreUnloadHandler(IPreUnloadHook)`. `BeginTransitionAsync` invokes `Task.WhenAll(handlers.OrderBy(h => h.Priority).Select(h => h.OnBeforeUnloadAsync(from, ct)))` with a 1000 ms timeout. Timeout → cancellation token fires, Scene Manager proceeds with the unload anyway and logs a BLOCKING error in `DEVELOPMENT_BUILD`. MVP priorities: Event Bus = 10 (flush deferred queue + transition to `Disposed`); Save/Load = 20 (`RequestFlush` fire-and-forget on retry path per R11); Input = 30 (disable action maps + unsubscribe). All handlers use `async Task`, never `WaitForSeconds` coroutines — the latter would stall under `Time.timeScale = 0` (e.g., transition from a paused state).

**Async-contract constraints (pass-3 BLOCKING #5).** `IPreUnloadHook.OnBeforeUnloadAsync` implementations:

- **MUST NOT use `ConfigureAwait(false)`** — Unity's main-thread context is required to safely touch `JsonUtility`, `SceneManager`, `Resources`, action maps, and any `MonoBehaviour`-scoped state. `ConfigureAwait(false)` would resume the continuation on a thread-pool thread and risk `UnityException: get_isPlaying can only be called from the main thread`. Code-review enforcement: CI grep flags `ConfigureAwait(false)` inside any class implementing `IPreUnloadHook`.
- **MUST NOT use `Task.Run` to dispatch work to a background thread for Unity-API operations.** Save/Load is the explicit exception — its background-thread JSON serialization is by design and isolated to its own thread; the public-facing `RequestFlush` returns to the main-thread caller. Other Foundation siblings (Event Bus dispatch, Input action map teardown) MUST execute synchronously on the main thread inside `OnBeforeUnloadAsync` — `await Task.Yield()` is permitted to give the engine a frame slice but does not change the execution context.
- **Failure mode if violated:** `MissingReferenceException` or thread-affinity exceptions during teardown — typically reproducible only under load and easy to miss in QA. The constraint is documented here so each Foundation sibling's bidirectional amendment can mirror it in its own GDD's R-rules.

**R8 — Activation Handshake with Modal/Dialog (Cross-GDD Contract).** The `load_ready` event's payload carries an `Action<bool> activationCallback`. Modal/Dialog subscribes to `load_ready`, begins overlay fade-out, and invokes `activationCallback(true)` when the fade-out completes. Scene Manager awaits this callback (500 ms timeout fallback) before setting `allowSceneActivation = true`. This is a NAMED cross-GDD contract — Modal/Dialog GDD (undesigned) must implement exactly one subscriber in production builds; test scenes proceed via the timeout fallback.

**R9 — Pause Overlay Lifecycle.** `AppStateManager.RequestPause()`:

1. If `IsPaused == true` → no-op (double-pause guard).
2. If `SceneManager.GetSceneByName("PauseOverlay").isLoaded == true` → no-op (concurrent-load guard).
3. **ATOMIC: Set `IsPaused = true` AND `Time.timeScale = 0f` together (same statement, no awaits between).** This freezes gameplay at the moment the player taps pause — eliminating the 100-200ms phantom-gameplay window of pass-1 that violated the Rush-phase pause contract. Player perception: "pause is instant." Single-frame frozen-board visual gap (16ms at 60fps) is the correct expression of "game is paused now"; the overlay loads on top of the already-frozen board.
4. `SceneManager.LoadSceneAsync("PauseOverlay", LoadSceneMode.Additive)`; await `sceneLoaded`. (Time.timeScale = 0 does NOT block scene loading; `LoadSceneAsync` runs on Unity's loading subsystem independent of game-time scaling.)
5. Fire `gameplay.app.state_changed { isPaused: true }` AFTER overlay's `sceneLoaded` confirms the visual is up. Subscribers (HUD dim, Audio Bus fade) react in their natural cadence.

`RequestUnpause()` (pass-3 BLOCKING #7 + #11 — explicit guard step 1; explicit reclaim step 3):

1. **If `IsPaused == false` → no-op (double-unpause guard, mirrors `RequestPause` step 1).** Pass-3 BLOCKING #7: pass-2 prose stated this guard but did not number it; making it numbered step 1 establishes parity with `RequestPause` and gives AC-R9e a concrete step to verify.
2. `await SceneManager.UnloadSceneAsync("PauseOverlay")`.
3. **`await Resources.UnloadUnusedAssets()` (pass-3 BLOCKING #11 — memory cycle stability).** Pause overlay assets (textures, materials, audio clips for the pause UI) are NOT auto-reclaimed by additive scene unload (Unity 6 contract — see R6). Without this step, every pause/unpause cycle leaks the overlay's working set; 10 cycles in a typical 30-min Rush session can accumulate 5–20 MB of orphaned references. Verified by AC-R9f (new pass-3 AC).
4. **ATOMIC: Set `Time.timeScale = 1f` AND `IsPaused = false` together.** Mirror of `RequestPause` step 3 — gameplay resumes only once the overlay is gone AND its assets reclaimed.
5. Fire `gameplay.app.state_changed { isPaused: false }`.

Pause overlay scene constraints (cross-GDD, enforced in Main Menu / Settings / Pause GDD + CI): (a) MUST NOT contain an `EventSystem` component — uses the persistent Boot-scene EventSystem per R14.e; (b) `Canvas.sortingOrder ∈ [30, 39]` per the project sort-order budget (R14.g) — this cross-references Input GDD AC-EC-HT2's duplicate-`sortingOrder` dev validator; (c) MUST NOT use `DontDestroyOnLoad` — that authority is exclusively `BootLoader`'s.

**R10 — OS Background Forced-Pause Path.** `AppStateManager` implements Unity's `OnApplicationPause(bool paused)` MonoBehaviour callback. When `paused == true`:

1. Call `SaveManager.FlushSync()` — synchronous; same code path as `Application.quitting`. If the OS terminates the process, the save persisted.
2. **Capture `_userPausedBeforeBackground = IsPaused` BEFORE mutating `IsPaused`** (pass-3 BLOCKING #1 fix — pass-2 step 5 silently cleared user pause on resume; capture-then-restore is the surgical fix).
3. Set `IsPaused = true`.
4. Set `AudioListener.pause = true` (audio mute on OS background; NOT during in-game pause — Audio Bus owns in-game audio pause separately via `gameplay.app.state_changed`).
5. Record `_backgroundedAtUtc = DateTime.UtcNow` AND `_backgroundedDuringRush = (AppState == Level && LevelRuntime.IsInRushPhase)` AND `_backgroundedDuringPrep = (AppState == Level && !LevelRuntime.IsInRushPhase)` (the booleans are read via Level Runtime's phase getter; both `false` if Level Runtime is not yet authored / scene is not Level). Pass-3 BLOCKING #13 introduces `_backgroundedDuringPrep` to support the third-tier threshold.
6. NO overlay load (player isn't looking at screen).
7. NO `gameplay.app.state_changed` fire (Event Bus may already be torn down; subscribers could not execute meaningfully).

When `paused == false` (resume) — **three-tier threshold per pass-3 fantasy correction (Rush / Prep / general):**

1. `AudioListener.pause = false` (unconditional, BEFORE the threshold branch — pass-3 REC-3: ensures audio is restored even when the resume path branches into `BeginTransitionAsync` and the post-branch `else` would otherwise be skipped).
2. Compute `elapsed = (DateTime.UtcNow - _backgroundedAtUtc).TotalSeconds` (clamp to 0 if negative per E12).
3. Determine threshold (three-way):
   - If `_backgroundedDuringRush` → `bg_expiry_in_rush_seconds` (default **1800s / 30 min**) — protects mid-Rush continuity.
   - Else if `_backgroundedDuringPrep` → `bg_expiry_in_prep_seconds` (default **600s / 10 min**) — Prep Phase planning state has real but lower value than mid-Rush; pass-3 BLOCKING #13 (CD adjudicated game-designer "1800s general" vs systems-designer "third tier" → third tier wins for surgical scope).
   - Else (MainMenu, BiomeMap, Shop) → `bg_expiry_general_seconds` (default **60s**) — eject quickly from non-gameplay contexts.
4. If `elapsed > threshold` → `BeginTransitionAsync("MainMenu")` — treat as fresh session return. (Note: `_userPausedBeforeBackground` is implicitly discarded; the new MainMenu session is unpaused.)
5. Else → set `IsPaused = _userPausedBeforeBackground` (pass-3 BLOCKING #1: restores the user's pre-background pause state — if the user paused before the OS backgrounded them, they remain paused on resume; if they were unpaused, resume runs in place). Player sees mid-Rush/Prep board exactly as they left it, with their pause state preserved.

**Rationale (pass-3 BLOCKING #1 + #13 resolution).** Pass-2's step 5 unconditional `IsPaused = false` silently cleared user pause on resume — a Section B "seams are where the chaos isn't" violation: the player paused, the phone rang, two minutes later their pause was gone. `_userPausedBeforeBackground` capture-then-restore preserves the user's intent across the OS-background round trip. Pass-3 also adds a third-tier `bg_expiry_in_prep_seconds` (600s default) so the Prep Phase — where the player has invested real planning time configuring tools, even though Rush hasn't started — receives proportionate continuity protection. The 60s general threshold ejected Prep planners on any phone call > 1 min; 1800s would have been overkill for non-Rush state. 600s is the surgical middle ground.

**Rationale (pass-2 BLOCK-1 carry-forward).** The pass-1 single 300s threshold ejected players from mid-Rush after any phone call exceeding 5 minutes — directly violating Section B's "seams are where the chaos isn't" fantasy. Anti-stale safeguard: 30 min upper bound on `bg_expiry_in_rush_seconds` preserves the "session-freshness" fantasy at the long tail.

**iOS phone-call path (pass-3 BLOCKING #4).** On iOS, the phone-call interruption fires `OnApplicationFocus(false)` (loss of focus to the phone-call modal) WITHOUT firing `OnApplicationPause(true)` for short interruptions — the app remains foregrounded behind the system phone UI. To honor the Section B "phone call comes in" fantasy on iOS, `AppStateManager` ALSO implements `OnApplicationFocus(bool hasFocus)`. When `hasFocus == false` on iOS (gated by `Application.platform == RuntimePlatform.IPhonePlayer`):
1. Call `SaveManager.FlushSync()` (defensive — short interruption may still escalate to background).
2. Set `AudioListener.pause = true` (mute audio for the phone-call duration).
3. Do NOT set `IsPaused = true` (the OS-modal phone UI does not represent a player-intended pause; if the call ends quickly, Unity continues rendering and we want gameplay to resume seamlessly without a pause-overlay round trip).
4. Do NOT record `_backgroundedAtUtc` (this is not a session-expiry-tracking event).

When `hasFocus == true` returns: set `AudioListener.pause = false`. No state restoration is needed because no state was changed beyond audio.

Android does not need this path — `OnApplicationPause(true)` fires reliably on phone-call. The path is iOS-only by platform check. Tracked as **OQ-21** (verify behavior across iOS focus-loss scenarios — phone call, banner notification, Control Center swipe — at first iOS device test cycle).

OS-background path is NOT `RequestPause()`. An OS background while the player was already in `RequestPause()` re-enters the forced-pause path on top; resume restores both states in order via the `_userPausedBeforeBackground` capture (step 2 above) — the user pause persists.

**R11 — Retry Performance Contract (< 2 sec, Mali-G52).** Level → Level retry on the ADR-003 reference Android device must complete within **2000 ms** wall-clock time from retry-button tap to first-interactive-frame of the new Level scene. Budget split (warm-Addressables-cache target):

- **Scene Manager: 1100 ms** — Level → Level retry uses `LoadSceneMode.Single`, so Unity auto-reclaims the outgoing Level scene's assets between `Awake` of the new Level and `sceneLoaded` firing (no explicit `Resources.UnloadUnusedAssets()` call needed — pass-3 BLOCKING #8 corrects pass-2's redundant manual call). Budget covers: scene unload (Unity-internal) + `LoadSceneAsync` activation + pre-unload hook completion. Additive `PauseOverlay` unload (R9) is a separate code path that DOES require an explicit `await Resources.UnloadUnusedAssets()` and is budgeted under R9, not R11.
- **Level Runtime (downstream, undesigned): 600 ms** — LevelData populate from Data Registry catalogs + first frame ready. Note: 600 ms is a residual allocation (2000 - 1100 - 300), not an evidence-based measurement; Level Runtime GDD authoring may surface a need to re-balance.
- **Shared slack: 300 ms** — Mali-G52 thermal variance, Addressables cold-cache, GC spikes.

**Cold-cache reality (pass-2 perf-analyst R-1; pass-6 R4 — distinct-severity advisory).** The 1100 ms `T_sm_budget` assumes warm Addressables cache. On Mali-G52 cold cache, `LoadSceneAsync` alone may consume 800-1000 ms; combined with unload + handshake + pre-unload hooks, raw `T_sm` may reach 1300-1500 ms. The 2000 ms `T_retry_max` is the real gate; `T_slack_consumed` (D.1) absorbs up to 300 ms of `T_sm` overrun. AC-R11 measures end-to-end `T_retry`, not `T_sm` in isolation. **Distinct-severity advisory log (pass-6 R4):** when `T_sm_overrun >= T_slack_budget` (i.e., Scene Manager alone consumes 100% or more of the shared slack — even if `T_retry` still passes via `T_lr` running well under its 600 ms budget), Budget Analyzer logs `[SceneManager] T_sm_overrun {ms}ms ≥ T_slack_budget — Scene Manager is consuming entire shared slack pool; cold-cache mitigation may be required` as an ADVISORY (distinct severity from the BLOCKING `T_slack_consumed > T_slack_budget` gate failure). This catches the silent regression where `T_sm` is steadily climbing while `T_lr` runs faster than budgeted — the gate stays green, but the system has burned through its margin against any subsequent `T_lr` regression. Surfaces budget-consumption-trend before the gate fails.

Measured as a DEVICE acceptance criterion (AC-P-Retry, Section H) against ADR-003's named reference device (which does not yet exist — see Section F Dependencies). Save/Load flush on the retry path is **fire-and-forget** — Scene Manager fires `RequestFlush` and immediately proceeds to scene unload; Save/Load completes the flush on its background thread concurrently. Awaiting the flush would add 50–150 ms to the critical path per gameplay-programmer analysis. This fire-and-forget contract is a cross-GDD obligation — Save/Load GDD's retry-path behavior must not assume Scene Manager awaits, AND must guarantee that public read APIs are safe to call on the new scene's `Awake` while a background flush is in flight (read-during-write race prevention; Save/Load owns this contract per AC-INT-SL).

**R12 — Re-Entrancy and Interruption.** `AppStateManager` maintains `_transitionInProgress` (set on first `BeginTransitionAsync` call, cleared on `app.state_changed` fire). While `true`:

- Any `BeginTransitionAsync` call is placed in a single-slot `_pendingTransition` queue; any prior queued entry is overwritten and a `Debug.LogWarning` fires in `DEVELOPMENT_BUILD`. On completion, the pending entry (if any) fires immediately.
- `RequestPause()` during a transition is placed in a single-slot `_pendingPause` queue; fires after the destination scene is active. This guarantees no `Loading → Paused` transition (prevented by R14.c).
- **Firing order when both queues are populated at transition completion (pass-2 R2 specification):** `_pendingTransition` fires FIRST; `_pendingPause` defers to the subsequent transition's completion (it queues against the new transition cycle). Rationale: pause-overlay-against-the-OLD-scene immediately followed by unloading-the-OLD-scene would orphan the additive overlay against the loading screen, violating R14.c. Verified by AC-R12c.
- **Destination-scene permission guard for `_pendingPause` (pass-3 BLOCKING #10):** when the in-flight transition completes and `_pendingPause` is non-empty, BEFORE firing it, check the new `AppState`. At MVP, pause is only valid in `AppState == Level` (E7 already prohibits pause during Boot; same logic extends to MainMenu, BiomeMap, Shop where there is no pausable gameplay state). If the destination state does NOT permit pause: discard `_pendingPause`, log `[SceneManager] _pendingPause discarded — destination AppState {newState} does not permit pause` as ADVISORY in DEVELOPMENT_BUILD, no error thrown. Pass-2 prose did not specify this guard; without it, a player who tapped pause while transitioning to MainMenu would see pause-overlay try to layer onto MainMenu, where it has no defined behavior. Verified by AC-R12d (new pass-3 AC, see Section H known coverage gaps).
- Double-tap on the retry button consequently retries exactly once, not twice.
- `RequestPause()` while `IsPaused == true`: no-op (R9.1). `RequestUnpause()` while `IsPaused == false`: no-op (R9 RequestUnpause step 1).

**R13 — App-Quit Contract.** `Application.quitting` triggers `AppStateManager.OnApplicationQuit()`, executing teardown via Foundation siblings' **synchronous teardown methods** (NOT the async `IPreUnloadHook` path used at R7 scene-transitions). **Pass-3 BLOCKING #5 contract:** the Editor quit path cannot `await` or `Task.Wait()` on async hooks — domain reload after Editor-quit can interleave with awaited tasks that touch managed Unity objects, throwing `InvalidOperationException` on the stalled domain. Player builds COULD theoretically `Task.Wait()` because no more frames will render, but a uniform sync path is simpler and more auditable. Therefore each Foundation sibling MUST expose a synchronous teardown method callable from `OnApplicationQuit()`:

- **Save/Load**: `SaveManager.FlushSync()` (already specified) — synchronous JSON write to disk.
- **Event Bus**: `EventBus.DisposeSync()` — synchronous deferred-queue drain + `Disposed` state transition. **Cross-GDD obligation (Phase 5 Event Bus bidirectional amendment):** Event Bus GDD must add `DisposeSync()` alongside the existing async `OnBeforeUnloadAsync` to support the quit path.
- **Input**: `InputDispatcher.TeardownSync()` — synchronous action-map disable + subscription unhook. **Cross-GDD obligation (Phase 5 Input bidirectional amendment):** Input GDD must add `TeardownSync()` alongside the async pre-unload path.

Editor quit path (`#if UNITY_EDITOR`) calls these sync methods in priority order (10 → 20 → 30). Player quit path uses the same sync methods for uniformity. Implementation does NOT branch on `#if UNITY_EDITOR / #else / #endif` for the teardown method calls themselves; only the surrounding `Time.timeScale = 1f; AudioListener.pause = false; AppState = Disposed` cleanup is shared. After all sync teardowns complete, `AppState = Disposed`. Note: `Application.quitting` is NOT reliable under iOS memory pressure or Android force-kill per engine-programmer §5 — R10's `OnApplicationPause(true)` FlushSync is the real save-safety net on mobile.

**R14 — Invariants.**

- **(a)** At most one non-additive scene active at any time. `PauseOverlay` is the only permitted additive scene at MVP; Vertical Slice may add Settings / Accessibility overlays following the same pattern.
- **(b)** No gameplay MonoBehaviour may execute Foundation-dependent logic in `Awake` or `Start` before `AppStateManager.FoundationReady == true`. Consumers guard with the flag or rely on `sceneLoaded` ordering (since activation handshake R8 guarantees Foundation is ready before activation).
- **(c)** `AppState` never enters a `Loading + user-paused-via-RequestPause` composite. Note that `Paused` is NOT in the `AppState` enum (R4) — it is a flag layered on top, so this invariant constrains *flag composition*, not enum transitions. `RequestPause()` calls during Loading queue to `_pendingPause` per R12 and fire after activation completes. Exception: `OnApplicationPause(true)` (OS-forced background) during Loading lawfully sets `IsPaused = true` per R10 step 2 (the OS-background path is NOT a user pause; it is a forced-pause safety flag). On resume, `IsPaused` is cleared per R10's two-tier threshold logic regardless of the in-progress transition state. E9 walks the composite-state cleanup path.
- **(d)** `Time.timeScale` is written exclusively by `AppStateManager`. Any other call-site `Time.timeScale = X` is a code-review BLOCKING issue.
- **(e)** `EventSystem` lives exclusively in the Boot scene's `DontDestroyOnLoad` hierarchy. No other scene (including `PauseOverlay`) may contain an `EventSystem` component. **Pass-3 BLOCKING #12: the Boot scene's `EventSystem` MUST use `InputSystemUIInputModule` (the new Input System UI gateway); `StandaloneInputModule` (the legacy module) MUST NOT be present on the same `EventSystem` GameObject.** Mixed-module setup silently drops UI events because the two modules race to consume input — the project pins the new Input System (per technical-preferences.md), and `InputSystemUIInputModule` is the only valid UI input module. CI scene-validation script enforces both invariants — single `EventSystem` instance count AND module-type assertion (AC-INV2 condition (d), Section H).
- **(f)** `AudioListener.pause` is written by Scene Manager ONLY on `OnApplicationPause` (R10). In-game pause does NOT touch `AudioListener.pause`; Audio Bus owns in-game audio pause by subscribing to `gameplay.app.state_changed` and fading individual audio sources.
- **(g)** Canvas `sortingOrder` budget project-wide: base gameplay 0–9, HUD 10–19, modals 20–29, pause overlay 30–39, system overlays 40+. Enforced by dev-build scene validator (builds on Input GDD AC-EC-HT2's duplicate-`sortingOrder` BLOCKING-in-dev validator).

### States and Transitions

| State | Entry Condition | Exit Condition | Behavior | Allowed Transitions |
|-------|-----------------|----------------|----------|---------------------|
| **Boot** | Application launch; `BootLoader.Awake()` runs; `Reset()` + R2 init chain in progress | All four Foundation siblings reach `Ready`; `BeginTransitionAsync("MainMenu")` called | No scene-change events published (Event Bus not yet dispatching own-transition events); Foundation init is synchronous here | → Loading |
| **Loading** | `BeginTransitionAsync` called for any destination | `sceneLoaded` confirms destination active AND activation handshake (R8) complete AND `app.state_changed` fired | Loading overlay visible (owned by Modal/Dialog); `allowSceneActivation = false`; pre-unload hooks running; `will_unload` + `load_ready` published here; gameplay input disabled (overlay may still receive input via Boot-scene `EventSystem` per R14.e) | → MainMenu / BiomeMap / Level / Shop |
| **MainMenu** | `sceneLoaded` confirms `MainMenu.unity` active; handshake complete | Player navigates → BiomeMap / Shop | Standard scene; Input active; Save state readable; Data Registry catalogs consumed | → Loading (→ BiomeMap, Shop) |
| **BiomeMap** | `sceneLoaded` confirms `BiomeMap.unity` active | Player selects a level (→ Level) or returns (→ MainMenu) | Level-select, biome pan; Data Registry catalogs readable per AC-C10 | → Loading (→ Level, MainMenu) |
| **Level** | `sceneLoaded` confirms `Level.unity` active; LevelData populated within R11's 600 ms sub-budget | Level complete / failed / retry / exit — each triggers `BeginTransitionAsync` | Gameplay runs; `Time.timeScale = 1f` unless `IsPaused == true`; T1/T2 budgets enforced per Event Bus + Input; first tap post-`sceneLoaded` honors `t2_latency_max = 100 ms` (60 fps) / 133 ms (30 fps) | → Loading (→ Level retry, BiomeMap, MainMenu); `IsPaused` flag may toggle (R9) |
| **Shop** | `sceneLoaded` confirms `Shop.unity` active | Purchase complete or exit | Currency / upgrade transactions; Save writes triggered by `gameplay.currency.*` / `gameplay.tool.*` per Save/Load contract | → Loading (→ MainMenu) |
| **Disposed** | `Application.quitting` + R13 teardown complete | Terminal — no exit | All Foundation siblings torn down; `Time.timeScale = 1f`; `AudioListener.pause = false`; process terminates | (none) |

**Sub-flags (modify the active state, not a separate state):**

| Flag | Set By | Cleared By | Behavior |
|------|--------|------------|----------|
| `IsPaused` (in-game pause) | `RequestPause()` from Pause UI (MVP: Level-only; VS: Shop / BiomeMap eligible) | `RequestUnpause()` from Pause UI | PauseOverlay loaded additively; `Time.timeScale = 0`; `AudioListener.pause` UNCHANGED (Audio Bus handles in-game audio pause); `gameplay.app.state_changed` fired |
| `_backgroundedAtUtc != default` (OS background) | `OnApplicationPause(true)` per R10 | `OnApplicationPause(false)` resume | `FlushSync`; `AudioListener.pause = true`; no overlay; resume threshold check against three-tier `bg_expiry_*_seconds`; `_userPausedBeforeBackground` captured at step 2 and restored on in-place resume per R10 step 5 (pass-3 BLOCKING #1) |
| `_userPausedBeforeBackground` (OS-background user-pause snapshot) | `OnApplicationPause(true)` step 2 — captures `IsPaused` BEFORE OS-background mutation | `OnApplicationPause(false)` step 5 — value is read once to restore `IsPaused`, then logically cleared (next OS-background recaptures fresh) | Pass-3 BLOCKING #1: ensures user pause is preserved across OS round trip rather than silently cleared on resume |

### Interactions with Other Systems

| System | Data Flow In (→ Scene Manager) | Data Flow Out (Scene Manager →) | Ownership of Interface |
|--------|-------------------------------|--------------------------------|------------------------|
| **Input System** | `Initialize()` hooks `SceneManager.sceneLoaded` to populate `DeviceDisplayMetrics`; registers as `IPreUnloadHook` priority 30 (disable action maps + unsubscribe) | Scene Manager guarantees the activation handshake (R8) completes before `allowSceneActivation = true` — enforces Input EC-CFG / AC-EC-CFG "query-before-Initialize" prohibition at scene boundaries | Unity owns `sceneLoaded`; Scene Manager owns activation gate; Input owns `IPreUnloadHook` + `Initialize` contract |
| **Event Bus** | Registers as `IPreUnloadHook` priority 10 (flush deferred queue, transition to `Disposed`); `ScriptableObject.OnEnable` resets bootstrap buffer on every scene load (Event Bus R8.a); mid-dispatch unload handling per Event Bus AC-EC7 | Scene Manager publishes `gameplay.scene.will_unload`, `gameplay.scene.load_ready`, `gameplay.app.state_changed`; `Application.quitting` triggers bus `Dispose()` via R13 | Event Bus owns dispatch + pre-unload impl + bootstrap buffer reset; Scene Manager is publisher; activation handshake is Scene Manager contract |
| **Save/Load** | Registers as `IPreUnloadHook` priority 20 (`RequestFlush` fire-and-forget on retry per R11; `FlushSync` on quit/OS-background); `OnDestroy` `SemaphoreSlim` disposal per AC-EC-Race3 must complete within pre-unload window | Scene Manager calls `FlushSync()` in R10 (OS background) and R13 (app quit); does NOT await flush on retry (fire-and-forget) | Save/Load owns flush APIs (`RequestFlush` async, `FlushSync` sync); Scene Manager owns when each is called; Save/Load GDD must document that Scene Manager does NOT await on retry |
| **Data Registry** | Catalog SO `Loaded → Consumed → terminal` uses scene unload as terminal trigger; AC-C1 + AC-C10 guarantee Level / HUD catalogs populated at `sceneLoaded` frame | Scene Manager's `UnloadSceneAsync` completion signals Catalog termination via Unity's `sceneUnloaded` callback | Data Registry owns Catalog lifecycle; Scene Manager owns unload sequencing; Data Registry intentionally has NO pre-unload hook (Catalog cleanup is terminal-only) |
| **Modal/Dialog** (downstream, undesigned — **PROVISIONAL**) | Subscribes to `gameplay.scene.load_ready`; owns loading overlay; invokes `activationCallback(true)` when fade-out ready | Scene Manager publishes `load_ready`; awaits callback with 500 ms timeout fallback | Scene Manager owns handshake protocol; Modal/Dialog owns overlay visual + callback invocation; cross-GDD contract NAMED here, to be absorbed into Modal/Dialog GDD |
| **Level Runtime** (downstream, undesigned — **PROVISIONAL**) | Subscribes to `gameplay.app.state_changed` for Level entry/exit; Prep / Rush / Scoring sub-states are entirely internal (Scene Manager neither reads nor drives them); LevelData populate must complete within R11's 600 ms sub-budget | Scene Manager does not push data; pause state readable via `AppStateManager.IsPaused`; `Time.timeScale` exclusively Scene Manager's per R14.d | Scene Manager owns `AppState` + `IsPaused` + `Time.timeScale`; Level Runtime reads them; Prep→Rush is Level Runtime's internal concern |
| **Main Menu / Settings / Pause UI** (downstream, undesigned — **PROVISIONAL**) | Pause UI calls `AppStateManager.RequestPause()` / `RequestUnpause()` as synchronous method calls (NOT via Event Bus — `Time.timeScale` must be set synchronously with overlay load per R9) | `gameplay.app.state_changed` with `isPaused = true / false` is published so Audio Bus, analytics, HUD can react | Scene Manager owns method contract; Pause UI owns call site; Event Bus publish is notification path for other systems |
| **Audio Bus** (downstream, undesigned — **PROVISIONAL**) | Subscribes to `gameplay.app.state_changed` to fade individual audio sources on in-game pause (`isPaused == true`); does NOT subscribe to OS-background (Scene Manager handles via direct `AudioListener.pause` per R14.f) | Scene Manager publishes `state_changed`; sets `AudioListener.pause` directly on OS-background only | Split ownership per R14.f: Scene Manager owns OS-background `AudioListener.pause`; Audio Bus owns in-game audio fade via `state_changed` subscription. Cross-referenced here; Audio Bus GDD must document the same split |

## Formulas

### D.1 — Retry Budget Decomposition

**Formula (pass-3 added per-term overrun assertions; pass-2 reformulated to eliminate slack double-count):**

```
T_retry = T_sm_actual + T_lr_actual                          (two-term sum; the actual contract)
T_retry ≤ T_retry_max = 2000 ms                              (hard gate on ADR-003 reference device)

// Per-term overruns (pass-3 BLOCKING #3 — false-negative defense).
T_sm_overrun = max(0, T_sm_actual - T_sm_budget)             (Scene Manager overage, ≥ 0)
T_lr_overrun = max(0, T_lr_actual - T_lr_budget)             (Level Runtime overage, ≥ 0)
T_slack_consumed = T_sm_overrun + T_lr_overrun               (derived; equals max(0, ...) sum-form)
assert T_slack_consumed ≤ T_slack_budget                     (300 ms ceiling on combined overage)
```

`T_slack_consumed` is an accounting label reporting how much of the 300 ms shared slack was used to absorb overage from the two real budget holders. It is NOT a third additive term in the sum.

**Per-term overrun logging (pass-3 BLOCKING #3 false-negative defense).** Pass-2's `T_slack_consumed` could mask a single-term overage when the other term ran under budget — e.g., `T_sm = 1300 ms (200 over)` and `T_lr = 400 ms (200 under)` produces `T_slack_consumed = 200 - 200 = 0 ms` under naive subtraction, hiding a Scene Manager regression. The `max(0, ...)` floor in each term prevents the under-budget side from offsetting the over-budget side at all (i.e., `T_sm_overrun = 200`, `T_lr_overrun = 0`, `T_slack_consumed = 200`). The DEVELOPMENT_BUILD Budget Analyzer logs `T_sm_overrun` and `T_lr_overrun` separately and **unconditionally when either is > 0** (independent of whether `T_slack_consumed` exceeds `T_slack_budget`). This surfaces budget-creep before it crosses the gate threshold. AC-D1 verifies the unconditional logging.

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Scene Manager actual | T_sm_actual | float (ms) | [0, ∞) | Wall-clock from retry tap to `sceneLoaded` + handshake complete for the new Level scene. Owned by Scene Manager (R11). Includes pre-unload hooks, unload + `Resources.UnloadUnusedAssets()`, load, activation handshake. |
| Scene Manager budget | T_sm_budget | float (ms) | = 1100 | Warm-Addressables-cache target. Cold-cache may exceed (R11). |
| Level Runtime actual | T_lr_actual | float (ms) | [0, ∞) | Wall-clock from `sceneLoaded` to first-interactive-frame of new Level (LevelData populate + initial render + Input ready). Owned by Level Runtime (downstream, undesigned). |
| Level Runtime budget | T_lr_budget | float (ms) | = 600 | Residual allocation (2000 - 1100 - 300); not evidence-based. Cross-GDD obligation — Level Runtime GDD must include a blocking AC for `T_lr_actual ≤ T_lr_budget + slack-share`. |
| Slack consumed (derived) | T_slack_consumed | float (ms) | [0, ∞) | Reported diagnostic. If `> T_slack_budget`, AC-R11 fails. Logged separately in DEVELOPMENT_BUILD Budget Analyzer. |
| Slack budget | T_slack_budget | float (ms) | = 300 | Combined overage ceiling shared between T_sm and T_lr. |
| Total retry ceiling | T_retry_max | float (ms) | = 2000 | Hard ceiling. DEVICE AC-P-Retry fails the gate if T_retry > T_retry_max. |

**Output Range:** 0 to 2000 ms under normal operation. Exceeding 2000 ms is a BLOCKING regression on the ADR-003 reference device. In DEVELOPMENT_BUILD, the Budget Analyzer logs per-line-item attribution AND the derived `T_slack_consumed`; Release builds log only the total.

**Behavior when `T_slack_consumed > T_slack_budget`:** by definition `T_retry > T_retry_max`, so AC-R11 fails. Budget Analyzer logs the actual overage value separately so post-mortem can identify which budget holder caused the breach.

**Example (warm Addressables cache, steady thermal):**
- T_sm_actual = 550 ms (Level unload 180 ms + LoadSceneAsync 250 ms — Single-mode auto-reclaims, see R6 BLOCKING #8 fix + handshake 20 ms + pre-unload hooks 100 ms)
- T_lr_actual = 380 ms (LevelData populate 120 ms + first frame 250 ms + Input re-initialize 10 ms)
- T_retry = 930 ms ≤ 2000 ms → PASS with 1070 ms margin.
- **Pass-3 per-term overruns:** `T_sm_overrun = max(0, 550-1100) = 0 ms`; `T_lr_overrun = max(0, 380-600) = 0 ms`; `T_slack_consumed = 0 ms` (both under budget; no overrun log fires).

**Example (cold Addressables cache, thermal throttle):**
- T_sm_actual = 1050 ms (Level unload 200 ms + cold-cache LoadSceneAsync 530 ms + handshake 50 ms + pre-unload hooks 270 ms — auto-reclaim is now Unity-internal in Single mode, see R6 BLOCKING #8 fix)
- T_lr_actual = 680 ms (uncached LevelData SOs 280 ms + first frame 380 ms + Input 20 ms)
- T_retry = 1730 ms ≤ 2000 ms → PASS with 270 ms margin.
- **Pass-3 per-term overruns:** `T_sm_overrun = max(0, 1050-1100) = 0 ms`; `T_lr_overrun = max(0, 680-600) = 80 ms`; `T_slack_consumed = 0 + 80 = 80 ms` (≤ 300 ms ceiling). DEVELOPMENT_BUILD logs `T_lr_overrun = 80ms` unconditionally because it is > 0, surfacing the Level Runtime budget creep before it crosses the gate.

**Example (pathological — combined overage exceeds slack):**
- T_sm_actual = 1300 ms; T_lr_actual = 850 ms.
- T_retry = 2150 ms > 2000 ms → **FAIL** AC-R11.
- **Pass-3 per-term overruns:** `T_sm_overrun = 200 ms`; `T_lr_overrun = 250 ms`; `T_slack_consumed = 450 ms` (exceeds 300 ms ceiling; both T_sm and T_lr flagged in unconditional log).

**Example (pass-3 BLOCKING #3 false-negative defense — pre-fix would have masked):**
- T_sm_actual = 1300 ms (200 over); T_lr_actual = 400 ms (200 under).
- T_retry = 1700 ms ≤ 2000 ms → gate PASSES.
- **Naive subtraction (pass-2 formula):** `T_slack_consumed = max(0, 200 + (-200)) = 0 ms` — masks the Scene Manager regression entirely.
- **Pass-3 per-term overruns:** `T_sm_overrun = 200 ms`; `T_lr_overrun = 0 ms`; `T_slack_consumed = 200 ms`. DEVELOPMENT_BUILD logs `T_sm_overrun = 200ms` unconditionally. Without this fix, a 200ms Scene Manager regression would have been invisible to the budget analyzer until the gate failed.

### D.2 — Pre-Unload Hook Aggregate Window

```
T_preunload = max(T_handler_i for i in registered_handlers)   // parallel Task.WhenAll
T_preunload ≤ T_preunload_timeout = 1000 ms                   // cancellation token fires if exceeded
```

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Handler i duration | T_handler_i | float (ms) | [0, 1000] | Wall-clock for the i-th registered `IPreUnloadHook.OnBeforeUnloadAsync` task. Parallel with other handlers via `Task.WhenAll`. |
| Aggregate window | T_preunload | float (ms) | [0, 1000] | The longest-running handler's duration (since they run in parallel). **NOT the sum.** |
| Timeout | T_preunload_timeout | float (ms) | = 1000 | Hard cancellation-token trigger. If reached, Scene Manager proceeds to unload anyway with BLOCKING log in DEVELOPMENT_BUILD. |

**MVP handler budgets (R7 priorities):**

| Handler | Priority | Nominal T_handler | Soft cap | Hard cap (cancel) |
|---------|----------|-------------------|----------|-------------------|
| Event Bus (flush deferred queue + transition to Disposed) | 10 | < 5 ms | 50 ms | 1000 ms |
| Save/Load (`RequestFlush` fire-and-forget on retry; `FlushSync` on quit) | 20 | < 1 ms (fire-and-forget) | 200 ms | 1000 ms |
| Input (disable action maps + unsubscribe) | 30 | < 3 ms | 20 ms | 1000 ms |

**Output Range:** 0 to 1000 ms. Nominal operation should see T_preunload < 10 ms (handlers are all lightweight). 1000 ms is a defensive ceiling for pathological Save/Load blocking — which R11 prevents on the retry path via fire-and-forget.

**Example (retry):**
- Event Bus: 4 ms (deferred queue was empty — typical of Rush-end state)
- Save/Load: 1 ms (`RequestFlush` returns immediately; background thread continues)
- Input: 2 ms
- T_preunload = max(4, 1, 2) = 4 ms → PASS.

**Edge case:** If a handler's `OnBeforeUnloadAsync` returns a never-completing Task, the cancellation token fires at 1000 ms, the handler's completion is abandoned, and Scene Manager logs `[SceneManager] Pre-unload handler timeout for {handler.GetType().Name}` as BLOCKING in `DEVELOPMENT_BUILD`. Release builds log as Error (non-blocking). The scene still unloads; the offending handler is likely deadlocked and needs investigation.

**Edge case (pass-6 R6 — zero handlers):** If no `IPreUnloadHook` handlers are registered (e.g., test scene with only `BootLoader` skeleton, or future scope where Foundation siblings opt out), `Task.WhenAll([])` completes immediately with `Task.CompletedTask` — Unity's `Task.WhenAll` overload accepts an empty array and returns synchronously-completed. `T_preunload = max(empty) = 0 ms` (definition: max over empty set conventionally returns 0 here, treating the aggregate window as "no work to do"). The transition proceeds without delay. No log fires (zero handlers is a legitimate runtime configuration, not an error). The 1000 ms timeout is irrelevant in this case (the cancellation token is set up but never approached). This codifies the previously-implicit behavior — relying on it would have been a maintenance trap if a future implementation switched to a max-of-non-empty-checked variant.

### D.3 — OS-Background Session-Expiry Threshold (pass-3 three-tier + user-pause preservation)

```
// Step 0 (pass-3 REC-3): unconditional audio restore BEFORE the branch.
AudioListener.pause = false

elapsed = t_resume - t_background_utc        // clamp to 0 if negative (E12)

// Pass-3 BLOCKING #13: three-tier threshold (Rush / Prep / general).
if _backgroundedDuringRush:
    threshold = bg_expiry_in_rush_seconds    // default 1800 sec (30 min)
elif _backgroundedDuringPrep:
    threshold = bg_expiry_in_prep_seconds    // default 600 sec (10 min)
else:
    threshold = bg_expiry_general_seconds    // default 60 sec

if elapsed > threshold:
    BeginTransitionAsync("MainMenu")
    // _userPausedBeforeBackground implicitly discarded; new session is unpaused.
else:
    // Pass-3 BLOCKING #1: restore the user's pre-background pause state, NOT a hardcoded false.
    IsPaused = _userPausedBeforeBackground
```

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Timestamp at background | t_background_utc | `DateTime` UTC | n/a | Captured at `OnApplicationPause(true)` per R10 step 5. |
| Was in Rush at background? | _backgroundedDuringRush | bool | n/a | Captured at `OnApplicationPause(true)` per R10 step 5 — `true` iff `AppState == Level && LevelRuntime.IsInRushPhase` at that moment. |
| Was in Prep at background? | _backgroundedDuringPrep | bool | n/a | Captured at `OnApplicationPause(true)` per R10 step 5 — `true` iff `AppState == Level && !LevelRuntime.IsInRushPhase` at that moment (i.e., Level scene loaded but Prep Phase still active). Pass-3 BLOCKING #13. |
| User pause state captured at background | _userPausedBeforeBackground | bool | n/a | Captured at `OnApplicationPause(true)` per R10 step 2 — value of `IsPaused` BEFORE the OS-background path mutates it. Pass-3 BLOCKING #1. |
| Timestamp at resume | t_resume | `DateTime` UTC | n/a | Captured at `OnApplicationPause(false)`. |
| Elapsed background duration | elapsed | float (sec) | [0, ∞) | `(t_resume - t_background_utc).TotalSeconds`. Device clock drift may produce small negative values in edge cases; clamp to 0 if negative. |
| In-Rush expiry threshold | bg_expiry_in_rush_seconds | int | [900, 3600] | Tuning knob (Section G); default 1800 sec (30 min). Protects mid-Rush continuity through typical interruptions. |
| In-Prep expiry threshold | bg_expiry_in_prep_seconds | int | [300, 1200] | Tuning knob (Section G); default 600 sec (10 min). Protects Prep Phase planning state — pass-3 BLOCKING #13 surgical fix; CD adjudicated against widening the general threshold to 1800s. |
| General expiry threshold | bg_expiry_general_seconds | int | [30, 300] | Tuning knob (Section G); default 60 sec. Eject from non-gameplay states quickly. |

**Output:** branch decision — `MainMenu` transition vs in-place resume (with user-pause restored from pre-background state).

**Example (brief phone call mid-Rush, 4 min, user was NOT paused):**
- `_backgroundedDuringRush = true`; `_userPausedBeforeBackground = false`; threshold = 1800 sec.
- elapsed = 240 sec < 1800 sec → in-place resume. `IsPaused = false` (restored from `_userPausedBeforeBackground`). Player sees their mid-Rush board exactly as they left it, gameplay resumes. **Section B fantasy preserved.**

**Example (brief phone call mid-Rush, 4 min, user HAD paused before the call):**
- `_backgroundedDuringRush = true`; `_userPausedBeforeBackground = true`; threshold = 1800 sec.
- elapsed = 240 sec < 1800 sec → in-place resume. `IsPaused = true` (restored from `_userPausedBeforeBackground`). Player remains paused; the user's intent to pause survives the OS round trip. **Pass-3 BLOCKING #1 fix verified: pass-2 would have silently set `IsPaused = false` here.**

**Example (long phone call mid-Rush, 25 min):**
- `_backgroundedDuringRush = true`; threshold = 1800 sec.
- elapsed = 1500 sec < 1800 sec → in-place resume with `IsPaused = _userPausedBeforeBackground`. Mid-Rush state retained even for a typical lunchtime call.

**Example (player abandoned mid-Rush, 45 min):**
- `_backgroundedDuringRush = true`; threshold = 1800 sec.
- elapsed = 2700 sec > 1800 sec → `BeginTransitionAsync("MainMenu")`. The 30-min upper bound preserves session-freshness; resuming an in-flight Rush 45 min later would feel jarring.

**Example (Prep Phase planner, 8-min interruption — pass-3 BLOCKING #13 worked example):**
- `_backgroundedDuringPrep = true`; threshold = 600 sec.
- elapsed = 480 sec < 600 sec → in-place resume. The player who was configuring tools in Prep Phase sees their tool layout preserved, with `IsPaused = _userPausedBeforeBackground`. Pass-2 (no Prep tier) would have ejected to MainMenu at the 60s general threshold within the first minute, destroying their planning. **Pass-3 BLOCKING #13 fix verified.**

**Example (Prep Phase planner, 12-min interruption):**
- `_backgroundedDuringPrep = true`; threshold = 600 sec.
- elapsed = 720 sec > 600 sec → `BeginTransitionAsync("MainMenu")`. 10 min is the calibrated cutoff: long enough for typical interruptions (call, errand) but short enough that returning Prep state past that point feels stale rather than helpful.

**Example (notification-shade pull on MainMenu, 15 sec):**
- `_backgroundedDuringRush = false`; `_backgroundedDuringPrep = false`; threshold = 60 sec.
- elapsed = 15 sec < 60 sec → in-place. MainMenu remains visible. `IsPaused = _userPausedBeforeBackground` (effectively `false` since pause is not valid in MainMenu per E7).

**Example (player returns to MainMenu after 5 min):**
- All flags `false`; threshold = 60 sec.
- elapsed = 300 sec > 60 sec → `BeginTransitionAsync("MainMenu")`. Re-init feels appropriate after a non-trivial absence from a non-gameplay state.

**Edge case:** If the system clock was adjusted during the background window (NTP sync, DST), `elapsed` may be negative or spuriously large. If `elapsed < 0` → treat as 0 (in-place resume; user-pause state is restored from `_userPausedBeforeBackground`). If `elapsed > 86400 sec` (24 hr) → clamp to "expired" unconditionally AND trigger the session-continuity integrity check (Save/Load's responsibility, undesigned). The three-tier threshold logic still applies: even with `_backgroundedDuringRush = true`, a 24+ hour absence triggers MainMenu — no Rush state is worth preserving across that gap.

**PauseOverlay teardown on bg-expiry eject (pass-6 R16).** When the OS-background resume path branches into `BeginTransitionAsync("MainMenu")` (elapsed > applicable tier threshold), the active `PauseOverlay` scene (if loaded — possible if the player paused before backgrounding) must be unloaded BEFORE the MainMenu transition fires. Otherwise, `BeginTransitionAsync` runs in `LoadSceneMode.Single` against the Level scene while `PauseOverlay` (additive) remains loaded, leading to a brief intermediate state where MainMenu is the new active scene and PauseOverlay is still visually present (orphaned overlay against MainMenu — sortingOrder budget gives it band [30, 39] which would visually layer over MainMenu's UI). **Resolution:** the OS-background resume path's `BeginTransitionAsync("MainMenu")` call is preceded by an explicit `PauseOverlay` cleanup: if `SceneManager.GetSceneByName("PauseOverlay").isLoaded == true`, await `SceneManager.UnloadSceneAsync("PauseOverlay")` AND `await Resources.UnloadUnusedAssets()` (additive unload does not auto-reclaim per R6) BEFORE invoking `BeginTransitionAsync`. The user-facing experience: brief MainMenu-without-overlay frame, then the standard MainMenu transition. This is acceptable on the bg-expiry path (a session that has been backgrounded past threshold is being intentionally re-initialized; the player expects a fresh state). The cleanup adds ~50-100 ms to the resume-eject path which is well within mobile-resume tolerance (no Pillar 5 budget applies — this is not a gameplay touch). Code-review enforcement: any future bg-expiry path implementation MUST include the PauseOverlay cleanup; missing cleanup is a regression even though no AC currently catches it (low-frequency event, hard to deterministically reproduce in CI).

### D.4 — Activation Handshake Timeout

```
T_activation_wait = min(T_overlay_fade_out, T_activation_max)
T_activation_max = 500 ms
```

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Actual overlay fade-out duration | T_overlay_fade_out | float (ms) | [0, ∞) | Time Modal/Dialog takes to fade out its loading overlay before invoking `activationCallback(true)`. Unknown to Scene Manager. |
| Observed wait | T_activation_wait | float (ms) | [0, 500] | How long Scene Manager actually waited. Capped by timeout. |
| Timeout ceiling | T_activation_max | float (ms) | = 500 | If exceeded, Scene Manager proceeds anyway with BLOCKING log in `DEVELOPMENT_BUILD`. Release builds proceed silently (test-scene fallback path). |

**Output Range:** 0 to 500 ms. Nominal Modal/Dialog fade-out should be 150–250 ms (per Modal/Dialog GDD when authored); 500 ms is a defensive ceiling.

**Example (production Modal/Dialog, 200 ms fade):**
- T_overlay_fade_out = 200 ms
- T_activation_wait = 200 ms → `allowSceneActivation = true` fires.

**Example (no subscriber, test scene):**
- No Modal/Dialog subscriber exists → `activationCallback` never invoked.
- T_activation_wait = 500 ms (timeout) → Scene Manager proceeds with activation. Log in DEVELOPMENT_BUILD: `[SceneManager] load_ready activation ack timeout; proceeding (possibly a test scene without Modal/Dialog)`.

**Edge case:** If `activationCallback(false)` is invoked (rejection), Scene Manager holds the scene at 0.9 indefinitely and logs a BLOCKING error. Currently unused — reserved for VS / post-MVP if Modal/Dialog needs to abort a transition (e.g., accessibility-confirm interrupt). At MVP, the boolean argument is always `true` or never invoked (timeout case).

### D.5 — Bootstrap Cumulative Init Duration

```
T_boot = T_dr + T_eb + T_sl + T_input   // sequential per R2
T_boot_target = 3000 ms                  // ADVISORY soft target
```

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Data Registry init | T_dr | float (ms) | [0, 1000] | `DataRegistry.Initialize()` — Catalog SO warm-up (includes `OnEnable` firing across all catalog SOs + lookup dictionary construction). |
| Event Bus init | T_eb | float (ms) | [0, 200] | `EventBus.Initialize()` — PlayerLoop injection + subscriber dictionaries. |
| Save/Load init | T_sl | float (ms) | [0, 800] | `SaveManager.Initialize()` — load `save.json` from `Application.persistentDataPath`, run migration if needed, populate in-memory state. Variance: fresh install ~5 ms; v1→v2 migration ~200 ms; `.bak` recovery ~400 ms. |
| Input init | T_input | float (ms) | [0, 400] | `InputDispatcher.Initialize()` — populate `DeviceDisplayMetrics` from `Screen.dpi` / `Screen.safeArea`, wire action maps. |
| Cumulative boot duration | T_boot | float (ms) | [0, 2400] | Sum of above; sequential per R2. |
| Target | T_boot_target | float (ms) | = 3000 | ADVISORY (not BLOCKING). Exceeding → flagged in performance regression dashboard but not gated. Unity engine cold-boot + splash screen are additional and outside this budget (Unity controls them). |

**Output Range:** 0 to 2400 ms under nominal conditions; 3000 ms is the advisory soft target.

**Example (fresh install, cold cache):**
- T_dr = 450 ms (first-time Catalog SO load)
- T_eb = 60 ms
- T_sl = 5 ms (no save file, `FirstRun` path)
- T_input = 120 ms
- T_boot = 635 ms → well within target.

**Example (mid-campaign, v1→v2 migration on resume):**
- T_dr = 180 ms (warm asset cache)
- T_eb = 50 ms
- T_sl = 220 ms (migration)
- T_input = 110 ms
- T_boot = 560 ms → within target.

**Edge case:** If `T_boot > T_boot_target`, the game still works — the bootstrap just took longer than desired. Advisory logging in DEVELOPMENT_BUILD; no gate. If a single Foundation sibling exceeds its per-sibling ceiling (e.g., `T_sl > 800`), the sibling's own GDD owns the diagnosis and mitigation; Scene Manager does not arbitrate.

**Per-sibling boot timeout (pass-6 R7 — R2 hang protection).** R2's `BootLoader.Awake()` awaits each Foundation sibling's `Task<bool> Ready` sequentially. A buggy sibling that hangs on `Ready` (e.g., Save/Load awaiting a never-resolving file I/O on a corrupted device, or Input awaiting an OS API on an emulator with a missing extension) produces a Boot scene that never advances — the splash screen displays indefinitely, with no diagnostic. The player force-kills and loses the session. **Mitigation:** wrap each sibling's `Ready` await in a per-sibling timeout: `T_sibling_max = T_x_ceiling × 2` (e.g., `T_dr_ceiling = 1000 ms` → `T_dr_max = 2000 ms` for the timeout; the safe-range ceiling is the nominal upper bound; 2× absorbs cold-cache and one-time migrations). On timeout, `BootLoader` logs `[BootLoader] Foundation sibling {name} Ready timed out at {T_x_max}ms — aborting boot` as fatal, runs `R13 quit teardown` to release whatever Foundation state has been initialized, and exits to the OS error screen (or Editor stops Play). The timeout is per-sibling, not aggregate — a 2-second `T_dr_max` plus a 2-second `T_eb_max` plus etc. would otherwise compound into an 8-second boot before failure surfaces. Per-sibling means the first hang is caught at its own ceiling, not after the cumulative budget. **R2 amendment:** the await sequence becomes `await Task.WhenAny(sibling.Ready, Task.Delay(T_x_max))` with a post-check; if `Ready` did not complete, fail. This is BLOCKING for R2 implementation; tracked as a code-review enforcement (no separate AC since AC-EC-E13 already covers the failure path's quit-teardown obligation).

**Sums-of-ceilings invariant (pass-2 systems-designer B3; pass-6 BLOCKING #4 — arithmetic correction).** The Section G tuning knobs allow each per-sibling ceiling to be raised independently to safe-range maxima. Pass-2 reported the safe-range maxima sum as `1000+500+1500+800 = 3800 ms` — that arithmetic used the **current values** for `T_dr_ceiling` (1000 ms) and `T_input_ceiling` (400 ms — note: pass-2 also used 800 here, not the current value), not the safe-range MAXIMA. The correct sum across **safe-range maxima** per Section G (`T_dr_ceiling [500, 2000]`, `T_eb_ceiling [100, 500]`, `T_sl_ceiling [400, 1500]`, `T_input_ceiling [200, 800]`) is `2000 + 500 + 1500 + 800 = **4800 ms**` (vs `T_boot_target = 3000 ms`). The pass-2 `+800 ms` margin allowance was derived against the wrong sum; the corrected delta is `4800 - 3000 = 1800 ms` between the safe-range max sum and the advisory target — meaning a tuner operating within "safe" ranges can collectively exceed the advisory target by up to **1.8×**, not the 0.27× pass-2's arithmetic implied. The invariant must be re-stated against the correct delta: when raising any `T_x_ceiling` toward its safe-range max, verify `sum(T_x_ceiling) ≤ T_boot_target × 1.6 = 4800 ms` (a 60% over-target headroom that allows the safe ranges to be exercised but flags the pathological case where every ceiling is simultaneously raised to its max — a configuration that almost certainly indicates a deeper performance regression elsewhere and warrants design-time review, not silent acceptance). If the sum would exceed `4800 ms`, the corresponding lower-ceiling sibling must be tightened in the same change. Add DEVELOPMENT_BUILD log: `[BootLoader] sum(T_x_ceiling) = {sum}ms exceeds 1.6× T_boot_target = 4800ms; tighten one ceiling or revisit T_boot_target` (advisory, non-gating). The pass-2 framing of "+800 ms accounts for legitimate margin between nominal worst-case and safe-range max" was reasoning about the wrong arithmetic — the safe-range maxima are not "nominal worst-case + 800 ms"; they are intentional Tuning Knob ceilings several multiples above nominal. CD adjudicated this as BLOCKING (formula correctness is canonical contract; runtime DEVELOPMENT_BUILD log is mitigation, not fix). AC-D3 logs per-sibling ceilings AND the sum vs the corrected `4800 ms` invariant.

## Edge Cases

**E1 — If `SceneManager.sceneUnloaded` is used for Foundation cleanup**: the callback fires AFTER scene objects are destroyed; by that point Event Bus subscribers may already be GC'd. R7 `IPreUnloadHook` is the correct pattern; use of `sceneUnloaded` for Foundation teardown is a code-review BLOCKING issue.

**E2 — If a pre-unload handler's `OnBeforeUnloadAsync` never completes**: the 1000 ms cancellation token fires (D.2), Scene Manager logs `[SceneManager] Pre-unload handler timeout for {type}` as BLOCKING in DEVELOPMENT_BUILD, and proceeds with the unload. The handler is presumed deadlocked; post-mortem investigation required.

**E3 — If an `IPreUnloadHook` registers after the first transition has begun**: `RegisterPreUnloadHandler` ignores the registration and logs a warning in DEVELOPMENT_BUILD. All Foundation siblings must register during `BootLoader.Awake()` before `BeginTransitionAsync("MainMenu")` is called.

**E4 — If `sceneLoaded` fires for the additive `PauseOverlay` scene**: Scene Manager distinguishes main-scene-load (triggers `app.state_changed`) from overlay-load (no `app.state_changed` — R9 explicitly owns the pause-state publish). The distinction is by scene name match against the current pause overlay scene constant.

**E5 — If `BeginTransitionAsync` is called with a scene name not in Build Settings**: Unity's `LoadSceneAsync` throws `ArgumentException`; Scene Manager catches, logs `[SceneManager] Invalid scene: {name}` as BLOCKING, clears `_transitionInProgress`, and drops any queued pending transition. The `_pendingPause` queue is preserved if the current scene was user-paused at the time of the invalid request.

**E6 — If the retry button is double-tapped**: R12 covers — second tap queues to `_pendingTransition` (overwriting any prior pending entry with a dev warning), single retry executes exactly once.

**E7 — If pause is requested during Boot state**: `RequestPause()` returns with a BLOCKING log `[SceneManager] Cannot pause during Boot state`. No scene has loaded yet; there is no user-visible context to pause. MainMenu must be reached before pause is valid.

**E8 — If two or more subscribers register for `gameplay.scene.load_ready`**: DEVELOPMENT_BUILD fires BLOCKING error `[SceneManager] Exactly one subscriber required for gameplay.scene.load_ready (found: N)`. Release builds invoke the first-registered subscriber's callback only and log a warning. Modal/Dialog GDD owns the single production subscriber; test scenes that need to override this must unregister the production subscriber explicitly.

**E9 — If `OnApplicationPause(true)` fires mid-transition**: R10 forced-pause path runs ON TOP of the in-progress transition. `SaveManager.FlushSync()` runs; `AudioListener.pause = true`; `_backgroundedAtUtc` captured. On resume: if elapsed ≤ threshold, the transition resumes from wherever it was suspended (Unity's async operation is still in memory); if elapsed > threshold, the pending transition is abandoned and `BeginTransitionAsync("MainMenu")` fires fresh.

**E10 — If the app is force-killed (iOS SIGKILL, Android swipe-from-recents)**: `Application.quitting` does NOT fire; but R10's `OnApplicationPause(true)` fired before process suspension and executed the synchronous `FlushSync`. On relaunch, `BootLoader` loads the persisted save state. This is why R10 uses `FlushSync` (not `RequestFlush`) — force-kill resilience.

**E11 — If `OnApplicationPause(false)` fires without a preceding `(true)` (Android OEM quirk)**: defensive check — if `_backgroundedAtUtc == default`, no-op (we have no record of having backgrounded). Do not attempt to compute elapsed against a zero timestamp.

**E12 — If the system clock adjusts during OS background (NTP sync, DST, user-initiated)**: `elapsed = t_resume - t_background_utc` may be negative or spuriously large. Clamp `elapsed < 0` to `0` (in-place resume); treat `elapsed > 86400 sec` (24 hrs) as "expired" unconditionally AND trigger the Save/Load session-continuity integrity check (Save/Load GDD owns, undesigned).

**E13 — If a Foundation sibling's `Initialize()` throws during boot**: `BootLoader` catches, logs fatal `[BootLoader] Foundation sibling {name} failed to initialize: {exception}`, displays Unity's default error screen (or at VS, a custom "Initialization Error" splash owned by Main Menu GDD), and exits to OS. Do NOT attempt to proceed to MainMenu with partial Foundation; invariant R14.b would be violated.

**E14 — If the developer runs Play from a non-Boot scene in the Editor (pass-2 revised — FoundationStub replaced by enforcement):** A `FoundationStub` mock in the Editor would diverge from production state and create a "tests pass in dev, fails in production" footgun (game-designer REC-4 + qa-lead R4 convergent finding). Pass-2 mitigation: an Editor-only `BootEnforcer` script (`[InitializeOnLoad]` + `EditorApplication.playModeStateChanged`) intercepts Play-from-non-Boot, stores the target scene name in an `EditorPrefs` key (`MM.Editor.PostBootJumpTo`), loads `Boot.unity`, and enters Play mode. `BootLoader` checks `MM.Editor.PostBootJumpTo` after the R2 init chain completes; if set, it issues `BeginTransitionAsync(targetScene)` instead of the default `MainMenu` transition. Result: every Play session goes through the production R2/R2.a init chain on real Foundation siblings; production fidelity is guaranteed; developer ergonomics preserved (the Play target scene still loads). Implementation lives in `Assets/Editor/BootEnforcer.cs`. Verified by AC-EC-E14a (EditMode interception) + AC-EC-E14b (PlayMode post-jump) — pass-3 BLOCKING #6 split the pass-2 AC-EC-E14 into the two halves so each test's executable mode (EditMode vs PlayMode) matches what it can actually verify. The pass-1 `FoundationStub` prefab pattern is **DISCARDED** — do not implement.

**E15 — If two `AppStateManager` instances exist simultaneously (Fast Enter Play Mode without domain reload + Boot scene re-entry)**: the second instance's `Awake` detects the prior `DontDestroyOnLoad` instance, logs warning `[AppStateManager] Duplicate instance detected; destroying self`, and calls `Destroy(this.gameObject)` immediately. The original instance continues operating.

**E16 — If any system writes `Time.timeScale` outside `AppStateManager`**: R14.d invariant holds at code-review level (grep CI rule fails on `Time.timeScale =` outside `AppStateManager.cs`). Runtime safeguard: `RequestUnpause` asserts `Time.timeScale == 0f` before setting to `1f`; mismatch fires `[SceneManager] Time.timeScale violated R14.d — expected 0, got {value}` as BLOCKING log in DEVELOPMENT_BUILD with stack trace (best-effort attribution to the violating call site).

**E17 — If Audio Bus is not yet authored (MVP ordering) and fails to subscribe to `gameplay.app.state_changed`**: audio continues playing through in-game pause. Documented MVP limitation; closes at VS when Audio Bus GDD lands. Scene Manager does NOT implement fallback audio fade — that would violate R14.f's split-ownership rule.

**E19 — If `Application.quitting` fires mid-transition (pass-6 R8 — quit-during-loading documentation)**: The R13 sync-teardown path runs while a `BeginTransitionAsync` cycle is still in flight. Pass-3 BLOCKING #5 spec'd `EventBus.DisposeSync()` + `InputDispatcher.TeardownSync()` + `SaveManager.FlushSync()` as the synchronous quit path; this edge case clarifies sequencing relative to the in-flight transition. **Behavior:** R13 does NOT await the in-flight `BeginTransitionAsync` to complete — the player has tapped quit (or the OS issued `Application.quitting`); finishing the transition adds latency to the quit while delivering no value (the destination scene will be torn down moments later). Instead, R13: (1) cancels the `BeginTransitionAsync` cancellation token (causing any in-flight pre-unload hooks to abandon their continuations); (2) skips `LoadSceneAsync` activation (any held `AsyncOperation` is allowed to be GC'd; no `allowSceneActivation = true` is fired); (3) executes the sync-teardown chain (Event Bus → Save/Load → Input in priority order); (4) sets `AppState = Disposed`. The in-flight transition's `_pendingTransition` and `_pendingPause` queues are cleared without firing. **Why this is not E9 (mid-transition OS background):** OS background is recoverable — the player may resume — so E9 preserves the in-flight transition state for resume. Quit is terminal — there is no resume; the transition state is unrecoverable; preserving it would waste teardown latency. Verified by code review (no formal AC since `Application.quitting` is rare in QA test environments and the failure mode — quit hangs while loading — is highly visible).

**E18 — If `Application.persistentDataPath` becomes invalid during OS-background (user cleared app data from Android settings while app was backgrounded)**: Save/Load's `FlushSync` in R10 throws `DirectoryNotFoundException`. Scene Manager catches, logs `[SceneManager] Save flush failed on OS background: storage invalidated` as BLOCKING, continues the R10 path (sets `AudioListener.pause = true`, `IsPaused = true`, records `_backgroundedAtUtc`). On resume, Save/Load transitions to `CorruptedRecovery` per its own AC; Scene Manager does not need a special-case recovery path.

## Dependencies

**Upstream**: None. Scene Manager is Foundation-layer; it depends only on Unity 6.3 LTS engine APIs and build/runtime infrastructure.

### Cross-system dependencies

| System | Direction | Nature | Interface | Status |
|--------|-----------|--------|-----------|--------|
| **Input System** | Input depends on SM | Lifecycle + pre-unload hook | `Initialize()` hooks `SceneManager.sceneLoaded` to populate `DeviceDisplayMetrics`. Registers as `IPreUnloadHook` priority 30 (disable action maps + unsubscribe). Activation handshake (R8) enforces Input's AC-EC-CFG "query-before-Initialize" prohibition at scene boundaries. | Approved (pass-6 post-Phase 6D 2026-04-24) |
| **Event Bus** | Bidirectional | Lifecycle driver + channel publisher | SM publishes three Gameplay-namespace channels: `gameplay.scene.will_unload` (T2), `gameplay.scene.load_ready` (T2, single-subscriber), `gameplay.app.state_changed` (T2). Event Bus registers as `IPreUnloadHook` priority 10 (flush deferred queue + transition to `Disposed`). Event Bus's `Disposed` terminal state is triggered by scene unload or `Application.quitting` per R13. | Approved (pass-3 + pass-4 amendment) |
| **Save/Load** | Save/Load depends on SM | Pre-unload hook + quit/background flush | Registers as `IPreUnloadHook` priority 20 (`RequestFlush` fire-and-forget on retry per R11; NOT awaited). On OS background (R10) + app quit (R13), SM calls `SaveManager.FlushSync()` synchronously. `OnDestroy` `SemaphoreSlim` disposal per AC-EC-Race3 must complete within pre-unload window. | Approved (pass 4 targeted verification 2026-04-21) |
| **Data Registry** | Data Registry depends on SM | Lifecycle terminal trigger | Catalog SO `Loaded → Consumed → terminal` uses scene unload as terminal trigger. SM's `UnloadSceneAsync` completion signals Catalog termination via Unity's `sceneUnloaded`. Data Registry intentionally has NO pre-unload hook — Catalog cleanup is terminal-only, idempotent, cheap. | Approved (pass 4) |
| **Modal/Dialog + Level End** | Modal/Dialog depends on SM | Activation handshake + loading overlay | Subscribes to `gameplay.scene.load_ready` as the **exactly-one** production subscriber. Owns the loading overlay visual. Invokes `activationCallback(true)` when fade-out completes. SM awaits callback with 500 ms timeout fallback (D.4). | Undesigned (MVP minimal priority) — must absorb R8 activation handshake contract when authored |
| **Level Runtime** | Level Runtime depends on SM | State subscriber + 600 ms populate budget + pause-flag reader | Subscribes to `gameplay.app.state_changed` for Level entry/exit. `LevelData` populate from Data Registry catalogs must complete within R11's 600 ms sub-budget (cross-GDD obligation — Level Runtime GDD must include a blocking AC against `T_lr ≤ 600 ms` on ADR-003 reference device). Prep / Rush / Scoring sub-states are entirely internal to Level Runtime. Reads `IsPaused`; `Time.timeScale` is exclusively SM's per R14.d. | Undesigned (MVP priority — Core layer #8) |
| **Main Menu / Settings / Pause** | Main Menu depends on SM | Method-call integration + transition consumer | Calls `AppStateManager.RequestPause()` / `RequestUnpause()` synchronously (not via Event Bus — `Time.timeScale` must be set synchronously with overlay load per R9). Triggers `BeginTransitionAsync` on nav actions (Play → Loading → BiomeMap; Quit → process exit). Owns pause-button UI + persistent "Accessibility" button (per Input System pass-5 B16). | Undesigned (MVP minimal priority) |
| **Biome Map / Level Select** | Biome Map depends on SM | Transition consumer | Calls `BeginTransitionAsync("Level")` on level-select tap. No special pre-unload needs. | Undesigned (MVP priority) |
| **Shop** | Shop depends on SM | Transition consumer | Calls `BeginTransitionAsync("MainMenu")` on exit. No special pre-unload needs. | Undesigned (MVP minimal priority) |
| **HUD** | HUD depends on SM | State subscriber | Subscribes to `gameplay.app.state_changed` — react to Level entry (show HUD), Level exit (hide HUD), `isPaused` toggle (dim HUD during pause). | Undesigned (MVP priority — Presentation layer) |
| **Audio Bus** | Audio Bus depends on SM (split ownership) | State subscriber + OS-background direct write | Subscribes to `gameplay.app.state_changed`, fades individual audio sources on `isPaused == true`. Does NOT subscribe to OS-background events — SM handles `AudioListener.pause = true/false` directly per R14.f (R10 fires before Event Bus may safely dispatch). Audio Bus GDD must document this split-ownership. | Undesigned (MVP priority — Foundation layer #6) |
| **Analytics / Telemetry** | Analytics depends on SM | Event subscriber | Subscribes to `gameplay.scene.will_unload` + `gameplay.app.state_changed` for scene-duration analytics. | Undesigned (Shippable v1.0) |
| **Ads SDK** | Ads SDK depends on SM | Transition injection (post-MVP) | Registers a `BeginTransitionAsync` wrapper that injects interstitial ad scene between gameplay scenes at configured trigger points. Not MVP-scoped. | Undesigned (Shippable v1.0 — Plumbing Sprint Months 4–5) |

### Platform dependencies

| Dependency | Usage | Risk |
|------------|-------|------|
| Unity 6.3 LTS `UnityEngine.SceneManagement` | `LoadSceneAsync`, `UnloadSceneAsync`, `sceneLoaded`, `sceneUnloaded`, `activeSceneChanged`, `AsyncOperation.allowSceneActivation` | Scene API surface post-Unity-2022-LTS is UNVERIFIED against Unity 6.3 LTS docs (no `modules/scene.md` in engine-reference); 7 claims flagged for Phase 6 `unity-specialist` verification (see Open Questions). |
| Unity `Application.quitting` | R13 app-quit contract | NOT reliable under iOS memory pressure or Android force-kill; R10's `OnApplicationPause(true)` `FlushSync` is the actual save-safety net. |
| Unity `OnApplicationPause(bool)` | R10 OS-background forced-pause path | Android fires reliably on home/call/notification-pull; iOS only on background. OEM-specific quirks per E11 defensive check. |
| Unity `RuntimeInitializeOnLoadMethod` | R3 restricted to `SubsystemRegistration` + `AfterAssembliesLoaded` only | `BeforeSplashScreen` tier 6.x behavior UNVERIFIED; usage prohibited outside the two permitted tiers. |
| Unity `Time.timeScale` | R9 pause overlay + R14.d exclusive-write invariant | `timeScale = 0` does NOT pause audio (`AudioListener.pause` is separate) or `Update` (only `FixedUpdate`); R7 handlers must use `async Task` or `WaitForSecondsRealtime`. |
| iOS / Android storage (`Application.persistentDataPath`) | `FlushSync` target at R10 / R13 | Android: user can clear app data mid-session (E18 edge case). iOS: sandboxed, more robust. |

### Pending ADRs triggered by this GDD

| ADR | Topic | Status | Trigger |
|-----|-------|--------|---------|
| **ADR-003** (shared — Save/Load + Input + Scene Manager) | Named Android + iOS reference devices | Pending (pre-production action item #3 from TD-FEASIBILITY gate) | AC-P-Retry (DEVICE AC) blocks on a named reference device. No MVP implementation until ADR-003 is resolved. |
| **ADR-008** (new — Scene Manager) | Scene Loading Strategy (async-only `SceneManager.LoadSceneAsync` + activation-handshake contract + Addressables deferral rationale + **Addressables 2.4 API mismatch note**) | Pending ADR authoring (pre-production scope) | R1 + R8 lock the async-only + handshake pattern at GDD level; ADR formalizes the decision record for architectural traceability and records Addressables deferral rationale. **Pass-2 unity-specialist R-5: Addressables 2.4 uses `SceneInstance.Activate()` not `AsyncOperation.allowSceneActivation` — when OQ-13 resolves and Addressables ships at VS, an adapter layer is required (not a drop-in replacement). ADR-008 must document this API surface delta.** |
| **ADR-009** (new — Scene Manager) | Bootstrap Architecture (dedicated Boot scene + `BootLoader` single-construction authority + Foundation `Reset()` discipline for Fast Enter Play Mode + **Editor `BootEnforcer` Play-from-non-Boot interception per E14**) | Pending ADR authoring (pre-production scope) | R2 + R2.a + R3 lock the Boot scene + RIOLM usage restrictions at GDD level; ADR formalizes the architectural record and documents Fast Enter Play Mode CI smoke-test requirement. **Pass-2: ADR-009 also captures the `BootEnforcer` decision (FoundationStub discarded; Editor-script enforcement adopted) — see E14.** |

### Editor & build-pipeline tooling required by this GDD

| Tool | Purpose | Trigger | Status |
|------|---------|---------|--------|
| `Assets/Editor/BootEnforcer.cs` | Intercepts Play-from-non-Boot; relaunches via Boot.unity preserving target scene in `EditorPrefs` | E14 + AC-EC-E14a (EditMode interception) + AC-EC-E14b (PlayMode post-jump) | Pre-implementation — author at first BootLoader story |
| Build-step Build Settings index 0 validator | Ensures `Boot.unity` is index 0 (R2) | AC-BOOT-R2 prerequisite | Pre-implementation — author with ADR-009 |
| Scene-validation script (CI) | Validates per-scene `EventSystem` count, `sortingOrder` budget, `DontDestroyOnLoad` exclusions | AC-INV2, AC-R9d, R14.e/g | Pre-implementation — author at Story 2 |

ADR-007 (CoreHaptics / `ProcessInfo.thermalState` / `PowerManager.ThermalStatus` / `VibrationEffect` native-plugin bridge) is triggered by Input System pass-6 Phase 6D and remains in flight for the same Plumbing Sprint (Months 4–5). Scene Manager does NOT add to ADR-007's scope.

### Bidirectional consistency amendments

Per project rule "dependencies must be bidirectional": when this GDD is approved, the following existing Foundation GDDs require minor additive edits — non-breaking, no full re-review:

- **Input System** — add Scene Manager row to Section F Dependencies (currently only references `SceneManager.sceneLoaded` in Section C prose). **Pass-4 amendment (BLOCKING #5):** add `InputDispatcher.TeardownSync()` synchronous teardown method alongside the async `IPreUnloadHook` path; document that `OnBeforeUnloadAsync` MUST NOT use `ConfigureAwait(false)` or `Task.Run` per Scene Manager R7 async-contract constraints.
- **Event Bus** — add Scene Manager as publisher of `gameplay.scene.*` + `gameplay.app.state_changed` channels in Section C Interactions + Section F Dependencies. Add `IPreUnloadHook` priority 10 registration rule to Event Bus R-rules. **Pass-4 amendment (BLOCKING #5):** add `EventBus.DisposeSync()` synchronous teardown method for the R13 quit path; document the same async-contract constraints (`ConfigureAwait(false)` and `Task.Run` prohibited in `OnBeforeUnloadAsync`).
- **Save/Load** — add Scene Manager as `FlushSync()` caller on OS background + quit in Section F. Add `IPreUnloadHook` priority 20 registration. (Pass-4 — Save/Load already exposes `FlushSync()`; no additional sync-teardown method required.)
- **Data Registry** — add Scene Manager as unload-sequence driver (informational only, no behavioral change).

These amendments are tracked as Phase 5 work items.

## Tuning Knobs

### Timing budgets (per-transition)

| Parameter | Current Value | Safe Range | Effect of Increase | Effect of Decrease |
|-----------|--------------|------------|-------------------|-------------------|
| `T_retry_max` | 2000 ms | [1500, 3000] | Relaxes the < 2 sec fast-retry promise. Above 2500 ms: retry-as-beat fantasy (Section B) starts to read as penalty. Above 3000 ms: Pillar 5 first-tap-post-load at risk | Tightens the gate. Below 1500 ms: unrealistic for Mali-G52 with cold Addressables cache — triggers flaky CI |
| `T_sm_budget` | 1100 ms | [800, 1400] | More Scene Manager margin; less Level Runtime margin (zero-sum against 2000 ms total minus slack) | Below 800 ms: `LoadSceneAsync` cold-cache on Mali-G52 alone can consume this |
| `T_lr_budget` | 600 ms | [400, 900] | More Level Runtime margin (owned by Level Runtime GDD when authored; cross-GDD knob) | Tighter LR budget; risks LevelData populate exceeding |
| `T_slack_budget` | 300 ms | [100, 500] | More absorption headroom for thermal / cold-cache variance | Below 100 ms: essentially no slack; any single line item exceeding nominal fails the 2000 ms gate |
| `T_preunload_timeout` | 1000 ms | [500, 2000] | More patience for slow handlers; longer transition window | Catches deadlocks faster but risks false-positive cancellation under thermal stress |
| `T_activation_ack_timeout` | 500 ms | [300, 1000] | More Modal/Dialog overlay animation budget | Below 300 ms: fade-out animation may not complete before Scene Manager proceeds (jarring transition) |

### OS-lifecycle thresholds (pass-3 BLOCKING #13 resolution: three-tier replaces pass-2 two-tier)

| Parameter | Current Value | Safe Range | Effect of Increase | Effect of Decrease |
|-----------|--------------|------------|-------------------|-------------------|
| `bg_expiry_general_seconds` | **60 sec** | [30, 300] | Longer general-resume window; risks false-positive MainMenu being delayed for legitimately abandoned sessions in MainMenu/BiomeMap/Shop | Shorter; more aggressive return to MainMenu when player was NOT in Level. Below 30 sec: notification-shade pulls + brief screen-locks may trigger forced re-init |
| `bg_expiry_in_prep_seconds` | **600 sec (10 min)** | [300, 1200] | Longer Prep Phase continuity window; preserves tool-configuration planning state through typical interruptions. Above 1200 sec (20 min): players returning to a stale Prep layout the next morning erodes the "fresh session" feel | Shorter; ejects mid-Prep planners on longer phone calls. Below 300 sec (5 min): typical phone calls start triggering false MainMenu returns — destroys planning investment. Pass-3 BLOCKING #13: CD adjudicated this third tier vs game-designer's "widen general to 1800s" — third tier wins for surgical scope (general MainMenu/BiomeMap/Shop ejection stays 60s) |
| `bg_expiry_in_rush_seconds` | **1800 sec (30 min)** | [900, 3600] | Longer mid-Rush continuity window; preserves the "seams are where the chaos isn't" fantasy through phone calls, errands, child interruptions. Above 3600 sec: session-freshness fantasy erodes | Shorter; more frequent forced MainMenu on long mid-Rush interruptions. Below 900 sec (15 min): typical commuter-route phone calls (median 3-4 min, long tail 10-20 min) start triggering false MainMenu returns — directly violates Section B's third fantasy |

The three-tier split protects mid-Rush continuity (the highest-stakes touchpoint per Section B), Prep Phase planning state (real but lower-value continuity), and ejects quickly from non-gameplay contexts. `_backgroundedDuringRush` and `_backgroundedDuringPrep` booleans are captured at OS-background time per R10 step 5. Pass-3 also captures `_userPausedBeforeBackground` at R10 step 2 — on resume below threshold, `IsPaused` is restored from this captured value (BLOCKING #1: pass-2 silently cleared user pause). Anti-stale upper bound on `bg_expiry_in_rush_seconds` preserves the fantasy at the long tail.

Future tuning (post-soft-launch retention data): all three knobs are candidates for telemetry-driven adjustment. If retention data reveals players abandoning at any threshold boundary, raise that tier; if telemetry shows return-from-MainMenu friction, raise `bg_expiry_general_seconds`. Decision deferred to live-ops. Tracked as OQ-19 expanded scope.

**Pass-6 R5 — `bg_expiry_general_seconds` revisit for BiomeMap / Shop context.** The 60 sec default lumps three distinct non-gameplay contexts together: MainMenu (where re-init feels appropriate after any non-trivial absence), BiomeMap (where the player has invested attention in level selection — a 60-sec ejection feels punitive after a quick notification check), and Shop (where transactional state — pending IAP confirmation, reviewing upgrade choice — has higher local investment than MainMenu but lower than mid-gameplay). Performance-analyst flagged that 60s may be too aggressive for BiomeMap/Shop. **Resolution:** retain 60s default at MVP (the conservative choice — eject-quickly is reversible if telemetry shows friction; eject-slowly that loses transactional state is harder to walk back). Defer split-into-three-non-gameplay-tiers (MainMenu / BiomeMap / Shop) to a live-ops tuning candidate, tracked as OQ-19 expansion. If post-soft-launch telemetry shows BiomeMap-resume friction (players returning after 90-180s, finding themselves dumped to MainMenu, expressing frustration in app-store reviews), the surgical resolution is to introduce a fourth-tier `bg_expiry_in_biomemap_seconds` (likely 300s) and `bg_expiry_in_shop_seconds` (likely 180s — short enough to invalidate stale IAP confirmation context) without touching `bg_expiry_general_seconds`. Pass-6 takes no MVP action — knob-tuning candidacy is the entire output.

### Bootstrap (ADVISORY — no gate, dashboards only)

| Parameter | Current Value | Safe Range | Effect of Increase | Effect of Decrease |
|-----------|--------------|------------|-------------------|-------------------|
| `T_boot_target` | 3000 ms | [2000, 5000] | More tolerance for slow boots | Below 2000 ms: unrealistic for Mali-G52 cold app launch with Unity engine boot + Catalog SO warm-up |
| `T_dr_ceiling` | 1000 ms | [500, 2000] | More Data Registry warm-up margin; signals growing Catalog SO count | Tightens — may require Data Registry Layer-2 latency gate re-tuning |
| `T_eb_ceiling` | 200 ms | [100, 500] | More Event Bus bootstrap margin | Below 100 ms: unrealistic; PlayerLoop injection + subscriber dictionary pre-alloc take time |
| `T_sl_ceiling` | 800 ms | [400, 1500] | Margin for larger saves / deeper migrations | Tightens — may require Save/Load migration performance budget adjustment |
| `T_input_ceiling` | 400 ms | [200, 800] | Margin for Input safe-area + `DeviceDisplayMetrics` populate | Below 200 ms: some Android OEMs query `Screen.dpi` asynchronously; tight enough to flake |

### Pre-unload handler ordering

| Parameter | Current Value | Safe Range | Effect of Reorder | Rationale |
|-----------|--------------|------------|-------------------|-----------|
| Event Bus priority | 10 | [5, 15] | Must run FIRST — its `Disposed` transition must complete before any other handler publishes events | Upstream of all other handlers |
| Save/Load priority | 20 | [15, 25] | Middle tier — after Event Bus (so late `gameplay.*` events still route to Save/Load) and before Input (so save-triggering input isn't lost) | Middle tier |
| Input priority | 30 | [25, 35] | Must run LAST — teardown stops action maps; subsequent handlers can no longer receive input events | Downstream of others |

### Cross-GDD `sortingOrder` budget (R14.g)

| Parameter | Current Value | Reallocation Risk | Authority |
|-----------|--------------|-------------------|-----------|
| Base gameplay band | sortingOrder [0, 9] | Overlap with HUD | Scene Manager + downstream gameplay GDDs |
| HUD band | sortingOrder [10, 19] | Overlap with modal | HUD GDD (undesigned) |
| Modal band | sortingOrder [20, 29] | Overlap with pause overlay | Modal/Dialog GDD (undesigned) |
| Pause overlay band | sortingOrder [30, 39] | Overlap with system overlay | Scene Manager (R9 + R14.g) |
| System overlay band | sortingOrder [40+] | Reserved for debug + accessibility overlays | Scene Manager (R14.g) — subject to Accessibility GDD additions at VS |

Dev-build scene validator enforces — cross-references Input System AC-EC-HT2 (duplicate-`sortingOrder` BLOCKING-in-dev validator). Band re-tuning requires coordinated amendment across affected UI GDDs (not unilateral).

### Configurable constants (rarely changed)

| Parameter | Current Value | Notes |
|-----------|--------------|-------|
| `BootSceneName` | `"Boot"` | Must match Build Settings index 0. CI build-step validator confirms. |
| `PauseOverlaySceneName` | `"PauseOverlay"` | Must match the additive overlay scene file name. Dev-build validator confirms no `EventSystem` / `DontDestroyOnLoad` / `sortingOrder` violation per R9 + R14.e. |

## Visual/Audio Requirements

Scene Manager has no direct ownership of visual or audio feedback. Transition-related assets and audio are owned by downstream systems:

- **Loading overlay visuals (fade-in/out animation, optional spinner)** — owned by Modal/Dialog GDD (undesigned). Scene Manager publishes `gameplay.scene.load_ready` with `activationCallback(true)` so Modal/Dialog can control the fade-out timing per R8 + D.4.
- **Pause overlay visuals (UI the player sees during in-game pause)** — owned by Main Menu/Settings/Pause GDD (undesigned). Scene Manager only additive-loads the scene and toggles `Time.timeScale`; it neither draws nor animates.
- **Pause audio fade (lowering SFX/music volume during in-game pause)** — owned by Audio Bus GDD (undesigned) via the split ownership rule in R14.f. Audio Bus subscribes to `gameplay.app.state_changed { isPaused: true }` and fades individual audio sources.
- **OS-background audio mute** — owned by Scene Manager directly (R10: `AudioListener.pause = true`). This is the only audio write Scene Manager performs, and it is platform-lifecycle-triggered, not designer-configurable.

Scene Manager has NO sounds, NO particles, NO screen-space UI, NO shaders. Any visual or audio change that occurs at a scene boundary is owned by the system responsible for that boundary.

**Asset Spec Flag: N/A** — Scene Manager produces no per-asset specifications. No `/asset-spec` run is required for this GDD.

## UI Requirements

Scene Manager has no direct UI ownership. UI surfaces that appear Scene-Manager-adjacent are owned by downstream systems:

- **Retry button, Pause button, Continue button, Level End screen** — owned by Main Menu / Settings / Pause GDD and Modal / Dialog + Level End GDD (both undesigned). These surfaces *invoke* Scene Manager methods (`BeginTransitionAsync`, `RequestPause`, `RequestUnpause`) synchronously — they are call-sites, not Scene Manager UI.
- **Loading overlay (the visual that covers the transition window)** — owned by Modal/Dialog GDD (undesigned). Scene Manager's only UI-adjacent contract is the R8 activation handshake `activationCallback(true)`.
- **Accessibility button (title-screen persistent button per Input System pass-5 B16)** — owned by Main Menu GDD (undesigned). Scene Manager is unaware of this button at the GDD level.
- **`EventSystem` — project-wide UI input gateway** — Scene Manager owns the INVARIANT that exactly one `EventSystem` exists (in the Boot scene's `DontDestroyOnLoad` hierarchy per R14.e) and that no other scene contains one (enforced by AC-INV2 BLOCKING CI). This is an architectural invariant, not a Scene Manager UI surface, but it IS a cross-GDD rule every UI system must respect.
- **Sort-order budget (base 0–9 / HUD 10–19 / modals 20–29 / pause overlay 30–39 / system 40+ per R14.g)** — Scene Manager owns the budget bands as a cross-GDD invariant. Individual band allocations within a system belong to that system's GDD. Re-tuning the budget requires coordinated amendment across affected UI GDDs.

**📌 UX Flag — Scene / App State Manager**: This system does NOT require a `/ux-design` UX spec because it has no user-facing screens. The systems that DO require UX specs in pre-production (Main Menu / Settings / Pause, Biome Map / Level Select, Modal / Dialog + Level End, HUD) are all downstream consumers and will each trigger their own `/ux-design` invocations when their GDDs are authored.

## Acceptance Criteria

Scene Manager's ACs were validated by `qa-lead` via /design-system Phase 4 mandatory consultation; pass-2 applied 6 BLOCKING + 6 RECOMMENDED; pass-3 `/design-review` applied 12 BLOCKING + 4 RECOMMENDED; pass-5 `/design-review` (verification-only) found 5 NEW BLOCKING + 16 RECOMMENDED line-edits applied via pass-6 author-led revision 2026-04-28. **56 ACs across 9 severity tiers** (pass-6 recount: H.1 36 + H.2 3 + H.3 9 + H.4 2 + H.5 6 = 56 — pass-3 was 49; pass-6 net +7 from BLOCKING #2 AC-BOOT-FEPM split into A+B sub-cases (+1 in H.1), BLOCKING #3 AC-R10c/R10d each split into 3 tier-specific fixtures (+4 in H.1), BLOCKING #5 AC-R9e tier promotion ADVISORY → BLOCKING CI PLAYMODE (not a count change), R3 added AC-R9g RequestUnpause Pillar 5 latency ADVISORY (+1 in H.1), R11 promoted AC-R12d from coverage gap into formal H.3 entry (+1 in H.3)), distributed per the project's standard gate pattern. Coverage prioritizes: R1/R6 grep enforcement, bootstrap ordering (R2/R2.a/R2.b/R3 — including the Fast-Enter-Play-Mode smoke test introduced by R2.a, with pass-3 R2.b rationale rewrite to "stale AsyncOperation in static field"), three-channel event taxonomy (R5 with `>= 0.9f` correction), pre-unload hook contract (R7 with concrete max-vs-sum criterion + pass-3 BLOCKING #5 async-contract constraints), activation handshake (R8 with asymmetric tolerance windows), pause lifecycle (R9 with atomic timeScale ordering + pass-3 BLOCKING #7/#11 RequestUnpause sequence + memory cycle), OS-background path (R10 + D.3 three-tier threshold via pass-3 BLOCKING #13 + pass-3 BLOCKING #1 user-pause preservation + pass-3 BLOCKING #4 iOS OnApplicationFocus path + AC-EC-E9 mid-transition + AC-EC-E18 storage-invalidated), retry performance (R11 on ADR-003 device with pass-3 REC-4 10-retry blocking + 20-retry P1 soak), re-entrancy (R12 + AC-R12c firing order + pass-3 BLOCKING #10 destination guard), app-quit (R13 with pass-3 BLOCKING #5 sync teardown contract), invariants (R14.d/e/f + pass-3 BLOCKING #12 InputSystemUIInputModule), all five formulas (D.1 with pass-3 BLOCKING #3 per-term overrun, D.2–D.5 stable), edge cases (E3/E13/E14a/E14b/E15/E9/E18), and cross-GDD integration with each Foundation sibling.

### H.1 Rule Compliance (36 ACs — pass-6 BLOCKING #2 AC-BOOT-FEPM-A + AC-BOOT-FEPM-B (+1); pass-6 BLOCKING #3 AC-R10c/R10d each splits to 3 fixtures (Rush/Prep/General) (+4); pass-6 R3 adds AC-R9g RequestUnpause Pillar 5 latency (+1); pass-6 BLOCKING #5 AC-R9e ADVISORY → BLOCKING CI PLAYMODE (no count change))

- **AC-R1 — No Direct LoadSceneAsync Call-Site (R6)**. GIVEN the full codebase, WHEN a CI grep scans for `SceneManager\.LoadSceneAsync\|SceneManager\.UnloadSceneAsync` across all `.cs` files, THEN zero matches appear outside `AppStateManager.cs`. **BLOCKING CI**
- **AC-R2 — Async-Only Primitive: No Blocking LoadScene (R1)**. GIVEN a PlayMode test, WHEN `BeginTransitionAsync` runs, THEN `SceneManager.LoadScene` (sync overload) is never called; main thread is never blocked by `Task.Wait()` or equivalent in the transition pipeline. **BLOCKING CI (PLAYMODE)**
- **AC-BOOT-R2 — Bootstrap Sequence Order (R2)**. GIVEN `BootLoader.Awake()` fires, WHEN initialization is sequence-logged, THEN `DataRegistry.Initialize()` completes before `EventBus.Initialize()` starts, EB before SL, SL before Input, and `BeginTransitionAsync("MainMenu")` is called only after all four `Ready` tasks fire. **BLOCKING CI (PLAYMODE)**
- **AC-BOOT-FEPM — Fast Enter Play Mode Smoke Test (R2.a) (pass-3 REC-2 — CI prerequisite documented; pass-6 BLOCKING #1+#2 — bit-encoding correction + `[SetUp]` enforcement)**. **CI prerequisite (pass-6 corrected bit encoding):** before this AC can run, `ProjectSettings/EditorSettings.asset` MUST contain `m_EnterPlayModeOptionsEnabled: 1` AND `m_EnterPlayModeOptions: 1` — which encodes `EnterPlayModeOptions.DisableDomainReload` per Unity 6 serialization (the flag enum: `DisableDomainReload = 1`, `DisableSceneReload = 2`, both = `3`). **Pass-5 BLOCKING #1 caught a pass-4 author defect**: pass-4 wrote `m_EnterPlayModeOptions: 2` annotated as "encodes `DisableDomainReload`" — value `2` actually encodes `DisableSceneReload`, leaving Domain Reload enabled, exactly the false-positive coverage this AC was designed to prevent. CD recommended value `1` (DisableDomainReload only — Scene Reload may run, which is the harder case for `Reset()` discipline because static fields persist across cycles while scene state is freshly reloaded each Play). The settings file change MUST be committed and the CI runner must check out a tree containing those settings. The settings flag is a one-time project-level commit, not a per-test fixture. **`[SetUp]` enforcement (pass-6 BLOCKING #2):** every test fixture for this AC MUST execute the following assertion in its `[SetUp]` (or equivalent fixture-pre-test method) BEFORE invoking the Play→Stop→Play sequence: `Assert.IsTrue(EditorSettings.enterPlayModeOptionsEnabled, "FEPM disabled — AC requires enabled flag"); Assert.AreEqual(EnterPlayModeOptions.DisableDomainReload, EditorSettings.enterPlayModeOptions & EnterPlayModeOptions.DisableDomainReload, "DisableDomainReload bit not set in project settings");`. The bitwise mask preserves correctness across both sub-cases below. Without the `[SetUp]` assertion, a developer (or CI agent) who silently flips the flag off in `ProjectSettings.asset` for any reason produces a green test that is not actually exercising domain-reload-disabled behavior — the same recursive false-positive class as the bit-encoding miss. **Two sub-cases (separate test fixtures, both BLOCKING):**
   - **Sub-case A — `DisableDomainReload` only (`m_EnterPlayModeOptions: 1`, Scene Reload enabled).** The harder regression case — domain reload disabled means static state persists across Play cycles; Scene Reload is enabled so each Play loads scenes fresh. `[SetUp]` mask asserts bit `1` is set. GIVEN this configuration, WHEN Play is pressed twice in sequence (Play → Stop → Play), THEN the second cycle completes `BootLoader.Awake()` without `InvalidOperationException` or duplicate-singleton warning; all four Foundation siblings reach `Ready` in both cycles; `R2.a Reset()` clears static fields between cycles.
   - **Sub-case B — both Domain Reload AND Scene Reload disabled (`m_EnterPlayModeOptions: 3`).** Same regression target plus scene state persistence. `[SetUp]` mask still asserts bit `1` is set (the AND with bit `2` for Scene Reload is asserted via a separate `Assert.AreEqual((EnterPlayModeOptions)3, EditorSettings.enterPlayModeOptions, ...)` in this sub-case's `[SetUp]`). GIVEN this configuration, WHEN Play is pressed twice in sequence, THEN the second cycle completes identically; the Boot scene's static `EventSystem` reference (R14.e) is not duplicated across cycles, and `BootLoader.Awake()` correctly re-initializes despite the scene being persisted from cycle 1.

   Both sub-cases are BLOCKING CI. Failing either means the FEPM regression-prevention contract is broken on at least one valid Unity FEPM configuration. **BLOCKING CI** (new pattern introduced by this GDD)
- **AC-BOOT-RIOLM — RIOLM Usage Restriction (R3)**. GIVEN the codebase, WHEN a CI grep scans for `[RuntimeInitializeOnLoadMethod`, THEN every match uses only `SubsystemRegistration` or `AfterAssembliesLoaded` tier; matches at `BeforeSplashScreen`, `AfterSceneLoad`, or `BeforeSceneLoad`, or any `SubsystemRegistration` match with MonoBehaviour instantiation, fail CI. **BLOCKING CI**
- **AC-R4 — AppState Sole-Writer Invariant (R4)**. GIVEN the codebase, WHEN a CI grep scans for `AppState =` or `IsPaused =` assignments, THEN zero matches appear outside `AppStateManager.cs`. **BLOCKING CI**
- **AC-R5 — Three-Channel Publish Ordering (R5) (pass-2 float-equality fix)**. GIVEN a PlayMode test with all three channels subscribed, WHEN `BeginTransitionAsync` completes, THEN events fire in strict order: (1) `gameplay.scene.will_unload` before any unload work; (2) `gameplay.scene.load_ready` when `AsyncOperation.progress >= 0.9f` (NOT exact equality — Unity progress can step from 0.85 to 0.95 within a single frame; IEEE 754 float comparison `== 0.9` is unreliable), AND `allowSceneActivation` is still `false` at fire time; (3) `gameplay.app.state_changed` only after `sceneLoaded` confirms the new scene is active and `AppState` is updated. Strict ordering is enforced by the implementation via `await` sequencing (not by timestamp comparison alone — same-frame events would defeat timestamp-based ordering). **BLOCKING CI (PLAYMODE)**
- **AC-R6 — BeginTransitionAsync Is Only Legal Entry (R6)**. Structural check complementing AC-R1: CI validates that `AppStateManager.BeginTransitionAsync` exists as the single callable public API surface for scene changes. **BLOCKING CI**
- **AC-R7 — Pre-Unload `Task.WhenAll` + 1000 ms Timeout (R7, D.2) (pass-2 concrete criterion; pass-6 R9 — dual-bound disambiguation)**. GIVEN three `IPreUnloadHook` handlers at priorities 10/20/30 with mock task durations 50ms / 30ms / 70ms (introduced via `Task.Delay`), WHEN `BeginTransitionAsync` runs, THEN: (a) all three handlers are invoked (verified by sequence counters incremented in each handler); (b) the wall-clock elapsed from first-handler-invoke to all-handlers-complete is `≤ max(50, 30, 70) + 70ms = 140ms` (max-with-tolerance — pass-6 R9 widens the upper bound from 120ms to 140ms because the pass-3 [120, 130) ambiguity created a 10ms dead-zone where the test was undefined: a 130ms result was simultaneously failing check (b) "elapsed > 120ms" AND failing check (c) "elapsed < 130ms" boundary, but check (b)'s strict `≤` made the boundary inclusive — leaving 130ms in a logically-impossible state. Widening (b) to ≤140ms eliminates the dead-zone and gives check (c) — the parallelism proof — a clean 10ms gap to operate in. The 70ms tolerance still catches a serialized-execution regression: serial would be 150ms+ which fails (b), and pseudo-parallel-with-mid-handler-stall would be ≥130ms which fails (c)); (c) the elapsed is `< sum(50, 30, 70) - 20ms = 130ms` — proves serialization did not occur (the test fails if elapsed is in [130, ∞)); (d) a never-completing handler test fixture verifies cancellation fires at 1000ms ± 100ms with BLOCKING log in DEVELOPMENT_BUILD; Scene Manager proceeds with unload. **BLOCKING CI (PLAYMODE)** — note: AC-D2 (EditMode) provides the cleaner max-vs-sum proof in pure C#; AC-R7's role is to verify the integration in a Unity scene context.
- **AC-R7a — `WaitForSeconds` Prohibition in Hooks (R7)**. GIVEN the codebase, WHEN a CI grep scans `IPreUnloadHook` implementations for `yield return new WaitForSeconds`, THEN zero matches appear. **BLOCKING CI**
- **AC-R8 — Activation Handshake: Single-Subscriber + 500 ms Fallback (R8, D.4) (pass-2 asymmetric tolerance)**. GIVEN a PlayMode test with a mock `load_ready` subscriber that invokes `activationCallback(true)` after 200 ms, WHEN a transition fires, THEN `allowSceneActivation = true` is set within `[150, 400]` ms after `load_ready` fires (lower bound 150ms catches "fired too early — Modal/Dialog wasn't waited on"; upper bound 400ms absorbs game-ci scheduling jitter on GitHub Actions runners while still catching real regressions). A separate test with no subscriber sets activation within `[400, 700]` ms (timeout fallback path). **BLOCKING CI (PLAYMODE)** — the asymmetric windows replace the pass-1 ±50ms symmetric tolerance which was flake-prone on game-ci/unity-test-runner@v4.
- **AC-R8a — Exactly-One Subscriber Enforcement (R8, E8)**. GIVEN a DEVELOPMENT_BUILD with two subscribers registered for `gameplay.scene.load_ready`, WHEN a transition fires, THEN a BLOCKING error `[SceneManager] Exactly one subscriber required for gameplay.scene.load_ready (found: 2)` is logged; the first-registered subscriber's callback is used; no exception thrown. **PLAYMODE**
- **AC-R9a — Double-Pause Guard (R9)**. GIVEN `IsPaused == true`, WHEN `RequestPause()` is called again, THEN no-op — `IsPaused` remains true, `PauseOverlay` is not loaded a second time, no `app.state_changed` fires, no error thrown. **BLOCKING CI (PLAYMODE)**
- **AC-R9b — Pause Overlay Lifecycle Order (R9) (pass-2 timing fix + sub-millisecond timestamps)**. GIVEN `AppState == Level` and `IsPaused == false`, WHEN `RequestPause()` is called, THEN — verified via a sequence-counter log (integer step counter, NOT wall-clock timestamps; `System.Diagnostics.Stopwatch` provides sub-millisecond fallback if a tie-break is needed) — the steps execute in this order: (1) `IsPaused = true` AND `Time.timeScale = 0f` happen ATOMICALLY (same statement, same step counter — verified by reading both values at step counter increment); (2) `LoadSceneAsync("PauseOverlay", Additive)` is invoked AFTER step 1; (3) `PauseOverlay.sceneLoaded` callback fires AFTER step 2; (4) `app.state_changed { isPaused: true }` fires LAST. Wall-clock-only ordering verification is rejected — same-frame events would defeat timestamp comparison. **BLOCKING CI (PLAYMODE)**
- **AC-R9c — Concurrent-Load Guard (R9, R12, R14.c)**. GIVEN `_transitionInProgress == true`, WHEN `RequestPause()` is called, THEN the pause request is placed in `_pendingPause` queue and fires only after the transition completes; `AppState` never transitions from `Loading` directly to `Paused`. **PLAYMODE**
- **AC-R9d — Pause Overlay Scene Constraints (R9, R14.e, R14.g)**. GIVEN `PauseOverlay.unity` is loaded additively, WHEN a CI scene-validation script inspects it, THEN (a) it contains no `EventSystem` component; (b) all Canvas components have `sortingOrder ∈ [30, 39]`; (c) no `DontDestroyOnLoad` is called from any of its scripts. **BLOCKING CI**
- **AC-R9e — RequestUnpause Step Sequence (R9 RequestUnpause; pass-3 BLOCKING #7 promotion from coverage gap to ADVISORY; pass-6 BLOCKING #5 — tier promotion ADVISORY → BLOCKING PLAYMODE)**. GIVEN `IsPaused == true` and `PauseOverlay` loaded, WHEN `RequestUnpause()` is called, THEN — verified via sequence-counter log — the steps execute in this order: (1) double-unpause guard checked (`IsPaused == false` → early no-op verified separately by calling `RequestUnpause()` twice in succession); (2) `UnloadSceneAsync("PauseOverlay")` invoked AND awaited to completion; (3) `Resources.UnloadUnusedAssets()` invoked AND awaited (verified by reading the returned `AsyncOperation.isDone == true` after the `await`); (4) ATOMIC `Time.timeScale = 1f` AND `IsPaused = false` (same statement, same step counter); (5) `app.state_changed { isPaused: false }` fires LAST. **BLOCKING CI (PLAYMODE)** — pass-6 BLOCKING #5 promotion (qa-lead + game-designer convergent + CD adjudicated): missing the double-unpause guard (step 1) creates the same permanent `Time.timeScale = 0f` freeze failure mode as missing the double-pause guard in `RequestPause()`. AC-R9a (pause direction) is already BLOCKING CI PLAYMODE; the asymmetry of leaving the unpause-direction symmetry as ADVISORY was unjustified — same bug class, same severity. CD: "Tier should follow failure severity." Replaces the pass-3 ADVISORY tier; pass-3 BLOCKING #7 promotion language ("from coverage gap to ADVISORY") is now superseded — the AC is BLOCKING CI PLAYMODE, no longer ADVISORY-tier.
- **AC-R9f — Pause Overlay Memory Stability Across Cycles (R9 RequestUnpause; pass-3 BLOCKING #11; pass-6 R1 + R2 — tolerance band widened to 5MB; baseline timing disambiguated)**. GIVEN a PlayMode test that records `GC.GetTotalMemory(forceFullCollection: true)` **immediately after the first `RequestUnpause()` returns and `gameplay.app.state_changed { isPaused: false }` has fired** (post-warm-up unpause baseline — the warm-up cycle is exactly one `RequestPause()` + `RequestUnpause()` round-trip preceding the measurement, completed before the baseline read; pass-6 R2 disambiguates the pass-3 ambiguity about whether "baseline" was post-pause or post-unpause), WHEN 10 additional `RequestPause()` + `RequestUnpause()` cycles execute back-to-back (total 11 cycles), THEN `GC.GetTotalMemory(true)` after cycle 11 is within `[baseline - 1MB, baseline + 5MB]` (asymmetric tolerance — pass-6 R1 widens the upper bound from 2MB to **5MB** to absorb Mali-G52 Mono GC noise floor: empirical Mono GC behavior on Cortex-A53/A55 ARM with 256-512MB heap shows ~0.5MB-per-cycle steady-state working-set churn that does not represent a real leak; 2MB ceiling produced flaky CI failures on game-ci/unity-test-runner@v4 across 10 cycles. The 5MB ceiling still catches the failure mode without `await Resources.UnloadUnusedAssets()` (5–20 MB typical drift) while ignoring noise-floor thrash). **PLAYMODE** — measures the pass-3 BLOCKING #11 fix actually prevents accumulation. Failure mode without `await Resources.UnloadUnusedAssets()` in `RequestUnpause` step 3: typical drift is 5–20 MB across 10 cycles (pause-overlay textures + audio clips not reclaimed). The test runs the warm-up cycle separately to isolate steady-state cycle leak from first-run setup.
- **AC-R9g — RequestUnpause Pillar 5 Latency Bound (R9 RequestUnpause + Pillar 5; pass-6 R3 — new ADVISORY)**. GIVEN `IsPaused == true` and `PauseOverlay` loaded, WHEN `RequestUnpause()` is invoked AND the next player tap fires immediately on the post-`isPaused: false` `app.state_changed`, THEN `t_unpause_to_first_input_event ≤ 100 ms` at 60 fps (`≤ 133 ms` at 30 fps) — measured as wall-clock from `RequestUnpause()` entry to `InputDispatcher` first-event-emit on the next tap, on the ADR-003 reference Android device. The Pillar 5 Feedback Floor applies at the seam from paused-back-to-active gameplay just as it does at fresh scene entry (AC-INT-FF). The `await Resources.UnloadUnusedAssets()` step 3 introduced by pass-3 BLOCKING #11 has nominal cost of 5–30 ms on Mali-G52 warm cache; cold cache or a paged-out PauseOverlay scene asset working-set may push the call to 60–80 ms, eating most of the 100 ms budget. AC-R9g surfaces this risk as a dashboard ADVISORY rather than a gate (the gate is AC-R9f memory stability). **DEVICE (ADVISORY)** — non-gating, blocked on ADR-003. Failure mode (advisory-log only): `[SceneManager] RequestUnpause → first-input latency {ms}ms exceeds Pillar 5 100ms target`.
- **AC-R10a — OS Background Forced-Pause Path (R10)**. GIVEN `AppState == Level` and `IsPaused == false`, WHEN `OnApplicationPause(true)` fires, THEN six-condition assertion passes: (1) `SaveManager.FlushSync()` called synchronously before any state change; (2) `IsPaused = true`; (3) `AudioListener.pause = true`; (4) `_backgroundedAtUtc` set to non-default `DateTime.UtcNow`; (5) NO PauseOverlay load; (6) NO `app.state_changed` fires. Verified by a PlayMode test using Unity's `RuntimePlatform` simulation + mock `SaveManager`. **BLOCKING CI (PLAYMODE)**
- **AC-R10b — OS Background + Already Paused: User-Pause Preservation (R10, E9) (pass-3 BLOCKING #2 — promoted ADVISORY → BLOCKING CI (PLAYMODE); pass-6 R10 — condition (c) rewritten as positive state assertion)**. GIVEN user-paused (`IsPaused == true`) + `OnApplicationPause(true)` fires, WHEN resume fires with elapsed below threshold, THEN: (a) at OS-background time, `_userPausedBeforeBackground = true` is captured (R10 step 2); (b) on resume, `IsPaused = _userPausedBeforeBackground = true` is restored — the user's pause state is preserved; (c) **PauseOverlay scene IS still loaded after resume (positive assertion):** `SceneManager.GetSceneByName("PauseOverlay").isLoaded == true`. Pass-3 condition (c) was a no-op-with-N/A — it asserted "PauseOverlay is NOT auto-loaded" with a parenthetical explaining the OS-background path does not unload it, leaving nothing concrete to assert. Pass-6 inverts to a positive state assertion: the OS-background path preserves the additive PauseOverlay scene across the round trip (it does not unload — the scene's `Awake` already ran pre-background, and `Time.timeScale = 0` does not destroy scenes), so on resume the overlay is still present in the loaded-scenes list and renders against the restored `IsPaused = true` state without a fresh load. Failure mode this catches: a future refactor that unloads PauseOverlay on background (e.g., to free memory) would silently break the user's pause state on resume — they'd see the un-paused gameplay flash for a frame before any code re-pauses, violating the seam fantasy. The pass-2 ADVISORY tier was wrong because user-pause silent clearance is a P0 contract violation (Section B "seams are where the chaos isn't" fantasy + Pillar 2 player-trust); BLOCKING CI ensures the regression cannot ship. **BLOCKING CI (PLAYMODE)** — uses Unity's `Application.RuntimePlatform` simulation hooks; runs on game-ci/unity-test-runner@v4.
- **AC-R10c — Session Expiry Above Threshold (R10, D.3) (pass-6 BLOCKING #3 — split into three tier-specific fixtures replacing stale `bg_session_expiry_seconds`)**. The pass-3 BLOCKING #13 three-tier resolution introduced `bg_expiry_in_rush_seconds` / `bg_expiry_in_prep_seconds` / `bg_expiry_general_seconds` (Section G + D.3) but the pass-2-era ACs continued to reference the obsolete single-threshold name `bg_session_expiry_seconds`. Pass-6 splits each AC into three fixtures, one per tier, asserting against the applicable tier threshold. Each fixture is a separate test method (e.g., `[Test] AboveThreshold_InRush_TransitionsToMainMenu`) so a failure pinpoints which tier regressed. Fixtures (all PLAYMODE):
   - **AC-R10c-Rush — In-Rush Above Threshold.** GIVEN `_backgroundedDuringRush == true` AND elapsed > `bg_expiry_in_rush_seconds` (default 1800 sec), WHEN `OnApplicationPause(false)` fires, THEN `BeginTransitionAsync("MainMenu")` is called; `IsPaused` is NOT set to false in place; `_userPausedBeforeBackground` is implicitly discarded.
   - **AC-R10c-Prep — In-Prep Above Threshold.** GIVEN `_backgroundedDuringPrep == true` AND elapsed > `bg_expiry_in_prep_seconds` (default 600 sec), WHEN `OnApplicationPause(false)` fires, THEN `BeginTransitionAsync("MainMenu")` is called; same post-conditions as Rush variant.
   - **AC-R10c-General — General Above Threshold.** GIVEN both `_backgroundedDuringRush` and `_backgroundedDuringPrep` are `false` (MainMenu / BiomeMap / Shop) AND elapsed > `bg_expiry_general_seconds` (default 60 sec), WHEN `OnApplicationPause(false)` fires, THEN `BeginTransitionAsync("MainMenu")` is called.

   **PLAYMODE** (each of the three fixtures)
- **AC-R10d — Session Expiry Below Threshold (R10, D.3) (pass-6 BLOCKING #3 — split into three tier-specific fixtures replacing stale `bg_session_expiry_seconds`)**. Mirror of AC-R10c for the below-threshold in-place-resume branch. Each fixture asserts against the applicable tier threshold. Fixtures (all PLAYMODE):
   - **AC-R10d-Rush — In-Rush Below Threshold.** GIVEN `_backgroundedDuringRush == true` AND elapsed < `bg_expiry_in_rush_seconds`, WHEN resume fires, THEN in-place resume: `AudioListener.pause = false`; `IsPaused = _userPausedBeforeBackground` (per pass-3 BLOCKING #1); no `BeginTransitionAsync`.
   - **AC-R10d-Prep — In-Prep Below Threshold.** GIVEN `_backgroundedDuringPrep == true` AND elapsed < `bg_expiry_in_prep_seconds`, WHEN resume fires, THEN in-place resume with same post-conditions; player's Prep planning state is preserved.
   - **AC-R10d-General — General Below Threshold.** GIVEN both `_backgroundedDuringRush` and `_backgroundedDuringPrep` are `false` AND elapsed < `bg_expiry_general_seconds`, WHEN resume fires, THEN in-place resume; effectively `IsPaused = false` since pause is not valid in MainMenu/BiomeMap/Shop per E7.

   **PLAYMODE** (each of the three fixtures)
- **AC-R10e — Clock Drift Edge Cases (D.3, E11, E12)**. GIVEN three fixtures: (a) `_backgroundedAtUtc == default` (resume without prior background per E11); (b) elapsed is negative (NTP clock-backward per E12); (c) elapsed > 86400 sec (clock-forward per E12), WHEN `OnApplicationPause(false)` fires, THEN (a) → no-op; (b) → clamped to 0, in-place resume; (c) → unconditional `MainMenu` transition. Three assertions in one EditMode test. **EDITMODE**
- **AC-R11 / AC-P-Retry — Retry Performance on Reference Device (R11, D.1) (pass-3 REC-4 — raised to 10 retries; 20-retry soak deferred to P1 post-Alpha)**. GIVEN the ADR-003 reference Android device (Mali-G52 class) in steady-state (warmed up by 60 sec of Level gameplay; thermal junction temp logged), WHEN **ten** consecutive Level → Level retries are measured via wall-clock instrumentation, THEN: (a) **max** of all ten ≤ 2000 ms (pass-3 REC-4 raises the sample size from 5 to 10 because n=5's `max` was statistically thin — a 1-in-100 tail event is invisible at n=5 but appears reliably at n=10; CD recommended 10 for the BLOCKING tier with 20 deferred to P1 soak rather than blocking MVP); (b) target **mean ≤ 1500 ms** with 500 ms headroom; (c) **monotonic-increase advisory check**: if any retry-N+1 exceeds retry-N by more than 300 ms, log a thermal-degradation ADVISORY warning (does not fail the gate but flags for performance regression dashboard); (d) per-retry `T_sm_actual`, `T_lr_actual`, `T_sm_overrun`, `T_lr_overrun`, `T_slack_consumed` (per pass-3 BLOCKING #3 D.1) are reported individually. **DEVICE** (blocked on ADR-003). **20-retry soak deferred to P1 post-Alpha** — runs as a separate sprint task during pre-launch perf soak, NOT a blocking gate; tracked as a P1 sprint task in the production backlog (`production/sprints/post-alpha/perf-soak-20-retry.md` to be authored when P1 sprint begins).
- **AC-R12a — Re-Entrancy Single-Slot Queue (R12)**. GIVEN a transition in progress (`_transitionInProgress == true`), WHEN `BeginTransitionAsync("Level")` is called twice in rapid succession (double-tap simulation), THEN exactly one retry transition executes; the second call overwrites the first in `_pendingTransition` with a dev warning; only one `app.state_changed` fires for the Level → Level cycle. **PLAYMODE**
- **AC-R12b — Invalid Scene Name Handling (E5)**. GIVEN `BeginTransitionAsync("NonExistentScene")` is called, WHEN Unity throws `ArgumentException`, THEN Scene Manager catches; logs `[SceneManager] Invalid scene: NonExistentScene` as BLOCKING; clears `_transitionInProgress`; `_pendingPause` queue is preserved; no crash. **PLAYMODE**
- **AC-R13 — App-Quit Teardown + Disposed State (R13) (pass-2 explicit verification step + Editor guard)**. GIVEN `Application.quitting` fires, WHEN `OnApplicationQuit()` executes, THEN: (1) pre-unload hook sequence runs synchronously — `Task.Wait()` permitted in Player builds, FORBIDDEN in Editor builds (uses `#if UNITY_EDITOR` synchronous-only path per R13); (2) `SaveManager.FlushSync()` called; (3) `Time.timeScale = 1f`; (4) `AudioListener.pause = false`; (5) `AppState = Disposed`; (6) **VERIFIED EXPLICITLY**: a subsequent test-fixture call to `BeginTransitionAsync("MainMenu")` after `AppState = Disposed` returns immediately without executing the transition, logs `[SceneManager] Cannot transition from Disposed state`, does NOT change `AppState`, and throws no exception. The pass-1 condition 6 ("AppState stays Disposed — no further transition can fire") was unverifiable as written; this revision makes the rejection path testable. **PLAYMODE**
- **AC-INV1 — Exclusive `Time.timeScale` Write (R14.d, E16) (pass-2 scope clarification)**. GIVEN the full codebase, WHEN a CI grep scans for `Time\.timeScale\s*=` across all `.cs` files outside `AppStateManager.cs`, THEN zero matches appear. **BLOCKING CI**. **NOTE — best-effort syntactic check.** The grep does NOT catch: reflection-based assignment (`typeof(Time).GetProperty("timeScale").SetValue(null, 0f)`); aliased wrappers (`static void SetTimeScale(float v) { Time.timeScale = v; }` defined outside `AppStateManager.cs` but called from anywhere); or any future API-renamed equivalent. Reflection bypass is covered by code review. The runtime defense is E16's `RequestUnpause` `Time.timeScale == 0f` precondition assertion in `DEVELOPMENT_BUILD`.
- **AC-INV2 — EventSystem Single-Instance + InputSystemUIInputModule (R14.e) (pass-3 BLOCKING #12 — added condition (d))**. GIVEN the Boot scene and all non-Boot scenes (MainMenu, BiomeMap, Level, Shop, PauseOverlay), WHEN a CI scene-validation script inspects each scene file, THEN: (a) the Boot scene contains exactly one `EventSystem` component; (b) all non-Boot scenes contain ZERO `EventSystem` components; (c) the Boot scene's `EventSystem` GameObject has an `InputSystemUIInputModule` component attached; (d) the same `EventSystem` GameObject has NO `StandaloneInputModule` component (the legacy Input Manager UI module). All four conditions are BLOCKING — any failing scene blocks the build. **BLOCKING CI** (build-time scene validator). The pass-3 BLOCKING #12 conditions (c)+(d) prevent the silently-dropped-UI-events failure mode that occurs when both UI modules coexist on the same EventSystem.
- **AC-INV3 — `AudioListener.pause` Split Ownership (R14.f, R10)**. GIVEN `RequestPause()` is called for in-game pause, WHEN `Time.timeScale = 0f` is set, THEN `AudioListener.pause` is NOT written by `AppStateManager` on this path (its value is unchanged — Audio Bus owns in-game audio fade via `app.state_changed`). Verified by inspecting `AudioListener.pause` before and after `RequestPause()`. **PLAYMODE**

### H.2 Formula Correctness (3 ACs)

- **AC-D1 — Retry Budget Per-Line-Item Attribution + Per-Term Overrun (D.1) (pass-3 BLOCKING #3)**. GIVEN a DEVELOPMENT_BUILD, WHEN a Level → Level retry completes, THEN: (a) the budget analyzer logs at minimum four per-line-item entries — scene-unload duration, `LoadSceneAsync` duration, activation-handshake duration, pre-unload-hooks duration; each entry includes its value in ms; total `T_sm` equals their sum; (b) `T_sm_overrun = max(0, T_sm - 1100)` is logged unconditionally when `> 0`, INDEPENDENT of whether `T_slack_consumed > T_slack_budget` (this defends against the false-negative case where `T_sm` overage offsets a `T_lr` undershoot in the naive formula); (c) similarly `T_lr_overrun = max(0, T_lr - 600)` is logged unconditionally when `> 0`; (d) `T_slack_consumed = T_sm_overrun + T_lr_overrun` is logged; (e) the `T_sm ≤ 1100 ms` advisory is logged but is no longer used as the gate (the gate is `T_retry ≤ 2000 ms` per R11). **PLAYMODE** — verified with three test fixtures matching D.1's three worked examples (warm cache, cold cache + thermal, pathological pre-fix mask scenario).
- **AC-D2 — Pre-Unload Aggregate Is Max Not Sum (D.2)**. GIVEN three mock handlers with simulated durations 50 ms / 30 ms / 70 ms running in parallel via `Task.Delay`, WHEN `BeginTransitionAsync` invokes `Task.WhenAll`, THEN the measured aggregate window is ≈ 70 ms (max), NOT ≈ 150 ms (sum); ±25 ms tolerance on CI hardware. **EDITMODE**
- **AC-D3 — Bootstrap Cumulative ADVISORY Log (D.5)**. GIVEN a DEVELOPMENT_BUILD, WHEN `BootLoader.Awake()` completes, THEN per-sibling timing (`T_dr`, `T_eb`, `T_sl`, `T_input`) is logged; if cumulative `T_boot > 3000 ms`, the advisory log `[BootLoader] Boot duration {ms}ms exceeded T_boot_target 3000ms` fires (not a blocking error); build continues. **ADVISORY**

### H.3 Edge Cases (9 ACs — pass-3 split AC-EC-E14 → AC-EC-E14a + AC-EC-E14b; pass-6 R11 promoted AC-R12d from coverage gap to formal entry (+1))

- **AC-EC-E3 — Late Hook Registration Rejected (E3)**. GIVEN a Foundation sibling attempts to call `RegisterPreUnloadHandler` AFTER `BeginTransitionAsync("MainMenu")` has fired, WHEN the call is made, THEN the registration is silently ignored; a dev warning fires in DEVELOPMENT_BUILD; no crash; the hook does NOT run on this or any future transition. **ADVISORY**
- **AC-EC-E13 — Foundation Init Failure (E13)**. GIVEN `EventBus.Initialize()` throws during `BootLoader.Awake()`, WHEN the exception propagates, THEN `BootLoader` catches; logs `[BootLoader] Foundation sibling EventBus failed to initialize: {exception}`; does NOT call `BeginTransitionAsync("MainMenu")`; process exits (or in Editor, play mode stops). Verified by a stub-injected throw on the second sibling. **PLAYMODE**
- **AC-EC-E14a — BootEnforcer EditMode Interception (E14) (pass-3 BLOCKING #6 — split from pass-2 AC-EC-E14)**. GIVEN the Unity Editor with `Level.unity` open in the hierarchy and Play mode NOT yet entered, WHEN a test fixture sets `EditorApplication.isPlaying = true` programmatically, THEN — verified by an EditMode test with `[InitializeOnLoad]` `BootEnforcer` registered — the `playModeStateChanged` callback fires with state `ExitingEditMode` BEFORE Play mode fully enters; the callback sets `EditorPrefs.SetString("MM.Editor.PostBootJumpTo", "Level")` AND replaces the active scene with `Boot.unity` via `EditorSceneManager.OpenScene("Assets/Scenes/Boot.unity", OpenSceneMode.Single)`; the EditorPrefs key value matches `"Level"` after the callback. Note: this AC verifies only the EditMode interception side — the post-jump verification (BootLoader reads the pref + transitions correctly) is owned by AC-EC-E14b. **EDITMODE** — Editor-only by definition; cannot run on a build server without an Editor license, but game-ci/unity-test-runner@v4 supports EditMode execution.
- **AC-EC-E14b — BootEnforcer PlayMode Post-Jump (E14) (pass-3 BLOCKING #6 — split from pass-2 AC-EC-E14)**. GIVEN `EditorPrefs.GetString("MM.Editor.PostBootJumpTo") == "Level"` is pre-set BEFORE Play mode enters AND the active scene is `Boot.unity`, WHEN Play mode runs `BootLoader.Awake()` and the R2.a → R2 init chain completes, THEN: (a) `BootLoader` reads `MM.Editor.PostBootJumpTo`, finds `"Level"`, clears the EditorPrefs key (`EditorPrefs.DeleteKey`), and calls `BeginTransitionAsync("Level")` instead of the default `BeginTransitionAsync("MainMenu")`; (b) all four Foundation siblings reach `Ready` before the transition fires (sequence-log assertion); (c) post-transition, the active scene is `Level.unity` with `AppState == AppState.Level`; (d) the EditorPrefs key is no longer set (`EditorPrefs.HasKey("MM.Editor.PostBootJumpTo") == false`). The pass-2 "programmatically trigger Play" step is dropped — Play mode triggering is owned by AC-EC-E14a (EditMode); this AC starts with Play mode already entered and verifies the post-jump behavior. **PLAYMODE** (Editor-only) — runs in the Editor's Play mode only; not a build target. The pass-1 `FoundationStub` prefab approach remains **DISCARDED** (see E14 + game-designer REC-4 + qa-lead R4).
- **AC-EC-E15 — Duplicate AppStateManager (E15)**. GIVEN a second `AppStateManager` instance is instantiated (simulated FEPM re-entry), WHEN its `Awake()` runs, THEN `[AppStateManager] Duplicate instance detected; destroying self` is logged; `Destroy(this.gameObject)` is called; the original instance continues operating with no state mutation. **PLAYMODE**
- **AC-EC-E9 — OS Background Mid-Transition (E9) (pass-2 added)**. GIVEN `_transitionInProgress == true` (transition is mid-flight), WHEN `OnApplicationPause(true)` fires, THEN: (a) `SaveManager.FlushSync()` is called per R10 step 1; (b) `IsPaused = true`; (c) `AudioListener.pause = true`; (d) `_backgroundedAtUtc` is set; (e) `_backgroundedDuringRush` is captured (false unless the transition was Level→Level retry from Rush phase); (f) Unity's `AsyncOperation` (the in-flight transition) remains in memory (not abandoned). On resume below threshold: the original transition either resumes from where Unity suspended it OR a fresh `BeginTransitionAsync` fires for the same target if the original is unrecoverable; either way `_transitionInProgress` is correctly cleared by `app.state_changed`. On resume above threshold: original transition is abandoned, `BeginTransitionAsync("MainMenu")` fires fresh. Verified by injecting OS-background mid-transition via Unity's lifecycle simulation. **PLAYMODE**
- **AC-EC-E18 — Storage Invalidated During OS Background (E18) (pass-2 added)**. GIVEN `OnApplicationPause(true)` fires AND `SaveManager.FlushSync()` throws `DirectoryNotFoundException` (mock-injected to simulate user clearing app data on Android), WHEN R10 executes, THEN: (a) Scene Manager catches the exception; (b) logs `[SceneManager] Save flush failed on OS background: storage invalidated` as BLOCKING; (c) continues the R10 path — `AudioListener.pause = true`, `IsPaused = true`, `_backgroundedAtUtc` recorded, `_backgroundedDuringRush` recorded; (d) does NOT crash. On resume, Save/Load's own `CorruptedRecovery` AC takes over. Verified by mock `ISaveManager.FlushSync` injection. **PLAYMODE**
- **AC-R12c — Pending Queue Firing Order (R12) (pass-2 added)**. GIVEN both `_pendingTransition` and `_pendingPause` are populated during a transition (simulating retry-double-tap + pause-tap scenario), WHEN the in-flight transition completes and `app.state_changed` fires, THEN: (a) `_pendingTransition` fires FIRST (the new retry transition begins); (b) `_pendingPause` is preserved through the new transition (it queues against the new `_transitionInProgress` cycle); (c) on the new transition's completion, `_pendingPause` then fires; (d) no pause-overlay-against-old-scene is loaded; (e) no `Loading + IsPaused-via-RequestPause` composite is ever observed. Verified by sequence-counter PlayMode test. **PLAYMODE**
- **AC-R12d — `_pendingPause` Destination Permission Guard (R12; pass-3 BLOCKING #10; pass-6 R11 — promoted from coverage gap to formal H.3 entry)**. GIVEN `_pendingPause` is queued during a transition that lands in MainMenu / BiomeMap / Shop (any non-Level state), WHEN the transition completes and `app.state_changed` fires for the new state, THEN: (a) `_pendingPause` is discarded — no `RequestPause()` call is dispatched; (b) Scene Manager logs `[SceneManager] _pendingPause discarded — destination AppState {newState} does not permit pause` as ADVISORY in `DEVELOPMENT_BUILD`; (c) no `PauseOverlay` scene is loaded; (d) `Time.timeScale` remains `1f`; (e) subsequent player input on the new scene proceeds normally. Verified by PlayMode test injecting a pause-tap during a transition to MainMenu. **PLAYMODE (DEFERRED)** — activates at Story 2 alongside AC-R9c (concurrent-load guard) — both verify the queue's correctness against destination state.

### H.4 Performance (2 ACs beyond AC-R11 — pass-2 added AC-P-UnityBoot)

- **AC-P-Boot — Foundation Cold-Boot Duration on Reference Device (D.5) (pass-2 t=0 reconciliation)**. GIVEN a cold-start launch (no background process, no Addressables warm cache) on the ADR-003 reference Android device, WHEN timed **from `BootLoader.Awake()` start to `BeginTransitionAsync("MainMenu")` call** (matches D.5's actual budget — Unity engine cold-boot is excluded), THEN `T_boot ≤ 3000 ms`; per-sibling attribution (`T_dr` / `T_eb` / `T_sl` / `T_input`) is logged. **DEVICE** (ADR-003 required). The pass-1 "OS process launch" t=0 was inconsistent with D.5's exclusion of Unity engine boot (~2-3s on Mali-G52); this rewrite measures only what D.5 actually budgets.
- **AC-P-UnityBoot — End-to-End Cold-Boot Visibility (ADVISORY) (pass-2 added)**. GIVEN a cold-start launch on the ADR-003 reference device, WHEN timed from OS process launch to `BootLoader.Awake()` start, THEN `T_unity_boot` is logged for visibility. No hard gate (Unity engine cold-boot is outside our control); flagged for performance regression dashboard if it crosses 3000ms (signals splash-screen / asset-bundle-preload bloat). Combined `T_total_cold_boot = T_unity_boot + T_boot` is logged for end-to-end visibility against the concept doc's "8-second player patience window" promise. **DEVICE (ADVISORY)** — no gate, dashboard only.

### H.5 Cross-System Integration (6 ACs — pass-2 corrected from pass-1 mis-count of 5)

- **AC-INT-EB — Event Bus Pre-Unload Hook (Event Bus dependency)**. GIVEN Event Bus registered as `IPreUnloadHook` priority 10, WHEN `BeginTransitionAsync` fires, THEN Event Bus `OnBeforeUnloadAsync` is called before priorities 20 and 30; Event Bus transitions to `Disposed` state before Scene Manager calls `LoadSceneAsync`; no events are dispatched on Event Bus channels after `Disposed`. Cross-references Event Bus AC-EC7. **PLAYMODE**
- **AC-INT-SL — Save/Load Fire-and-Forget on Retry (Save/Load dependency)**. GIVEN a Level → Level retry transition, WHEN Scene Manager calls `RequestFlush` (fire-and-forget), THEN Scene Manager does NOT await the flush before calling `LoadSceneAsync`; `RequestFlush` returns within 2 ms; the Save/Load background thread continues independently. Verified by mock `ISaveManager` recording await vs fire-and-forget. Cross-references Save/Load GDD AC-EC-Race1. **PLAYMODE**
- **AC-INT-Input — Input Teardown Before Activation (Input dependency)**. GIVEN Input registered as `IPreUnloadHook` priority 30, WHEN `BeginTransitionAsync` runs, THEN Input's `OnBeforeUnloadAsync` (disable action maps + unsubscribe) completes BEFORE `allowSceneActivation = true`; the new scene's `Awake` can safely call `InputDispatcher.Initialize()` without encountering an already-active action map. Cross-references Input System AC-EC-CFG. **PLAYMODE**
- **AC-INT-Level — Level Runtime Sub-Budget Gate**. GIVEN Level Runtime GDD is authored and its `T_lr ≤ 600 ms` BLOCKING AC is implemented, WHEN a Level scene load completes and Level Runtime populates `LevelData`, THEN Scene Manager's responsibility is to log `T_sm` so Level Runtime can verify the combined `T_sm + T_lr ≤ 1700 ms` budget (excluding slack). **PLAYMODE (DEFERRED — activates at Level Runtime GDD authoring)**
- **AC-INT-Modal — Activation Handshake with Modal/Dialog**. GIVEN Modal/Dialog GDD is authored and the loading-overlay subscriber is implemented, WHEN `gameplay.scene.load_ready` fires, THEN Modal/Dialog's `activationCallback(true)` is invoked after fade-out; Scene Manager receives the callback before the 500 ms timeout; `allowSceneActivation = true` fires immediately on callback (not after the full 500 ms). **PLAYMODE (DEFERRED — activates at Modal/Dialog GDD authoring)**
- **AC-INT-FF — Feedback Floor Handoff at Scene Boundary (Pillar 5 cross-GDD, added post-CD-GDD-ALIGN Condition 2; pass-2 single-owner clarification)**. **Owning GDD: Scene Manager** (this AC). Cross-references Input System AC-P1 — but the integration test owned by THIS AC. GIVEN a Level scene load completes and `gameplay.app.state_changed` fires, WHEN the player's first tap occurs after activation, THEN: (a) `InputDispatcher` is callable (`enabled == true`, action maps active) within 1 frame of `sceneLoaded` — verified by reading `InputDispatcher.IsReady` on the next `Update()` after `sceneLoaded` callback fires; (b) the first tap registered in the post-`sceneLoaded` frame is delivered to subscribers without being silently dropped — verified by tap-injection fixture firing immediately on `app.state_changed` and asserting at least one subscriber receives the event. Scene Manager's portion: Input is enabled and no taps are dropped. **Input System's portion (Input AC-P1) is the sub-100ms latency assertion** — that test belongs to Input AC-P1 and lives in `tests/playmode/input/`. If the integration test fails, the bug owner is determined by which side's assertion failed: failing (a) or (b) → Scene Manager bug; failing the latency assertion in Input AC-P1 → Input System bug. The pass-1 dual-ownership ambiguity is resolved: Scene Manager owns the readiness guarantee, Input owns the latency guarantee, and a failing integration is attributed to whichever sub-assertion broke. **PLAYMODE (Integration)**

### Shift-Left Implementation Notes (pass-6 R13 — test file paths added)

Test file paths follow the project test layout (`tests/unit/[system]/`, `tests/integration/[system]/`, `tests/playmode/[system]/`, `tests/editmode/[system]/`, `tests/device/[system]/`) per `.claude/docs/coding-standards.md` — Scene Manager's tests live under `scene-manager/`.

**Scaffold at Story 1 (BootLoader + AppStateManager skeleton):**
- AC-BOOT-R2: sequence-log stub for all four Foundation `Initialize()` methods; `IPreUnloadHook` stub implementations for priorities 10/20/30. — `tests/playmode/scene-manager/boot_sequence_test.cs`
- AC-R5: event-capture fixture that records channel name, payload type, and wall-clock timestamp for all three `gameplay.scene.*` channels; required before the first `BeginTransitionAsync` implementation is merged. — `tests/playmode/scene-manager/transition_event_taxonomy_test.cs`
- AC-BOOT-FEPM (Sub-cases A + B): the Play-twice CI smoke test fixture must ship alongside the `Reset()` implementation — deferring means FEPM regressions survive until QA hand-off, which is exactly what R2.a was written to prevent. Each sub-case is its own `[Test]` method in the same fixture; both share `[SetUp]` enforcement assertion. — `tests/editmode/scene-manager/boot_fepm_smoke_test.cs` (the Play-mode round-trip is driven from EditMode via `EditorApplication.isPlaying`)
- AC-R10e + AC-D2: pure C# fixtures (< 30 min each), no Unity scene required; ship at Story 1. — `tests/editmode/scene-manager/clock_drift_test.cs`, `tests/editmode/scene-manager/preunload_max_vs_sum_test.cs`

**Scaffold at Story 2 (Pause overlay + OS background):**
- AC-R9a/b: `RequestPause()` sequence-log fixture with timestamped `sceneLoaded` vs `timeScale` write. — `tests/playmode/scene-manager/pause_lifecycle_test.cs`
- AC-R9e: `RequestUnpause()` sequence-counter fixture (BLOCKING CI PLAYMODE per pass-6 BLOCKING #5). — same file: `tests/playmode/scene-manager/pause_lifecycle_test.cs`
- AC-R9f: 11-cycle pause/unpause memory-stability fixture with `GC.GetTotalMemory(true)` baseline + post-cycle assertion. — `tests/playmode/scene-manager/pause_memory_stability_test.cs`
- AC-R10a: `OnApplicationPause` 6-condition assertion fixture using mock `ISaveManager`. — `tests/playmode/scene-manager/os_background_test.cs`
- AC-R10b: User-pause preservation fixture (positive PauseOverlay.isLoaded assertion per pass-6 R10). — same file: `tests/playmode/scene-manager/os_background_test.cs`
- AC-R10c-Rush/Prep/General + AC-R10d-Rush/Prep/General: six fixtures, one per tier-direction combination. — `tests/playmode/scene-manager/bg_expiry_threshold_test.cs`
- AC-R12d: `_pendingPause` destination guard fixture. — `tests/playmode/scene-manager/pending_pause_destination_test.cs`

**Scaffold at integration sprint:**
- AC-INT-EB / AC-INT-SL / AC-INT-Input: cross-Foundation integration tests. — `tests/integration/scene-manager/foundation_handoff_test.cs`
- AC-INT-FF (Feedback Floor handoff at scene boundary): integration test owned by Scene Manager; readiness assertion. Latency sub-assertion lives in Input AC-P1's `tests/playmode/input/`. — `tests/integration/scene-manager/feedback_floor_handoff_test.cs`

**Scaffold at DEVICE / Plumbing Sprint (post ADR-003):**
- AC-R11 / AC-P-Retry, AC-P-Boot, AC-P-UnityBoot: device perf measurements. — `tests/device/scene-manager/retry_perf_test.cs`, `tests/device/scene-manager/boot_perf_test.cs`
- AC-R9g (RequestUnpause Pillar 5 latency advisory). — `tests/device/scene-manager/unpause_latency_test.cs`

**Deferrable to QA hand-off:**
- AC-R12a, AC-R12b, AC-R12c: re-entrancy + invalid-scene + firing-order tests; post-integration-sprint. — `tests/playmode/scene-manager/reentrancy_test.cs`
- AC-EC-E14a, AC-EC-E14b, AC-EC-E15: Editor-only guards; low regression risk mid-sprint. — `tests/editmode/scene-manager/boot_enforcer_test.cs`, `tests/playmode/scene-manager/duplicate_appstatemanager_test.cs`

### Gate Summary (pass-6 recount — body authoritative; pass-3 was 49; pass-6 BLOCKING #5 promotes AC-R9e ADVISORY → BLOCKING CI PLAYMODE; pass-6 BLOCKING #2 splits AC-BOOT-FEPM into 2 sub-cases; pass-6 BLOCKING #3 splits AC-R10c/R10d into 6 tier-specific fixtures; pass-6 RECOMMENDED R3 adds AC-R9g RequestUnpause Pillar 5 latency ADVISORY)

| Gate Level | Count | AC IDs | Operability Status |
|------------|-------|--------|--------------------|
| **BLOCKING CI** | 10 | AC-R1, AC-BOOT-FEPM-A, AC-BOOT-FEPM-B, AC-BOOT-RIOLM, AC-R4, AC-R6, AC-R7a, AC-R9d, AC-INV1, AC-INV2 | **ACTIVE** — AC-R9d requires scene-validator script (Story 2); AC-INV2 condition (d) requires `InputSystemUIInputModule` assertion (pass-3 BLOCKING #12); AC-BOOT-FEPM split into Sub-case A (DisableDomainReload only, `m_EnterPlayModeOptions: 1`) and Sub-case B (both reloads disabled, `m_EnterPlayModeOptions: 3`) per pass-6 BLOCKING #2 — `[SetUp]` enforcement assertion required in both fixtures |
| **BLOCKING CI (PLAYMODE)** | 10 | AC-R2, AC-BOOT-R2, AC-R5, AC-R7, AC-R8, AC-R9a, AC-R9b, **AC-R9e**, AC-R10a, AC-R10b | **ACTIVE** at first AppStateManager implementation story; AC-R5 activates once Event Bus absorbs `gameplay.scene.*` channels + `ExclusiveSubscribe` facet (Phase 5 amendment); AC-R10b promoted from ADVISORY in pass-3 BLOCKING #2 (user-pause preservation is a P0 contract); **AC-R9e promoted from ADVISORY in pass-6 BLOCKING #5** (RequestUnpause double-unpause guard creates same permanent `Time.timeScale=0f` freeze as RequestPause double-pause guard — same severity class as AC-R9a) |
| **PLAYMODE** | 20 | AC-R8a, AC-R9c, AC-R9f, AC-R10c-Rush, AC-R10c-Prep, AC-R10c-General, AC-R10d-Rush, AC-R10d-Prep, AC-R10d-General, AC-R12a, AC-R12b, AC-R12c, AC-R13, AC-INV3, AC-D1, AC-EC-E13, AC-EC-E14b, AC-EC-E15, AC-EC-E9, AC-EC-E18 | **PARTIALLY ACTIVE** — most ACTIVE at Stories 1–3; AC-R9f new in pass-3 BLOCKING #11 (memory stability); AC-EC-E14b is the PlayMode side of the pass-3 split; **AC-R10c/R10d each split into 3 tier-specific fixtures (Rush / Prep / General) per pass-6 BLOCKING #3** — net +4 vs pass-3 single-fixture pair |
| **PLAYMODE (DEFERRED)** | 3 | AC-INT-Level, AC-INT-Modal, **AC-R12d** | **DEFERRED** — activates when Level Runtime / Modal/Dialog GDDs land; AC-R12d promoted from "Known coverage gaps" to formal Gate Summary tier per pass-6 R11 (still PLAYMODE deferred until Story 2) |
| **PLAYMODE (Integration)** | 4 | AC-INT-EB, AC-INT-SL, AC-INT-Input, AC-INT-FF | **ACTIVE** — cross-system integration sprint; AC-INT-FF owns the integration test (Input AC-P1 owns the latency sub-assertion) |
| **EDITMODE** | 3 | AC-R10e, AC-D2, AC-EC-E14a | **ACTIVE** (pure C# logic + Editor-only interception); AC-EC-E14a new in pass-3 BLOCKING #6 (split from pass-2 AC-EC-E14) |
| **DEVICE** | 2 | AC-R11 / AC-P-Retry, AC-P-Boot | **BLOCKED on ADR-003** — activate at Plumbing Sprint / VS milestone; AC-R11 raised to 10 retries per pass-3 REC-4 |
| **DEVICE (ADVISORY)** | 1 | AC-P-UnityBoot | **BLOCKED on ADR-003** — dashboard only when activated |
| **ADVISORY** | 3 | AC-D3, AC-EC-E3, **AC-R9g** | **ACTIVE** at dashboard level — non-gating; AC-R9g new in pass-6 R3 (RequestUnpause Pillar 5 latency bound — `t_unpause_to_first_input_event ≤ 100 ms` 60 fps / 133 ms 30 fps measured on ADR-003 device, dashboard-only); AC-R9e exited this tier via pass-6 BLOCKING #5 promotion |

**Total: 56 ACs** across 9 severity tiers (sum: 10+10+20+3+4+3+2+1+3 = 56). Pass-3 was 49; pass-6 net change is +7: +1 from BLOCKING #2 (AC-BOOT-FEPM splits to A+B; net +1), +4 from BLOCKING #3 (AC-R10c splits to 3 + AC-R10d splits to 3; net +4), +1 from R3 (AC-R9g new ADVISORY), +1 from R11 (AC-R12d promoted from coverage gaps into formal tier; net +1 since the deferred-coverage entry was always tracked but not Gate-Summary-counted). AC-R9e tier promotion is not a count change. The pass-6 sum is consistent across H subsections — see "AC count reconciliation" note at end of section H. AC-R10b tier value preserved (already promoted in pass-3 BLOCKING #2).

### Known coverage gaps (flagged for later addition at story time)

- **AC-R4b** (full state-machine walk — each valid transition fires `app.state_changed` with correct prev/next values) — add at Story 3 if regressions surface.
- **R14.a** (at most one non-additive scene) — CI scene-validator concern, partially covered by AC-INV2; add a runtime assertion in DEVELOPMENT_BUILD `OnSceneLoaded` (implementation note, not a separate AC).
- **R14.b** (no gameplay MB calling Foundation APIs before `FoundationReady`) — un-testable by grep (semantic, not syntactic); covered by code review + documentation.
- **AC-EC-E7** (pause-during-Boot guard) — trivial; add only if it regresses.
- **E17** (Audio Bus absent at MVP) — documented MVP limitation; closes when Audio Bus GDD lands.
- **(pass-6 R12 — AC-R9e stale entry deleted)**: pass-3 BLOCKING #7 promoted AC-R9e into Section H.1 Rule Compliance as a real test (initially ADVISORY, pass-6 BLOCKING #5 promoted it again to BLOCKING CI PLAYMODE). The change-log breadcrumb here is no longer needed.
- **(pass-6 R11 — AC-R12d promoted to formal Section H.3)**: AC-R12d (`_pendingPause` Destination Permission Guard, pass-3 BLOCKING #10) is no longer "deferred to story time" — it is now a formal AC in H.3 with PLAYMODE (DEFERRED) tier, activates at Story 2. See H.3 entry below.

## Open Questions

Track the unverified claims, provisional assumptions, and deferred design questions this GDD has introduced. Not blocking — all have documented fallback behavior; each will be resolved at a named milestone.

### Knowledge gaps — Unity 6.3 API verification (0 open + 7 RESOLVED — pass-6 R14 closed OQ-1)

Pass-2 unity-specialist verified 6 of the 7 pass-1 OQs against Unity 6.0 official docs (`https://docs.unity3d.com/6000.0/...`); pass-5 unity-specialist verification of `LoadSceneParameters` closes the last open OQ. All 7 are **CLOSED** as of pass-6.

| OQ | Topic | Status | Resolution |
|----|-------|--------|-----------|
| **OQ-1** | Does `LoadSceneParameters` (Unity 2019+) have new flags in Unity 6.x? | **CLOSED — unchanged in Unity 6.3 LTS** | Pass-5 unity-specialist verified the `LoadSceneParameters` struct against Unity 6.3 LTS docs: same two fields as Unity 2019+ (`LoadSceneMode loadSceneMode` + `LocalPhysicsMode localPhysicsMode`), no new flags introduced across Unity 6.0/6.1/6.2/6.3. R1's `LoadSceneMode` enum assumption is correct. (Pass-6 R14 fold-in) |
| **OQ-2** | Is `Application.focusChanged` (`Action<bool>`) promoted in Unity 6.x as preferred over `OnApplicationFocus`? | **CLOSED — keep `OnApplicationPause`** | Unity 6 docs describe `Application.focusChanged` as a delegate "for code that does not use the MonoBehaviour namespace" — NOT a replacement. Android docs continue to recommend `OnApplicationPause` for background detection. R10 design unchanged. |
| **OQ-3** | `RuntimeInitializeOnLoadMethod(BeforeSplashScreen)` tier behavior in Unity 6.x builds vs editor — confirmed same? | **CLOSED — `BeforeSplashScreen` reliable; R3's prohibition is architectural** | Unity 6 docs confirm all five RIOLM tiers fire identically in Player and Editor. R3's prohibition stands but the rationale should be read as "architectural — prevent Foundation construction outside `BootLoader.Awake()`" not "reliability concern." |
| **OQ-4** | Unity 6.x Fast Enter Play Mode separates Scene Reload from Domain Reload as independent toggles | **CLOSED — confirmed independent** | Unity 6 Manual "Details of disabling domain and scene reload" lists 4 independent states. AC-BOOT-FEPM "play twice without domain reload" smoke test is implementable with Domain Reload disabled + Scene Reload enabled (the harder case for `Reset()` discipline). |
| **OQ-5** | Does Unity 6.x provide an official pre-unload callback in `UnityEngine.SceneManagement`? | **CLOSED — none exists** | Unity 6 `SceneManager` API still has `sceneUnloaded` (post-unload) only. `IPreUnloadHook` custom interface remains the correct pattern. |
| **OQ-6** | `UnloadSceneAsync` + automatic GC integration — does Unity 6.x still require separate `Resources.UnloadUnusedAssets()` for memory reclamation? | **CLOSED — manual reclamation REQUIRED for `UnloadSceneAsync`; AUTOMATIC for `LoadSceneAsync` in `LoadSceneMode.Single`** | Unity 6 `SceneManager.UnloadSceneAsync` docs explicitly state: "Assets are currently not unloaded. In order to free up asset memory call `Resources.UnloadUnusedAssets`." `LoadSceneAsync` in single mode auto-reclaims. **R6 + R11 + D.1 updated in pass-2 to add explicit `Resources.UnloadUnusedAssets()` step** (additive `PauseOverlay` unload path was the leak vector). |
| **OQ-7** | Unity 6.x "app backgrounded for N minutes" native lifecycle API — does one exist, or is R10's `DateTime.UtcNow` polling right? | **CLOSED — no native API; `DateTime.UtcNow` polling is correct** | No Unity 6 API exposes a backgrounded-duration query. R10 design unchanged. |

### Cross-GDD provisional assumptions (5 items)

These are contracts this GDD names but which depend on downstream GDDs that don't yet exist. Each will be absorbed (or contested) when the downstream GDD is authored.

| OQ | Topic | Resolution |
|----|-------|-----------|
| **OQ-8** | Modal/Dialog's R8 activation-handshake contract: "exactly one production subscriber" invoking `activationCallback(true)` on fade-out | Modal/Dialog GDD authoring — if a different handshake is needed (e.g., two subscribers for Accessibility announcer + visual fade), R8 needs amendment |
| **OQ-9** | Level Runtime's 600 ms `T_lr` sub-budget (R11): cross-GDD performance obligation | Level Runtime GDD authoring — must include BLOCKING AC against `T_lr ≤ 600 ms` on ADR-003 device; if LR's actual work exceeds, R11 budget split must re-balance |
| **OQ-10** | Audio Bus's split ownership of `AudioListener.pause` (R14.f): SM owns OS-background writes; AB owns in-game pause audio fade | Audio Bus GDD authoring — must explicitly document the split; if AB design prefers unified ownership, R14.f must be amended |
| **OQ-11** | Main Menu's `RequestPause()` call-site (R9): synchronous method call, NOT via Event Bus | Main Menu GDD authoring — if event-based triggering is preferred, R9's synchronous-method contract must be renegotiated |
| **OQ-12** | Cross-GDD `sortingOrder` budget re-tuning (R14.g): band widths may be too narrow for dense HUD at Rush Phase | UI GDD authoring — re-tune via coordinated amendment across all affected UI GDDs if needed |

### Design questions deferred to downstream phases (2 items)

| OQ | Topic | Resolution |
|----|-------|-----------|
| **OQ-13** | Addressables-driven scene loading (`Addressables.LoadSceneAsync`) — deferred to VS per Section C Decision 1 | If Mali-G52 memory pressure during Rush Phase becomes critical (concept doc flags as real risk), may become MVP-required. Decision deferred to Architecture phase (ADR-008 scope) |
| **OQ-14** | Pause overlay as persistent-loaded-but-hidden vs load-on-demand per R9 | Trade-off: memory (always-loaded overlay assets) vs latency (200+ ms overlay load). Defer to Main Menu GDD — measure actual pause-latency on ADR-003 device and decide |

### Project-level (carried forward from session state)

| OQ | Topic | Resolution |
|----|-------|-----------|
| **OQ-15** | ADR-003 reference Android device naming — blocks AC-R11 (AC-P-Retry), AC-P-Boot, AC-P-UnityBoot DEVICE ACs | Pre-production ADR authoring (TD-FEASIBILITY pre-production action item #3). Cross-linked to Save/Load + Input System — all three block together. |

### New OQs added in pass-2 /design-review

| OQ | Topic | Resolution |
|----|-------|-----------|
| **OQ-16** | Event Bus `ExclusiveSubscribe` channel facet — Phase 5 Event Bus bidirectional amendment must add `ExclusiveSubscribe<T>(channel, handler)` API that throws `InvalidOperationException` on second registration. | Phase 5 Event Bus amendment (post Scene Manager pass-2 approval). Modal/Dialog GDD authoring will be the first consumer. |
| **OQ-17** | Addressables 2.4 `SceneInstance.Activate()` API surface vs `AsyncOperation.allowSceneActivation` — when OQ-13 resolves, the activation gate mechanism requires an adapter layer. | ADR-008 must document this API surface delta (pass-2 unity-specialist R-5). Not a near-term blocker (Addressables deferred to VS), but ADR-008 authoring should capture it now to prevent surprise. |
| **OQ-18** | Save/Load read-during-write safety — when Scene Manager fire-and-forgets `RequestFlush` on retry path, the new scene's `Awake` may read SaveManager state during in-flight background flush. AC-INT-SL only tests handler-return time. | Save/Load GDD bidirectional amendment (Phase 5) must explicitly document that public read APIs are safe to call during background flush, OR add a cross-GDD AC verifying read-during-write safety. |
| **OQ-19** | Two-tier `bg_session_expiry` retention validation — pass-2 BLOCK-1 set defaults to 60s general / 1800s in-Rush based on Section B fantasy reasoning; live-ops retention data may suggest different values post-soft-launch. | Live-ops A/B test post-launch. Tracked as a tuning candidate, not a blocker. |
| **OQ-20** | Cold-cache T_sm overshoot on Mali-G52 — pass-2 perf-analyst R-1 noted that 1100ms `T_sm_budget` is optimistic for cold Addressables cache; the 2000ms `T_retry_max` gate should still hold via T_slack absorption, but real Mali-G52 device measurement is needed to confirm. | DEVICE measurement on first ADR-003-resolved test cycle; if T_sm consistently exceeds 1300ms on cold cache, re-balance budget split (raise `T_sm_budget`, reduce `T_slack_budget` or `T_lr_budget`). |

### New OQ added in pass-4 (author-led revision)

| OQ | Topic | Resolution |
|----|-------|-----------|
| **OQ-21** | iOS `OnApplicationFocus(false)` behavior across focus-loss scenarios — pass-3 BLOCKING #4 added an iOS-only path for phone-call interruption (the call modal triggers focus loss, not background). The path needs verification across distinct focus-loss events: (a) phone call coming in; (b) banner notification swipe; (c) Control Center pull; (d) lock-screen-without-background. Each may fire `OnApplicationFocus(false)` with different recovery patterns; the GDD currently assumes uniform `AudioListener.pause` toggle behavior. **Pass-6 R15 fold-in (escalation path documentation):** the short-call-to-long-call escalation sequence is currently safe-by-construction but undocumented. If `OnApplicationFocus(false)` fires for a phone call (R10 iOS path: `FlushSync` + `AudioListener.pause = true`, no `IsPaused` mutation), and the call subsequently extends past iOS's foreground-keepalive window (variable, ~30-60 sec depending on iOS version + audio session category), iOS escalates to a true background and fires `OnApplicationPause(true)`. The R10 OS-background path then runs ON TOP of the focus-loss state: `FlushSync` (idempotent — already saved), `_userPausedBeforeBackground = IsPaused = false` capture, `IsPaused = true`, `AudioListener.pause = true` (already true — idempotent), `_backgroundedAtUtc` recorded. On resume: the OS-background path's three-tier threshold logic runs (likely `_backgroundedDuringRush = true` if Level was active at focus-loss time), `IsPaused` restores to `false` (per the captured `_userPausedBeforeBackground`), `AudioListener.pause = false`. The `OnApplicationFocus(true)` event also fires on resume, harmlessly setting `AudioListener.pause = false` redundantly. **Sequence is safe** because: every state mutation is idempotent, `_userPausedBeforeBackground` is captured at OS-background time (the more authoritative event), and iOS does not re-fire `OnApplicationFocus(false)` for an already-backgrounded app. Document escalation as "expected behavior, no special-case code required" so a future maintainer reading R10 understands the long-call resume path is the same as any OS-background resume path. | First iOS device test cycle (post-ADR-003 device naming). If certain focus-loss events do NOT fire `OnApplicationFocus`, R10's iOS path needs additional Unity API integration (e.g., `Application.focusChanged` delegate) or iOS native plugin instrumentation. |

**Total: 21 OQs** — 0 open Unity API + 7 RESOLVED Unity API (pass-6 R14 closed OQ-1) + 5 cross-GDD provisional + 2 deferred design + 1 ADR-003 shared + 5 pass-2 (4 cross-GDD/architecture, 1 live-ops, 1 device-measurement) + 1 pass-4 (iOS focus-loss device measurement, expanded scope per pass-6 R15 escalation-path documentation). Net active: 13 (excluding 7 RESOLVED + 1 deferred-low-risk).
