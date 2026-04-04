---
icon: material/chart-line
---

# Scalable Floats and Curve Tables

`FScalableFloat` is GAS's primary mechanism for level-based scaling. It pairs a raw float value with an optional curve table reference, producing `Value * Curve[Level]` at runtime. It is one of the four [magnitude calculation](../gameplay-effects/magnitude-calculations.md) types and appears throughout [modifier](../gameplay-effects/modifiers.md) configuration.

---

## FScalableFloat

### Properties

| Property | Type | Description |
|---|---|---|
| `Value` | `float` | The raw coefficient. If no curve is set, this is the final value. |
| `Curve` | `FCurveTableRowHandle` | Reference to a row in a `UCurveTable`. Evaluated at the specified level. |
| `RegistryType` | `FDataRegistryType` | Optional: Data Registry containing the curve. If set, `Curve.RowName` is used as the item name. |

### Evaluation

At runtime, `GetValueAtLevel(Level)` computes:

```
FinalValue = Value * CurveTable[RowName].Eval(Level)
```

If no curve is set (`IsStatic() == true`), the raw `Value` is returned directly.

### Key Methods

| Method | Description |
|---|---|
| `GetValueAtLevel(float Level)` | Returns `Value * Curve[Level]`. The primary evaluation method. |
| `GetValue()` | Returns `GetValueAtLevel(0)`. Convenience for level-independent lookups. |
| `AsBool(float Level)` | Returns `true` if the evaluated value is non-zero |
| `AsInteger(float Level)` | Returns the evaluated value cast to `int32` |
| `IsStatic()` | Returns `true` if there is no curve reference (raw float only) |
| `SetValue(float)` | Sets the raw `Value` without changing the curve reference |
| `SetScalingValue(float, FName, UCurveTable*)` | Sets both the value and curve reference |
| `IsValid()` | Returns `false` if the curve reference is broken |

---

## Where FScalableFloat Appears in GAS

`FScalableFloat` is used extensively throughout the Gameplay Effect system:

| Location | Property | What It Controls |
|---|---|---|
| `FGameplayModifierInfo` | `ModifierMagnitude` (when using `ScalableFloat` type) | Modifier magnitude at a given GE level |
| `UGameplayEffect` | `DurationMagnitude` | How long a `HasDuration` effect lasts |
| `UGameplayEffect` | `Period` | Time between periodic executions |
| `FGameplayAbilitySpecConfig` | `LevelScalableFloat` | Ability level granted by `AbilitiesGameplayEffectComponent` |
| `UChanceToApplyGameplayEffectComponent` | `ChanceToApplyToTarget` | Probability of GE application (0.0 to 1.0) |
| Various ability tasks | Duration/delay parameters | Task-specific timing |

---

## Curve Tables

A `UCurveTable` is a collection of named curves, each mapping a float input (the "level") to a float output.

### Creating a Curve Table

=== "In the Editor"

    1. **Content Browser** > Add > Miscellaneous > **Curve Table**
    2. Choose **Rich Curve** (for interpolated float curves) or **Simple Curve** (for linear segments)
    3. Add rows — each row name becomes a key you reference from `FScalableFloat`
    4. Add columns for each level (1, 2, 3, ...) and set the values

=== "From CSV/JSON"

    1. Create a CSV file with a header row of level numbers:

        ```csv
        ---,1,2,3,4,5
        Damage,10,15,22,30,40
        Cooldown,5.0,4.5,4.0,3.5,3.0
        ManaCost,20,25,30,35,40
        ```

    2. **Content Browser** > Import, select the CSV, choose **CurveTable** as the import type
    3. Configure interpolation type (Constant, Linear, or Cubic)

### Linking a Scalable Float to a Curve Table

In a Gameplay Effect's modifier:

1. Set **Magnitude Calculation Type** to **Scalable Float**
2. Set the `Value` field (the multiplier coefficient, often `1.0`)
3. In the `Curve` dropdown, select your curve table and row
4. At runtime, the GE's level is passed as the curve input

### Global Curve Table

You can designate a global curve table in Project Settings > Gameplay Abilities Settings > **Global Curve Table**. This table is loaded once at startup and available to all `FScalableFloat` references. Useful for centralizing all scaling data in one asset.

---

## Data Registry Integration

`FScalableFloat` supports `FDataRegistryType` as an alternative to direct curve table references. When `RegistryType` is set:

1. The `Curve.RowName` is used as the Data Registry item name
2. The curve table reference itself may be empty — the registry resolves it
3. This enables async loading and late-binding of curve data

This is an advanced feature primarily useful for large projects that need lazy-loaded scaling data.

---

## Practical Patterns

### Flat Value (No Scaling)

Set `Value` to the desired number, leave `Curve` empty. The value is constant regardless of level.

```
Value: 25.0
Curve: None
Result at any level: 25.0
```

### Level Curve Only

Set `Value` to `1.0`, set `Curve` to a row that contains the actual values per level.

```
Value: 1.0
Curve: DamageTable.Fireball
Level 1 → 1.0 * 50  = 50
Level 5 → 1.0 * 120 = 120
```

### Coefficient + Curve

Set `Value` to a coefficient, set `Curve` to a normalized progression curve. Useful when multiple abilities share the same curve shape but different magnitudes.

```
Value: 2.5
Curve: SharedProgression.Linear
Level 1 → 2.5 * 10 = 25
Level 5 → 2.5 * 30 = 75
```

This allows designers to tune individual ability magnitudes (the `Value`) independently from the overall scaling curve.

## Related Pages

- [Modifiers](../gameplay-effects/modifiers.md) -- where ScalableFloat is most commonly used as a modifier magnitude
- [Magnitude Calculations](../gameplay-effects/magnitude-calculations.md) -- the four magnitude calculation policies including ScalableFloat
- [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) -- using ScalableFloat for level-scaled cooldown durations and costs
