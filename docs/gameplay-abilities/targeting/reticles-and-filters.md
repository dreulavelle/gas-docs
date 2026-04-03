---
title: Reticles and Filters
description: AGameplayAbilityWorldReticle for visual feedback during targeting, and FGameplayTargetDataFilter for controlling which actors are valid targets.
---

# Reticles and Filters

When a player is in the middle of targeting, they need two things: **visual feedback** showing what they are aiming at, and **validation rules** controlling which targets are valid. Reticles and filters handle these two concerns.

## World Reticles

`AGameplayAbilityWorldReticle` is a world-space actor that provides visual feedback during targeting. It is the circle on the ground for an AoE ability, the highlight on a valid target, or the preview mesh for a placement ability.

### The Base Class: AGameplayAbilityWorldReticle

Reticles are designed to be subclassed in Blueprint. The base class provides the framework, and you fill in the visuals:

```cpp
UCLASS(Blueprintable, notplaceable)
class AGameplayAbilityWorldReticle : public AActor
{
    // Initialization
    void InitializeReticle(
        AGameplayAbilityTargetActor* InTargetingActor,
        APlayerController* PlayerController,
        FWorldReticleParameters InParameters);

    // Blueprint events to override
    void OnValidTargetChanged(bool bNewValue);      // Target validity changed
    void OnTargetingAnActor(bool bNewValue);         // Targeting an actor vs empty space
    void OnParametersInitialized();                  // Parameters are ready

    // Material parameter helpers
    void SetReticleMaterialParamFloat(FName ParamName, float Value);
    void SetReticleMaterialParamVector(FName ParamName, FVector Value);

    // Utility
    void FaceTowardSource(bool bFaceIn2D);
};
```

### Key Properties

| Property | Type | Purpose |
|:---|:---|:---|
| `Parameters` | `FWorldReticleParameters` | Contains an `AOEScale` vector for sizing the reticle. Usage depends on your reticle implementation. |
| `bFaceOwnerFlat` | `bool` | If true, the reticle faces the owner in 2D (yaw only). Default `true`. |
| `bSnapToTargetedActor` | `bool` | If true, snaps to the targeted actor's location instead of the trace hit point. |
| `bIsTargetValid` | `bool` (read-only) | Whether the current target passes validation. Set by the target actor. |
| `bIsTargetAnActor` | `bool` (read-only) | Whether the reticle is currently over an actor (vs. the ground or empty space). |

### Blueprint Events

The reticle base class exposes several `BlueprintImplementableEvent` functions that you override in your Blueprint subclass:

- **OnValidTargetChanged** — fires when `bIsTargetValid` changes. Use this to change the reticle color (green for valid, red for invalid).
- **OnTargetingAnActor** — fires when the reticle transitions between targeting an actor and targeting empty space.
- **OnParametersInitialized** — fires when the reticle receives its parameters. Use this to set up scale, materials, etc.
- **SetReticleMaterialParamFloat/Vector** — set material instance parameters on the reticle mesh. Called by the target actor to communicate data.

### Creating a Reticle in Blueprint

1. Create a new Blueprint inheriting from `AGameplayAbilityWorldReticle`
2. Add visual components (a decal, a mesh, a niagara system, etc.)
3. Implement `OnParametersInitialized` to size your visuals based on `Parameters.AOEScale`
4. Implement `OnValidTargetChanged` to change color/opacity based on validity
5. Assign the reticle class to your target actor's `ReticleClass` property

### AGameplayAbilityWorldReticle_ActorVisualization

A specialized reticle for **actor placement** targeting. Instead of a generic decal or mesh, it visualizes the actual actor that will be placed:

```cpp
UCLASS(notplaceable)
class AGameplayAbilityWorldReticle_ActorVisualization
    : public AGameplayAbilityWorldReticle
{
    void InitializeReticleVisualizationInformation(
        AGameplayAbilityTargetActor* InTargetingActor,
        AActor* VisualizationActor,
        UMaterialInterface* VisualizationMaterial);

    UPROPERTY()
    TArray<TObjectPtr<UActorComponent>> VisualizationComponents;
};
```

