---
title: "Example: Custom Damage Calculation"
description: Complete UGameplayEffectExecutionCalculation implementing a full damage formula with attribute captures, elemental resistances, crit, and armor.
---

# Example: Custom Damage Calculation

<div class="example-badges" markdown>
  <span class="badge badge--advanced">Advanced</span>
</div>

A custom `UGameplayEffectExecutionCalculation` that implements a full damage formula: `(BaseDamage * CritMultiplier) - ArmorReduction`, then reduced by elemental resistance. This is the most powerful (and complex) way to calculate effect magnitudes in GAS — it can capture attributes from both the source and target, read tags from the effect, and output arbitrary modifiers.

ExecCalcs **must** be written in C++. There is no Blueprint equivalent. If your project needs a custom damage formula that reads multiple attributes from multiple actors, this is the tool for the job.

## What We're Building

- A C++ class (`UExecCalc_Damage`) that GAS calls when the damage effect executes
- **Source captures:** `BaseDamage`, `CriticalHitChance`, `CriticalHitMultiplier`
- **Target captures:** `Armor`, `Health`
- Reads the **damage type** from the effect's asset tags (`Damage.Type.Fire`, `Damage.Type.Ice`, etc.)
- Reads the matching **resistance** from the target (`Resistance.Fire`, `Resistance.Ice`, etc.)
- Formula: `FinalDamage = ((BaseDamage + BonusDamage) * CritMultiplier - Armor) * (1 - Resistance)`
- Outputs a modifier to `PendingDamage`

## Prerequisites

Everything from [Project Setup](../getting-started/project-setup.md), plus these attributes in your AttributeSet:

| Attribute | Owner | Purpose |
|:---|:---|:---|
| `BaseDamage` | Source | Base damage value for the attacker |
| `CriticalHitChance` | Source | Chance to crit (0.0 - 1.0) |
| `CriticalHitMultiplier` | Source | Crit damage multiplier (e.g., 2.0 = double damage) |
| `Armor` | Target | Flat damage reduction |
| `Health` | Target | Current health |
| `PendingDamage` | Target | Meta attribute for damage processing |

And these gameplay tags defined in your project:

- `Damage.Type.Physical`, `Damage.Type.Fire`, `Damage.Type.Ice`, `Damage.Type.Lightning`
- `Resistance.Physical`, `Resistance.Fire`, `Resistance.Ice`, `Resistance.Lightning`

For elemental resistances, you'll also need matching attributes: `ResistancePhysical`, `ResistanceFire`, `ResistanceIce`, `ResistanceLightning` (float, 0.0 - 1.0, where 0.5 means 50% resistance).

---

## Step 1: The ExecCalc Class

### Header

```cpp
// ExecCalc_Damage.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffectExecutionCalculation.h"
#include "ExecCalc_Damage.generated.h"

UCLASS()
class YOURPROJECT_API UExecCalc_Damage : public UGameplayEffectExecutionCalculation
{
    GENERATED_BODY()

public:
    UExecCalc_Damage();

    virtual void Execute_Implementation(
        const FGameplayEffectCustomExecutionParameters& ExecutionParams,
        FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;
};
```

The class is simple: a constructor that registers which attributes to capture, and `Execute_Implementation` which contains the actual formula.

!!! info "Why Execute_Implementation?"
    `Execute` is declared as a `BlueprintNativeEvent` in the engine source (see `UGameplayEffectExecutionCalculation`). That means the actual C++ override is `Execute_Implementation`, not `Execute`. The engine dispatches to your implementation through the native event system.

### Source

