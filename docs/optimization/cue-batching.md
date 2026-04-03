---
title: Gameplay Cue Batching
description: Reducing cue RPC overhead with FScopedGameplayCueSendContext, reliability tuning, and measuring the bandwidth impact.
---

# Gameplay Cue Batching

Every gameplay cue that fires through the ASC generates a network multicast RPC. One cue is fine. But apply a damage effect, a slow, and a burn in the same frame and that's three RPCs per target -- multiplied by every nearby player who receives the multicast. In a 32-player match with an AoE hitting 10 targets, a single ability activation can produce 30+ cue RPCs in one frame.

This page covers how to batch those RPCs, when to make them unreliable, and how to measure the results.

!!! note "This page focuses on optimization"
    For the full explanation of the cue send context system and ability-level batching, see [Batching](../networking/batching.md). This page is about squeezing bandwidth when you've already profiled and know cues are the problem.

## Why Batch

The engine warns you when cue RPCs exceed the budget. The `CheckForTooManyRPCs` function in `UGameplayCueManager` tracks this at runtime:

| Scenario | Cue RPCs/Frame (per target) | With Batching |
|:---|:---|:---|
| Single damage hit | 1 | 1 (no benefit) |
| Damage + slow + burn | 3 | 1 |
| AoE hitting 10 targets (3 effects each) | 30 | 10 |
| Periodic tick (0.1s) + visual cue each tick | 10/sec | 10/sec (no benefit -- reduce tick rate instead) |

Batching helps when multiple cues fire in the same callstack. It does nothing for cues spread across frames.

## FScopedGameplayCueSendContext

This is an RAII wrapper around the cue manager's send context reference count. When the count is above zero, cue RPCs are queued instead of sent immediately. When the last `FScopedGameplayCueSendContext` goes out of scope, `FlushPendingCues()` fires and sends everything in a batch.

```cpp
void AMyCharacter::ApplyAoEEffects(const TArray<AActor*>& Targets)
{
    // One scope for the entire AoE -- all cues batch into a single flush
    FScopedGameplayCueSendContext CueBatchScope;

    for (AActor* Target : Targets)
    {
        UAbilitySystemComponent* TargetASC =
            UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Target);
        if (!TargetASC) continue;

        TargetASC->ApplyGameplayEffectToSelf(DamageEffect, Level, Context);
        TargetASC->ApplyGameplayEffectToSelf(SlowEffect, Level, Context);
        TargetASC->ApplyGameplayEffectToSelf(BurnEffect, Level, Context);
    }
}
// CueBatchScope destructor calls EndGameplayCueSendContext()
// which calls FlushPendingCues() since context count hits 0
```

### How It Works Internally

The source in `GameplayCueManager.cpp` is straightforward:

1. **Constructor** calls `StartGameplayCueSendContext()`, which increments `GameplayCueSendContextCount`
2. While the count is > 0, cues go into the `PendingExecuteCues` array instead of firing RPCs
3. **Destructor** calls `EndGameplayCueSendContext()`, which decrements the count
4. When the count reaches 0, `FlushPendingCues()` broadcasts `OnFlushPendingCues`, then iterates and sends
5. During flush, `DoesPendingCueExecuteMatch()` deduplicates -- if two pending cues match, only one RPC is sent

### Nesting

Send contexts are reference-counted, so nesting works correctly:

```cpp
{
    FScopedGameplayCueSendContext OuterScope;  // count = 1
    ApplyDamage(Target);  // cues queued
    {
        FScopedGameplayCueSendContext InnerScope;  // count = 2
        ApplyDebuffs(Target);  // cues queued
    }  // count = 1, no flush yet
    ApplyBurn(Target);  // still queued
}  // count = 0, ALL cues flush here
```

!!! warning "Don't let the count go negative"
    Calling `EndGameplayCueSendContext()` more times than `StartGameplayCueSendContext()` logs a warning and puts the system in a bad state. Always use the RAII wrapper instead of calling start/end manually.

## Reliability Settings

All 10 `NetMulticast_Invoke*` methods on the ASC are declared with `UFUNCTION(NetMulticast, Unreliable)`. This is intentional and correct for most games.

