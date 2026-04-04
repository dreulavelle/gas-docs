---
title: Effect Extensions (Legacy)
icon: material/puzzle-plus
description: The legacy GameplayEffectExtension framework — what it was, why it was removed, and what replaced it.
---

# Effect Extensions (Legacy)

If you've encountered `GameplayEffectExtension.h` in the GAS source and wondered what it does — the short answer is: **nothing anymore**. The original `UGameplayEffectExtension` classes were removed from the engine. The header file survives only as a legacy include point for `FGameplayEffectModCallbackData`, a struct that is still actively used throughout GAS.

## What It Was

The GameplayEffectExtension system was an early attempt at modular GE behavior. Extensions were `UObject` subclasses that could be attached to Gameplay Effects to run custom logic during effect execution. Think of them as a precursor to [GE Components](ge-components.md).

The original design allowed extensions to hook into:

- Pre/post execution of modifiers
- Custom handling when an effect was applied or removed
- Additional data attached to the effect

## Why It Was Removed

The extension system had several problems that led to its removal:

- **No clear ownership model** — extensions weren't proper subobjects, making serialization and garbage collection fragile
- **Limited hook points** — the extension API was too narrow for the variety of behaviors games needed
- **Replaced by better systems** — Execution Calculations, Modifier Magnitude Calculations, and eventually GE Components all solved the same problems more robustly

## What Replaced It

The functionality that extensions were meant to provide is now covered by three systems:

| Need | Modern Solution | Docs |
|:---|:---|:---|
| Custom logic during effect execution | [Execution Calculations](execution-calculations.md) | `UGameplayEffectExecutionCalculation` |
| Custom modifier math | [Magnitude Calculations](magnitude-calculations.md) | `UGameplayModifierMagnitudeCalculation` |
| Modular behavior on GE assets | [GE Components (5.3+)](ge-components.md) | `UGameplayEffectComponent` subclasses |
| Blocking effect application with custom logic | [Custom Application Requirements](custom-application.md) | `UGameplayEffectCustomApplicationRequirement` |

GE Components in particular are the spiritual successor to extensions. They provide a proper component architecture with clean lifecycle hooks, correct serialization as instanced subobjects, and a well-defined callback API.

## FGameplayEffectModCallbackData

The one thing that survives from `GameplayEffectExtension.h` is the `FGameplayEffectModCallbackData` struct:

```cpp
struct FGameplayEffectModCallbackData
{
    const FGameplayEffectSpec& EffectSpec;     // The spec the modifier came from
    FGameplayModifierEvaluatedData& EvaluatedData; // The computed modifier data
    UAbilitySystemComponent& Target;           // The target ASC
};
```

This struct is passed to `UAttributeSet::PreAttributeChange`, `PostGameplayEffectExecute`, and related callbacks on attribute sets. It tells you *what* is modifying *which* attribute and *who* is being modified.

Even though the extension system is gone, this struct is load-bearing — it's part of the attribute modification pipeline and isn't going anywhere.

## If You See Extension References in Old Code

If you're working with a codebase that references `UGameplayEffectExtension`:

1. The class no longer exists — any code using it won't compile against modern engine versions
2. Identify what the extension was doing
3. Migrate to the appropriate modern system (ExecCalc, ModMagCalc, or GE Component)
4. The `GameplayEffectExtension.h` include can be kept if you need `FGameplayEffectModCallbackData`, but the extension classes themselves are gone

## Further Reading

- [GE Components](ge-components.md) — the modern replacement for modular GE behavior
- [Execution Calculations](execution-calculations.md) — custom logic during effect execution
- [Magnitude Calculations](magnitude-calculations.md) — custom modifier math
- [Attributes and Attribute Sets](../core-concepts/attributes-and-attribute-sets.md) — where `FGameplayEffectModCallbackData` is used in practice
