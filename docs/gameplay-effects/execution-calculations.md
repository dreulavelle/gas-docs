---
title: Execution Calculations
description: Using UGameplayEffectExecutionCalculation for complex multi-attribute formulas — attribute capture, custom parameters, output modifiers, and a full damage example.
---

# Execution Calculations

When a single modifier on a single attribute isn't enough, Execution Calculations let you write arbitrary C++ that reads multiple attributes, performs complex logic, and outputs multiple attribute modifications — all within a single GE execution.

## When to Use Execution Calculations

Execution Calculations (`UGameplayEffectExecutionCalculation`, often called "ExecCalcs") are the right tool when:

- Your formula reads from **multiple attributes** on the source and/or target
- You need to **modify multiple attributes** in one pass (e.g., damage reduces Health _and_ applies Stagger)
- The logic is too complex for a single modifier's magnitude calculation
- You need **conditional output** — different attribute changes based on runtime conditions

The classic use case is a **damage formula** where the final damage depends on the attacker's Attack, the defender's Armor, elemental resistances, critical hit chance, and so on.

!!! info "ExecCalc vs. Custom Magnitude Calculation"
    A `UGameplayModMagnitudeCalculation` produces one number for one modifier. An `UGameplayEffectExecutionCalculation` can read many attributes and output many modifications. If you only need one number, use [CustomCalculationClass](magnitude-calculations.md#customcalculationclass). If you need a multi-input, multi-output formula, use an ExecCalc.

## Attribute Capture

Before an ExecCalc can read attributes, those attributes must be **captured**. Capture means the engine grabs the attribute's value (or a reference to its aggregator) and makes it available inside the calculation.

### Capture Definitions

You declare what to capture in the constructor using the `DECLARE_ATTRIBUTE_CAPTUREDEF` and `DEFINE_ATTRIBUTE_CAPTUREDEF` helper macros:

```cpp
// In the header or a shared static struct
struct FDamageStatics
{
    DECLARE_ATTRIBUTE_CAPTUREDEF(AttackPower);
    DECLARE_ATTRIBUTE_CAPTUREDEF(Armor);
    DECLARE_ATTRIBUTE_CAPTUREDEF(Health);

    FDamageStatics()
    {
        // Capture AttackPower from the Source, snapshotted at spec creation
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, AttackPower, Source, true);

        // Capture Armor from the Target, live (not snapshotted)
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, Armor, Target, false);

        // Capture Health from the Target, live
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, Health, Target, false);
    }
};

static const FDamageStatics& DamageStatics()
{
    static FDamageStatics Statics;
    return Statics;
}
```

### Source vs. Target

- **Source** — The actor that applied the effect (the attacker, the healer, etc.)
- **Target** — The actor receiving the effect (the one being damaged, healed, etc.)

### Snapshot vs. Live

- **Snapshot (`true`)** — The attribute value is captured once when the `FGameplayEffectSpec` is created and frozen. Even if the source's attribute changes between spec creation and application, the captured value stays the same.
- **Live (`false`)** — The attribute value is read at the time the calculation actually executes. This means it reflects the most current state.

**When to snapshot:** Source attributes (like AttackPower) are commonly snapshotted because the attacker's stats at the moment they "decide" to attack is what should matter.

**When to use live:** Target attributes (like Armor) are commonly live because the target's state at the moment of impact is what should matter.

!!! tip "Snapshot Default"
    There's no universal rule. Pick based on your game's design. Snapshotting source and reading target live is common, but some games snapshot everything for determinism.

### Registering Captures

In your ExecCalc constructor, register the capture definitions with the engine:

```cpp
UMyDamageExecCalc::UMyDamageExecCalc()
{
    RelevantAttributesToCapture.Add(DamageStatics().AttackPowerDef);
    RelevantAttributesToCapture.Add(DamageStatics().ArmorDef);
    RelevantAttributesToCapture.Add(DamageStatics().HealthDef);
}
```

## The Execute Function

Override `Execute_Implementation` to write your calculation:

```cpp
void UMyDamageExecCalc::Execute_Implementation(
    const FGameplayEffectCustomExecutionParameters& ExecutionParams,
    FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    // ... your logic here ...
}
```

### FGameplayEffectCustomExecutionParameters

The input parameter gives you access to:

| Method | Returns |
|:---|:---|
| `GetOwningSpec()` | The `FGameplayEffectSpec` that triggered this execution |
| `GetTargetAbilitySystemComponent()` | The target ASC |
| `GetSourceAbilitySystemComponent()` | The source ASC (can be null) |
| `GetPassedInTags()` | Tags passed in via the execution definition |
| `AttemptCalculateCapturedAttributeMagnitude(...)` | Evaluate a captured attribute |
| `AttemptCalculateCapturedAttributeBaseValue(...)` | Get the base value of a captured attribute |
| `AttemptCalculateCapturedAttributeBonusMagnitude(...)` | Get the bonus (modifiers only) of a captured attribute |
| `AttemptCalculateTransientAggregatorMagnitude(...)` | Evaluate a transient "temporary variable" aggregator |

### FGameplayEffectCustomExecutionOutput

The output struct is where you add the attribute modifications your calculation produces:

```cpp
// Add a modifier to reduce Health by computed damage
OutExecutionOutput.AddOutputModifier(
    FGameplayModifierEvaluatedData(
        DamageStatics().HealthProperty,
        EGameplayModOp::Additive,
        -FinalDamage
    )
);
```

You can add as many output modifiers as needed. Each one specifies which attribute to modify, which operation to use, and the magnitude.

Additional output control:

| Method | What It Does |
|:---|:---|
| `MarkStackCountHandledManually()` | Tell the engine you've already accounted for stack count in your calculation |
| `MarkConditionalGameplayEffectsToTrigger()` | Allow conditional effects on the execution definition to fire |
| `MarkGameplayCuesHandledManually()` | Tell the engine you've already triggered gameplay cues yourself |

## Scoped Modifiers (Calculation Modifiers)

The `FGameplayEffectExecutionDefinition` that references your ExecCalc class can also specify **CalculationModifiers** — modifiers that are folded into attribute evaluation _only for the scope of this execution_. They don't permanently modify the attribute.

This lets designers tweak captured attribute values in the GE asset without changing your C++ code. For example, a GE could specify a scoped +20 bonus to AttackPower that only applies within the damage calculation.

Scoped modifiers can target either captured attributes or **transient aggregators** — temporary variables identified by gameplay tags that exist only during the execution.

```cpp
// Transient aggregator example: reading a "temporary variable"
float BonusDamage = 0.f;
ExecutionParams.AttemptCalculateTransientAggregatorMagnitude(
    FGameplayTag::RequestGameplayTag("Calc.BonusDamage"),
    EvalParams,
    BonusDamage);
```

## Passing Data into Execution Calculations

There are several ways to get runtime data into your ExecCalc:

### 1. SetByCaller

The most common approach. Set values on the spec before application, read them inside the ExecCalc:

```cpp
// Setting (before apply)
SpecHandle.Data->SetSetByCallerMagnitude(DamageTag, 100.f);

// Reading (inside Execute_Implementation)
float BaseDamage = ExecutionParams.GetOwningSpec().GetSetByCallerMagnitude(DamageTag);
```

### 2. Scoped Modifiers (Backing Data Attribute)

GE designers can add CalculationModifiers that target captured attributes. These are automatically folded into `AttemptCalculateCapturedAttributeMagnitude`.

### 3. Scoped Modifiers (Backing Data Temp Variable)

Same as above but targeting transient aggregators identified by tags. Read via `AttemptCalculateTransientAggregatorMagnitude`.

### 4. Effect Context

The `FGameplayEffectContext` (accessible via `GetOwningSpec().GetEffectContext()`) carries the instigator, effect causer, hit result, origin, and any custom data you've added by subclassing it.

```cpp
const FGameplayEffectContextHandle& ContextHandle = ExecutionParams.GetOwningSpec().GetEffectContext();
const FHitResult* HitResult = ContextHandle.GetHitResult();
if (HitResult)
{
    // Use hit location for headshot detection, etc.
}
```

## Complete Damage Calculation Example

Here's a full ExecCalc implementation showing a realistic damage formula:

```cpp
// DamageExecCalc.h
#pragma once

#include "GameplayEffectExecutionCalculation.h"
#include "DamageExecCalc.generated.h"

UCLASS()
class MYGAME_API UDamageExecCalc : public UGameplayEffectExecutionCalculation
{
    GENERATED_BODY()

public:
    UDamageExecCalc();

    virtual void Execute_Implementation(
        const FGameplayEffectCustomExecutionParameters& ExecutionParams,
        FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;
};
```

```cpp
// DamageExecCalc.cpp
#include "DamageExecCalc.h"
#include "MyAttributeSet.h"

struct FDamageStatics
{
    DECLARE_ATTRIBUTE_CAPTUREDEF(AttackPower);
    DECLARE_ATTRIBUTE_CAPTUREDEF(CritChance);
    DECLARE_ATTRIBUTE_CAPTUREDEF(Armor);
    DECLARE_ATTRIBUTE_CAPTUREDEF(Health);

    FDamageStatics()
    {
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, AttackPower, Source, true);
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, CritChance, Source, true);
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, Armor, Target, false);
        DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, Health, Target, false);
    }
};

static const FDamageStatics& DamageStatics()
{
    static FDamageStatics Statics;
    return Statics;
}

UDamageExecCalc::UDamageExecCalc()
{
    RelevantAttributesToCapture.Add(DamageStatics().AttackPowerDef);
    RelevantAttributesToCapture.Add(DamageStatics().CritChanceDef);
    RelevantAttributesToCapture.Add(DamageStatics().ArmorDef);
    RelevantAttributesToCapture.Add(DamageStatics().HealthDef);
}

void UDamageExecCalc::Execute_Implementation(
    const FGameplayEffectCustomExecutionParameters& ExecutionParams,
    FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

    // Gather tags for evaluation
    const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    FAggregatorEvaluateParameters EvalParams;
    EvalParams.SourceTags = SourceTags;
    EvalParams.TargetTags = TargetTags;

    // Read captured attributes
    float AttackPower = 0.f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().AttackPowerDef, EvalParams, AttackPower);

    float CritChance = 0.f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().CritChanceDef, EvalParams, CritChance);

    float Armor = 0.f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().ArmorDef, EvalParams, Armor);

    // Read SetByCaller base damage
    static FGameplayTag DamageTag = FGameplayTag::RequestGameplayTag("SetByCaller.Damage");
    float BaseDamage = Spec.GetSetByCallerMagnitude(DamageTag, false, 0.f);

    // --- Damage Formula ---
    // Raw damage = base damage + attack power contribution
    float RawDamage = BaseDamage + (AttackPower * 0.5f);

    // Critical hit check
    float CritMultiplier = 1.0f;
    if (FMath::FRand() < FMath::Clamp(CritChance / 100.f, 0.f, 1.f))
    {
        CritMultiplier = 2.0f;
    }

    // Armor mitigation: damage reduced by Armor / (Armor + 100)
    float ArmorMitigation = 1.0f - (Armor / (Armor + 100.f));

    float FinalDamage = RawDamage * CritMultiplier * ArmorMitigation;
    FinalDamage = FMath::Max(FinalDamage, 0.f); // No negative damage

    // Output: reduce Health
    OutExecutionOutput.AddOutputModifier(
        FGameplayModifierEvaluatedData(
            DamageStatics().HealthProperty,
            EGameplayModOp::Additive,
            -FinalDamage));
}
```

Then in the GE asset, add an **Execution** entry pointing to `UDamageExecCalc`. No modifiers needed on the GE itself — the ExecCalc handles everything.

## Conditional Gameplay Effects

The `FGameplayEffectExecutionDefinition` supports `ConditionalGameplayEffects` — additional GEs that will be applied if the execution succeeds and you call `MarkConditionalGameplayEffectsToTrigger()` on the output. Each conditional effect can have source tag requirements.

This is useful for "on hit" effects: if the damage calc succeeds, also apply a burn DoT.

## Passed-In Tags

The execution definition has a `PassedInTags` container. These tags are made available inside the ExecCalc via `ExecutionParams.GetPassedInTags()`. Enable `bRequiresPassedInTags` on your ExecCalc class to use them. This provides a lightweight way for GE designers to pass context to the calculation without SetByCaller.

## Tips

!!! warning "ExecCalcs Only Run on Authority"
    Execution Calculations only run on the server (or in standalone). They don't run on predicted clients. If you need client-side prediction for something an ExecCalc does, you'll need a different approach (like predicting the entire GE application with modifiers).

!!! tip "Keep It Testable"
    Extract your damage formula into a pure static function and call it from the ExecCalc. This makes it easy to write unit tests for the formula itself without needing a full GAS setup.

!!! note "Output Modifier Operations"
    Output modifiers from ExecCalcs are typically `Additive` (adding/subtracting from an attribute). But you can use any `EGameplayModOp`. Override is particularly useful when you want to _set_ an attribute to a specific value.
