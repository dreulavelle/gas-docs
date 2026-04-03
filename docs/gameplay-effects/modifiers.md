---
title: Modifiers
description: How Gameplay Effect modifiers change attributes — operations, the aggregation formula, evaluation channels, and modifier tag requirements.
---

# Modifiers

Modifiers are the primary way Gameplay Effects change attributes. Each modifier targets a single attribute, applies a single operation, and gets its magnitude from one of [four calculation policies](magnitude-calculations.md). When multiple modifiers affect the same attribute, they are aggregated together using a well-defined formula.

## FGameplayModifierInfo

Every modifier on a Gameplay Effect is defined as an `FGameplayModifierInfo`:

```cpp
struct FGameplayModifierInfo
{
    // Which attribute this modifier targets
    FGameplayAttribute Attribute;

    // The operation: Add, Multiply, Divide, Override, etc.
    TEnumAsByte<EGameplayModOp::Type> ModifierOp;

    // How the magnitude is calculated (ScalableFloat, AttributeBased, Custom, SetByCaller)
    FGameplayEffectModifierMagnitude ModifierMagnitude;

    // Which evaluation channel this modifier operates in
    FGameplayModEvaluationChannelSettings EvaluationChannelSettings;

    // Tag requirements on the source for this modifier to qualify
    FGameplayTagRequirements SourceTags;

    // Tag requirements on the target for this modifier to qualify
    FGameplayTagRequirements TargetTags;
};
```

You configure these in the `Modifiers` array on a `UGameplayEffect`. Each entry in the array is one modifier.

## Modifier Operations (EGameplayModOp)

There are six operations, and their order of evaluation matters. Here they are, in the order the engine applies them:

| Operation | Enum Value | What It Does |
|:---|:---|:---|
| **Add (Base)** | `EGameplayModOp::AddBase` | Adds to the base value before any multiplication |
| **Multiply (Additive)** | `EGameplayModOp::MultiplyAdditive` | Multipliers are summed, then applied once |
| **Divide (Additive)** | `EGameplayModOp::DivideAdditive` | Divisors are summed, then applied once |
| **Multiply (Compound)** | `EGameplayModOp::MultiplyCompound` | Each multiplier is applied individually (compounding) |
| **Add (Final)** | `EGameplayModOp::AddFinal` | Flat addition after all multiplication |
| **Override** | `EGameplayModOp::Override` | Replaces the entire computed value |

!!! warning "Override Precedence"
    If any modifier uses Override, it wins — the entire computed result is replaced with the Override value. If multiple Overrides exist, the last one applied takes effect. Use Override sparingly.

## The Aggregation Formula

When multiple modifiers target the same attribute, the engine aggregates them using this formula:

```
((BaseValue + SumOf(AddBase)) * SumOf(MultiplyAdditive) / SumOf(DivideAdditive) * ProductOf(MultiplyCompound)) + SumOf(AddFinal)
```

Spelled out more precisely, from the source code comment in `GameplayEffectTypes.h`:

```
((BaseValue + AddBase) * MultiplyAdditive / DivideAdditive * MultiplyCompound) + AddFinal
```

Where:

- **AddBase** values are summed together: `SumOf(all AddBase magnitudes)`
- **MultiplyAdditive** values use a "bias" of 1.0, so they are summed and applied as a single multiplier: `1.0 + SumOf(all MultiplyAdditive magnitudes - 1.0 each)`
- **DivideAdditive** values also use a bias of 1.0, summed as a single divisor: `1.0 + SumOf(all DivideAdditive magnitudes - 1.0 each)`
- **MultiplyCompound** values are multiplied individually: `ProductOf(all MultiplyCompound magnitudes)`
- **AddFinal** values are summed together: `SumOf(all AddFinal magnitudes)`

### Worked Example

Let's say we have a base **Attack Power** of **100** and the following active modifiers:

| Modifier | Operation | Magnitude |
|:---|:---|:---|
| Strength Buff | AddBase | +20 |
| War Shout | MultiplyAdditive | 1.5 (50% increase) |
| Battle Fury | MultiplyAdditive | 1.3 (30% increase) |
| Armor Pen | DivideAdditive | 2.0 |
| Enchantment | MultiplyCompound | 1.1 |
| Rune Bonus | AddFinal | +15 |

**Step 1: AddBase**
```
100 + 20 = 120
```

**Step 2: MultiplyAdditive**

The two multiplicative modifiers use biased addition. The bias for Multiply is 1.0, so:
```
Additive Sum = (1.5 - 1.0) + (1.3 - 1.0) = 0.5 + 0.3 = 0.8
Multiplier = 1.0 + 0.8 = 1.8
120 * 1.8 = 216
```

