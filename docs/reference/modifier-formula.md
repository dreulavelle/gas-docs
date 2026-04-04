---
icon: material/calculator-variant
---

# The Modifier Aggregation Formula

Every attribute in GAS is evaluated by an `FAggregator` that collects all active modifiers and computes a final value using a fixed formula. This page documents the exact formula from UE 5.7 source (`GameplayEffectAggregator.cpp`).

For modifier concepts, see [Modifiers](../gameplay-effects/modifiers.md).

---

## The Formula

```
((BaseValue + Additive) * Multiplicative / Division * CompoundMultiply) + FinalAdd
```

From `FAggregatorModChannel::EvaluateWithBase`:

```cpp
return ((InlineBaseValue + Additive) * Multiplicitive / Division * CompoundMultiply) + FinalAdd;
```

**Override short-circuits the entire formula.** If any qualifying `Override` modifier exists, its value is returned immediately — no other modifiers are evaluated.

---

## EGameplayModOp Values

The enum `EGameplayModOp::Type` defines six modifier operations, evaluated in this order:

| Enum Value | Display Name | Internal Index | Bias | Aggregation | Description |
|---|---|---|---|---|---|
| `AddBase` | Add (Base) | 0 | 0.0 | Summed | Added to the base value before any multiplication |
| `MultiplyAdditive` | Multiply (Additive) | 1 | 1.0 | Summed | Multipliers are summed together, then applied as a single multiplication |
| `DivideAdditive` | Divide (Additive) | 2 | 1.0 | Summed | Divisors are summed together, then applied as a single division |
| `Override` | Override | 3 | -- | First wins | Replaces the entire computation. First qualifying Override wins. |
| `MultiplyCompound` | Multiply (Compound) | 4 | -- | Multiplied | Each modifier is multiplied against the running product |
| `AddFinal` | Add (Final) | 5 | 0.0 | Summed | Added after all multiplication/division is complete |

!!! warning "Override index discontinuity"
    In the enum, `Override = 3` and `MultiplyCompound = 4`. This is **not** the evaluation order — Override is checked first before any other operations.

---

## Bias and SumMods

For `AddBase`, `MultiplyAdditive`, `DivideAdditive`, and `AddFinal`, modifiers are aggregated using `SumMods`:

```cpp
float FAggregatorModChannel::SumMods(const TArray<FAggregatorMod>& InMods, float Bias, ...)
{
    float Sum = Bias;
    for (const FAggregatorMod& Mod : InMods)
    {
        if (Mod.Qualifies())
        {
            Sum += (Mod.EvaluatedMagnitude - Bias);
        }
    }
    return Sum;
}
```

The **bias** is the identity value for the operation. It is subtracted from each modifier's magnitude before summing, then the sum starts at the bias.

| Operation | Bias | Identity Meaning |
|---|---|---|
| AddBase | 0.0 | Adding 0 changes nothing |
| MultiplyAdditive | 1.0 | Multiplying by 1 changes nothing |
| DivideAdditive | 1.0 | Dividing by 1 changes nothing |
| AddFinal | 0.0 | Adding 0 changes nothing |
| Override | 0.0 | N/A (not summed) |
| MultiplyCompound | -- | Uses direct multiplication, not SumMods |

**What this means in practice:** A `MultiplyAdditive` modifier with magnitude `1.5` contributes `+0.5` to the sum (1.5 - 1.0 bias). Two such modifiers sum to `1.0 + 0.5 + 0.5 = 2.0`.

### CompoundMultiply

`MultiplyCompound` does not use SumMods. Each modifier is multiplied directly:

```cpp
float MultiplyMods(const TArray<FAggregatorMod>& InMods)
{
    float Result = 1.0f;
    for (const FAggregatorMod& Mod : InMods)
    {
        if (Mod.Qualifies())
            Result *= Mod.EvaluatedMagnitude;
    }
    return Result;
}
```

Two `MultiplyCompound` modifiers of `1.5` produce `1.5 * 1.5 = 2.25` (not `2.0`).

---

