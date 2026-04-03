---
title: Batching
description: Ability activation batching, cue batching, FScopedServerAbilityRPCBatcher, and measuring the impact on bandwidth and performance.
---

# Batching

In a networked GAS game, abilities and cues generate RPCs. A single ability activation can produce several: the activation RPC, cue RPCs for effects applied during activation, and montage RPCs. When many abilities fire simultaneously (a crowd of players all attacking at once), this can saturate the RPC budget.

Batching reduces this by combining multiple RPCs into fewer, larger ones.

## Ability Activation Batching

### The Problem

A typical predicted ability activation generates:

1. `ServerTryActivateAbility` RPC (client → server)
2. `ClientActivateAbilitySucceed` or `ClientActivateAbilityFailed` RPC (server → client)
3. Gameplay Cue multicast RPCs for any cues triggered during activation
4. Potentially a montage replication update

For a fast-attacking character, this can be 3-5 RPCs per attack.

### FScopedServerAbilityRPCBatcher

The `FScopedServerAbilityRPCBatcher` is an RAII object that batches ability-related RPCs within its scope:

```cpp
// On the server, batch RPCs for this ASC
{
    FScopedServerAbilityRPCBatcher Batcher(AbilitySystemComponent, ActivationInfo);
    // All RPCs generated in this scope are batched
    // They're sent as a single combined RPC when the scope ends
}
```

The ASC uses this internally during ability activation. You typically don't need to manage it manually unless you're doing custom activation flows.

### How It Works

While the batcher is active:

- Cue RPCs are queued instead of sent immediately
- The ability activation response is held
- When the scope closes, everything is packaged into a single RPC

This is especially effective when an ability triggers multiple cues or applies multiple effects during its activation callstack.

## Cue Batching

Gameplay Cue RPCs can be batched independently of ability batching through the `GameplayCueManager`'s send context system.

### Send Context

```cpp
UGameplayCueManager* CueManager = UAbilitySystemGlobals::Get().GetGameplayCueManager();

CueManager->StartGameplayCueSendContext();

// Apply multiple effects that trigger cues...
ASC->ApplyGameplayEffectToSelf(DamageEffect, Level, Context);
ASC->ApplyGameplayEffectToSelf(BurnEffect, Level, Context);
ASC->ApplyGameplayEffectToSelf(SlowEffect, Level, Context);
// Each would normally fire its own cue RPC

CueManager->EndGameplayCueSendContext();
// All three cues are flushed in one batch
```

The RAII wrapper `FScopedGameplayCueSendContext` is the preferred way to use this:

```cpp
{
    FScopedGameplayCueSendContext CueContext;
    // Everything in this scope batches cue RPCs
    ASC->ApplyGameplayEffectToSelf(Effect1, Level, Context);
    ASC->ApplyGameplayEffectToSelf(Effect2, Level, Context);
}
// Cues flushed here
```

Send contexts are reference-counted -- you can nest them, and cues only flush when the last context closes.

### Pending Cue Processing

While batching, cues accumulate in the manager's `PendingExecuteCues` array. During `FlushPendingCues()`:

1. Each pending cue goes through `ProcessPendingCueExecute()` -- which can reject it
2. Duplicates are detected via `DoesPendingCueExecuteMatch()` and merged
3. The remaining cues are sent as a batch

Override `ProcessPendingCueExecute` or `DoesPendingCueExecuteMatch` in a custom `UGameplayCueManager` subclass for game-specific batching logic.

### The RPC Budget

The ASC logs a warning when too many cue RPCs are generated in a single frame. The `CheckForTooManyRPCs` function in the manager tracks this. If you're seeing these warnings, batching or reliability changes are your primary tools.

## Reliability Settings

By default, most gameplay cue RPCs are **unreliable multicast**. This is intentional -- cues are cosmetic, and dropping one occasionally is acceptable. The alternative (reliable RPCs) can cause RPC saturation and connection issues under load.

The multicast functions on `IAbilitySystemReplicationProxyInterface` are defined as unreliable:

```cpp
// These are all unreliable NetMulticast
virtual void NetMulticast_InvokeGameplayCueExecuted_FromSpec(...) = 0;
virtual void NetMulticast_InvokeGameplayCueExecuted(...) = 0;
virtual void NetMulticast_InvokeGameplayCuesExecuted(...) = 0;
// ... etc
```

For duration cues (Add/Remove), the state is replicated through the `FActiveGameplayCueContainer` (a `FFastArraySerializer`), which uses reliable property replication. So even if the "add" RPC is lost, the client will eventually see the cue state through property replication catch-up.

!!! tip "Unreliable is usually fine"
    A dropped hit impact cue is almost never noticeable. A dropped "add aura" event is caught by property replication. Only override to reliable if you have a specific, measurable need.

## Measuring Impact

### Network Profiler

Use UE's built-in network profiler to measure RPC counts and bandwidth:

1. Run with `-networkprofiler` command line argument
2. Open the profiler from **Window > Developer Tools > Network Profiler**
3. Look for `GameplayCue` and `AbilitySystem` RPCs
4. Compare before/after batching changes

### Console Commands

```
net.ListNetRPCs          // List all RPCs and their call counts
stat net                 // General networking stats
net.TrackRPCFrequency 1  // Track RPC frequency
```

### What to Look For

| Metric | Healthy | Concerning |
|:---|:---|:---|
| Cue RPCs per frame (per ASC) | 0-3 | 5+ |
| Total RPCs per frame | < 50 | 100+ |
| RPC saturation warnings | None | Any |
| Bandwidth per ASC | < 1 KB/s | 5+ KB/s |

## Practical Recommendations

1. **Always use `FScopedGameplayCueSendContext`** when applying multiple effects in a loop
2. **Keep cues unreliable** unless you have a specific reason not to
3. **Batch server-side processing** -- if your game logic applies 5 effects to a target in one action, wrap it in a send context
4. **Measure before optimizing** -- batching adds complexity; only do it if you've measured a problem
5. **Consider cue frequency** -- if an effect ticks every 0.1s and triggers a cue each tick, that's 10 cue RPCs/second per target. Reduce tick rate or suppress intermediate cues.

## Related Pages

- [Prediction](prediction.md) -- how batching interacts with prediction
- [Cue Manager](../gameplay-cues/cue-manager.md) -- the cue send context system
- [Cue Batching (Optimization)](../optimization/cue-batching.md) -- performance-focused guidance
- [Replication Proxy](replication-proxy.md) -- moving RPCs to a proxy actor