### When Unreliable Is Acceptable

**Almost always.** Here's the reasoning:

| Cue Type | Unreliable OK? | Why |
|:---|:---|:---|
| Hit impacts (Executed) | Yes | Cosmetic-only. Missing one hit spark is invisible at game speed. |
| Footstep/movement cues | Yes | High frequency, cosmetic. Dropping one is unnoticeable. |
| Periodic tick visuals | Yes | The next tick fires 0.1-0.5s later anyway. |
| Duration cue Add/Remove | Yes | The `FActiveGameplayCueContainer` replicates the **state** reliably via property replication. Even if the "add" RPC is dropped, the client sees the cue state on the next property update. |
| One-shot critical gameplay feedback | Maybe not | See below. |

### When You Might Need Reliable

If you have a cue that:

1. Fires exactly once (not periodic)
2. Has no property replication fallback (instant cue, not duration-based)
3. Communicates gameplay-critical information (not just cosmetic)

Then you might need a reliable path. But the right fix is usually to **not use a cue for gameplay-critical state** -- use a replicated property or a reliable RPC from your ability instead.

!!! tip "Don't make cues reliable to fix a design problem"
    If dropping a cue breaks gameplay, the problem is that gameplay depends on a cosmetic channel. Fix the dependency, not the reliability.

## Custom Batching Logic

Override these on your `UGameplayCueManager` subclass for game-specific control:

```cpp
UCLASS()
class UMyGameplayCueManager : public UGameplayCueManager
{
    GENERATED_BODY()

public:
    // Return false to reject a pending cue entirely
    virtual bool ProcessPendingCueExecute(
        FGameplayCuePendingExecute& PendingCue) override
    {
        // Example: suppress cues on actors beyond a distance threshold
        if (IsActorTooFarFromAnyViewer(PendingCue.OwningComponent))
        {
            return false;
        }
        return Super::ProcessPendingCueExecute(PendingCue);
    }

    // Return true if two pending cues should merge into one
    virtual bool DoesPendingCueExecuteMatch(
        FGameplayCuePendingExecute& PendingCue,
        FGameplayCuePendingExecute& ExistingCue) override
    {
        return Super::DoesPendingCueExecuteMatch(PendingCue, ExistingCue);
    }
};
```

Register it in your `UAbilitySystemGlobals` subclass or via the `GameplayAbilitiesDeveloperSettings` (`GlobalGameplayCueManagerClass`).

## Measuring Impact

### Before/After Methodology

1. Set up a reproducible test scenario (e.g., 10 bots using abilities in a confined area)
2. Measure **without** batching -- note RPC count and bandwidth
3. Add `FScopedGameplayCueSendContext` around your effect application code
4. Measure again

### Console Commands

```
net.ListNetRPCs              // RPC call counts per function
net.TrackRPCFrequency 1      // Log RPC frequency
stat net                     // Bandwidth overview
```

### What Good Looks Like

| Metric | Before Batching | After Batching |
|:---|:---|:---|
| Cue RPCs per AoE hit | 3 per target | 1 per target |
| Total multicast RPCs/frame | 30-50 | 10-15 |
| `CheckForTooManyRPCs` warnings | Frequent | None |

## Practical Checklist

- [ ] Wrap any code that applies 2+ effects to the same target in `FScopedGameplayCueSendContext`
- [ ] Wrap AoE loops (multiple targets) in a single scope
- [ ] Keep cue multicasts unreliable (they already are by default -- don't change them)
- [ ] Profile with `net.ListNetRPCs` to verify batching is working
- [ ] If periodic effects trigger cues, reduce tick rate rather than batching (batching doesn't help across frames)
- [ ] Consider `ProcessPendingCueExecute` overrides for distance-based suppression in large-scale games

## Related Pages

- [Batching](../networking/batching.md) -- the full networking explanation of ability and cue batching
- [Cue Manager](../gameplay-cues/cue-manager.md) -- how the manager routes and dispatches cues
- [Replication Optimization](replication-optimization.md) -- reducing GE replication overhead
- [Attribute Proxy](attribute-proxy.md) -- proxy interface for large-scale replication
