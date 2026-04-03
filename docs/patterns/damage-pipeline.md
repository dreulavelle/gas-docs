---
title: Damage Pipeline
description: End-to-end damage flow in GAS — from GE spec creation through ExecCalc to PostGameplayEffectExecute, with a full code example.
---

# Damage Pipeline

The damage pipeline is arguably the most important pattern in any GAS-based action game. This page walks through the entire flow from "character swings a sword" to "target takes damage, might die, damage number pops up."

## The Full Flow

```
1. Source Ability creates a GE Spec for the damage effect
     │
2. GE Spec is applied to target's ASC
     │
3. Execution Calculation runs:
     │  - Captures source attack, crit chance, weapon damage
     │  - Captures target armor, resistances
     │  - Calculates final damage after all modifiers
     │  - Outputs to a "meta attribute" (e.g., IncomingDamage)
     │
4. Modifier applies IncomingDamage to target's AttributeSet
     │
5. PostGameplayEffectExecute fires on the AttributeSet:
     │  - Reads IncomingDamage
     │  - Subtracts from Health
     │  - Clamps Health to 0
     │  - Resets IncomingDamage to 0
     │  - Checks for death
     │
6. GameplayCue fires for damage numbers / hit feedback
```

## Why Meta Attributes?

A **meta attribute** is a transient attribute that acts as a pipeline staging area. It's never replicated, never displayed to players, and always reset to 0 after use. The classic example is `IncomingDamage`.

Why not modify `Health` directly in the ExecCalc? Because `PostGameplayEffectExecute` is the right place to do final processing:

- Clamping Health to min/max
- Checking for death
- Triggering shields or damage absorption
- Broadcasting damage events to UI

If the ExecCalc directly modified Health, you'd lose this processing hook. Meta attributes give you a clean separation: the ExecCalc figures out the number, `PostGameplayEffectExecute` applies it to the real attribute.

## Step-by-Step Implementation

### 1. The Attribute Set

```cpp
UCLASS()
class UMyAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    // --- Combat Attributes ---
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealth)
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Armor)
    FGameplayAttributeData Armor;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Armor)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_AttackPower)
    FGameplayAttributeData AttackPower;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, AttackPower)

    // --- Meta Attributes (not replicated) ---
    UPROPERTY(BlueprintReadOnly)
    FGameplayAttributeData IncomingDamage;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, IncomingDamage)

    // OnRep functions
    UFUNCTION() void OnRep_Health(const FGameplayAttributeData& OldValue);
    UFUNCTION() void OnRep_MaxHealth(const FGameplayAttributeData& OldValue);
    UFUNCTION() void OnRep_Armor(const FGameplayAttributeData& OldValue);
    UFUNCTION() void OnRep_AttackPower(const FGameplayAttributeData& OldValue);

    virtual void PostGameplayEffectExecute(
        const FGameplayEffectModCallbackData& Data) override;
};
```

### 2. The Damage Gameplay Effect

Create `GE_Damage` (Blueprint or C++):

- **Duration Policy:** Instant
- **Executions:** Add your `UExecCalc_Damage` class
- **GameplayCue tag:** `GameplayCue.Damage` (via the GameplayCue GE Component)

The effect itself is simple -- the complexity lives in the ExecCalc.

### 3. The Execution Calculation

```cpp
UCLASS()
class UExecCalc_Damage : public UGameplayEffectExecutionCalculation
{
    GENERATED_BODY()

public:
    UExecCalc_Damage();

    virtual void Execute_Implementation(
        const FGameplayEffectCustomExecutionParameters& ExecutionParams,
        FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;
};
```

**Constructor -- declare attribute captures:**

```cpp
UExecCalc_Damage::UExecCalc_Damage()
{
    // Source captures
    FGameplayEffectAttributeCaptureDefinition AttackPowerDef(
        UMyAttributeSet::GetAttackPowerAttribute(),
        EGameplayEffectAttributeCaptureSource::Source,
        true); // Snapshot at application time
    RelevantAttributesToCapture.Add(AttackPowerDef);

    // Target captures
    FGameplayEffectAttributeCaptureDefinition ArmorDef(
        UMyAttributeSet::GetArmorAttribute(),
        EGameplayEffectAttributeCaptureSource::Target,
        false); // Don't snapshot, use current value
    RelevantAttributesToCapture.Add(ArmorDef);
}
```

**Execution -- the damage formula:**