It creates a visual copy of the actor being placed, optionally with a special material (e.g., a translucent holographic material). Used with `AGameplayAbilityTargetActor_ActorPlacement`.

## Target Data Filters

`FGameplayTargetDataFilter` controls which actors pass the targeting check. It is a simple struct with basic filtering capabilities, designed to be subclassed for complex filtering needs.

### FGameplayTargetDataFilter

```cpp
USTRUCT(BlueprintType)
struct FGameplayTargetDataFilter
{
    // Returns true if the actor passes the filter
    virtual bool FilterPassesForActor(const AActor* ActorToBeFiltered) const;

    UPROPERTY()
    AActor* SelfActor;              // "Self" for self-filtering

    UPROPERTY(EditAnywhere)
    TSubclassOf<AActor> RequiredActorClass;  // Must be this class (or subclass)

    UPROPERTY(EditAnywhere)
    ETargetDataFilterSelf SelfFilter;   // Self-targeting rule

    UPROPERTY(EditAnywhere)
    bool bReverseFilter;            // Invert the filter result
};
```

### Self-Targeting Rules

The `SelfFilter` enum controls whether the caster can target themselves:

| Value | Meaning |
|:---|:---|
| `TDFS_Any` | Allow self and others (no self-filtering) |
| `TDFS_NoSelf` | Exclude self from targets |
| `TDFS_NoOthers` | Only allow self as a target |

### Filter Evaluation

The default `FilterPassesForActor` logic:

1. Check `SelfFilter` — if the actor matches the self rule, apply it
2. Check `RequiredActorClass` — if set, the actor must be of this class
3. Apply `bReverseFilter` — if true, invert the result

### FGameplayTargetDataFilterHandle

The filter is wrapped in a `FGameplayTargetDataFilterHandle` for polymorphic usage:

```cpp
USTRUCT(BlueprintType)
struct FGameplayTargetDataFilterHandle
{
    TSharedPtr<FGameplayTargetDataFilter> Filter;

    bool FilterPassesForActor(const AActor* ActorToBeFiltered) const;
};
```

If no filter is set (the `TSharedPtr` is null), all actors pass. This is the default behavior.

### Creating Custom Filters

Subclass `FGameplayTargetDataFilter` for project-specific filtering logic:

```cpp
struct FMyFilter_TeamBased : public FGameplayTargetDataFilter
{
    ETeamAttitude::Type RequiredAttitude = ETeamAttitude::Hostile;

    virtual bool FilterPassesForActor(const AActor* ActorToBeFiltered) const override
    {
        if (!FGameplayTargetDataFilter::FilterPassesForActor(ActorToBeFiltered))
        {
            return false;
        }

        // Check team attitude
        if (SelfActor && ActorToBeFiltered)
        {
            // Your team comparison logic here
            ETeamAttitude::Type Attitude = GetAttitude(SelfActor, ActorToBeFiltered);
            return Attitude == RequiredAttitude;
        }

        return true;
    }
};
```

Set it up via the handle:

```cpp
FGameplayTargetDataFilterHandle FilterHandle;
auto* TeamFilter = new FMyFilter_TeamBased();
TeamFilter->RequiredAttitude = ETeamAttitude::Hostile;
TeamFilter->InitializeFilterContext(OwnerActor);
FilterHandle.Filter = MakeShareable(TeamFilter);

// Pass to your target actor
TargetActor->Filter = FilterHandle;
```

## Putting It Together

Here is how reticles and filters connect to the [target actor](target-actors.md) system:

1. When you configure a target actor (either in Blueprint or via code), you set:
    - `ReticleClass` — the reticle Blueprint to spawn
    - `ReticleParams` — size/scale parameters
    - `Filter` — the target data filter handle

2. When `StartTargeting` is called, the target actor spawns the reticle and begins updating it each tick with position and validity info.

3. Each tick (for trace-based actors), the trace result is run through the filter. If it passes, the reticle shows "valid" feedback. If not, "invalid."

4. On confirm, the final target data has already been filtered — only valid targets are included.

The filter applies at two levels: during the visual feedback phase (showing valid/invalid on the reticle) and during the final data production (only including targets that pass).
