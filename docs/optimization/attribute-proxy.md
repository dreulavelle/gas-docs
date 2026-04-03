---
title: Attribute Replication Proxy
description: IAbilitySystemReplicationProxyInterface for large-scale games, how the proxy splits replication concerns, the Serialization/ net serializers, and implementation tradeoffs.
---

# Attribute Replication Proxy

When your game has 50+ replicated actors with ASCs -- think battle royale, MMO overworld, or large-scale PvP -- the default ASC replication model starts to hurt. Every ASC replicates its own attributes, tags, and cues independently, which means the server is doing redundant work and burning bandwidth on actors that most clients don't care about.

The `IAbilitySystemReplicationProxyInterface` lets you redirect that replication through a proxy actor with better relevancy control.

!!! note "This page focuses on optimization"
    For the full interface reference, multicast method signatures, and basic setup, see [Replication Proxy](../networking/replication-proxy.md). This page is about when the proxy pays for itself and how to get the most out of it.

## When You Need It

The proxy is **not** a default choice. It adds architectural complexity. Here's a rough decision guide:

| Game Scale | Replication Mode | Need Proxy? |
|:---|:---|:---|
| < 16 players | Full or Mixed | No |
| 16-32 players | Mixed | Probably not |
| 32-64 players | Mixed (players), Minimal (AI) | Maybe, profile first |
| 64+ players | Minimal for most | Yes |
| MMO (hundreds visible) | Minimal + proxy | Yes, mandatory |

The cost of **not** using a proxy at scale: every ASC replicates to every relevant client independently, even when the only data that matters to non-owners is "are they on fire?" (a single tag). The proxy lets you replicate that minimal data through an actor with proper distance-based culling.

## How It Works

The proxy pattern splits ASC replication into two paths:

```
Without proxy:
  ASC (on PlayerState) → replicates everything to all relevant clients

With proxy:
  ASC (on PlayerState) → full data to owning client only
  Proxy (on Pawn)      → minimal data (tags, cues) to other clients
                          inherits Pawn's distance-based relevancy
```

The key insight: the `PlayerState` is always relevant to its owner, so the ASC on it gives the owning client full fidelity. But the `PlayerState` has no distance culling -- it's relevant to everyone. The Pawn does have culling. By routing minimal GAS data through a proxy on the Pawn, you only send tag/cue updates to clients who are actually close enough to see the effects.

### What the Proxy Replicates

In a typical setup:

| Data | Replicated By | To Whom |
|:---|:---|:---|
| Full GE specs, stacking, duration | ASC (on PlayerState) | Owner only |
| Attributes (Health, MaxHealth) | ASC AttributeSet `DOREPLIFETIME` | Depends on condition |
| Gameplay tags | `FMinimalReplicationTagCountMap` on proxy | Relevant clients |
| Duration cues (active state) | `FMinimalGameplayCueReplicationProxy` on proxy | Relevant clients |
| Instant cue RPCs | `NetMulticast_Invoke*` on proxy | Relevant clients |

## The Interface

`IAbilitySystemReplicationProxyInterface` defines three categories:

### ForceReplication

```cpp
virtual void ForceReplication() = 0;
```

Called when the ASC needs to ensure pending data is sent immediately. Your implementation should call `ForceNetUpdate()` on the proxy actor.

### Call Proxies (10 methods)

These intercept outgoing cue events before they become RPCs. The default implementation just forwards to the corresponding multicast. Override them on your proxy if you need to filter, throttle, or redirect cue RPCs:

```cpp
// Default: just forwards to the multicast
void IAbilitySystemReplicationProxyInterface::Call_InvokeGameplayCueExecuted(
    const FGameplayTag GameplayCueTag,
    FPredictionKey PredictionKey,
    FGameplayEffectContextHandle EffectContext)
{
    NetMulticast_InvokeGameplayCueExecuted(GameplayCueTag, PredictionKey, EffectContext);
}
```

### NetMulticast Proxies (10 methods, pure virtual)

These are the actual RPCs. They **must** be implemented as `UFUNCTION(NetMulticast, Unreliable)` on whatever actor does the replication:

```cpp
UFUNCTION(NetMulticast, Unreliable)
void NetMulticast_InvokeGameplayCueExecuted_FromSpec(...) override;

UFUNCTION(NetMulticast, Unreliable)
void NetMulticast_InvokeGameplayCueExecuted(...) override;

// ... 8 more
```

## Implementation Guide

### Step 1: Decide Where the Proxy Lives

The most common pattern: the proxy is the Pawn itself. The Pawn already has distance-based relevancy and net culling, so it's a natural fit.

```cpp
UCLASS()
class AMyPawn : public ACharacter, public IAbilitySystemReplicationProxyInterface
{
    GENERATED_BODY()

public:
    // Minimal cue proxy for non-owner clients
    UPROPERTY(Replicated)
    FMinimalGameplayCueReplicationProxy MinimalCueProxy;

    // Minimal tag map for non-owner clients
    UPROPERTY(Replicated)
    FMinimalReplicationTagCountMap MinimalTagMap;
};
```

### Step 2: Implement the Interface

