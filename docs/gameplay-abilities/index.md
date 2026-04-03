---
title: Gameplay Abilities Deep Dive
description: Comprehensive guide to Gameplay Abilities in Unreal Engine's Gameplay Ability System — lifecycle, instancing, tasks, input, targeting, and more.
---

# Gameplay Abilities Deep Dive

Gameplay Abilities are the **verbs** of GAS. They represent discrete actions an actor can perform: casting a fireball, dodging, jumping, entering a rage state, channeling a heal. Every ability goes through a well-defined lifecycle of activation, execution, and termination — and the system provides a rich set of tools for controlling each step.

If you are looking for a high-level overview, start with the [Core Concepts: Gameplay Abilities Overview](../core-concepts/gameplay-abilities-overview.md) first, then come back here for the deep dive.

## The Ability Lifecycle (Quick Recap)

At its core, every ability flows through the same phases:

```
CanActivateAbility()  →  ActivateAbility()  →  CommitAbility()  →  EndAbility()
       ↑                                             ↑
  Tag checks                                  Cost + Cooldown
  Cooldown check                              applied here
  Cost check
  Net role check
```

Between `ActivateAbility` and `EndAbility`, your ability is alive and running. It can spawn [Ability Tasks](ability-tasks.md) to do asynchronous work, request [targeting data](targeting/index.md), play montages, apply [Gameplay Effects](../gameplay-effects/index.md), and fire [Gameplay Cues](../gameplay-cues/index.md). The ability is not complete until `EndAbility` is called — and forgetting to call it is one of the most common bugs new GAS developers run into.

We cover the lifecycle in full detail on the [Lifecycle and Activation](lifecycle-and-activation.md) page.

## Decision Tree: Where Should I Go?

Not sure which page answers your question? Start here.

**"I want to..."**

- **...understand how abilities are activated, committed, and ended** — [Lifecycle and Activation](lifecycle-and-activation.md)
- **...figure out which instancing policy to use** — [Instancing Policy](instancing-policy.md)
- **...do async work inside my ability (wait for events, play montages, etc.)** — [Ability Tasks](ability-tasks.md)
- **...listen for GAS events from a UI widget or AI controller** — [Async Ability Tasks](async-ability-tasks.md)
- **...bind abilities to player input (Enhanced Input)** — [Input Binding](input-binding.md)
- **...grant a set of abilities when a character spawns or equips something** — [Ability Sets](ability-sets.md)
- **...implement interactive targeting (aim at the ground, select actors, etc.)** — [Targeting](targeting/index.md)
- **...see what the built-in example abilities look like** — [Built-in Abilities](built-in-abilities.md)

**"I have a problem with..."**

- **...my ability never ending / leaking** — Check the [EndAbility section](lifecycle-and-activation.md#endability) and make sure every code path calls it.
- **...tasks crashing when the ability ends** — Check your [Instancing Policy](instancing-policy.md) — you may be using NonInstanced with tasks (don't do that).
- **...abilities not activating** — Walk through the [full activation check sequence](lifecycle-and-activation.md#the-full-check-sequence) to see which check is failing. Use `ShowDebug AbilitySystem` to inspect tags.
- **...input not working with abilities** — See [Input Binding](input-binding.md) for the Enhanced Input integration patterns.

## Suggested Reading Order

If you are working through this section for the first time, this order builds concepts progressively:

1. [Lifecycle and Activation](lifecycle-and-activation.md) — the foundation, how abilities live and die
2. [Instancing Policy](instancing-policy.md) — how abilities are allocated in memory (affects everything else)
3. [Ability Tasks](ability-tasks.md) — the async building blocks for ability logic
4. [Async Ability Tasks](async-ability-tasks.md) — the newer alternative for non-ability contexts
5. [Input Binding](input-binding.md) — connecting player input to ability activation
6. [Ability Sets](ability-sets.md) — organizing and granting groups of abilities
7. [Targeting](targeting/index.md) — the full targeting subsystem
8. [Built-in Abilities](built-in-abilities.md) — learning from Epic's examples

!!! tip "Already comfortable with GAS?"
    If you have shipped a project with GAS before and just need to look something up, jump straight to whatever page interests you. Each page is designed to stand on its own.

## Key Source Files

For engine spelunkers, the relevant headers live under `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/`:

| File | What It Contains |
|:---|:---|
| `Abilities/GameplayAbility.h` | `UGameplayAbility` — the full ability class with lifecycle, commit, cancel, tags, cues |
| `GameplayAbilitySpec.h` | `FGameplayAbilitySpec`, `FGameplayAbilityActivationInfo`, `FGameplayAbilitySpecDef` |
| `Abilities/GameplayAbilityTypes.h` | `FGameplayAbilityActorInfo`, instancing/net execution/trigger enums |
| `Abilities/Tasks/AbilityTask.h` | `UAbilityTask` base class |
| `Abilities/Tasks/AbilityTask_*.h` | All 27+ built-in ability tasks |
| `Abilities/Async/AbilityAsync.h` | `UAbilityAsync` base class for async tasks |
| `Abilities/Async/AbilityAsync_*.h` | All 6 async ability task types |
| `Abilities/GameplayAbilityTargetActor*.h` | Target actor classes for interactive targeting |
| `Abilities/GameplayAbilityTargetTypes.h` | `FGameplayAbilityTargetData` and its handle |
| `Abilities/GameplayAbilityTargetDataFilter.h` | `FGameplayTargetDataFilter` |
| `Abilities/GameplayAbilityWorldReticle*.h` | World reticle actors for targeting visualization |
| `GameplayAbilitySet.h` | `UGameplayAbilitySet` data asset |
| `Abilities/GameplayAbility_CharacterJump.h` | Built-in jump ability example |
| `Abilities/GameplayAbility_Montage.h` | Built-in montage ability example |
