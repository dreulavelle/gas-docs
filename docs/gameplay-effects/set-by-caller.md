---
title: SetByCaller
description: How to pass runtime values into Gameplay Effects using SetByCaller — setting by tag vs. name, reading in modifiers and ExecCalcs, common patterns, and debugging.
---

# SetByCaller

SetByCaller is the mechanism for passing **runtime values** from code into a Gameplay Effect. Instead of the GE defining its own magnitude (via a ScalableFloat, attribute, or custom calc), the calling code provides the number when creating the spec.

This is one of the most commonly used features in GAS. Any time you compute a value in gameplay code and need to feed it into a GE — damage from a weapon, heal amount from a potion, duration scaled by a stat — SetByCaller is how you do it.

## How It Works

The flow is:

1. **GE Definition:** A modifier (or execution, or duration) declares its magnitude as `SetByCaller`, with a `DataTag` (or `DataName`) that identifies which value to look up.

2. **Spec Creation:** Code creates an `FGameplayEffectSpec` from the GE.

3. **Value Setting:** Code sets the magnitude on the spec, keyed by the same tag or name.

4. **Application:** The spec is applied. When the engine evaluates the modifier, it looks up the SetByCaller value.

## Setting Values: Tag vs. Name

There are two ways to identify SetByCaller values: by **Gameplay Tag** and by **FName**. Tags are the modern, preferred approach.

=== "By Tag (Preferred)"

    ```cpp
    // On the GE: Modifier magnitude is SetByCaller with DataTag = "SetByCaller.Damage"

    // In code:
    FGameplayEffectSpecHandle SpecHandle =
        ASC->MakeOutgoingGameplayEffectSpec(DamageEffectClass, Level);

    SpecHandle.Data->SetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag("SetByCaller.Damage"),
        150.f);

    ASC->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
    ```

=== "By Name (Legacy)"

    ```cpp
    // On the GE: Modifier magnitude is SetByCaller with DataName = "Damage"

    // In code:
    SpecHandle.Data->SetSetByCallerMagnitude(FName("Damage"), 150.f);
    ```

!!! tip "Use Tags"
    Tags are type-safe, hierarchical, and visible in the tag editor. Names are just strings. New code should always use tags.

### The FSetByCallerFloat Struct

On the GE, a SetByCaller modifier uses this struct:

```cpp
struct FSetByCallerFloat
{
    FName DataName;        // Legacy: lookup by name
    FGameplayTag DataTag;  // Preferred: lookup by tag
};
```

If both are set, the engine will try the tag first, then fall back to the name.

## Reading SetByCaller Values

### In Modifiers

You don't need to do anything special — the engine reads the value automatically when evaluating the modifier's magnitude. Just make sure you set the value on the spec before applying.

### In Execution Calculations

Inside `Execute_Implementation`, read values from the owning spec:

```cpp
void UMyExecCalc::Execute_Implementation(
    const FGameplayEffectCustomExecutionParameters& ExecutionParams,
    FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

    // By tag
    float Damage = Spec.GetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag("SetByCaller.Damage"),
        false,  // WarnIfNotFound
        0.f);   // DefaultIfNotFound

    // By name (legacy)
    float Healing = Spec.GetSetByCallerMagnitude(
        FName("Healing"),
        false,
        0.f);
}
```

### In Custom Magnitude Calculations

Inside `CalculateBaseMagnitude_Implementation`:

```cpp
float UMyMagnitudeCalc::CalculateBaseMagnitude_Implementation(
    const FGameplayEffectSpec& Spec) const
{
    float BaseDamage = GetSetByCallerMagnitudeByTag(
        Spec,
        FGameplayTag::RequestGameplayTag("SetByCaller.Damage"));

    return BaseDamage * SomeMultiplier;
}
```

The `UGameplayModMagnitudeCalculation` base class provides `GetSetByCallerMagnitudeByTag` and `GetSetByCallerMagnitudeByName` as Blueprint-callable helpers.

## The Storage

SetByCaller values are stored on the `FGameplayEffectSpec` in two maps:

```cpp
TMap<FName, float>         SetByCallerNameMagnitudes;  // Legacy name-based
TMap<FGameplayTag, float>  SetByCallerTagMagnitudes;   // Tag-based
```

These maps are **replicated** as part of the spec. When a predicted spec is sent to the server (or an authoritative spec is replicated to clients), the SetByCaller values travel with it.

## Common Patterns

### Damage Scaling

The most common pattern. A weapon or ability computes damage based on game logic, then passes it into a damage GE:

```cpp
float BaseDamage = Weapon->GetBaseDamage();
float BonusDamage = CalculateBonusDamage(AttackerStats);

FGameplayEffectSpecHandle SpecHandle =
    ASC->MakeOutgoingGameplayEffectSpec(GE_ApplyDamage);
SpecHandle.Data->SetSetByCallerMagnitude(DamageTag, BaseDamage + BonusDamage);
ASC->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
```

