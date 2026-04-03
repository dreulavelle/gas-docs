---
title: Magnitude Calculations
description: The four magnitude calculation policies for Gameplay Effect modifiers — ScalableFloat, AttributeBased, CustomCalculationClass, and SetByCaller.
---

# Magnitude Calculations

Every modifier needs a number — its magnitude. The `EGameplayEffectMagnitudeCalculation` enum defines four ways to produce that number, from dead simple to fully custom:

```cpp
enum class EGameplayEffectMagnitudeCalculation : uint8
{
    ScalableFloat,            // A simple float, optionally scaled by a curve table
    AttributeBased,           // Derived from an attribute's value
    CustomCalculationClass,   // A custom C++ (or BP) class computes it
    SetByCaller,              // The code that creates the spec provides the value
};
```

Each policy has a dedicated data structure that configures it. Let's walk through all four.

## ScalableFloat

The simplest option. You specify a base value and optionally reference a row in a Curve Table to scale it by level.

```
Magnitude = BaseValue * CurveTable[Level]
```

If no curve table is specified, the magnitude is just the flat value you enter.

**When to use:** Simple, static values. A buff that always gives +20 armor. A poison that always does -5 HP per tick. Anything where the number doesn't depend on runtime context.

**With Curve Tables:** Scalable floats shine when paired with curve tables for level scaling. You define a curve that maps level to a multiplier, and the engine looks up the appropriate value at the GE's level. This lets designers create one GE asset that scales across an entire progression range.

!!! tip "Curve Tables Are Powerful"
    Don't underestimate ScalableFloat. With curve tables, a single GE asset can produce different magnitudes at level 1 vs. level 50 without any code. Many shipped games use ScalableFloat + curve tables for the vast majority of their effects.

## AttributeBased

Derives the magnitude from a captured attribute value. The full formula is:

```
Magnitude = (Coefficient * (PreMultiplyAdditiveValue + [Attribute Value])) + PostMultiplyAdditiveValue
```

Where `[Attribute Value]` depends on the `EAttributeBasedFloatCalculationType`:

