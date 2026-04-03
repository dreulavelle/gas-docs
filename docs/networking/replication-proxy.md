---
title: Replication Proxy
description: IAbilitySystemReplicationProxyInterface for offloading GAS replication, the net serializers in the Serialization folder, and setup guide.
---

# Replication Proxy

The `IAbilitySystemReplicationProxyInterface` lets you move GAS replication off the `UAbilitySystemComponent` and onto a separate "proxy" actor. This is primarily useful when you need the ASC to live on one actor (like a PlayerState) but want replication to flow through a different actor (like a Pawn) that has better relevancy control.

## When You Need This

The proxy interface becomes important when:

- **50+ players**: You need Minimal replication mode, and the default ASC replication isn't sufficient
- **ASC on PlayerState, replication on Pawn**: The PlayerState is always relevant to its owner, but the Pawn has distance-based relevancy. You want cue and tag replication to follow pawn relevancy.
- **Custom replication actors**: Your game has a dedicated replication actor per player (common in large-scale games)
- **Splitting replication concerns**: You want different actors to handle different parts of GAS replication

For most games (< 32 players), you don't need this. The ASC's built-in replication works fine.

## IAbilitySystemReplicationProxyInterface

The interface defines two categories of methods:

### Call Proxies

These intercept outgoing GameplayCue RPCs. By default, they forward to the corresponding multicast methods:

```cpp
virtual void Call_InvokeGameplayCueExecuted_FromSpec(...);
virtual void Call_InvokeGameplayCueExecuted(...);
virtual void Call_InvokeGameplayCuesExecuted(...);
virtual void Call_InvokeGameplayCueExecuted_WithParams(...);
virtual void Call_InvokeGameplayCuesExecuted_WithParams(...);
virtual void Call_InvokeGameplayCueAdded(...);
virtual void Call_InvokeGameplayCueAdded_WithParams(...);
virtual void Call_InvokeGameplayCueAddedAndWhileActive_FromSpec(...);
virtual void Call_InvokeGameplayCueAddedAndWhileActive_WithParams(...);
virtual void Call_InvokeGameplayCuesAddedAndWhileActive_WithParams(...);
```

Override these on your proxy actor to intercept and redirect cue RPCs.

### Multicast Proxies

These are the actual network multicast functions. They **must** be implemented as `UFUNCTION(NetMulticast, Unreliable)`:

```cpp
virtual void NetMulticast_InvokeGameplayCueExecuted_FromSpec(...) = 0;
virtual void NetMulticast_InvokeGameplayCueExecuted(...) = 0;
virtual void NetMulticast_InvokeGameplayCuesExecuted(...) = 0;
// ... (10 total)
```

### ForceReplication

```cpp
virtual void ForceReplication() = 0;
```

Called when the ASC needs to ensure pending replication data is sent. Typically calls `ForceNetUpdate()` on the proxy actor.

## FMinimalGameplayCueReplicationProxy

For Minimal replication mode, the `FMinimalGameplayCueReplicationProxy` provides a lightweight alternative to the full `FActiveGameplayCueContainer`:

```cpp
USTRUCT()
struct FMinimalGameplayCueReplicationProxy
{
    // Set owning ASC
    void SetOwner(UAbilitySystemComponent* ASC);

    // Copy data from ASC's cue container (call from PreReplication)
    void PreReplication(const FActiveGameplayCueContainer& SourceContainer);

    // Custom net serialization
    bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess);

    // Clear all cues
    void RemoveAllCues();

    // Skip updates for owner connection
    void SetRequireNonOwningNetConnection(bool b);
};
```

Instead of replicating the full `FActiveGameplayCueContainer` with FastArraySerializer overhead, this packs cue tags and locations into a compact format.

## Net Serializers

The `Serialization/` folder under the GAS plugin contains specialized net serializers for various GAS types. These work with UE's Iris replication system:

| Serializer | What It Serializes |
|:---|:---|
| `GameplayAbilityRepAnimMontageNetSerializer` | Ability montage replication data |
| `GameplayAbilityTargetDataHandleNetSerializer` | Target data handles |
| `GameplayAbilityTargetingLocationInfoNetSerializer` | Targeting location info |
| `GameplayEffectContextHandleNetSerializer` | Effect context handles |
| `GameplayEffectContextNetSerializer` | Effect context structs |
| `GameplayTagCountContainerNetSerializer` | Tag count containers |
| `MinimalGameplayCueReplicationProxyNetSerializer` | Minimal cue proxy data |
| `MinimalReplicationTagCountNetSerializer` | Minimal tag count data |

Plus the replication fragment:

| Fragment | Purpose |
|:---|:---|
| `MinimalGameplayCueReplicationProxyReplicationFragment` | Iris replication fragment for the minimal cue proxy |

These are internal to the engine's networking layer and you typically don't interact with them directly unless you're building a custom replication system.

## Setup Guide

### Step 1: Create Your Proxy Actor

```cpp
UCLASS()
class AMyReplicationProxy : public AActor, public IAbilitySystemReplicationProxyInterface
{
    GENERATED_BODY()

public:
    virtual void ForceReplication() override
    {
        ForceNetUpdate();
    }

    // Implement all NetMulticast functions as UFUNCTION(NetMulticast, Unreliable)
    UFUNCTION(NetMulticast, Unreliable)
    virtual void NetMulticast_InvokeGameplayCueExecuted_FromSpec(
        const FGameplayEffectSpecForRPC Spec,
        FPredictionKey PredictionKey) override;

    UFUNCTION(NetMulticast, Unreliable)
    virtual void NetMulticast_InvokeGameplayCueExecuted(
        const FGameplayTag GameplayCueTag,
        FPredictionKey PredictionKey,
        FGameplayEffectContextHandle EffectContext) override;

    // ... implement all 10 multicast methods ...

    // For Minimal mode: the lightweight cue proxy
    UPROPERTY(Replicated)
    FMinimalGameplayCueReplicationProxy MinimalCueProxy;

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(AMyReplicationProxy, MinimalCueProxy);
    }

    virtual void PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker) override
    {
        Super::PreReplication(ChangedPropertyTracker);
        if (OwningASC)
        {
            MinimalCueProxy.PreReplication(OwningASC->GetActiveGameplayCueContainer());
        }
    }

    void SetOwningASC(UAbilitySystemComponent* ASC)
    {
        OwningASC = ASC;
        MinimalCueProxy.SetOwner(ASC);
    }

private:
    UPROPERTY()
    TObjectPtr<UAbilitySystemComponent> OwningASC;
};
```

### Step 2: Wire Up the ASC

The ASC already implements `IAbilitySystemReplicationProxyInterface`. Its default implementation sends RPCs through itself. To redirect through your proxy, override the `Call_*` methods on the ASC to forward to your proxy actor, or set up the ASC to use a different replication proxy.

### Step 3: Handle Minimal Cue Proxy

For `FMinimalGameplayCueReplicationProxy`:

1. Call `SetOwner()` with the ASC after both are initialized
2. Call `PreReplication()` in the proxy actor's `PreReplication()` to sync the cue state
3. Optionally call `SetRequireNonOwningNetConnection(true)` to skip sending to the owning client (they already have the data from the ASC)
4. Call `RemoveAllCues()` during cleanup

### Step 4: Set Replication Mode

```cpp
ASC->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);
```

## Practical Considerations

**Relevancy:** The whole point of using a proxy is often to tie GAS replication to an actor with distance-based relevancy. Make sure your proxy actor has the right `NetCullDistanceSquared` and relevancy settings.

**Ownership:** The proxy actor's net owner determines who receives owner-only replicated properties. Make sure ownership is set correctly.

**Lifecycle:** The proxy needs to outlive the ASC, or at minimum handle the case where the ASC is destroyed first. Use weak pointers and null checks.

**Testing:** This setup is complex. Test with `net.PktLoss` and `net.PktLag` to verify behavior under adverse conditions.

## Related Pages

- [Replication Modes](replication-modes.md) -- Minimal mode which requires this proxy
- [Replication Optimization](../optimization/replication-optimization.md) -- performance guidance
- [Attribute Proxy](../optimization/attribute-proxy.md) -- optimization-focused proxy guide
- [Batching](batching.md) -- reducing RPC count through the proxy