!!! info "Why Biased Addition?"
    MultiplyAdditive uses biased addition so that two "+50% damage" buffs give you +100% total (double damage), not +125% (1.5 * 1.5). This is intuitive for designers — percentages stack additively with each other.

**Step 3: DivideAdditive**

The divisor also uses biased addition. Bias for Divide is 1.0:
```
Additive Sum = (2.0 - 1.0) = 1.0
Divisor = 1.0 + 1.0 = 2.0
216 / 2.0 = 108
```

**Step 4: MultiplyCompound**

Compound multipliers are applied individually (no bias):
```
108 * 1.1 = 118.8
```

If there were two compound multipliers of 1.1, they would compound: `108 * 1.1 * 1.1 = 130.68`

**Step 5: AddFinal**
```
118.8 + 15 = 133.8
```

**Final Value: 133.8**

### Another Quick Example: Two Additive vs. Two Compound Multipliers

Suppose base value is **100** and two 50% multipliers are applied:

=== "MultiplyAdditive (Both)"

    ```
    Additive Sum = (1.5 - 1.0) + (1.5 - 1.0) = 1.0
    Multiplier = 1.0 + 1.0 = 2.0
    100 * 2.0 = 200
    ```
    Result: **200** (the multipliers add together: +50% + +50% = +100%)

=== "MultiplyCompound (Both)"

    ```
    100 * 1.5 * 1.5 = 225
    ```
    Result: **225** (the multipliers compound on each other)

This distinction is why both operation types exist. Additive multiplication gives designers predictable percentage stacking. Compound multiplication gives true multiplicative scaling.

## Evaluation Channels

Evaluation channels (Channel0 through Channel9) let you segment modifier evaluation into independent layers. Within each channel, the full aggregation formula runs independently, and the channels are then combined.

Most projects never touch channels and everything runs in `Channel0` by default. Channels are useful when you need an ordering guarantee across different systems — for example, ensuring that a base stat calculation in Channel0 completes before a buff layer in Channel1 reads from it.

!!! note "Channels Need Configuration"
    Evaluation channels are hidden by default in the editor. To enable and name them, configure them in your project's `AbilitySystemGlobals` subclass. If you don't use them, don't worry about them.

The `AttributeMagnitudeEvaluatedUpToChannel` calculation type in [Magnitude Calculations](magnitude-calculations.md) uses channels to read an attribute value computed only up to a certain channel, which can be useful for layered calculations.

## Modifier Tag Requirements

Each modifier can have its own **Source Tags** and **Target Tags** requirements:

```cpp
struct FGameplayModifierInfo
{
    // ...
    FGameplayTagRequirements SourceTags;  // Tags the source must have for this modifier to apply
    FGameplayTagRequirements TargetTags;  // Tags the target must have for this modifier to apply
};
```

These are evaluated at the time the modifier is aggregated. If the tag requirements aren't met, the modifier is skipped — it's as if it doesn't exist for that evaluation.

This is different from GE-level tag requirements (which control whether the entire effect can _apply_). Modifier-level tags let you have a single GE with multiple modifiers where some are conditional.

**Example:** A "Fire Shield" effect that grants +20 fire resistance always, but also grants +10 ice resistance only if the target has `State.Frozen`:

```
Modifier 1: AddBase +20 to FireResistance (no tag requirements)
Modifier 2: AddBase +10 to IceResistance (TargetTags: Require State.Frozen)
```

## Instant vs. Duration Modifier Behavior

How modifiers interact with attributes depends on the effect's duration policy:

- **Instant modifiers** change the **base value** of the attribute. The modifier is applied, the base value is permanently altered, and the effect disappears.

- **Duration/Infinite modifiers** change the **current value** while active. The base value is untouched. When the effect is removed, the current value recalculates without that modifier's contribution.

This means if you have Health = 100 (base) and apply an Infinite AddBase +50:

- Base value stays 100
- Current value becomes 150
- Remove the effect: current value returns to 100

But an Instant AddBase +50:

- Base value becomes 150
- Current value becomes 150
- The effect is gone — the +50 is permanent

## Stacking and Modifiers

When an effect stacks (see [Stacking](stacking.md)), the modifier magnitude can optionally be multiplied by the stack count. This is controlled by the `bFactorInStackCount` property on `UGameplayEffect`.

## What's Next?

Now that you know what operations a modifier can perform, the next question is: where does the magnitude number come from? That's [Magnitude Calculations](magnitude-calculations.md).