| Calculation Type | What It Returns |
|:---|:---|
| `AttributeMagnitude` | The final evaluated magnitude of the attribute (base + all active modifiers) |
| `AttributeBaseValue` | Only the base value of the attribute (ignoring active modifiers) |
| `AttributeBonusMagnitude` | The bonus from modifiers only: `FinalMag - BaseValue` |
| `AttributeMagnitudeEvaluatedUpToChannel` | The attribute value computed only up to a specified [evaluation channel](modifiers.md#evaluation-channels) |

The `FAttributeBasedFloat` struct:

```cpp
struct FAttributeBasedFloat
{
    FScalableFloat Coefficient;           // Default: 1.0
    FScalableFloat PreMultiplyAdditiveValue;   // Default: 0.0
    FScalableFloat PostMultiplyAdditiveValue;  // Default: 0.0

    FGameplayEffectAttributeCaptureDefinition BackingAttribute; // Which attribute to read
    FCurveTableRowHandle AttributeCurve;  // Optional: use attribute value as lookup into curve

    EAttributeBasedFloatCalculationType AttributeCalculationType;
    EGameplayModEvaluationChannel FinalChannel; // For EvaluatedUpToChannel type

    FGameplayTagContainer SourceTagFilter; // Only count modifiers from sources with these tags
    FGameplayTagContainer TargetTagFilter; // Only count modifiers on targets with these tags
};
```

**When to use:** When a modifier's magnitude should scale off an attribute. A heal that restores HP proportional to the healer's Wisdom. A damage bonus based on the target's missing health. A shield that absorbs damage equal to 200% of your Armor.

### Worked Example

A "Power Strike" ability that deals damage equal to 150% of the attacker's Attack Power, plus a flat 10:

- **Coefficient**: 1.5
- **PreMultiplyAdditiveValue**: 0
- **BackingAttribute**: Source's AttackPower (captured, final magnitude)
- **PostMultiplyAdditiveValue**: 10

If the attacker has 80 AttackPower:
```
(1.5 * (0 + 80)) + 10 = 120 + 10 = 130
```

### Curve Table Lookup

If `AttributeCurve` is specified, the attribute value is used as the **lookup key** into the curve, rather than being used directly. This lets you create non-linear scaling: a diminishing returns curve for armor, a breakpoint system for critical hit chance, etc.

### Tag Filters

The `SourceTagFilter` and `TargetTagFilter` let you narrow which modifiers are considered when evaluating the backing attribute. For example, if your SourceTagFilter requires `Buff.Fire`, only modifiers from effects with that tag will be included when computing the attribute value.

## CustomCalculationClass

When ScalableFloat and AttributeBased aren't flexible enough, you can write a custom C++ or Blueprint class that computes the magnitude.

Your class extends `UGameplayModMagnitudeCalculation`:

```cpp
UCLASS()
class UMyCustomMagnitudeCalc : public UGameplayModMagnitudeCalculation
{
    GENERATED_BODY()

public:
    UMyCustomMagnitudeCalc()
    {
        // Declare which attributes we need captured
        // These must be added to RelevantAttributesToCapture in the constructor
        FGameplayEffectAttributeCaptureDefinition AttackCapture;
        AttackCapture.AttributeToCapture = UMyAttributeSet::GetAttackPowerAttribute();
        AttackCapture.AttributeSource = EGameplayEffectAttributeCaptureSource::Source;
        AttackCapture.bSnapshot = true;

        RelevantAttributesToCapture.Add(AttackCapture);
    }

    virtual float CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec& Spec) const override
    {
        // Gather tags for filtering
        const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
        const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

        FAggregatorEvaluateParameters EvalParams;
        EvalParams.SourceTags = SourceTags;
        EvalParams.TargetTags = TargetTags;

        float AttackPower = 0.f;
        GetCapturedAttributeMagnitude(
            RelevantAttributesToCapture[0], Spec, EvalParams, AttackPower);

        // Custom logic: Attack scales quadratically after 100
        if (AttackPower > 100.f)
        {
            return 100.f + FMath::Sqrt(AttackPower - 100.f) * 10.f;
        }
        return AttackPower;
    }
};
```

The class is wrapped in `FCustomCalculationBasedFloat`, which adds the same Coefficient / PreMultiplyAdditiveValue / PostMultiplyAdditiveValue / FinalLookupCurve that AttributeBased has:

```
FinalMagnitude = (Coefficient * (PreMultiplyAdditiveValue + CalculateBaseMagnitude())) + PostMultiplyAdditiveValue
```

And optionally, the result can be looked up in a curve via `FinalLookupCurve`.

**When to use:** Non-linear scaling, multi-attribute formulas for a _single modifier_ (as opposed to [Execution Calculations](execution-calculations.md) which can output _multiple_ modifiers), or any calculation that needs game-specific logic.

!!! info "Custom Calc vs. Execution Calc"
    `UGameplayModMagnitudeCalculation` produces a single magnitude value for a single modifier. `UGameplayEffectExecutionCalculation` can read multiple attributes and output multiple modifier changes. If you need to modify multiple attributes in one formula, use an Execution Calculation. If you just need a clever number for one modifier, Custom Calc is the lighter-weight choice.

### Blueprint Support

`UGameplayModMagnitudeCalculation` is `Blueprintable`. You can override `CalculateBaseMagnitude` in Blueprint if you prefer, though C++ is more common for performance-critical calculations.

### Helper Functions

The class provides several Blueprint-callable helper functions:

- `GetCapturedAttributeMagnitude` — Read a captured attribute's value
- `GetSetByCallerMagnitudeByTag` / `GetSetByCallerMagnitudeByName` — Read SetByCaller values from the spec
- `GetSourceAggregatedTags` / `GetTargetAggregatedTags` — Get combined tags
- `GetSourceActorTags` / `GetTargetActorTags` — Get actor-level tags
- `GetSourceSpecTags` / `GetTargetSpecTags` — Get spec-level tags

## SetByCaller

The calling code provides the magnitude directly by setting it on the `FGameplayEffectSpec` before application. The GE just declares "I expect a value identified by this tag (or name)."

```cpp
// In the GE asset: Modifier uses SetByCaller with DataTag = "SetByCaller.Damage"

// In code, before applying:
FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingGameplayEffectSpec(DamageEffectClass);
SpecHandle.Data->SetSetByCallerMagnitude(
    FGameplayTag::RequestGameplayTag("SetByCaller.Damage"), 75.f);
ASC->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
```

**When to use:** When the magnitude is determined entirely by game logic at the call site. Damage values computed by a weapon. Heal amounts from a potion. Duration scaling by combo count. Anything where the number comes from _outside_ the GE definition.

SetByCaller is covered extensively in its [own page](set-by-caller.md).

## Decision Table

| Situation | Recommended Policy |
|:---|:---|
| Fixed value, optionally scaled by level | **ScalableFloat** |
| Scales off a single attribute with simple math | **AttributeBased** |
| Scales off an attribute with non-linear or complex math | **CustomCalculationClass** |
| Multiple attributes need complex interaction (single modifier) | **CustomCalculationClass** |
| The value is known at the call site, not by the GE | **SetByCaller** |
| Multiple attributes modified by one interconnected formula | Use an [Execution Calculation](execution-calculations.md) instead of modifiers |

!!! tip "Start Simple"
    Most effects in a typical project use ScalableFloat or SetByCaller. Don't reach for CustomCalculationClass until you actually need it — the simpler options are easier to debug and maintain.
