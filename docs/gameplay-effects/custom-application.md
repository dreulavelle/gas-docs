---
title: Custom Application Requirements
icon: material/shield-check
description: Writing native C++ logic to control when a Gameplay Effect can be applied, beyond tag-based checks.
---

# Custom Application Requirements

GAS has a rich tag-based system for controlling when effects can apply — required tags, blocked tags, immunity queries. But sometimes tags aren't enough. What if an effect should only apply when the target's health is below 50%? Or when the target is facing the source? Or when a gameplay-specific condition is met that doesn't map cleanly to tag presence?

That's where **custom application requirements** come in. They let you write arbitrary C++ (or Blueprint) logic that GAS consults before applying an effect. Return `true` and the effect applies normally. Return `false` and it's rejected — just like a failed tag check.

## UGameplayEffectCustomApplicationRequirement

The base class is straightforward:

```cpp
UCLASS(BlueprintType, Blueprintable, Abstract, MinimalAPI)
class UGameplayEffectCustomApplicationRequirement : public UObject
{
    GENERATED_BODY()

public:
    /** Return whether the gameplay effect should be applied or not */
    UFUNCTION(BlueprintNativeEvent, Category="Calculation")
    bool CanApplyGameplayEffect(const UGameplayEffect* GameplayEffect,
                                 const FGameplayEffectSpec& Spec,
                                 UAbilitySystemComponent* ASC) const;
};
```

Key things:

- **BlueprintNativeEvent** — you can override `CanApplyGameplayEffect` in either C++ or Blueprint
- **Blueprintable** — you can create Blueprint subclasses in the editor
- **Const method** — the requirement object is shared across all instances of the GE, so you must not mutate state
- **Parameters give you everything** — the GE definition, the full spec (with context, source info, magnitudes), and the target ASC

The `Spec` parameter is particularly powerful. Through it you can access:

- `Spec.GetContext()` — the effect context (instigator, causer, hit result, origin)
- `Spec.GetLevel()` — the effect level
- `Spec.CapturedSourceTags` — tags from the source at spec creation time
- `Spec.GetSetByCallerMagnitude(Tag)` — any SetByCaller values

## UCustomCanApplyGameplayEffectComponent

Since UE 5.3, custom application requirements are wired to a GE through the `UCustomCanApplyGameplayEffectComponent` — a GE component that holds an array of requirement classes.

```cpp
UCLASS(DisplayName="Custom Can Apply This Effect", MinimalAPI)
class UCustomCanApplyGameplayEffectComponent : public UGameplayEffectComponent
{
    GENERATED_BODY()

public:
    /** Custom application requirements */
    UPROPERTY(EditDefaultsOnly, Category = Application,
              meta = (DisplayName = "Custom Application Requirement"))
    TArray<TSubclassOf<UGameplayEffectCustomApplicationRequirement>> ApplicationRequirements;

    /** Determine if we can apply this GameplayEffect or not */
    virtual bool CanGameplayEffectApply(const FActiveGameplayEffectsContainer& ActiveGEContainer,
                                         const FGameplayEffectSpec& GESpec) const override;
};
```

When GAS is about to apply an effect, the component iterates through `ApplicationRequirements`, instantiates the CDO for each class, and calls `CanApplyGameplayEffect`. If *any* requirement returns `false`, the effect is blocked.

## Setting It Up

### Step 1: Write the Requirement Class

```cpp
// UHealthThresholdRequirement.h
#pragma once

#include "GameplayEffectCustomApplicationRequirement.h"
#include "HealthThresholdRequirement.generated.h"

/**
 * Only allows the effect to apply if the target's health is below
 * a configurable percentage of max health.
 */
UCLASS(DisplayName = "Health Below Threshold")
class MYGAME_API UHealthThresholdRequirement : public UGameplayEffectCustomApplicationRequirement
{
    GENERATED_BODY()

public:
    /** The health percentage threshold (0.0 - 1.0). Effect only applies below this. */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Requirement",
              meta = (ClampMin = "0.0", ClampMax = "1.0"))
    float HealthThreshold = 0.5f;

    virtual bool CanApplyGameplayEffect_Implementation(
        const UGameplayEffect* GameplayEffect,
        const FGameplayEffectSpec& Spec,
        UAbilitySystemComponent* ASC) const override;
};
```

```cpp
// UHealthThresholdRequirement.cpp
#include "HealthThresholdRequirement.h"
#include "AbilitySystemComponent.h"
#include "MyAttributeSet.h" // Your attribute set with Health and MaxHealth

bool UHealthThresholdRequirement::CanApplyGameplayEffect_Implementation(
    const UGameplayEffect* GameplayEffect,
    const FGameplayEffectSpec& Spec,
    UAbilitySystemComponent* ASC) const
{
    if (!ASC)
    {
        return false;
    }

    // Read current and max health from the target's attributes
    bool bFound = false;
    float CurrentHealth = ASC->GetNumericAttribute(UMyAttributeSet::GetHealthAttribute());
    float MaxHealth = ASC->GetNumericAttribute(UMyAttributeSet::GetMaxHealthAttribute());

    if (MaxHealth <= 0.f)
    {
        return false;
    }

    float HealthPercent = CurrentHealth / MaxHealth;
    return HealthPercent < HealthThreshold;
}
```

### Step 2: Add the Component to Your GE

