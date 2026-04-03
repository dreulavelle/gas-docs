---
title: Lazy Loading and Async Loading
description: Lazy loading GameplayCueNotifies, async loading ability classes with soft references, the GameplayCueManager's loading pipeline, and the memory vs startup-time tradeoff.
---

# Lazy Loading and Async Loading

GAS has a lot of content: cue notify Blueprints, ability classes, effect definitions, montages. Loading all of it at startup is safe (no hitches later) but slow (longer load screen) and memory-hungry (everything is resident whether it's used or not).

The alternative is lazy loading -- load things when they're first needed. The tradeoff is that first use can cause a hitch if the load isn't handled asynchronously.

This page covers both sides: how GAS loads things by default, how to make it async, and how to find the right balance for your game.

## GameplayCue Loading Pipeline

The `UGameplayCueManager` has a sophisticated loading system built around two `UObjectLibrary` instances and a set of virtual configuration methods that control when and how things load.

### The Two Object Libraries

| Library | Purpose | When Scanned |
|:---|:---|:---|
| `RuntimeGameplayCueObjectLibrary` | The "always loaded" cue notifies | At startup, paths from `GetAlwaysLoadedGameplayCuePaths()` |
| `EditorGameplayCueObjectLibrary` | All known cue notifies (editor only) | Editor only, for the GameplayCue editor UI |

At startup, `InitializeRuntimeObjectLibrary()` scans the runtime paths and discovers all cue notify assets via the Asset Registry. What happens next depends on four virtual booleans:

### The Four Loading Controls

```cpp
// GameplayCueManager.cpp defaults:

bool UGameplayCueManager::ShouldSyncScanRuntimeObjectLibraries() const
{
    return true;  // Always sync-scan the asset registry for cue assets
}

bool UGameplayCueManager::ShouldSyncLoadRuntimeObjectLibraries() const
{
    return false;  // Don't block startup to load every cue class
}

bool UGameplayCueManager::ShouldAsyncLoadRuntimeObjectLibraries() const
{
    return true;  // Start async loading cue classes in the background
}

bool UGameplayCueManager::ShouldDeferScanningRuntimeLibraries() const
{
    return false;  // Don't wait for AR to finish gathering
}
```

The default behavior: **sync scan** the Asset Registry to discover cue assets (fast -- just reads metadata), then **async load** the actual classes in the background. No sync load, no deferred scan.

This means:

1. Startup scans for cue notify assets (quick)
2. Background streaming loads the classes (non-blocking)
3. If a cue is triggered before its class finishes loading, `HandleMissingGameplayCue` decides what to do

### What Happens When a Cue Is Missing

When the cue manager routes a cue event but the notify class isn't loaded yet:

```cpp
bool UGameplayCueManager::HandleMissingGameplayCue(
    UGameplayCueSet* OwningSet,
    FGameplayCueNotifyData& CueData,
    AActor* TargetActor,
    EGameplayCueEvent::Type EventType,
    FGameplayCueParameters& Parameters)
{
    if (ShouldSyncLoadMissingGameplayCues())
    {
        // Block and load now -- causes a hitch
        CueData.LoadedGameplayCueClass =
            Cast<UClass>(StreamableManager.LoadSynchronous(CueData.GameplayCueNotifyObj));
        return true;  // loaded, continue execution
    }
    else if (ShouldAsyncLoadMissingGameplayCues())
    {
        // Start async load, cache the cue event, replay when loaded
        // ... queues into AsyncLoadPendingGameplayCues map
        return false;  // not loaded yet, cue is deferred
    }
    return false;
}
```

The defaults:

```cpp
bool UGameplayCueManager::ShouldSyncLoadMissingGameplayCues() const
{
    return false;  // Don't hitch
}

bool UGameplayCueManager::ShouldAsyncLoadMissingGameplayCues() const
{
    return true;  // Async load, then replay the cue event
}
```

So by default: if a cue fires before its class is loaded, the engine queues the cue event, starts an async load, and replays the cue when the load completes. The player sees the cue appear slightly late rather than experiencing a hitch.

!!! info "The `FAsyncLoadPendingGameplayCue` struct"
    When a cue is deferred, the manager stores the full cue context -- owning set, tag, target actor (weak pointer), event type, and parameters -- in `AsyncLoadPendingGameplayCues`. When the async load finishes (`OnMissingCueAsyncLoadComplete`), it replays each pending cue if the target actor is still alive.

## Async Loading Ability Classes

Abilities and effects can also be lazy-loaded using UE's soft reference system. This is independent of the cue manager -- it's a general UE pattern applied to GAS types.

### TSoftClassPtr for Abilities

Instead of hard-referencing ability classes (which forces them to load with the owning asset), use soft references:

```cpp
UCLASS()
class UMyAbilitySet : public UDataAsset
{
    GENERATED_BODY()

public:
    // Hard reference -- loads when this DataAsset loads
    // UPROPERTY()
    // TArray<TSubclassOf<UGameplayAbility>> Abilities;

    // Soft reference -- loads on demand
    UPROPERTY(EditDefaultsOnly)
    TArray<TSoftClassPtr<UGameplayAbility>> Abilities;

    void GrantAbilities(UAbilitySystemComponent* ASC)
    {
        for (const TSoftClassPtr<UGameplayAbility>& AbilityClass : Abilities)
        {
            if (UClass* Loaded = AbilityClass.LoadSynchronous())
            {
                ASC->GiveAbility(FGameplayAbilitySpec(Loaded, 1));
            }
        }
    }
};
```

### Async Loading with FStreamableManager

For non-blocking loading:

```cpp
void UMyAbilitySet::GrantAbilitiesAsync(
    UAbilitySystemComponent* ASC,
    FSimpleDelegate OnComplete)
{
    TArray<FSoftObjectPath> PathsToLoad;
    for (const TSoftClassPtr<UGameplayAbility>& AbilityClass : Abilities)
    {
        if (!AbilityClass.IsNull() && !AbilityClass.IsValid())
        {
            PathsToLoad.Add(AbilityClass.ToSoftObjectPath());
        }
    }

    if (PathsToLoad.IsEmpty())
    {
        // Everything already loaded
        GrantAbilitiesImmediate(ASC);
        OnComplete.ExecuteIfBound();
        return;
    }

    FStreamableManager& StreamableManager =
        UAssetManager::GetStreamableManager();

    StreamableManager.RequestAsyncLoad(
        PathsToLoad,
        FStreamableDelegate::CreateLambda([this, ASC, OnComplete]()
        {
            GrantAbilitiesImmediate(ASC);
            OnComplete.ExecuteIfBound();
        })
    );
}
```

### TSoftClassPtr for Effects

The ASC already supports soft-referenced effects via `GetGameplayEffectCount_IfLoaded`:

```cpp
// Check if an effect is active without forcing it to load
int32 Count = ASC->GetGameplayEffectCount_IfLoaded(
    TSoftClassPtr<UGameplayEffect>(FSoftObjectPath(TEXT("/Game/Effects/GE_Burn"))),
    nullptr,
    true  // bEnforceOnGoingCheck
);
```

This is useful for queries that shouldn't trigger a load -- like UI code checking whether an effect is active.

## Preloading Strategies

Full lazy loading can cause visible delays. Here are strategies to preload what matters while keeping memory lean:

### Strategy 1: Preload by Game Phase

Load abilities for the current phase, unload the rest:

```cpp
void AMyGameMode::OnMatchStart()
{
    // Preload combat abilities
    PreloadAbilitySet(CombatAbilitySet);
    // Don't preload lobby/menu abilities -- not needed
}

void AMyGameMode::OnMatchEnd()
{
    // Preload results/lobby abilities
    PreloadAbilitySet(LobbyAbilitySet);
}
```

### Strategy 2: Preload by Class Selection

If the player picks a class, preload that class's abilities during the loading screen:

```cpp
void AMyPlayerController::OnClassSelected(UMyCharacterClass* SelectedClass)
{
    // Start loading this class's abilities and cues during the transition
    SelectedClass->PreloadAbilitiesAsync();
    SelectedClass->PreloadCueNotifiesAsync();
}
```

### Strategy 3: Preload Cue Notify Actors

For frequently-used `AGameplayCueNotify_Actor` subclasses, set `NumPreallocatedInstances` on the CDO:

```cpp
AMyHitImpactCue::AMyHitImpactCue()
{
    // Pre-spawn 5 instances into the pool
    NumPreallocatedInstances = 5;
}
```

The cue manager calls `UpdatePreallocation()` to spawn these ahead of time. When a cue fires, it pulls from the pool instead of spawning a new actor. Combined with the recycling system (`Recycle()` / `ReuseAfterRecycle()` on `AGameplayCueNotify_Actor`), this eliminates spawn hitches for common cues.

!!! tip "Call `UpdatePreallocation` from your GameState"
    The cue manager's `UpdatePreallocation(UWorld*)` prespawns one actor per call, spreading the cost across frames. Call it from `Tick` on your game state or a manager actor until preallocation is complete.

### Strategy 4: Custom Cue Notify Paths

Control which directories get scanned and loaded by overriding `GetAlwaysLoadedGameplayCuePaths()`:

```cpp
TArray<FString> UMyGameplayCueManager::GetAlwaysLoadedGameplayCuePaths()
{
    TArray<FString> Paths;
    // Only scan these directories at startup
    Paths.Add(TEXT("/Game/Effects/Cues/Core"));
    Paths.Add(TEXT("/Game/Effects/Cues/Common"));
    // Don't include /Game/Effects/Cues/Rare -- load on demand
    return Paths;
}
```

You can also add paths at runtime with `AddGameplayCueNotifyPath()`:

```cpp
// When entering a new zone, add its cue path
CueManager->AddGameplayCueNotifyPath(TEXT("/Game/Effects/Cues/Zone_Volcano"));
```

## The Memory vs. Startup-Time Tradeoff

| Approach | Startup Time | Memory | First-Use Hitch | Complexity |
|:---|:---|:---|:---|:---|
| Load everything at startup | Slow | High | None | Low |
| Async load at startup (default) | Fast | Grows over time | Rare (brief) | Low |
| Full lazy load | Fast | Low | Yes (first cue) | Medium |
| Phase-based preload | Medium | Medium | None in-phase | Medium |
| Class-based preload | Fast | Low-Medium | None for selected class | High |

For most games, the default (async load at startup) is the right starting point. Only change it if you've measured a problem:

- **Startup too slow?** Reduce `GetAlwaysLoadedGameplayCuePaths()` to only core cues, let the rest lazy-load.
- **Hitches on first cue?** Verify `ShouldAsyncLoadMissingGameplayCues()` returns true (it does by default). If hitches persist, the cue is likely sync-loaded through a different code path -- check for `LoadSynchronous` calls in your project.
- **Too much memory?** Profile with `memreport -full`. If cue notify classes are a significant chunk, narrow the always-loaded paths and rely on async loading.

!!! warning "Don't override `ShouldSyncLoadMissingGameplayCues` to return true"
    This causes a blocking load on the game thread whenever a cue fires and its class isn't loaded. In a shipping game, this means hitches. The default async path (defer the cue, load in the background, replay when ready) is almost always better.

## Debugging Loading Issues

### Cue Not Playing at All

1. Check the cue tag matches a notify in the `GameplayCueSet` (`PrintGameplayCueNotifyMap` console command)
2. Verify the notify's directory is in the scanned paths
3. Check if the notify class failed to load (look for `"Late load of GameplayCueNotify ... failed!"` in the log)

### Cue Playing Late

This is expected behavior with async loading. The cue fires, the class isn't loaded, it's queued in `AsyncLoadPendingGameplayCues`, and replayed after the async load completes. If the delay is too long:

- Move the cue to a preloaded path
- Set `NumPreallocatedInstances` on frequently-used actor cues
- Use `Static` cue notifies instead of `Actor` cues where possible (they're lighter to load)

### Cue Causing a Hitch

If `ShouldSyncLoadMissingGameplayCues()` returns false (default) but you still see hitches, something else is forcing a sync load. Common culprits:

- A hard reference (`TSubclassOf` or direct asset reference) in the cue notify class that pulls in a heavy asset
- A Blueprint cue that references large meshes/materials without soft references
- A `LoadSynchronous` call in your own code

## Related Pages

- [Cue Manager](../gameplay-cues/cue-manager.md) -- how the cue manager discovers and routes cues
- [Cue Notify Types](../gameplay-cues/cue-types.md) -- Static vs Actor vs Burst cue types
- [Cue Batching](cue-batching.md) -- reducing cue RPC overhead
- [Replication Optimization](replication-optimization.md) -- reducing replication bandwidth