```cpp
// ExecCalc_Damage.cpp
#include "ExecCalc_Damage.h"
#include "AbilitySystemComponent.h"
#include "YourProjectAttributeSet.h"

// ---------------------------------------------------------------
// Attribute capture declarations using engine-provided macros
// ---------------------------------------------------------------
// DECLARE_ATTRIBUTE_CAPTUREDEF expands to:
//   FProperty* <Name>Property;
//   FGameplayEffectAttributeCaptureDefinition <Name>Def;
//
// DEFINE_ATTRIBUTE_CAPTUREDEF(AttributeSetClass, Property, CaptureSource, bSnapshot)
//   CaptureSource: Source or Target
//   bSnapshot: true = capture value at spec creation time
//              false = capture value at execution time

struct FDamageStatics
{
    DECLARE_ATTRIBUTE_CAPTUREDEF(BaseDamage);
    DECLARE_ATTRIBUTE_CAPTUREDEF(CriticalHitChance);
    DECLARE_ATTRIBUTE_CAPTUREDEF(CriticalHitMultiplier);
    DECLARE_ATTRIBUTE_CAPTUREDEF(Armor);

    FDamageStatics()
    {
        // Source attributes — snapshot at spec creation so they
        // reflect the attacker's stats at the moment of attack,
        // not when the projectile hits 2 seconds later
        DEFINE_ATTRIBUTE_CAPTUREDEF(
            UYourProjectAttributeSet, BaseDamage, Source, true);
        DEFINE_ATTRIBUTE_CAPTUREDEF(
            UYourProjectAttributeSet, CriticalHitChance, Source, true);
        DEFINE_ATTRIBUTE_CAPTUREDEF(
            UYourProjectAttributeSet, CriticalHitMultiplier, Source, true);

        // Target attributes — do NOT snapshot, we want the
        // target's current armor at the moment damage is applied
        DEFINE_ATTRIBUTE_CAPTUREDEF(
            UYourProjectAttributeSet, Armor, Target, false);
    }
};

static const FDamageStatics& DamageStatics()
{
    static FDamageStatics Statics;
    return Statics;
}

UExecCalc_Damage::UExecCalc_Damage()
{
    // Register all capture definitions with the parent class.
    // This tells GAS which attributes this calc needs access to.
    RelevantAttributesToCapture.Add(DamageStatics().BaseDamageDef);
    RelevantAttributesToCapture.Add(DamageStatics().CriticalHitChanceDef);
    RelevantAttributesToCapture.Add(DamageStatics().CriticalHitMultiplierDef);
    RelevantAttributesToCapture.Add(DamageStatics().ArmorDef);
}

void UExecCalc_Damage::Execute_Implementation(
    const FGameplayEffectCustomExecutionParameters& ExecutionParams,
    FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    // Get the owning spec and the source/target ASCs
    const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

    UAbilitySystemComponent* SourceASC = ExecutionParams.GetSourceAbilitySystemComponent();
    UAbilitySystemComponent* TargetASC = ExecutionParams.GetTargetAbilitySystemComponent();

    if (!SourceASC || !TargetASC)
    {
        return;
    }

    // Gather source and target tags for evaluation
    const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    FAggregatorEvaluateParameters EvalParams;
    EvalParams.SourceTags = SourceTags;
    EvalParams.TargetTags = TargetTags;

    // --------------------------------------------------
    // 1. Capture attribute values
    // --------------------------------------------------
    float BaseDamage = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().BaseDamageDef, EvalParams, BaseDamage);

    float CritChance = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().CriticalHitChanceDef, EvalParams, CritChance);

    float CritMultiplier = 1.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().CriticalHitMultiplierDef, EvalParams, CritMultiplier);

    float Armor = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
        DamageStatics().ArmorDef, EvalParams, Armor);

    // --------------------------------------------------
    // 2. Read bonus damage from SetByCaller (optional)
    // --------------------------------------------------
    // Abilities can pass additional damage via SetByCaller on the spec.
    // If no SetByCaller was set for this tag, it returns 0.
    float BonusDamage = Spec.GetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag(FName("SetByCaller.BonusDamage")),
        /*bWarnIfNotFound=*/ false,
        /*DefaultIfNotFound=*/ 0.0f);

    // --------------------------------------------------
    // 3. Crit roll
    // --------------------------------------------------
    float CritRoll = FMath::FRandRange(0.0f, 1.0f);
    float FinalCritMultiplier = (CritRoll <= CritChance) ? CritMultiplier : 1.0f;

    // --------------------------------------------------
    // 4. Elemental resistance
    // --------------------------------------------------
    // Read the damage type from the effect's asset tags
    float Resistance = 0.0f;

    struct FDamageTypeMapping
    {
        FGameplayTag DamageTypeTag;
        FGameplayAttribute ResistanceAttribute;
    };

    // Map damage type tags to resistance attributes
    const TArray<FDamageTypeMapping> DamageTypeMappings = {
        {
            FGameplayTag::RequestGameplayTag(FName("Damage.Type.Fire")),
            UYourProjectAttributeSet::GetResistanceFireAttribute()
        },
        {
            FGameplayTag::RequestGameplayTag(FName("Damage.Type.Ice")),
            UYourProjectAttributeSet::GetResistanceIceAttribute()
        },
        {
            FGameplayTag::RequestGameplayTag(FName("Damage.Type.Lightning")),
            UYourProjectAttributeSet::GetResistanceLightningAttribute()
        },
        {
            FGameplayTag::RequestGameplayTag(FName("Damage.Type.Physical")),
            UYourProjectAttributeSet::GetResistancePhysicalAttribute()
        },
    };

    // Check which damage type tag is on the effect spec's asset tags
    FGameplayTagContainer AssetTags;
    Spec.GetAllAssetTags(AssetTags);

    for (const FDamageTypeMapping& Mapping : DamageTypeMappings)
    {
        if (AssetTags.HasTagExact(Mapping.DamageTypeTag))
        {
            // Read the resistance value directly from the target's ASC
            bool bFound = false;
            Resistance = TargetASC->GetGameplayAttributeValue(
                Mapping.ResistanceAttribute, bFound);
            if (!bFound)
            {
                Resistance = 0.0f;
            }
            break; // Use the first matching damage type
        }
    }

    // Clamp resistance to [0, 1] range
    Resistance = FMath::Clamp(Resistance, 0.0f, 1.0f);

    // --------------------------------------------------
    // 5. Apply the formula
    // --------------------------------------------------
    // FinalDamage = ((BaseDamage + BonusDamage) * CritMultiplier - Armor) * (1 - Resistance)
    float DamageBeforeArmor = (BaseDamage + BonusDamage) * FinalCritMultiplier;
    float DamageAfterArmor = FMath::Max(DamageBeforeArmor - Armor, 0.0f);
    float FinalDamage = DamageAfterArmor * (1.0f - Resistance);

    // Ensure damage is non-negative
    FinalDamage = FMath::Max(FinalDamage, 0.0f);

    if (FinalDamage > 0.0f)
    {
        // --------------------------------------------------
        // 6. Output the modifier
        // --------------------------------------------------
        // AddOutputModifier takes a FGameplayModifierEvaluatedData:
        //   - Attribute: which attribute to modify
        //   - ModifierOp: how to modify it (EGameplayModOp::Type)
        //   - Magnitude: the value
        OutExecutionOutput.AddOutputModifier(
            FGameplayModifierEvaluatedData(
                UYourProjectAttributeSet::GetPendingDamageAttribute(),
                EGameplayModOp::Additive,    // Add to PendingDamage
                FinalDamage));
    }
}
```

