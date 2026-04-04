---
title: Target Data
description: FGameplayAbilityTargetData — the polymorphic payload that carries targeting results between abilities, tasks, and effects.
---

# Target Data

`FGameplayAbilityTargetData` is the currency of the targeting system. It is a polymorphic struct that carries targeting results — hit results, actor lists, world locations, or custom data — from the targeting step to wherever it needs to go (effect application, spawning, custom logic).

## FGameplayAbilityTargetData — The Base

The base struct is abstract and defines the interface that all target data types share:

```cpp
USTRUCT()
struct FGameplayAbilityTargetData
{
    // Returns all targeted actors
    virtual TArray<TWeakObjectPtr<AActor>> GetActors() const;

    // Override to return a hit result
    virtual const FHitResult* GetHitResult() const;

    // Returns an origin transform
    virtual FTransform GetOrigin() const;

    // Returns a target/end point
    virtual FVector GetEndPoint() const;
    virtual FTransform GetEndPointTransform() const;

    // Apply a gameplay effect to each target
    TArray<FActiveGameplayEffectHandle> ApplyGameplayEffect(
        const UGameplayEffect* GameplayEffect,
        const FGameplayEffectContextHandle& InEffectContext,
        float Level, FPredictionKey PredictionKey);

    // Apply a pre-built spec
    virtual TArray<FActiveGameplayEffectHandle> ApplyGameplayEffectSpec(
        FGameplayEffectSpec& Spec, FPredictionKey PredictionKey);
};
```

The key thing to understand is that target data is **polymorphic**. The base struct defines the interface, and subclasses provide the actual data. This lets the same consuming code work regardless of whether the target data came from a line trace, a radius overlap, or a custom selection method.

## FGameplayAbilityTargetDataHandle — The Container

You never pass raw `FGameplayAbilityTargetData` around. Instead, you use `FGameplayAbilityTargetDataHandle`, which is a smart container that:

- Holds one or more `FGameplayAbilityTargetData` entries (via `TSharedPtr`)
- Supports polymorphism (each entry can be a different subclass)
- Implements `NetSerialize` for replication
- Is the type used in Blueprint pins and function signatures

```cpp
FGameplayAbilityTargetDataHandle Handle;

// Add target data entries
FGameplayAbilityTargetData_SingleTargetHit* HitData =
    new FGameplayAbilityTargetData_SingleTargetHit(HitResult);
Handle.Add(HitData);

// Iterate
for (int32 i = 0; i < Handle.Num(); ++i)
{
    const FGameplayAbilityTargetData* Data = Handle.Get(i);
    if (Data)
    {
        TArray<TWeakObjectPtr<AActor>> Actors = Data->GetActors();
        // ...
    }
}
```

!!! warning "Ownership and lifetime"
    Target data inside a handle is managed via `TSharedPtr` with a custom deleter (`FGameplayAbilityTargetDataDeleter`) that properly destructs the polymorphic struct. Use `new` to create target data and `Handle.Add()` to transfer ownership — the handle will clean it up.

## Built-in Target Data Types

The engine provides several concrete target data types.

### FGameplayAbilityTargetData_SingleTargetHit

The most commonly used type. Wraps a single `FHitResult`:

```cpp
USTRUCT()
struct FGameplayAbilityTargetData_SingleTargetHit : public FGameplayAbilityTargetData
{
    UPROPERTY()
    FHitResult HitResult;

    UPROPERTY()
    bool bHitReplaced = false;
};
```

This is what you get from line traces. It provides the hit actor, impact location, normal, bone name, and all other `FHitResult` data.

### FGameplayAbilityTargetData_LocationInfo

Contains two `FGameplayAbilityTargetingLocationInfo` entries — a source location and a target location. Used for location-based targeting (e.g., "cast from here to there"):

```cpp
USTRUCT()
struct FGameplayAbilityTargetData_LocationInfo : public FGameplayAbilityTargetData
{
    UPROPERTY()
    FGameplayAbilityTargetingLocationInfo SourceLocation;

    UPROPERTY()
    FGameplayAbilityTargetingLocationInfo TargetLocation;
};
```