1. Open your Gameplay Effect Blueprint
2. In the GE Components section, click **+** and select **Custom Can Apply This Effect**
3. In the component's details, add your requirement class (e.g., `HealthThresholdRequirement`) to the **Application Requirements** array
4. If the requirement has editable properties (like `HealthThreshold`), note that those are set on the CDO — all GEs using this class share the same threshold value

!!! tip "Multiple Requirements"
    You can add multiple requirement classes to the same component. They're evaluated in order, and the effect is blocked if *any* returns `false`. This lets you compose requirements: "health below 50% AND target is facing source" without writing a single monolithic class.

### Step 3: Done

That's it. GAS will now call your requirement class before applying this effect. If `CanApplyGameplayEffect` returns `false`, the effect is rejected silently — it never enters the Active GE Container.

## Blueprint Implementation

Since `CanApplyGameplayEffect` is a `BlueprintNativeEvent`, you can also implement requirements entirely in Blueprint:

1. Create a new Blueprint class derived from `UGameplayEffectCustomApplicationRequirement`
2. Override `CanApplyGameplayEffect`
3. Implement your logic using the provided GE, Spec, and ASC parameters
4. Reference the Blueprint class in the GE component

This is perfectly viable for prototype and game-jam work. For shipping games with many effects checking requirements frequently, C++ is faster — but the overhead is typically negligible unless you have hundreds of effects applying per frame.

## The Old Way (Pre-5.3)

Before GE components existed, custom application requirements were configured directly on the `UGameplayEffect` class:

```cpp
// Pre-5.3 (deprecated property on UGameplayEffect)
UPROPERTY(EditDefaultsOnly, Category = Application)
TArray<TSubclassOf<UGameplayEffectCustomApplicationRequirement>> CustomApplicationRequirements;
```

In 5.3+, this property was replaced by `UCustomCanApplyGameplayEffectComponent`. Existing assets should migrate automatically, but if you're upgrading a project and your custom requirements stop being checked, verify that the component was created correctly on your GEs.

## When to Use Custom Requirements vs Tags

| Use Case | Solution |
|:---|:---|
| Effect requires target to have a specific state tag | Tag requirements (no code needed) |
| Effect requires target to NOT have a specific tag | Blocked tags or immunity |
| Effect requires target attribute below a threshold | Custom application requirement |
| Effect requires spatial relationship (facing, distance) | Custom application requirement |
| Effect requires game-specific logic (inventory check, quest state) | Custom application requirement |
| Effect should be blocked by an entire category of other effects | Immunity via `UImmunityGameplayEffectComponent` |

The general rule: **if a tag can represent the condition, use tags**. Tags are faster, data-driven, and visible in the debugger. Custom requirements are for conditions that fundamentally can't be expressed as tag presence/absence.

## Practical Examples

### "Execute Only" Requirement

An effect that can only apply to targets that are in an "execute" state (below 20% HP and not shielded):

```cpp
bool UExecuteOnlyRequirement::CanApplyGameplayEffect_Implementation(
    const UGameplayEffect* GameplayEffect,
    const FGameplayEffectSpec& Spec,
    UAbilitySystemComponent* ASC) const
{
    if (!ASC) return false;

    // Check health threshold
    float Health = ASC->GetNumericAttribute(UMyAttributeSet::GetHealthAttribute());
    float MaxHealth = ASC->GetNumericAttribute(UMyAttributeSet::GetMaxHealthAttribute());
    if (MaxHealth <= 0.f || (Health / MaxHealth) >= 0.2f)
    {
        return false;
    }

    // Also check that target has no shield tag (belt and suspenders with tag check)
    FGameplayTagContainer OwnedTags;
    ASC->GetOwnedGameplayTags(OwnedTags);
    return !OwnedTags.HasTag(FGameplayTag::RequestGameplayTag(FName("State.Shielded")));
}
```

### "Facing Source" Requirement

An effect that only applies if the target is facing the source (e.g., a taunt that only works on enemies looking at you):

```cpp
bool UFacingSourceRequirement::CanApplyGameplayEffect_Implementation(
    const UGameplayEffect* GameplayEffect,
    const FGameplayEffectSpec& Spec,
    UAbilitySystemComponent* ASC) const
{
    if (!ASC) return false;

    AActor* TargetActor = ASC->GetOwnerActor();
    const FGameplayEffectContextHandle& Context = Spec.GetEffectContext();
    AActor* SourceActor = Context.GetInstigator();

    if (!TargetActor || !SourceActor) return false;

    FVector ToSource = (SourceActor->GetActorLocation() - TargetActor->GetActorLocation()).GetSafeNormal();
    FVector TargetForward = TargetActor->GetActorForwardVector();

    // Dot product > 0 means target is roughly facing source (within 180 degrees)
    // Use a higher threshold for tighter cone
    float Dot = FVector::DotProduct(TargetForward, ToSource);
    return Dot > FacingThreshold; // e.g., 0.5 for ~60-degree half-cone
}
```

## Further Reading

- [GE Components](ge-components.md) — the component architecture that custom requirements plug into
- [Tags and Requirements](tags-and-requirements.md) — the tag-based application checks that run alongside custom requirements
- [Immunity](immunity.md) — another mechanism for blocking effect application
- [GE Component Catalog](../reference/ge-component-catalog.md) — all built-in GE components including `UCustomCanApplyGameplayEffectComponent`
