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

When an effect stacks (see [Stacking](stacking.md)), the modifier magnitude can optionally be multiplied by the stack count. This is controlled by the `bFactorInStackCount` property on each `FGameplayModifierInfo` (per modifier, not per effect).

## Attribute Capture Definitions

When Execution Calculations or Modifier Magnitude Calculations need to read attribute values to compute their results, they must declare **capture definitions** -- formal declarations of which attributes they need, from whom, and whether to snapshot the value.

This is covered in full detail in [Execution Calculations > Attribute Capture](execution-calculations.md#attribute-capture), including:

- The `DECLARE_ATTRIBUTE_CAPTUREDEF` / `DEFINE_ATTRIBUTE_CAPTUREDEF` macros
- Source vs. Target capture
- Snapshot vs. Live evaluation
- Registering captures via `RelevantAttributesToCapture`
- Common gotchas (wrong source, missing registration)

The short version: every attribute a calculation reads must be explicitly declared as a `FGameplayEffectAttributeCaptureDefinition`. This struct specifies three things:

```cpp
struct FGameplayEffectAttributeCaptureDefinition
{
    FGameplayAttribute AttributeToCapture;  // Which attribute
    EGameplayEffectAttributeCaptureSource AttributeSource;  // Source or Target
    bool bSnapshot;  // Freeze at spec creation, or read live at execution
};
```

If a calculation tries to read an attribute it didn't register in `RelevantAttributesToCapture`, the capture will silently fail and return a default value. This is one of the most common sources of "my ExecCalc always outputs zero" bugs.

## Aggregator Internals

Everything above describes what modifiers do from the outside. This section peels back the layer and explains how modifiers actually work inside the engine. Understanding aggregators isn't required for normal development, but it's essential for debugging modifier behavior that doesn't match your expectations.

### What the Aggregator Is

Every attribute that has active modifiers gets an `FAggregator`. The aggregator is the internal object that collects all modifiers targeting a single attribute, stores them organized by operation type and evaluation channel, and computes the final attribute value on demand.

When you apply a duration effect with an Additive +20 modifier on Health, the engine doesn't change the Health value directly. Instead, it adds a `FAggregatorMod` entry to the Health attribute's `FAggregator`. When the engine needs the current value of Health, it evaluates the aggregator: start with the base value, run through all qualified mods in order, and produce the result.

```
Base Value (100)
    │
    └─ FAggregator
        ├─ Channel 0
        │   ├─ AddBase mods:        [+20 from Strength Buff, +10 from Rune]
        │   ├─ MultiplyAdditive:    [1.5 from War Shout]
        │   ├─ DivideAdditive:      []
        │   ├─ MultiplyCompound:    [1.1 from Enchantment]
        │   ├─ AddFinal:            [+15 from Rune Bonus]
        │   └─ Override:            []
        │
        └─ Result: computed via the aggregation formula
```

### How Evaluation Works

Aggregator evaluation is **lazy**. The aggregator doesn't recompute the final value every time a mod is added or removed. Instead, it marks itself as **dirty** and broadcasts `OnDirty` to any listeners (including the `FActiveGameplayEffectsContainer` that owns the attribute). The actual evaluation happens when something asks for the current attribute value.

The evaluation flow:

1. `FAggregator::Evaluate(Parameters)` is called
2. For each mod in each channel, `FAggregatorMod::UpdateQualifies(Parameters)` checks whether the mod's source/target tag requirements are met
3. Only qualified mods participate in the aggregation formula
4. The result is computed channel by channel using `EvaluateWithBase`
5. The output of one channel becomes the input base value for the next channel

### FScopedAggregatorOnDirtyBatch

The engine batches dirty notifications to avoid cascading recalculations. When multiple mods change in the same frame (common during effect application or removal), `FScopedAggregatorOnDirtyBatch` delays all `OnDirty` callbacks until the batch scope exits:

```cpp
{
    FScopedAggregatorOnDirtyBatch ScopedBatch;
    // Add/remove multiple mods -- no callbacks fire yet
    Aggregator1.AddMod(...);
    Aggregator2.RemoveAggregatorMod(...);
    Aggregator3.AddMod(...);
} // All dirty aggregators fire their callbacks here
```

This is an important implementation detail if you're debugging why an attribute value seems "stale" during effect processing -- it might be inside a batch scope.

### FAggregatorMod

Each individual modifier entry stored in an aggregator:

```cpp
struct FAggregatorMod
{
    const FGameplayTagRequirements* SourceTagReqs;  // Source must match these
    const FGameplayTagRequirements* TargetTagReqs;  // Target must match these
    float EvaluatedMagnitude;     // The modifier's magnitude
    float StackCount;             // Stack multiplier
    FActiveGameplayEffectHandle ActiveHandle;  // The GE that owns this mod
    bool IsPredicted;             // Whether this is a predicted mod

    bool Qualifies() const;       // Does this mod pass tag checks?
    void UpdateQualifies(const FAggregatorEvaluateParameters& Params) const;
};
```

The `Qualifies()` check is what makes per-modifier tag requirements work. During evaluation, mods that don't qualify are skipped entirely -- they contribute nothing to the final value.

### FAggregatorEvaluateParameters

The context passed into evaluation, controlling which mods qualify:

```cpp
struct FAggregatorEvaluateParameters
{
    const FGameplayTagContainer* SourceTags;  // Current source tags
    const FGameplayTagContainer* TargetTags;  // Current target tags

    // Ignore specific active effects during evaluation
    TArray<FActiveGameplayEffectHandle> IgnoreHandles;

    // Additional tag filters (mod's source/target tags must match ALL)
    FGameplayTagContainer AppliedSourceTagFilter;
    FGameplayTagContainer AppliedTargetTagFilter;

    bool IncludePredictiveMods;  // Whether to include predicted mods
};
```

This is how the engine implements tag-filtered evaluation. When you set `SourceTags` and `TargetTags`, each mod's tag requirements are checked against them. If a mod requires `State.Enraged` on the source but the source doesn't have that tag, the mod is disqualified for this evaluation.

### OnAttributeAggregatorCreated

When an aggregator is first created for an attribute (the first time a duration/infinite effect targets that attribute), the engine calls `UAttributeSet::OnAttributeAggregatorCreated`:

```cpp
// In your attribute set
virtual void OnAttributeAggregatorCreated(
    const FGameplayAttribute& Attribute,
    FAggregator* NewAggregator) const override
{
    if (Attribute == GetHealthAttribute())
    {
        // Set custom evaluation metadata for Health
        NewAggregator->EvaluationMetaData =
            &FAggregatorEvaluateMetaDataLibrary::MostNegativeMod_AllPositiveMods;
    }
}
```

This callback lets you attach custom evaluation rules to specific attributes. The engine ships one pre-built metadata rule:

- **`MostNegativeMod_AllPositiveMods`** -- Only the single most negative modifier counts, but all positive modifiers stack normally. This is useful for "most-negative-debuff-only" rules where you don't want multiple damage-reducing debuffs to stack.

You can also provide your own `FAggregatorEvaluateMetaData` with a custom `FCustomQualifiesFunc` lambda that runs during evaluation and toggles individual mod qualifications.

### Evaluation Channels (Channel0 through Channel9)

Evaluation channels let you segment modifier evaluation into independent layers. Each channel has its own set of mods organized by operation type, and the channels evaluate in order -- the output of Channel0 becomes the base value input for Channel1, and so on.

```
Base Value: 100

Channel 0: has AddBase +20 mod
  → Channel 0 result: 120

Channel 1: has MultiplyCompound 1.5 mod
  → Channel 1 input: 120 (output of Channel 0)
  → Channel 1 result: 180

Final Value: 180
```

The engine supports channels 0 through 9 (defined by `EGameplayModEvaluationChannel`), but only channels that actually have mods incur any cost. The channel map is sparse -- unused channels don't exist.

**When you'd use multiple channels:**

- **Layered stat systems.** Base stats computed in Channel 0, buff/debuff layer in Channel 1, final multipliers in Channel 2. This guarantees that buffs always see the fully-computed base stats, not intermediate values.
- **Ordered evaluation.** When one modifier needs to operate on the result of another modifier, but they're applied by different systems that can't coordinate ordering.

Most projects use only Channel 0 and never think about channels. They're a power tool for edge cases.

!!! note "Channel Configuration"
    Channels are hidden in the editor by default. To enable and name them, override `UAbilitySystemGlobals` and configure the channel names. See [AbilitySystemGlobals](../reference/ability-system-globals.md) for details.

### Aggregator Snapshots

The `TakeSnapshotOf` method creates a copy of an aggregator's state at a point in time. This is used internally for snapshotted attribute captures in Execution Calculations -- the ExecCalc gets a frozen copy of the aggregator rather than a live reference.

### Why This Matters for Debugging

When a modifier "isn't working," the problem is usually one of:

1. **Tag filtering.** The mod exists in the aggregator but doesn't qualify because the source or target tags don't match its requirements. Use `showdebug abilitysystem` to check active tags.

2. **Wrong channel.** The mod is in a different evaluation channel than expected. If another effect in a higher channel is overriding the value, your mod's contribution gets masked.

3. **Stale evaluation.** Inside a `FScopedAggregatorOnDirtyBatch`, attribute values haven't been recomputed yet. If you're checking a value during effect application, you might be reading a stale result.

4. **Prediction filtering.** `IncludePredictiveMods` is false, and the mod is predicted. On the server, predicted mods may not be included in evaluation until confirmed.

5. **Ignored handles.** Some evaluation contexts explicitly ignore certain active effect handles. The mod exists but is being skipped.

6. **Custom qualifies function.** If `OnAttributeAggregatorCreated` set custom `EvaluationMetaData`, the custom function might be disqualifying your mod.

To inspect an aggregator's state at runtime, the `showdebug abilitysystem` display shows active effects and their modifiers. For deeper inspection, you can set a breakpoint in `FAggregatorModChannel::EvaluateWithBase` and inspect the mods array and their qualification states.

## What's Next?

Now that you know what operations a modifier can perform, the next question is: where does the magnitude number come from? That's [Magnitude Calculations](magnitude-calculations.md).

## Related Pages

- [Modifier Formula](../reference/modifier-formula.md) -- the exact aggregation formula with worked examples and edge cases