!!! warning "AttemptCalculateCapturedAttributeMagnitude can fail"
    The `Attempt...` prefix isn't decorative — the function returns `bool`. It fails if the spec doesn't have a valid capture for that attribute (misconfigured effect, missing AttributeSet on the source/target). In production code, you should check the return value and handle the failure case. The example above silently uses the default values (0.0) for simplicity.

---

## Step 2: Create the Gameplay Effect

Create a new Gameplay Effect asset: `GE_Damage_ExecCalc`

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Executions[0] — Calculation Class** | `ExecCalc_Damage` |

**No modifiers.** The ExecCalc outputs the modifier itself via `AddOutputModifier`. You don't configure modifiers in the effect editor when using an ExecCalc — the calc is the modifier.

**Tags (via AssetTagsGameplayEffectComponent):**

| Tag | Purpose |
|:---|:---|
| `Damage.Type.Fire` | Tells the ExecCalc this is fire damage (reads matching resistance) |

!!! tip "One effect, many damage types"
    You can create one `GE_Damage_ExecCalc` per damage type (fire, ice, physical) by changing only the asset tag. Or create a single effect and set the damage type tag dynamically via `FGameplayEffectSpec::DynamicAssetTags` when making the spec. The ExecCalc reads the tag at execution time either way.

### Using the Effect from an Ability