### Dynamic Duration

A buff whose duration depends on a stat:

```cpp
float BuffDuration = CasterStats.BuffDuration * DurationMultiplier;

FGameplayEffectSpecHandle SpecHandle =
    ASC->MakeOutgoingGameplayEffectSpec(GE_SpeedBuff);
SpecHandle.Data->SetSetByCallerMagnitude(DurationTag, BuffDuration);
ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
```

The GE's `DurationMagnitude` is configured as SetByCaller with the matching tag.

### Combo Step Modifier

A combo system where each hit in the sequence does progressively more damage:

```cpp
int32 ComboStep = GetCurrentComboStep(); // 1, 2, 3, ...
float DamageMultiplier = 1.0f + (ComboStep * 0.25f); // 1.0, 1.25, 1.5, ...

FGameplayEffectSpecHandle SpecHandle =
    ASC->MakeOutgoingGameplayEffectSpec(GE_ComboDamage);
SpecHandle.Data->SetSetByCallerMagnitude(DamageTag, BaseDamage * DamageMultiplier);
```

### Multiple SetByCaller Values

A single GE can use multiple SetByCaller values across different modifiers or in an ExecCalc:

```cpp
SpecHandle.Data->SetSetByCallerMagnitude(DamageTag, 100.f);
SpecHandle.Data->SetSetByCallerMagnitude(StaggerTag, 25.f);
SpecHandle.Data->SetSetByCallerMagnitude(KnockbackTag, 500.f);
```

Each modifier or ExecCalc reads the specific tag it cares about.

## Blueprint Usage

In Blueprints, the equivalent flow uses `MakeOutgoingGameplayEffectSpec` → `Assign Set By Caller Magnitude` → `Apply Gameplay Effect Spec to Target`:

1. Call `MakeOutgoingGameplayEffectSpec` on the ASC to get a spec handle
2. Call `AssignTagSetByCallerMagnitude` on the spec handle, passing the tag and the float value
3. Call `ApplyGameplayEffectSpecToTarget` with the modified spec

The Blueprint functions are defined in `UAbilitySystemBlueprintLibrary`.

## Debugging Missing Values

If you forget to set a SetByCaller value before applying the spec, the engine will log a warning:

```
SetByCaller Magnitude for tag [SetByCaller.Damage] was not found on GE Spec [GE_ApplyDamage].
```

This is the `WarnIfNotFound` parameter at work. The default behavior:

- `GetSetByCallerMagnitude(Tag, true, 0.f)` — Warns and returns 0.0
- `GetSetByCallerMagnitude(Tag, false, 0.f)` — Silently returns 0.0

!!! tip "Don't Suppress Warnings Carelessly"
    The warning exists for a reason. If you're seeing it, it usually means the spec wasn't set up correctly before application. Only use `WarnIfNotFound = false` when you genuinely expect the value to be optional.

### Common Causes of Missing Values

1. **Forgot to set the value** — The most common cause. Make sure every code path that applies the GE also sets all required SetByCaller values.

2. **Tag mismatch** — The tag used in `SetSetByCallerMagnitude` doesn't match the tag configured on the GE's modifier. Check for typos or tag hierarchy issues.

3. **Copied spec without SetByCaller** — If you copy or chain specs, SetByCaller values don't automatically transfer. Use `CopySetByCallerMagnitudes` or `MergeSetByCallerMagnitudes` to carry values forward.

4. **AdditionalEffectsGameplayEffectComponent** — When additional effects are applied on completion, SetByCaller values from the original spec are **not** copied by default. Set `bOnApplicationCopyDataFromOriginalSpec = true` on the component if you need them forwarded.

## Tips

!!! warning "SetByCaller and Prediction"
    SetByCaller values travel with the spec, so they work fine with prediction. The client predicts with the values it set, and the server independently computes (potentially different) values. If there's a mismatch, the server's version wins during reconciliation.

!!! note "SetByCaller vs. Effect Context"
    SetByCaller is for **numeric magnitudes**. If you need to pass non-numeric data (actors, hit results, arbitrary objects), use the `FGameplayEffectContext` instead. See [Dynamic Effects](dynamic-effects.md) for more on effect context.

## Related Pages

- [Cooldowns and Costs](cooldowns-and-costs.md) -- using SetByCaller for dynamic cooldown durations and resource costs
- [Dynamic Effects](dynamic-effects.md) -- the full spec modification workflow that SetByCaller is part of
- [Melee Attack Example](../examples/melee-attack.md) -- practical SetByCaller usage for passing weapon damage into a GE
