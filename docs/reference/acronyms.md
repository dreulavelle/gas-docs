---
icon: material/alphabetical-variant
---

# Acronyms

Quick-reference table of abbreviations used throughout GAS documentation and source code.

## GAS Acronyms

| Abbreviation | Full Name | Description |
|---|---|---|
| **ASC** | Ability System Component | `UAbilitySystemComponent` — the central hub that owns abilities, effects, attributes, and tags |
| **AT** | Ability Task | `UAbilityTask` — latent task that runs within an active ability's scope |
| **CDO** | Class Default Object | The template UObject instance; GAS uses CDOs extensively for instancing policies |
| **ExecCalc** | Execution Calculation | `UGameplayEffectExecutionCalculation` — custom C++ class for multi-attribute calculations |
| **GA** | Gameplay Ability | `UGameplayAbility` — a discrete action a character can perform |
| **GAS** | Gameplay Ability System | The full plugin/framework name |
| **GC** | Gameplay Cue | Cosmetic-only notification (VFX/SFX) triggered by gameplay events or effects |
| **GE** | Gameplay Effect | `UGameplayEffect` — data-driven object that modifies attributes and grants tags |
| **GEC** | Gameplay Effect Component | `UGameplayEffectComponent` — modular behavior component on a GE (UE 5.3+) |
| **MMC** | Modifier Magnitude Calculation | `UGameplayModifierMagnitudeCalculation` — custom C++ class for computing a single modifier's magnitude |
| **SBC** | SetByCaller | Magnitude type where the value is set on the `FGameplayEffectSpec` at runtime via a Gameplay Tag key |

## Related Engine Acronyms

| Abbreviation | Full Name | Description |
|---|---|---|
| **FGT** | Gameplay Tag | `FGameplayTag` — a hierarchical label like `Ability.Skill.Fireball` |
| **FGTC** | Gameplay Tag Container | `FGameplayTagContainer` — a set of Gameplay Tags |
| **FGTR** | Gameplay Tag Requirements | `FGameplayTagRequirements` — has/does-not-have tag filter |
| **FGES** | Gameplay Effect Spec | `FGameplayEffectSpec` — runtime instance of a GE with context, magnitude, and tags |
| **FGAS** | Gameplay Ability Spec | `FGameplayAbilitySpec` — runtime instance of a granted ability |
| **AGE** | Active Gameplay Effect | `FActiveGameplayEffect` — a GE that has been applied and is tracked by the ASC |