```cpp
void AMyPawn::ForceReplication()
{
    ForceNetUpdate();
}

// Implement all 10 NetMulticast methods
UFUNCTION(NetMulticast, Unreliable)
void AMyPawn::NetMulticast_InvokeGameplayCueExecuted_FromSpec(
    const FGameplayEffectSpecForRPC Spec,
    FPredictionKey PredictionKey)
{
    // Forward to the cue manager for local handling
    UAbilitySystemGlobals::Get().GetGameplayCueManager()
        ->InvokeGameplayCueExecuted_FromSpec(GetAbilitySystemComponent(), Spec, PredictionKey);
}

// ... repeat for all 10 methods
```

### Step 3: Wire Up Replication

```cpp
void AMyPawn::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // Only send to non-owner connections -- the owner gets full data from the ASC
    DOREPLIFETIME_CONDITION(AMyPawn, MinimalCueProxy, COND_SkipOwner);
    DOREPLIFETIME_CONDITION(AMyPawn, MinimalTagMap, COND_SkipOwner);
}

void AMyPawn::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);

    if (UAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        MinimalCueProxy.PreReplication(ASC->GetActiveGameplayCueContainer());
        // MinimalTagMap is updated by the ASC automatically
    }
}
```

### Step 4: Set the ASC's Replication Mode

```cpp
ASC->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);
```

With Minimal mode, the ASC stops replicating `FActiveGameplayEffect` entries. Tags and cues flow through the proxy instead.

## The Serialization/ Folder

The GAS plugin includes specialized net serializers that work with UE's Iris replication system. These handle the binary packing of GAS types over the network:

| Serializer | Purpose |
|:---|:---|
| `GameplayAbilityRepAnimMontageNetSerializer` | Montage replication data (sections, position, etc.) |
| `GameplayAbilityTargetDataHandleNetSerializer` | Target data handles for abilities |
| `GameplayAbilityTargetingLocationInfoNetSerializer` | Targeting location (transform + source) |
| `GameplayEffectContextHandleNetSerializer` | GE context handles (polymorphic) |
| `GameplayEffectContextNetSerializer` | GE context struct internals |
| `GameplayTagCountContainerNetSerializer` | Full tag count containers |
| `MinimalGameplayCueReplicationProxyNetSerializer` | Minimal cue proxy (compact format) |
| `MinimalReplicationTagCountNetSerializer` | Minimal tag count map (compact format) |
| `PredictionKeyNetSerializer` | Prediction keys |

Plus two replication fragments for Iris:

| Fragment | Purpose |
|:---|:---|
| `MinimalGameplayCueReplicationProxyReplicationFragment` | Iris fragment for the cue proxy |
| `MinimalReplicationTagCountMapReplicationFragment` | Iris fragment for the tag count map |

These are internal to the engine. You don't touch them directly unless you're building a custom replication backend. But knowing they exist explains why the minimal proxy types are so bandwidth-efficient -- they use purpose-built binary serializers instead of generic property replication.

## Tradeoffs

### What You Gain

- **Relevancy-based culling**: Non-owner clients only receive GAS data when the proxy actor is relevant (within net cull distance)
- **Bandwidth reduction**: Minimal proxy types pack tags and cues into compact binary formats instead of full `FActiveGameplayEffect` replication
- **Scalability**: At 100+ actors, the difference between "every ASC replicates to everyone" and "proxy replicates to relevant clients" is massive

### What You Pay

- **Complexity**: You're now managing two replication paths. Bugs in the proxy setup can cause cues to not play or tags to appear stale on non-owner clients.
- **10 boilerplate methods**: Every NetMulticast function must be implemented as a proper UFUNCTION on the proxy actor. That's a lot of forwarding code.
- **Lifecycle management**: The proxy must outlive the ASC, or handle graceful degradation. If the Pawn dies and respawns, the proxy state needs to reset.
- **Non-owner data is lossy**: Non-owner clients see Minimal data only. They can't query "what effects are active on that player?" -- they only see tags and cues.
- **Debugging is harder**: When a cue doesn't play on a remote client, you have to check both the ASC path and the proxy path.

### When It's Not Worth It

- Under 32 players with Mixed mode on players and Minimal on AI
- Games where all actors are always relevant (small arena, no distance culling)
- Singleplayer or local multiplayer

## Bandwidth Comparison

Rough numbers for a character with 5 active effects, 8 tags, and 2 active cues:

| Mode | Per-Connection Bandwidth | Notes |
|:---|:---|:---|
| Full (no proxy) | ~3-5 KB initial + deltas | Every client gets everything |
| Mixed (no proxy) | ~1-3 KB initial + deltas | GE specs to owner only, cues to all |
| Minimal + proxy | ~50-200 bytes/update | Tags + cue state only, compact serialization |

The difference compounds linearly with player count. At 64 players, the proxy can reduce total GAS bandwidth by 80-90%.

## Related Pages

- [Replication Proxy](../networking/replication-proxy.md) -- full interface reference and setup details
- [Replication Optimization](replication-optimization.md) -- choosing replication modes
- [Cue Batching](cue-batching.md) -- reducing cue RPC count
- [Lazy Loading](lazy-loading.md) -- avoiding hitches when cues first play
