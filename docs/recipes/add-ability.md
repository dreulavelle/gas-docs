---
title: "Recipe: Add an Ability"
description: Step-by-step procedure for creating a Gameplay Ability Blueprint, configuring it, and granting it to a character.
---

# Recipe: Add an Ability

**Goal:** Create a new Gameplay Ability, configure its class defaults, implement its activation logic, and grant it to a character.

**Prerequisites:** A C++ base ability class and a character with an ASC. See [C++ vs Blueprint](../patterns/cpp-vs-blueprint.md) for the recommended setup.

---

## Steps

### 1. Create the Blueprint

1. In Content Browser, right-click in your `GAS/Abilities/` folder
2. Select **Blueprint Class**
3. Under "All Classes," search for your base ability class (e.g., `MyGameplayAbility`) -- or use `GameplayAbility` if you don't have a custom base
4. Name it with the `GA_` prefix: `GA_Fireball`

### 2. Set Class Defaults

Open the Blueprint and go to the **Class Defaults** panel:

| Property | Recommended Setting | Notes |
|:---|:---|:---|
| **Ability Tags** | `Ability.Combat.Ranged.Fireball` | Identifies this specific ability |
| **Cancel Abilities with Tag** | (leave empty or set as needed) | Abilities to cancel when this activates |
| **Block Abilities with Tag** | (leave empty or set as needed) | Abilities that can't activate while this is active |
| **Activation Blocked Tags** | `CrowdControl.Stun`, `State.Dead` | Tags on the owner that block activation |
| **Activation Required Tags** | (usually empty) | Tags the owner must have to activate |
| **Instancing Policy** | `InstancedPerActor` | Recommended default. See [Instancing Policy](../gameplay-abilities/instancing-policy.md). |
| **Net Execution Policy** | `LocalPredicted` | For player-activated abilities. See [Net Execution Policies](../networking/net-execution-policies.md). |
| **Cost GE** | Your cost effect (e.g., `GE_Cost_Mana_30`) | Optional. See [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md). |
| **Cooldown GE** | Your cooldown effect (e.g., `GE_Cooldown_3s`) | Optional |

### 3. Wire ActivateAbility

In the Event Graph:

1. The **Event ActivateAbility** node is your entry point
2. Implement your ability logic:
    - Play montage (via `PlayMontageAndWait` task)
    - Wait for events (gameplay event, input, timer)
    - Apply effects to targets
3. Always call **EndAbility** when the ability finishes

```
Event ActivateAbility
  → Commit Ability (checks cost + cooldown, deducts cost)
    → [Branch: success]
      → Play Montage And Wait
        → On Completed: End Ability
        → On Cancelled: End Ability
    → [Branch: fail]
      → End Ability
```

!!! warning "Always call EndAbility"
    An ability that never ends will block other abilities (if it has blocking tags) and leak resources. Every code path must reach `EndAbility`.

### 4. Grant to Character

In your character's initialization (BeginPlay or after ASC is ready):

=== "C++"

    ```cpp
    void AMyCharacter::InitializeAbilities()
    {
        if (HasAuthority() && AbilitySystemComponent)
        {
            for (TSubclassOf<UGameplayAbility>& Ability : DefaultAbilities)
            {
                AbilitySystemComponent->GiveAbility(
                    FGameplayAbilitySpec(Ability, 1,
                        INDEX_NONE, this));
            }
        }
    }
    ```

=== "Blueprint"

    Call **Give Ability** on the ASC node, passing the ability class.

=== "Ability Set"

    Use a `UGameplayAbilitySet` data asset to grant a collection of abilities, effects, and attribute sets together. See [Ability Sets](../gameplay-abilities/ability-sets.md).

### 5. Test

1. PIE
2. Trigger the ability (via input binding or `AbilitySystem.AbilityActivate` console command)
3. Verify it activates, plays feedback, and ends cleanly
4. Check `showdebug abilitysystem` to confirm it appears in the ability list

---

## Checklist

- [ ] Blueprint created with `GA_` prefix
- [ ] Class Defaults configured (tags, instancing, net policy, cost, cooldown)
- [ ] `ActivateAbility` wired with gameplay logic
- [ ] `EndAbility` called on all exit paths
- [ ] Ability granted to the character
- [ ] Tested in PIE

## Related

- [Lifecycle and Activation](../gameplay-abilities/lifecycle-and-activation.md) -- how abilities activate and end
- [Bind Ability to Input](bind-ability-to-input.md) -- wiring to player input
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) -- async operations within abilities
- [Blueprint Function Library](../reference/blueprint-library.md) -- all available Blueprint nodes for GAS
