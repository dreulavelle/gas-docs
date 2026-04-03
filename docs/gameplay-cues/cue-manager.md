---
title: Cue Manager
description: UGameplayCueManager deep dive — routing, GameplayCueSet, preloading, async loading, batching, GameplayCueTranslator, and overriding the manager.
---

# Cue Manager

The `UGameplayCueManager` is a singleton (technically a `UDataAsset`) that acts as the central routing hub for all Gameplay Cue events. It discovers cue notify classes, loads them, matches incoming cue tags to the right notify, manages the actor recycle pool, and handles batching.

You'll rarely call into the manager directly during normal development -- the ASC handles routing for you. But understanding the manager is essential for configuration, optimization, and debugging cue issues.

## How Routing Works

When a cue event arrives (from an effect, manual trigger, or animation), the flow through the manager is:

```
HandleGameplayCue(TargetActor, Tag, EventType, Parameters)
    │
    ├─ 1. ShouldSuppressGameplayCues(TargetActor)
    │      → If true, cue is silently dropped
    │
    ├─ 2. TranslateGameplayCue(Tag, TargetActor, Parameters)
    │      → Tag may be remapped via GameplayCueTranslator
    │
    ├─ 3. RouteGameplayCue(TargetActor, Tag, EventType, Parameters)
    │      ├─ Check IGameplayCueInterface on target actor
    │      └─ Look up tag in GameplayCueSet
    │          ├─ Found → invoke notify class
    │          └─ Not found → HandleMissingGameplayCue()
    │              ├─ Sync load? → load and invoke
    │              ├─ Async load? → queue for later
    │              └─ Neither → silently ignored
    │
    └─ OnRouteGameplayCue delegate fires (for custom observers)
```

### Execution Options

The `EGameplayCueExecutionOptions` flags let callers skip parts of the pipeline:

| Flag | Effect |
|:---|:---|
| `IgnoreInterfaces` | Skip `IGameplayCueInterface` check |
| `IgnoreNotifies` | Skip notify class lookup/spawn |
| `IgnoreTranslation` | Skip tag translation step |
| `IgnoreSuppression` | Force execution even if suppressed |
| `IgnoreDebug` | Don't show debug visualizations |

## GameplayCueSet

The `UGameplayCueSet` is the lookup table that maps `FGameplayTag` to cue notify class/object data. The manager maintains two sets:

| Set | Purpose | When Used |
|:---|:---|:---|
| **RuntimeGameplayCueObjectLibrary.CueSet** | The "always loaded" set | At runtime |
| **EditorGameplayCueObjectLibrary.CueSet** | All discoverable cues | Editor only (GameplayCue editor) |

At startup, the manager scans the configured paths (see below), finds all cue notify assets, and registers them in the runtime CueSet.

## Scan Paths and Configuration

### Where the Manager Looks

The manager discovers cue notifies by scanning content directories. The paths are configured in **Project Settings > Gameplay Abilities Settings**:

```
GameplayCueNotifyPaths:
  - /Game/GAS/Cues
  - /Game/Effects/GameplayCues
```

Or in C++ via `UAbilitySystemGlobals`:

```cpp
// In your AbilitySystemGlobals subclass
TArray<FString> UMyAbilitySystemGlobals::GetGameplayCueNotifyPaths()
{
    TArray<FString> Paths = Super::GetGameplayCueNotifyPaths();
    Paths.Add(TEXT("/Game/MyPlugin/Cues"));
    return Paths;
}
```

You can also add/remove paths at runtime:

```cpp
UGameplayCueManager* CueManager = UAbilitySystemGlobals::Get().GetGameplayCueManager();
CueManager->AddGameplayCueNotifyPath(TEXT("/Game/DLC/Cues"), true);
CueManager->RemoveGameplayCueNotifyPath(TEXT("/Game/OldCues"), true);
```

The second parameter (`bShouldRescanCueAssets`) triggers a re-scan of the object libraries.

### Auto-Tagging from Asset Name

Cue notifies can have their tag set automatically based on their asset name. If you name your asset `GC_Hit_Physical_Slash`, the system can derive `GameplayCue.Hit.Physical.Slash`. This is handled by `UAbilitySystemGlobals::DeriveGameplayCueTagFromAssetName`.

!!! tip "Naming convention"
    Name your cue assets to match their tag hierarchy. Prefix with `GC_` by [convention](../patterns/naming-conventions.md), then replace dots with underscores: `GC_Status_Burning` maps to `GameplayCue.Status.Burning`.

## Loading Behavior

### Startup Loading

By default, the manager async-loads all cue notifies found in the configured paths at startup. The behavior is controlled by virtual functions you can override:

| Method | Default | Effect |
|:---|:---|:---|
| `ShouldSyncScanRuntimeObjectLibraries()` | Varies | Force sync scan of asset registry |
| `ShouldSyncLoadRuntimeObjectLibraries()` | `false` | Sync load all cues at startup |
| `ShouldAsyncLoadRuntimeObjectLibraries()` | `true` | Async load cues at startup |
| `ShouldAsyncLoadObjectLibrariesAtStart()` | `true` | Start async load on manager creation |
| `ShouldLoadGameplayCueAssetData()` | `true` | Per-asset filter for loading |

### Missing Cue Handling

When a cue event fires for a tag that hasn't been loaded yet:

| Method | Default | Behavior |
|:---|:---|:---|
| `ShouldSyncLoadMissingGameplayCues()` | `false` | Block and load immediately |
| `ShouldAsyncLoadMissingGameplayCues()` | `false` | Start async load, queue the event |

If both return false, the cue is silently dropped. Override `HandleMissingGameplayCue()` for custom handling.

!!! warning "Sync loading in shipping builds"
    Sync loading cues at runtime causes hitches. It's useful during development to catch missing cues, but disable it for shipping. See [Lazy Loading](../optimization/lazy-loading.md) for the recommended approach.

### Async Load Pending Cues

When a cue is async-loaded due to a missing class, the manager caches pending events in `AsyncLoadPendingGameplayCues`. Once the class finishes loading, the cached events are replayed. This is transparent to the gameplay code but means there's a brief delay before the cue plays.

## Actor Recycling

For instanced cue notifies (`AGameplayCueNotify_Actor` and subclasses), the manager maintains a **recycle pool** to avoid the cost of spawning and destroying actors repeatedly.

The flow:

1. Cue fires for the first time → actor is spawned normally
2. Cue ends → `Recycle()` is called on the actor → actor hides and goes into the pool
3. Same cue type needed again → `FindRecycledCue()` finds a pooled actor → `ReuseAfterRecycle()` is called

Check if recycling is enabled with `UGameplayCueManager::IsGameplayCueRecylingEnabled()`.

### Preallocation

For cue actors that set `NumPreallocatedInstances > 0`, the manager pre-spawns that many actors at startup via `UpdatePreallocation()`. Call this from your GameState or game mode during initialization:

```cpp
UGameplayCueManager* CueManager = UAbilitySystemGlobals::Get().GetGameplayCueManager();
CueManager->ResetPreallocation(GetWorld());
CueManager->UpdatePreallocation(GetWorld());
```

## GameplayCueTranslator

The `FGameplayCueTranslationManager` (accessible via `TranslationManager` on the cue manager) allows tag remapping before lookup. This is useful for:

- Mapping generic tags to game-specific ones based on context (e.g., `GameplayCue.Hit.Physical` → `GameplayCue.Hit.Physical.Sword` based on the equipped weapon)
- Overriding cues per character class or skin
- A/B testing different cue variants

Translation happens in step 2 of the routing pipeline, before the notify lookup. Implement custom translators by working with the `FGameplayCueTranslationManager` API.

## Batching and Pending Cues

The manager supports batching cue events to reduce RPC overhead. This is managed through a **send context**:

```cpp
// Start batching
CueManager->StartGameplayCueSendContext();

// ... multiple cue events are queued as FGameplayCuePendingExecute ...

// Stop batching and flush
CueManager->EndGameplayCueSendContext();
```

The engine uses `FScopedGameplayCueSendContext` as an RAII wrapper. While a send context is active, cues are accumulated in `PendingExecuteCues` instead of being sent immediately. When the last context closes, `FlushPendingCues()` is called and the batch goes out.

The `OnFlushPendingCues` delegate fires during flush, which is useful for custom batching strategies.

For more on batching optimization, see [Cue Batching](../optimization/cue-batching.md) and [Batching (Networking)](../networking/batching.md).

## Overriding the Manager

To customize the manager, create a subclass and register it:

### Step 1: Subclass

```cpp
UCLASS()
class UMyGameplayCueManager : public UGameplayCueManager
{
    GENERATED_BODY()

    virtual bool ShouldSuppressGameplayCues(AActor* TargetActor) override
    {
        // Example: suppress cues during loading screens
        if (IsInLoadingScreen())
            return true;
        return Super::ShouldSuppressGameplayCues(TargetActor);
    }
};
```

### Step 2: Register in Project Settings

In **Project Settings > Gameplay Abilities Settings**, set the **Global GameplayCue Manager Class** to your subclass.

Or via your `DefaultGame.ini`:

```ini
[/Script/GameplayAbilities.GameplayAbilitiesDeveloperSettings]
GlobalGameplayCueManagerClass=/Script/MyGame.MyGameplayCueManager
```

### Common Override Points

| Virtual | Purpose |
|:---|:---|
| `ShouldSuppressGameplayCues` | Reject cues globally (loading, dead actors) |
| `HandleMissingGameplayCue` | Custom behavior when a cue class isn't loaded |
| `ShouldLoadGameplayCueAssetData` | Filter which assets to include |
| `ProcessPendingCueExecute` | Custom batch processing |
| `DoesPendingCueExecuteMatch` | Custom deduplication of pending cues |
| `GetInstancedCueActor` | Custom instance management |
| `GetAlwaysLoadedGameplayCuePaths` | Dynamic path configuration |

## Debugging

The manager has a `PrintGameplayCueNotifyMap()` function that dumps the entire tag-to-class mapping to the log. Useful for verifying that your cues are discovered:

```
AbilitySystem.GameplayCue.PrintGameplayCueNotifyMap
```

See [Debugging](../debugging/index.md) for more diagnostic tools.

## Related Pages

- [Cue Notify Types](cue-types.md) -- the six notify classes
- [Cue Parameters](cue-parameters.md) -- data passed to cues
- [Cue Batching](../optimization/cue-batching.md) -- reducing RPC overhead
- [Lazy Loading](../optimization/lazy-loading.md) -- async loading strategies
- [AbilitySystemGlobals](../reference/ability-system-globals.md) -- global configuration
