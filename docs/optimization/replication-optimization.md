---
title: Replication Optimization
description: Choosing the right replication mode, Minimal mode deep dive, attribute subobject registration, and bandwidth measurement.
---

# Replication Optimization

The replication mode you choose for each ASC is the highest-impact optimization decision in GAS networking. This page is the practical follow-up to [Replication Modes](../networking/replication-modes.md) -- focused on implementation and measurement.

## Quick Wins

### 1. Use Mixed Mode for Players, Minimal for AI

```cpp
// Player characters
PlayerASC->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

// AI enemies
EnemyASC->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);
```

AI rarely needs to replicate full effect specs to clients. Tags and cues are usually sufficient.

### 2. Minimize Replicated Attributes

Only replicate attributes that clients actually need. Health, MaxHealth, and a few combat stats are typical. Internal calculation attributes (DamageMultiplier, InternalCooldownReduction) don't need replication.

```cpp
void UMyAttributeSet::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // Replicate what clients need to see
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);

    // DON'T replicate internal attributes
    // IncomingDamage - meta attribute, never replicated
    // InternalCritBonus - only used server-side
}
```

### 3. Condition-Based Replication

Use `COND_OwnerOnly` for attributes that only the owning player needs:

```cpp
DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Mana, COND_OwnerOnly, REPNOTIFY_Always);
DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Stamina, COND_OwnerOnly, REPNOTIFY_Always);
```

Other players don't need to know your mana count (unless your game shows it).

## Minimal Mode Deep Dive

Minimal mode stops replicating `FActiveGameplayEffect` entries entirely. What still works:

| Feature | Status in Minimal |
|:---|:---|
| Tags | Replicated via `FMinimalReplicationTagCountMap` |
| Cues (duration) | Via `FMinimalGameplayCueReplicationProxy` |
| Cues (instant) | Via unreliable multicast RPC |
| Attributes | Via `DOREPLIFETIME` on the AttributeSet (unchanged) |
| Active GE specs | **Not replicated** |
| GE duration/stacking info | **Not replicated** |
| Prediction reconciliation | **Not supported without proxy** |

### When to Use

- AI-controlled characters (50+ actors)
- Background/environmental actors with ASCs
- Any ASC where clients don't need the effect list

### Setting Up Minimal for Non-Player ASCs

```cpp
AMyAICharacter::AMyAICharacter()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("ASC"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);
}
```

For non-player ASCs, this is usually all you need. Tags replicate for tag queries, cues replicate for visual feedback, and you save all the bandwidth from effect spec replication.

## Attribute Subobject Registration

By default, attribute sets are discovered as subobjects of the ASC's owner. Make sure they're properly registered:

```cpp
AMyCharacter::AMyCharacter()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("ASC"));
    AttributeSet = CreateDefaultSubobject<UMyAttributeSet>(TEXT("AttributeSet"));
}
```

The attribute set is automatically registered because it's a subobject of the same actor that owns the ASC. If you create attribute sets dynamically, register them explicitly:

```cpp
ASC->AddAttributeSetSubobject(NewObject<UMyAttributeSet>(ASCOwner));
```

## Bandwidth Measurement

### Method 1: Network Profiler

Run with `-networkprofiler` and analyze the output:

1. Look for `AbilitySystemComponent` in the actor list
2. Check property replication bandwidth (attribute changes, GE array changes)
3. Check RPC bandwidth (cue multicasts, ability activation)

### Method 2: Stat Net

```
stat net
```

Watch `OutBunch` and `OutBytes` — compare before and after changing replication modes.

### Method 3: Manual Counting

For a rough estimate:

- Each `FActiveGameplayEffect` replicates ~200-500 bytes (varies with spec complexity)
- Each attribute change is ~8-12 bytes
- Each tag change is ~4-8 bytes
- Each cue RPC is ~20-50 bytes

With Mixed mode and 10 active effects on a character: ~2-5 KB initial, then delta updates.

With Minimal mode: only attribute changes and tag/cue deltas, often under 100 bytes/frame.

## Advanced: Attribute Replication Filtering

For very large player counts, you can implement custom replication filtering to only send attribute updates when they change by more than a threshold:

```cpp
void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    // Only mark dirty if change is significant
    // (This is a conceptual example -- actual implementation
    //  depends on your replication strategy)
}
```

This is an advanced technique for MMO-scale games and isn't needed for most projects.

## Related Pages

- [Replication Modes](../networking/replication-modes.md) -- the three modes explained
- [Attribute Proxy](attribute-proxy.md) -- proxy interface for large-scale
- [Cue Batching](cue-batching.md) -- reducing cue bandwidth