Here's how an ability creates and applies this damage effect:

=== "Blueprint"
    ```
    Make Outgoing Gameplay Effect Spec (GE_Damage_ExecCalc)
        |
        v
    Assign Set By Caller Magnitude
        +-- Data Tag: SetByCaller.BonusDamage
        +-- Magnitude: 10.0 (optional bonus from ability level, weapon, etc.)
        |
        v
    Apply Gameplay Effect Spec to Target
        +-- Spec: (from above)
        +-- Target: (hit actor's ASC)
    ```

=== "C++"
    ```cpp
    // Inside an ability's ActivateAbility or hit callback
    FGameplayEffectSpecHandle SpecHandle =
        MakeOutgoingGameplayEffectSpec(DamageEffectClass, GetAbilityLevel());

    if (SpecHandle.IsValid())
    {
        // Optional: set bonus damage via SetByCaller
        SpecHandle.Data->SetSetByCallerMagnitude(
            FGameplayTag::RequestGameplayTag(FName("SetByCaller.BonusDamage")),
            10.0f);

        // Optional: add a dynamic damage type tag
        // (if not already on the effect's asset tags)
        SpecHandle.Data->DynamicAssetTags.AddTag(
            FGameplayTag::RequestGameplayTag(FName("Damage.Type.Fire")));

        // Apply to target
        ApplyGameplayEffectSpecToTarget(
            CurrentSpecHandle,
            CurrentActorInfo,
            CurrentActivationInfo,
            SpecHandle,
            TargetData);
    }
    ```

---

## Step 3: Wire Input

This example has no input of its own — the ExecCalc is used inside a damage effect that abilities apply. See [Melee Attack](melee-attack.md) or [Ranged Attack](ranged-attack.md) for how abilities wire input and apply damage effects.

---

## Step 4: Test

### Verify Damage Numbers

1. Set up a source character with known attributes:
    - `BaseDamage` = 100
    - `CriticalHitChance` = 0.0 (disable crit for deterministic testing)
    - `CriticalHitMultiplier` = 2.0

2. Set up a target character with known attributes:
    - `Armor` = 20
    - `ResistanceFire` = 0.25 (25% fire resistance)
    - `Health` = 500

3. Apply `GE_Damage_ExecCalc` (with `Damage.Type.Fire` tag) from source to target

4. Expected result:
    - `(100 + 0) * 1.0 - 20 = 80` (no crit, no bonus damage, minus armor)
    - `80 * (1 - 0.25) = 60` (fire resistance)
    - `PendingDamage` receives `60`
    - After `PostGameplayEffectExecute` processes it: `Health = 500 - 60 = 440`

### Using showdebug

```
showdebug abilitysystem
```

