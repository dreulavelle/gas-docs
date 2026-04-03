---
title: "Recipe: Add an Attribute"
description: Step-by-step procedure for adding a new gameplay attribute with replication, clamping, and OnRep.
---

# Recipe: Add an Attribute

**Goal:** Add a new replicated attribute (e.g., `Stamina`) to your Attribute Set.

**Prerequisites:** An existing `UAttributeSet` subclass. See [Attributes and Attribute Sets](../core-concepts/attributes-and-attribute-sets.md) for the concepts.

---

## Steps

### 1. Declare the Attribute in the Header

```cpp
// MyAttributeSet.h

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Stamina, Category = "Attributes")
FGameplayAttributeData Stamina;
ATTRIBUTE_ACCESSORS(UMyAttributeSet, Stamina)
```

`ATTRIBUTE_ACCESSORS` is a macro that generates `GetStamina()`, `SetStamina()`, `GetStaminaAttribute()`, and `InitStamina()` helper functions.

### 2. Initialize in the Constructor

```cpp
// MyAttributeSet.cpp

UMyAttributeSet::UMyAttributeSet()
{
    InitStamina(100.f);
}
```

### 3. Register for Replication

```cpp
void UMyAttributeSet::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Stamina,
        COND_None, REPNOTIFY_Always);
}
```

`REPNOTIFY_Always` ensures the OnRep fires even if the value didn't change (needed for prediction reconciliation).

### 4. Add the OnRep Function

Header:
```cpp
UFUNCTION()
void OnRep_Stamina(const FGameplayAttributeData& OldStamina);
```

Implementation:
```cpp
void UMyAttributeSet::OnRep_Stamina(const FGameplayAttributeData& OldStamina)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Stamina, OldStamina);
}
```

`GAMEPLAYATTRIBUTE_REPNOTIFY` tells the ASC about the attribute change for internal bookkeeping.

### 5. (Optional) Add Clamping

Override `PreAttributeChange` for clamping the *current* value:

```cpp
void UMyAttributeSet::PreAttributeChange(
    const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    if (Attribute == GetStaminaAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.f, GetMaxStamina());
    }
}
```

And/or clamp in `PostGameplayEffectExecute` for base value clamping after instant effects:

```cpp
void UMyAttributeSet::PostGameplayEffectExecute(
    const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    if (Data.EvaluatedData.Attribute == GetStaminaAttribute())
    {
        SetStamina(FMath::Clamp(GetStamina(), 0.f, GetMaxStamina()));
    }
}
```

### 6. Compile and Test

1. Compile your project
2. In PIE, use `showdebug abilitysystem` to verify the attribute appears
3. Apply an effect that modifies Stamina and confirm the value changes

---

## Checklist

- [ ] `UPROPERTY` with `ReplicatedUsing`
- [ ] `ATTRIBUTE_ACCESSORS` macro
- [ ] `Init{AttributeName}()` in constructor (e.g., `InitStamina(100.f)`)
- [ ] `DOREPLIFETIME_CONDITION_NOTIFY` in `GetLifetimeReplicatedProps`
- [ ] `OnRep_` function with `GAMEPLAYATTRIBUTE_REPNOTIFY`
- [ ] Clamping in `PreAttributeChange` and/or `PostGameplayEffectExecute`

## Related

- [Attributes and Attribute Sets](../core-concepts/attributes-and-attribute-sets.md) -- concepts
- [Modifier Formula](../reference/modifier-formula.md) -- how modifiers aggregate