```cpp
void UExecCalc_Damage::Execute_Implementation(
    const FGameplayEffectCustomExecutionParameters& ExecutionParams,
    FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

    // Gather tags for conditional logic
    const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    FAggregatorEvaluateParameters EvalParams;
    EvalParams.SourceTags = SourceTags;
    EvalParams.TargetTags = TargetTags;

    // --- Capture attributes ---
    float AttackPower = 0.f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        AttackPowerDef, EvalParams, AttackPower);

    float Armor = 0.f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        ArmorDef, EvalParams, Armor);

    // --- Get base damage from SetByCaller ---
    float BaseDamage = Spec.GetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag(TEXT("SetByCaller.Damage")),
        false, 0.f);

    // --- Apply damage formula ---
    // Simple example: (BaseDamage + AttackPower) * (100 / (100 + Armor))
    float DamageReduction = 100.f / (100.f + FMath::Max(Armor, 0.f));
    float FinalDamage = (BaseDamage + AttackPower) * DamageReduction;

    // --- Damage type multipliers via tags ---
    if (SourceTags->HasTag(DamageType_Fire) && TargetTags->HasTag(Resistance_Fire))
    {
        FinalDamage *= 0.5f; // 50% fire resistance
    }

    // --- Critical hit ---
    // (Typically rolled here or passed via custom context)

    // Clamp to non-negative
    FinalDamage = FMath::Max(FinalDamage, 0.f);

    // --- Output to meta attribute ---
    OutExecutionOutput.AddOutputModifier(
        FGameplayModifierEvaluatedData(
            UMyAttributeSet::GetIncomingDamageAttribute(),
            EGameplayModOp::Additive,
            FinalDamage));
}
```

### 4. PostGameplayEffectExecute

```cpp
void UMyAttributeSet::PostGameplayEffectExecute(
    const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    if (Data.EvaluatedData.Attribute == GetIncomingDamageAttribute())
    {
        const float DamageDone = GetIncomingDamage();
        SetIncomingDamage(0.f); // Always reset meta attribute

        if (DamageDone > 0.f)
        {
            const float NewHealth = GetHealth() - DamageDone;
            SetHealth(FMath::Clamp(NewHealth, 0.f, GetMaxHealth()));

            if (GetHealth() <= 0.f)
            {
                // Broadcast death event
                // Your game handles death (respawn, ragdoll, etc.)
            }
        }
    }
}
```

### 5. Applying Damage from an Ability

```cpp
void UGA_MeleeAttack::ApplyDamageToTarget(AActor* Target)
{
    FGameplayEffectSpecHandle SpecHandle =
        MakeOutgoingGameplayEffectSpec(DamageEffectClass, GetAbilityLevel());

    // Pass base damage via SetByCaller
    SpecHandle.Data->SetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag(TEXT("SetByCaller.Damage")),
        WeaponBaseDamage);

    // Apply to target
    ApplyGameplayEffectSpecToTarget(
        CurrentActivationInfo, SpecHandle,
        UAbilitySystemBlueprintLibrary::AbilityTargetDataFromActor(Target));
}
```

### 6. Damage Number Cue

The `GameplayCue.Damage` tag on the GE triggers a [Burst cue](../gameplay-cues/cue-types.md#ugameplaycuenotify_burst) that:

- Reads `Parameters.RawMagnitude` for the damage number
- Reads the [custom context](../gameplay-cues/cue-parameters.md#passing-custom-data-via-effect-context) for crit flag, damage type
- Spawns a floating damage number widget at `Parameters.Location`

## Design Considerations

**SetByCaller vs hardcoded:** Use [SetByCaller](../gameplay-effects/set-by-caller.md) for base damage so different abilities can reuse the same GE with different damage values.

**One GE or many?** One `GE_Damage` with the ExecCalc is usually sufficient. The ExecCalc handles all damage types via tags. You don't need `GE_FireDamage`, `GE_PhysicalDamage`, etc.

**Armor formula:** The `100 / (100 + Armor)` formula is a soft cap -- diminishing returns. Other options include flat reduction (`Damage - Armor`), percentage reduction, or custom curves. Pick what fits your game feel.

**Multiple damage outputs:** An ExecCalc can output to multiple attributes. You could output `IncomingPhysicalDamage` and `IncomingMagicalDamage` separately if your game needs that distinction.

## Related Pages

- [Execution Calculations](../gameplay-effects/execution-calculations.md) -- deep dive on ExecCalcs
- [SetByCaller](../gameplay-effects/set-by-caller.md) -- passing values into effects
- [Cue Parameters](../gameplay-cues/cue-parameters.md) -- custom data in cues
- [Modifier Formula](../reference/modifier-formula.md) -- the math behind modifier aggregation
- [Attributes and Attribute Sets](../core-concepts/attributes-and-attribute-sets.md) -- attribute basics
