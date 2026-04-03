---
title: Ability Sets
description: Organizing, granting, and revoking groups of abilities — UGameplayAbilitySet, FGameplayAbilitySpec, and common grant patterns.
---

# Ability Sets

As your project grows, you will quickly find yourself needing to grant and revoke groups of abilities together. A character class might come with 8 abilities. A weapon might add 3 more. A talent tree might unlock abilities at specific levels. Managing these one-by-one gets tedious fast.

This page covers the mechanisms for organizing abilities into sets and the patterns for granting and revoking them.

## UGameplayAbilitySet — The Built-in Data Asset

The engine ships with `UGameplayAbilitySet`, a simple `UDataAsset` that pairs ability classes with input bindings:

```cpp
UCLASS()
class UGameplayAbilitySet : public UDataAsset
{
    UPROPERTY(EditAnywhere, Category = AbilitySet)
    TArray<FGameplayAbilityBindInfo> Abilities;

    void GiveAbilities(UAbilitySystemComponent* AbilitySystemComponent) const;
};
```

Each `FGameplayAbilityBindInfo` maps an `EGameplayAbilityInputBinds` enum to an ability class:

```cpp
USTRUCT()
struct FGameplayAbilityBindInfo
{
    UPROPERTY(EditAnywhere)
    TEnumAsByte<EGameplayAbilityInputBinds::Type> Command;

    UPROPERTY(EditAnywhere)
    TSubclassOf<UGameplayAbility> GameplayAbilityClass;
};
```

!!! note "An example, not a prescription"
    Epic explicitly calls this an "example" in the source comments. The built-in enum is limited to 9 slots (Ability1-9) and uses the legacy input system. Most projects create their own ability set implementation with richer features. That said, it is a fine starting point and shows the grant pattern clearly.

## The Core Mechanism: FGameplayAbilitySpec

Regardless of what data structure you use to organize abilities, the actual granting is always done through `FGameplayAbilitySpec`. This is the runtime representation of a granted ability on an ASC.

### Constructing a Spec

```cpp
// From a class — most common
FGameplayAbilitySpec Spec(UMyAbility::StaticClass(), AbilityLevel, InputID, SourceObject);

// From a CDO reference (backward compat)
FGameplayAbilitySpec Spec(AbilityCDO, AbilityLevel, InputID, SourceObject);
```

The constructor parameters:

| Parameter | Purpose | Default |
|:---|:---|:---|
| `InAbilityClass` | The ability class to grant | Required |
| `InLevel` | The ability level (affects scaled values) | `1` |
| `InInputID` | Input binding ID (or `INDEX_NONE` for no binding) | `INDEX_NONE` |
| `InSourceObject` | The object that granted this ability (e.g., weapon, talent node) | `nullptr` |

### Granting

```cpp
FGameplayAbilitySpecHandle Handle = AbilitySystemComponent->GiveAbility(Spec);
```

`GiveAbility` returns a `FGameplayAbilitySpecHandle` — save this if you need to refer to the ability later (for activation, removal, or level changes).

### Revoking

```cpp
AbilitySystemComponent->ClearAbility(Handle);
```

This removes the ability from the ASC. If the ability is currently active, it will be canceled first.

You can also clear by class:

```cpp
AbilitySystemComponent->ClearAllAbilitiesWithInputID(InputID);
```

### FGameplayAbilitySpecDef

When abilities are granted by Gameplay Effects (via the `GrantAbilities` GE Component), the grant configuration uses `FGameplayAbilitySpecDef`:

```cpp
USTRUCT()
struct FGameplayAbilitySpecDef
{
    TSubclassOf<UGameplayAbility> Ability;       // What ability to grant
    FScalableFloat LevelScalableFloat;            // Level (can be curve-driven)
    int32 InputID;                                // Input binding
    EGameplayEffectGrantedAbilityRemovePolicy RemovalPolicy;  // What happens on GE removal
    TWeakObjectPtr<UObject> SourceObject;         // Grant source
};
```

The `RemovalPolicy` controls what happens when the granting Gameplay Effect is removed:

| Policy | Behavior |
|:---|:---|
| `CancelAbilityImmediately` | Cancel any active activation, then remove the ability |
| `RemoveAbilityOnEnd` | Let any active activation finish, then remove the ability |
| `DoNothing` | Leave the granted ability in place even after the GE is removed |

## Common Grant Patterns

### Pattern 1: Class Defaults Array

The simplest pattern. Your character class (or a component) has an array of ability classes to grant on initialization:

```cpp
UPROPERTY(EditDefaultsOnly, Category = "Abilities")
TArray<TSubclassOf<UGameplayAbility>> DefaultAbilities;

void AMyCharacter::InitializeAbilities()
{
    if (!AbilitySystemComponent || bAbilitiesInitialized) return;

    for (const TSubclassOf<UGameplayAbility>& AbilityClass : DefaultAbilities)
    {
        if (AbilityClass)
        {
            AbilitySystemComponent->GiveAbility(
                FGameplayAbilitySpec(AbilityClass, 1, INDEX_NONE, this));
        }
    }

    bAbilitiesInitialized = true;
}
```