## Worked Examples

### Example 1: Simple Additive

An attribute with `BaseValue = 100`. One GE applies `AddBase +20`.

```
= ((100 + 20) * 1.0 / 1.0 * 1.0) + 0
= 120
```

### Example 2: Multiple Multiplicative

`BaseValue = 100`. Two GEs each apply `MultiplyAdditive 1.5` (i.e., +50%).

The two 1.5 values sum additively: `1.0 + (1.5 - 1.0) + (1.5 - 1.0) = 2.0`

```
= ((100 + 0) * 2.0 / 1.0 * 1.0) + 0
= 200
```

This is the "additive stacking" behavior: two +50% bonuses give +100%, not +125%.

### Example 3: Compound Multiply

`BaseValue = 100`. Two GEs each apply `MultiplyCompound 1.5`.

Compound multipliers multiply together: `1.5 * 1.5 = 2.25`

```
= ((100 + 0) * 1.0 / 1.0 * 2.25) + 0
= 225
```

This is true multiplicative stacking: each 50% bonus compounds on the previous result.

### Example 4: Mixed Operations

`BaseValue = 100`. Applied modifiers:

- `AddBase +20`
- `MultiplyAdditive 1.5` (+50%)
- `DivideAdditive 2.0` (divide by 2)
- `MultiplyCompound 1.1` (+10% compound)
- `AddFinal +5`

Aggregated values:

- Additive = 0 + (20 - 0) = **20**
- Multiplicative = 1.0 + (1.5 - 1.0) = **1.5**
- Division = 1.0 + (2.0 - 1.0) = **2.0**
- CompoundMultiply = **1.1**
- FinalAdd = 0 + (5 - 0) = **5**

```
= ((100 + 20) * 1.5 / 2.0 * 1.1) + 5
= (120 * 1.5 / 2.0 * 1.1) + 5
= (180 / 2.0 * 1.1) + 5
= (90 * 1.1) + 5
= 99 + 5
= 104
```

### Example 5: Override Wins

`BaseValue = 100`. Applied modifiers:

- `AddBase +50`
- `Override 75`
- `MultiplyAdditive 2.0`

Result: **75**. The Override modifier short-circuits all other computation.

---

## Edge Cases

### Division by Zero

If the `DivideAdditive` sum evaluates to approximately zero, the engine logs a warning and substitutes `1.0`:

```cpp
if (FMath::IsNearlyZero(Division))
{
    ABILITY_LOG(Warning, TEXT("Division summation was 0.0f in FAggregatorModChannel."));
    Division = 1.f;
}
```

### Multiple Overrides

If multiple qualifying `Override` modifiers exist, **the first one in the array wins**. The order is the order they were added to the aggregator, which generally corresponds to the order the GEs were applied.

### Override + Other Modifiers

Other modifiers are simply ignored when an Override exists. They are not evaluated at all.

### Reverse Evaluation

Clients use `ReverseEvaluate` to determine the server's base value from a replicated final value. This works by inverting the formula:

```cpp
ComputedValue = ((FinalValue - FinalAdd) / CompoundMultiply * Division / Multiplicitive) - Additive;
```

This fails (returns the raw `FinalValue`) if an `Override` modifier is active or if `Multiplicative` is near zero.

---

## Evaluation Channels

When `bAllowGameplayModEvaluationChannels` is enabled (off by default), modifiers can be assigned to different channels (`Channel0` through `Channel9`). Channels are evaluated in numeric order, with **the output of each channel becoming the base value for the next**.

Each channel independently applies the full formula above. This allows certain modifiers to be isolated from others — for example, putting a "damage reduction" step in a separate channel from "base damage modifiers."

See [AbilitySystemGlobals](ability-system-globals.md) for how to enable and name channels.

## Related Pages

- [Modifiers](../gameplay-effects/modifiers.md) -- how modifiers are configured on Gameplay Effects and feed into this formula
- [Execution Calculations](../gameplay-effects/execution-calculations.md) -- an alternative to modifiers for complex multi-attribute formulas