`FGameplayAbilityTargetingLocationInfo` supports three location types:

| Type | Description |
|:---|:---|
| `LiteralTransform` | A raw world transform |
| `ActorTransform` | The transform of an associated actor |
| `SocketTransform` | A socket on the player's skeletal mesh |

### FGameplayAbilityTargetData_ActorArray

Holds a list of actors, useful for AoE abilities or multi-target skills:

```cpp
USTRUCT()
struct FGameplayAbilityTargetData_ActorArray : public FGameplayAbilityTargetData
{
    UPROPERTY()
    FGameplayAbilityTargetingLocationInfo SourceLocation;

    UPROPERTY()
    TArray<TWeakObjectPtr<AActor>> TargetActorArray;
};
```

## Creating Custom Target Data

For project-specific targeting needs, subclass `FGameplayAbilityTargetData`:

```cpp
USTRUCT()
struct FMyTargetData_Terrain : public FGameplayAbilityTargetData
{
    GENERATED_USTRUCT_BODY()

    UPROPERTY()
    FVector TargetLocation;

    UPROPERTY()
    float AreaRadius;

    UPROPERTY()
    FGameplayTag TerrainType;

    virtual TArray<TWeakObjectPtr<AActor>> GetActors() const override
    {
        // Could do an overlap check here, or return empty
        return {};
    }

    virtual FVector GetEndPoint() const override
    {
        return TargetLocation;
    }

    virtual bool HasEndPoint() const override
    {
        return true;
    }

    virtual UScriptStruct* GetScriptStruct() const override
    {
        return FMyTargetData_Terrain::StaticStruct();
    }
};
```

!!! important "GetScriptStruct is required for serialization"
    If you want your custom target data to replicate, you **must** override `GetScriptStruct()` to return the correct struct type. This is how the serialization system knows which struct to construct on the receiving end.

### Registering for NetSerialize

Custom target data types need to be registered with the gameplay ability target data serialization system. You do this by implementing `NetSerialize` and registering the struct:

```cpp
bool FMyTargetData_Terrain::NetSerialize(
    FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
{
    Ar << TargetLocation;
    Ar << AreaRadius;
    TerrainType.NetSerialize(Ar, Map, bOutSuccess);
    bOutSuccess = true;
    return true;
}
```

## Using Target Data with Effects

The common flow is: receive target data, create an effect spec, apply it to the targets:

```cpp
void UMyAbility::OnTargetDataReady(
    const FGameplayAbilityTargetDataHandle& Data)
{
    FGameplayEffectSpecHandle SpecHandle =
        MakeOutgoingGameplayEffectSpec(DamageEffectClass, GetAbilityLevel());

    // Apply to all targets in the handle
    TArray<FActiveGameplayEffectHandle> AppliedHandles =
        K2_ApplyGameplayEffectSpecToTarget(SpecHandle, Data);

    K2_EndAbility();
}
```

The `ApplyGameplayEffectSpec` on `FGameplayAbilityTargetData` will route the effect to each targeted actor, using the hit result or actor list from the target data.

## Confirmation Types

When using target actors to produce target data, the confirmation behavior is controlled by `EGameplayTargetingConfirmation`:

| Confirmation | Behavior |
|:---|:---|
| `Instant` | Data is produced immediately, no user input |
| `UserConfirmed` | Waits for the user to press confirm |
| `Custom` | The ability decides when data is ready |
| `CustomMulti` | Like Custom, but does not destroy the target actor on data production |

These are set on the `WaitTargetData` [ability task](../ability-tasks.md) and control when the target actor sends its data back.

## Related Pages

- [Target Actors](target-actors.md) -- the world actors that produce target data via traces, overlaps, and player input
- [Reticles and Filters](reticles-and-filters.md) -- visual feedback and validation rules applied during targeting
- [Targeting Overview](index.md) -- how target data, target actors, and reticles fit together