Watch the target's attributes panel. When the damage effect is applied, you'll see `PendingDamage` spike to 60 (briefly, since it's processed and reset to 0 in `PostGameplayEffectExecute`) and `Health` drop to 440.

### Test Matrix

| Source BaseDamage | BonusDamage (SBC) | Crit? | Target Armor | Resistance | Expected |
|:---|:---|:---|:---|:---|:---|
| 100 | 0 | No | 20 | 0.0 | 80 |
| 100 | 0 | No | 20 | 0.25 | 60 |
| 100 | 0 | Yes (2x) | 20 | 0.0 | 180 |
| 100 | 50 | No | 20 | 0.0 | 130 |
| 100 | 0 | No | 200 | 0.0 | 0 (clamped) |
| 0 | 0 | No | 0 | 0.0 | 0 |

---

## The Full Flow

```mermaid
flowchart LR
    A["Ability applies\nGE_Damage_ExecCalc"]:::event --> B["GAS captures\nattributes"]:::func
    B --> C["Execute_Implementation()"]:::func
    C --> D["Read attributes\n+ SetByCaller"]:::func
    D --> E["Crit roll +\nResistance lookup"]:::branch
    E --> F["Compute formula\n(Base+Bonus)*Crit - Armor\n* (1-Resistance)"]:::func
    F --> G["Output modifier\nPendingDamage += Final"]:::func
    G --> H["PostGameplayEffectExecute\nPendingDamage -> Health"]:::endpoint

    classDef event fill:#5c1a1a,stroke:#ff6666,color:#fff
    classDef func fill:#2a2a4a,stroke:#9b89f5,color:#fff
    classDef branch fill:#5c4a1a,stroke:#ffb84a,color:#fff
    classDef endpoint fill:#1a4a2d,stroke:#6bcb3a,color:#fff
```

---

## Variations

??? example "Execution Calculations vs Modifier Magnitude Calculations"
    If your formula only needs one attribute (e.g., scale damage by the source's Strength), you don't need an ExecCalc. A `UGameplayModMagnitudeCalculation` (MMC) is simpler — it captures one attribute and outputs a single magnitude. Use ExecCalcs when you need:

    - Multiple source AND target attributes
    - Conditional logic (crit rolls, damage type branching)
    - Multiple output modifiers from one calculation
    - Access to the full effect context (instigator, hit result, etc.)

    See [Magnitude Calculations](../gameplay-effects/magnitude-calculations.md) vs [Execution Calculations](../gameplay-effects/execution-calculations.md) for a detailed comparison.

??? example "Adding hit location multiplier"
    Read the hit result from the effect context: `Spec.GetContext().GetHitResult()`. If the hit location is on the head (bone name check or physics asset region), multiply damage by a headshot multiplier. This data flows naturally through the `FGameplayEffectContext` that's set when the ability applies the spec.

??? example "Damage over time with ExecCalc"
    Use `Has Duration` with a `Period` on the effect. The ExecCalc fires on every periodic execution, not just once. This lets you implement damage-over-time with armor/resistance checking on every tick — so if the target gains armor mid-DoT, the remaining ticks deal less damage.

??? example "Logging damage breakdown"
    Add `UE_LOG` calls in your ExecCalc to print the damage pipeline: base, bonus, crit multiplier, armor reduction, resistance factor, and final damage. This is invaluable during tuning. Wrap it in a `#if !UE_BUILD_SHIPPING` guard so it compiles out of shipping builds.

---

## Related Pages

- [Execution Calculations](../gameplay-effects/execution-calculations.md) — deep dive on ExecCalc architecture
- [Magnitude Calculations](../gameplay-effects/magnitude-calculations.md) — simpler alternative for single-attribute formulas
- [SetByCaller](../gameplay-effects/set-by-caller.md) — passing runtime values into effect specs
- [Modifiers](../gameplay-effects/modifiers.md) — understanding `EGameplayModOp` and how output modifiers work
- [Damage Pipeline](../patterns/damage-pipeline.md) — how PendingDamage flows through PostGameplayEffectExecute
- [Tags and Requirements](../gameplay-effects/tags-and-requirements.md) — asset tags and how effects use them