This is common for abilities that every character of a given class always has — basic attacks, movement abilities, etc.

### Pattern 2: Equipment-Based (Grant on Equip, Revoke on Unequip)

When a weapon or piece of equipment provides abilities, you grant them when the item is equipped and revoke them when it is unequipped. Store the handles so you can clean up:

```cpp
USTRUCT()
struct FGrantedAbilitySet
{
    GENERATED_BODY()
    TArray<FGameplayAbilitySpecHandle> AbilityHandles;
};
```

```cpp
FGrantedAbilitySet UEquipmentComponent::GrantAbilities(
    UAbilitySystemComponent* ASC,
    const UEquipmentDefinition* Equipment)
{
    FGrantedAbilitySet GrantedSet;

    for (const FEquipmentAbilityEntry& Entry : Equipment->GrantedAbilities)
    {
        FGameplayAbilitySpec Spec(Entry.AbilityClass, Entry.AbilityLevel,
                                  INDEX_NONE, Equipment);
        FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(Spec);
        GrantedSet.AbilityHandles.Add(Handle);
    }

    return GrantedSet;
}

void UEquipmentComponent::RevokeAbilities(
    UAbilitySystemComponent* ASC,
    FGrantedAbilitySet& GrantedSet)
{
    for (const FGameplayAbilitySpecHandle& Handle : GrantedSet.AbilityHandles)
    {
        ASC->ClearAbility(Handle);
    }
    GrantedSet.AbilityHandles.Reset();
}
```

The `SourceObject` parameter is useful here — set it to the equipment item so you can query "which abilities came from this weapon" later if needed.

### Pattern 3: Granted by Gameplay Effect

Gameplay Effects can grant abilities through the `UAbilitiesGameplayEffectComponent` (the GrantAbilities GE Component). This is powerful because the ability's lifetime is tied to the effect's lifetime.

For example, a "Rage" buff GE that grants the `GA_RageStrike` ability while active. When Rage expires, the ability is automatically revoked based on the `RemovalPolicy`.

See [GE Components](../gameplay-effects/ge-components.md) for how to configure this.

### Pattern 4: Progression-Based (Grant at Level Thresholds)

For RPG-style level-up systems, grant abilities when the character reaches specific levels:

```cpp
USTRUCT(BlueprintType)
struct FAbilityUnlock
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly)
    int32 RequiredLevel;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UGameplayAbility> AbilityClass;
};

void AMyCharacter::OnLevelChanged(int32 NewLevel)
{
    for (const FAbilityUnlock& Unlock : AbilityUnlocks)
    {
        if (NewLevel >= Unlock.RequiredLevel
            && !GrantedUnlockHandles.Contains(Unlock.AbilityClass))
        {
            FGameplayAbilitySpec Spec(Unlock.AbilityClass, NewLevel);
            FGameplayAbilitySpecHandle Handle =
                AbilitySystemComponent->GiveAbility(Spec);
            GrantedUnlockHandles.Add(Unlock.AbilityClass, Handle);
        }
    }
}
```

## Ability Levels

When granting an ability, you specify a level. This level is accessible inside the ability via `GetAbilityLevel()` and is used by:

- Scalable floats in cost and cooldown GEs (level-based curve lookup)
- `MakeOutgoingGameplayEffectSpec` (the level is passed to effect specs)
- Any custom logic in your ability that branches on level

To change an ability's level after granting:

```cpp
FGameplayAbilitySpec* Spec = AbilitySystemComponent->FindAbilitySpecFromHandle(Handle);
if (Spec)
{
    Spec->Level = NewLevel;
    AbilitySystemComponent->MarkAbilitySpecDirty(*Spec);
}
```

## The Spec Container

Internally, the ASC stores all granted abilities in an `FGameplayAbilitySpecContainer`:

```cpp
UPROPERTY()
TArray<FGameplayAbilitySpec> Items;
```

This is a Fast Array Serializer for efficient replication. The specs replicate to clients so they know which abilities are available. The `ShouldReplicateAbilitySpec()` check on each spec (delegated to the ability's `ShouldReplicateAbilitySpec`) controls whether a given spec replicates.

## Building Your Own Ability Set

For most projects, you will want a richer ability set than the built-in one. Here is a starting point:

```cpp
USTRUCT(BlueprintType)
struct FMyAbilitySetEntry
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UGameplayAbility> AbilityClass;

    UPROPERTY(EditDefaultsOnly)
    int32 AbilityLevel = 1;

    UPROPERTY(EditDefaultsOnly, meta = (Categories = "Input"))
    FGameplayTag InputTag;
};

UCLASS(BlueprintType)
class UMyAbilitySet : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Abilities")
    TArray<FMyAbilitySetEntry> Abilities;

    // Also support granting effects and attribute sets
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    TArray<TSubclassOf<UGameplayEffect>> GrantedEffects;

    UPROPERTY(EditDefaultsOnly, Category = "Attributes")
    TArray<TSubclassOf<UAttributeSet>> GrantedAttributes;
};
```

This approach (used by Lyra and other Epic samples) groups abilities with their input tags and can also grant effects and attribute sets as a cohesive "character kit" or "equipment loadout."
