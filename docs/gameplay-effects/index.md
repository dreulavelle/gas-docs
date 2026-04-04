---
title: Gameplay Effects Deep Dive
description: Comprehensive guide to Gameplay Effects in Unreal Engine's Gameplay Ability System — duration, modifiers, calculations, stacking, components, and more.
---

# Gameplay Effects Deep Dive

Gameplay Effects (GEs) are the workhorse of GAS. They are the **data-driven bundles** that actually _do things_ to actors: heal them, burn them, speed them up, slow them down, grant them abilities, make them immune to crowd control. If Gameplay Abilities are the verbs, Gameplay Effects are the nouns — the tangible changes applied to an actor's state.

This section goes deep. If you are looking for a high-level overview, start with the [Core Concepts: Gameplay Effects Overview](../core-concepts/gameplay-effects-overview.md) first, then come back here.

## The Three Duration Types (Quick Recap)

Every Gameplay Effect has a **Duration Policy** that determines its lifetime:

| Duration Policy | What It Does | Lives In Active GE Container? |
|:---|:---|:---|
| **Instant** | Executes once, permanently changes base attribute values, then disappears | No |
| **Has Duration** | Lives on the target for a set amount of time, modifying attributes while active | Yes |
| **Infinite** | Lives on the target forever (until explicitly removed) | Yes |

Duration and Infinite effects are _added_ to the Active Gameplay Effects Container. Instant effects are _executed_ — they modify the base value of an attribute and are gone. This distinction matters a lot, and we cover it thoroughly in [Duration and Lifecycle](duration-and-lifecycle.md).

## Decision Tree: Where Should I Go?

Not sure which page answers your question? Start here.

**"I want to..."**

- **...understand how GEs are created, live, and die** — [Duration and Lifecycle](duration-and-lifecycle.md)
- **...add or multiply an attribute by some value** — [Modifiers](modifiers.md)
- **...figure out how a modifier gets its number** — [Magnitude Calculations](magnitude-calculations.md)
- **...write a complex formula that touches multiple attributes** — [Execution Calculations](execution-calculations.md)
- **...make a buff that stacks up to 5 times** — [Stacking](stacking.md)
- **...grant abilities, apply tags, or block other effects** — [GE Components](ge-components.md)
- **...control when effects can apply based on tags** — [Tags and Requirements](tags-and-requirements.md)
- **...make my character immune to fire damage** — [Immunity](immunity.md)
- **...implement cooldowns or resource costs for abilities** — [Cooldowns and Costs](cooldowns-and-costs.md)
- **...pass a damage number from code into an effect** — [SetByCaller](set-by-caller.md)
- **...create or modify effects at runtime in C++** — [Dynamic Effects](dynamic-effects.md)
- **...show effect names or icons in my UI** — [UI Data](ui-data.md)
- **...add custom logic to control when an effect applies** — [Custom Application](custom-application.md)
- **...understand the old extension system** — [Effect Extensions](effect-extensions.md) (Legacy)

## Suggested Reading Order

If you are working through this for the first time, this order builds concepts progressively:

1. [Duration and Lifecycle](duration-and-lifecycle.md) — the foundation
2. [Modifiers](modifiers.md) — how attributes get changed
3. [Magnitude Calculations](magnitude-calculations.md) — where the numbers come from
4. [SetByCaller](set-by-caller.md) — the most common way to pass data into effects
5. [Execution Calculations](execution-calculations.md) — the power tool for complex formulas
6. [Stacking](stacking.md) — what happens when the same effect is applied twice
7. [GE Components](ge-components.md) — the modular architecture (5.3+)
8. [Tags and Requirements](tags-and-requirements.md) — conditional application and ongoing requirements
9. [Immunity](immunity.md) — blocking effects
10. [Cooldowns and Costs](cooldowns-and-costs.md) — the ability economy
11. [Dynamic Effects](dynamic-effects.md) — runtime creation and modification
12. [UI Data](ui-data.md) — showing effect names, descriptions, and icons in UI
13. [Custom Application](custom-application.md) — custom logic controlling when effects apply
14. [Effect Extensions](effect-extensions.md) — the legacy extension system (pre-5.3)

!!! tip "Already comfortable with GAS?"
    If you have shipped a project with GAS before and just need to look something up, jump straight to whatever page interests you. Each page is designed to stand on its own.

## Key Source Files

For engine spelunkers, the relevant headers live under `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/`:

| File | What It Contains |
|:---|:---|
| `GameplayEffect.h` | `UGameplayEffect`, `FGameplayEffectSpec`, `FActiveGameplayEffect`, modifier structs, stacking enums |
| `GameplayEffectTypes.h` | `EGameplayModOp`, evaluation channels, `FGameplayEffectContext` |
| `GameplayEffectComponent.h` | `UGameplayEffectComponent` base class |
| `GameplayEffectComponents/*.h` | All 11 built-in component types |
| `GameplayEffectExecutionCalculation.h` | `UGameplayEffectExecutionCalculation` and its parameter/output structs |
| `GameplayModMagnitudeCalculation.h` | `UGameplayModMagnitudeCalculation` base class |
