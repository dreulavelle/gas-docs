---
icon: material/run-fast
description: Complete dodge roll ability with stamina cost, cooldown, i-frames via Activation Owned Tags, root motion montage, and ability blocking.
---

# Example: Dodge Roll

<div class="example-badges" markdown>
  <span class="badge badge--intermediate">Intermediate</span>
</div>

## Overview

A dodge roll ability that grants invulnerability frames (i-frames) during the animation, costs stamina, and blocks other abilities while rolling. This demonstrates how Activation Owned Tags create temporary character states, how those states integrate with the damage pipeline, and how Activation Blocked Tags prevent re-rolling mid-dodge. The roll uses root motion from the montage for physically grounded movement.

## What We're Building

- **Stamina cost** of 20 per roll
- **Short cooldown** of 0.5 seconds to prevent roll spam
- **I-frames** during the entire roll animation via `State.Invulnerable` tag
- **Root motion movement** -- the dodge distance comes from the animation, not manual velocity
- **Ability blocking** -- can't attack or re-roll while rolling
- **CC blocking** -- can't roll while stunned or dead
- **Tags as state** -- `State.Evading` and `State.Invulnerable` are granted automatically on activation and removed on end

## Prerequisites

!!! tip "What you need before starting"
    This example assumes you have completed [Project Setup](../getting-started/project-setup.md) and have:

    - A character with an **Ability System Component**
    - An **AttributeSet** with Health and Stamina attributes
    - A **base ability class** (`YourProjectGameplayAbility`) with an `InputTag` property
    - An **input binding system** that routes Enhanced Input actions to abilities by tag
    - A **dodge roll animation montage** (`AM_DodgeRoll`) with root motion, set up for your character's skeleton

    If any of that is missing, start with [Project Setup](../getting-started/project-setup.md).

## Step 1: Create the Effects

The dodge roll only needs two effects -- a cost and a cooldown. There is no damage effect because the dodge doesn't deal damage. The i-frame behavior comes entirely from tags, not from a Gameplay Effect.

### GE_Cost_DodgeRoll

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] -- Attribute** | `YourProjectAttributeSet.Stamina` |
| **Modifiers[0] -- Modifier Op** | Add |
| **Modifiers[0] -- Magnitude** | Scalable Float: `-20.0` |

### GE_Cooldown_DodgeRoll

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `0.5` (half a second) |
| **GrantedTags** | `Cooldown.Evade` |

We use `Cooldown.Evade` rather than `Cooldown.Ability.DodgeRoll` because you might have multiple evasion abilities (dodge, dash, blink) that share a single cooldown. If you want independent cooldowns per evasion type, use more specific tags like `Cooldown.Evade.DodgeRoll`.

## Step 2: Create the Ability

Create a new Blueprint class with **YourProjectGameplayAbility** as the parent. Name it `GA_DodgeRoll`.

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Input Tag** | `InputTag.Movement.Dodge` | Maps to your dodge input |
| **Ability Tags** | `Ability.Movement.DodgeRoll` | Identifies this ability for queries |
| **Activation Blocked Tags** | `State.Dead`, `CrowdControl.Hard`, `State.Evading` | Can't roll while dead, stunned, or already rolling |
| **Activation Owned Tags** | `State.Evading`, `State.Invulnerable` | Granted on activate, removed on end |
| **Block Abilities With Tag** | `Ability.Combat` | Can't attack mid-roll |
| **Instancing Policy** | `InstancedPerActor` | Required when using Ability Tasks |
| **Net Execution Policy** | `LocalPredicted` | Feels responsive on the client |
| **Cost Gameplay Effect Class** | `GE_Cost_DodgeRoll` | Stamina check and deduction |
| **Cooldown Gameplay Effect Class** | `GE_Cooldown_DodgeRoll` | 0.5s re-activation delay |

!!! info "How Activation Owned Tags create i-frames"
    **Activation Owned Tags** are the core mechanic here. When the ability activates, the ASC automatically grants `State.Evading` and `State.Invulnerable` to the owning actor. When the ability ends (for any reason -- completion, interruption, cancellation), those tags are automatically removed. No manual tag management required.

    `State.Invulnerable` is what your damage pipeline checks. `State.Evading` serves double duty: it blocks re-activation (via Activation Blocked Tags) and can be queried by other systems (animation, UI, AI perception).

!!! warning "State.Evading in Activation Blocked Tags"
    Notice `State.Evading` appears in both Activation Owned Tags *and* Activation Blocked Tags. This is intentional -- it prevents the player from re-activating the dodge while already dodging. Since Activation Owned Tags are granted before activation checks on subsequent attempts, the second activation sees `State.Evading` and fails. This is the standard "can't re-use while active" pattern.

### Event Graph

=== "Blueprint"

    !!! abstract "Event Graph"

        1. **ActivateAbility** fires -- at this point, `State.Evading` and `State.Invulnerable` are already granted by Activation Owned Tags
        2. **CommitAbility** checks cost and cooldown, deducts stamina, applies cooldown. If it fails (not enough stamina, still on cooldown), the ability ends immediately and the tags are removed
        3. **PlayMontageAndWait** plays `AM_DodgeRoll` with root motion. The character moves through space based on the animation
        4. When the montage completes or is interrupted/cancelled: **EndAbility**. Activation Owned Tags (`State.Evading`, `State.Invulnerable`) are automatically removed

    ```mermaid
    flowchart LR
        A["ActivateAbility"]:::event --> B["CommitAbility"]:::func
        B -->|Failed| C["EndAbility\n(cancelled)"]:::endpoint
        B -->|Success| D["PlayMontageAndWait\nAM_DodgeRoll"]:::task
        D -->|Completed / BlendOut| E["EndAbility"]:::endpoint
        D -->|Interrupted / Cancelled| E

        classDef event fill:#5c1a1a,stroke:#ff6666,color:#fff
        classDef func fill:#2a2a4a,stroke:#9b89f5,color:#fff
        classDef task fill:#1a3a5c,stroke:#4a9eff,color:#fff
        classDef endpoint fill:#1a4a2d,stroke:#6bcb3a,color:#fff
    ```

=== "C++"
    ```cpp
    void UGA_DodgeRoll::ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData)
    {
        if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }

        UAbilityTask_PlayMontageAndWait* MontageTask =
            UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
                this,
                NAME_None,
                DodgeMontage,  // UPROPERTY: TObjectPtr<UAnimMontage>
                1.0f);

        MontageTask->OnCompleted.AddDynamic(
            this, &UGA_DodgeRoll::OnMontageFinished);
        MontageTask->OnBlendOut.AddDynamic(
            this, &UGA_DodgeRoll::OnMontageFinished);
        MontageTask->OnInterrupted.AddDynamic(
            this, &UGA_DodgeRoll::OnMontageCancelled);
        MontageTask->OnCancelled.AddDynamic(
            this, &UGA_DodgeRoll::OnMontageCancelled);
        MontageTask->ReadyForActivation();
    }

    void UGA_DodgeRoll::OnMontageFinished()
    {
        EndAbility(
            CurrentSpecHandle,
            CurrentActorInfo,
            CurrentActivationInfo,
            true, false);
    }

    void UGA_DodgeRoll::OnMontageCancelled()
    {
        EndAbility(
            CurrentSpecHandle,
            CurrentActorInfo,
            CurrentActivationInfo,
            true, true);
    }
    ```

    The C++ version is straightforward -- the dodge roll is a simple montage-driven ability with no gameplay events or damage application. All the interesting behavior comes from the Class Defaults (Activation Owned Tags, Blocked Tags, Block Abilities With Tag).

!!! danger "Always End Ability"
    Every code path must call **End Ability**. This is especially critical for the dodge roll because Activation Owned Tags (`State.Invulnerable`) persist as long as the ability is active. A leaked dodge roll ability means permanent invulnerability.

### I-Frames: How State.Invulnerable Works in the Damage Pipeline

The `State.Invulnerable` tag doesn't do anything on its own -- your damage pipeline needs to check for it. There are two approaches:

=== "PostGameplayEffectExecute Check"
    The simplest approach. In your AttributeSet's `PostGameplayEffectExecute`, check for the tag before applying damage:

    ```cpp
    void UYourAttributeSet::PostGameplayEffectExecute(
        const FGameplayEffectModCallbackData& Data)
    {
        // ... other processing ...

        if (Data.EvaluatedData.Attribute ==
            GetPendingDamageAttribute())
        {
            UAbilitySystemComponent* TargetASC =
                Data.Target.AbilityActorInfo->AbilitySystemComponent.Get();

            if (TargetASC && TargetASC->HasMatchingGameplayTag(
                FGameplayTag::RequestGameplayTag(
                    FName("State.Invulnerable"))))
            {
                // Target is invulnerable -- zero out the damage
                SetPendingDamage(0.f);
                return;
            }

            // Normal damage processing continues...
            const float FinalDamage = GetPendingDamage();
            SetPendingDamage(0.f);
            SetHealth(FMath::Clamp(
                GetHealth() - FinalDamage, 0.f, GetMaxHealth()));
        }
    }
    ```

=== "Immunity GE Component"
    A cleaner approach for production. Create an infinite-duration Gameplay Effect with an `ImmunityGameplayEffectComponent` that blocks damage effects. The dodge ability applies this GE on activation and removes it on end. This is more setup but keeps your damage pipeline free of hardcoded tag checks. See [Immunity](../gameplay-effects/immunity.md) for details.

!!! tip "Which approach to use?"
    The `PostGameplayEffectExecute` check is simpler and works well for prototyping and small projects. The Immunity GE Component approach is more scalable -- it lets you define immunity rules in data (which tags block which effects) without modifying C++ code. Most shipped games use some form of the immunity system.

## Step 3: Wire Input

### 1. InputAction Asset

Create `IA_Dodge` (right-click > **Input > Input Action**). Set the Value Type to `Digital (Bool)`.

### 2. InputMappingContext

In your `IMC_Default`, add:

- **Input Action:** `IA_Dodge`
- **Key:** Space bar, B button, or your preferred dodge key

### 3. Route Input to the ASC

=== "Blueprint"
    In your Character Blueprint's Event Graph:

    1. Add an **Enhanced Input Action** event node for `IA_Dodge`
    2. From the exec pin, call **Get Ability System Component** on Self
    3. Iterate activatable abilities and try to activate any whose Input Tag matches `InputTag.Movement.Dodge`

=== "C++"
    ```cpp
    // Same pattern as the melee attack input binding,
    // but matching InputTag.Movement.Dodge instead
    EnhancedInput->BindAction(
        DodgeAction,  // UPROPERTY: TObjectPtr<UInputAction>
        ETriggerEvent::Started,
        this, &AYourCharacter::OnDodgeInput);
    ```

See [Input Binding](../gameplay-abilities/input-binding.md) for the full production input routing setup.

### 4. Grant the Ability

Add `GA_DodgeRoll` to your character's **Startup Abilities** array in Class Defaults.

## Step 4: Test

### Basic Test

1. Hit Play
2. Press your dodge key -- character should play the roll animation and move via root motion
3. Stamina should decrease by 20
4. Press dodge again immediately -- should not activate (0.5s cooldown)
5. Wait 0.5 seconds -- dodge should work again

### I-Frame Test

1. Set up a damage source (enemy attack, damage volume, or console command)
2. Time the damage to hit during the roll
3. Verify no damage is taken during the roll (`State.Invulnerable` is present)
4. Verify damage works normally before and after the roll

### ShowDebug Checklist

Open the console and type `showdebug abilitysystem`:

| Scenario | Expected Result |
|:---|:---|
| Press dodge with 20+ Stamina | Montage plays, Stamina drops by 20, `State.Evading` + `State.Invulnerable` appear in tags, `Cooldown.Evade` appears |
| Press dodge with < 20 Stamina | Nothing happens -- Commit Ability fails cost check |
| Press dodge during cooldown | Nothing happens -- cooldown tag blocks |
| Press dodge while already rolling | Nothing happens -- `State.Evading` in Activation Blocked Tags |
| Press attack during roll | Nothing happens -- `Ability.Combat` blocked by Block Abilities With Tag |
| Damage hits during roll | No health change -- `State.Invulnerable` present |
| Roll ends | `State.Evading` and `State.Invulnerable` disappear from tags |
| Press dodge while stunned (`CrowdControl.Hard`) | Nothing happens -- blocked by Activation Blocked Tags |

### Edge Cases

- **Dodge at exactly 20 stamina** -- should succeed (cost check is `>=`, not `>`)
- **Interrupted by a force (knockback)** -- montage interrupted, ability ends, tags removed, character is no longer invulnerable
- **Network: dodge on client** -- prediction should make the roll feel immediate. If the server rejects (out of stamina due to a race), the client rolls back

!!! warning "Nothing happening?"
    Common issues:

    1. **No root motion movement** -- make sure `bEnableRootMotion` is true on the montage and your Character Movement Component is configured to accept root motion
    2. **Permanent invulnerability** -- End Ability is not being called on all paths. Check that every montage delegate leads to End Ability
    3. **Can still attack during roll** -- verify `Block Abilities With Tag` includes `Ability.Combat` and that your combat abilities have `Ability.Combat` in their Ability Tags
    4. **Stamina not deducting** -- make sure `GE_Cost_DodgeRoll` is assigned to the Cost Gameplay Effect Class property

!!! tip "Connecting to UI"
    Show a cooldown sweep on the dodge icon, or flash the stamina bar when cost is deducted. See [Connecting GAS to UI](../patterns/gas-to-ui.md) for reactive UI patterns that listen to GAS events instead of polling.

## The Full Flow

Here's what happens from button press to roll completion:

1. **Input** fires `IA_Dodge`
2. Input handler finds abilities with `InputTag.Movement.Dodge` and calls `TryActivateAbility`
3. The ASC checks activation requirements:
    - Blocked tags: Does the owner have `State.Dead`, `CrowdControl.Hard`, or `State.Evading`?
4. If checks pass, the ability **activates**:
    - **Activation Owned Tags** are granted immediately: `State.Evading`, `State.Invulnerable`
    - `ActivateAbility` fires
5. `CommitAbility` checks cost and cooldown:
    - Cost: Do we have 20+ Stamina? If yes, apply `GE_Cost_DodgeRoll` (Stamina -20)
    - Cooldown: Apply `GE_Cooldown_DodgeRoll` (tag granted for 0.5 seconds)
    - If commit fails, End Ability is called immediately (tags are removed)
6. **Play Montage and Wait** starts `AM_DodgeRoll` with root motion. The character physically moves through space
7. During the roll, `State.Invulnerable` is on the ASC. Any incoming damage that checks this tag is zeroed out
8. During the roll, `Block Abilities With Tag: Ability.Combat` prevents combat abilities from activating
9. The montage completes. **End Ability** is called
10. Activation Owned Tags (`State.Evading`, `State.Invulnerable`) are automatically removed
11. After 0.5 seconds, the cooldown effect expires and `Cooldown.Evade` tag is removed. The dodge is available again

## Variations

### Directional Dodge

Instead of always rolling forward, use the player's movement input to determine roll direction. Before playing the montage, read the character's last movement input vector and select a directional montage section (`Forward`, `Back`, `Left`, `Right`). Pass the section name to `PlayMontageAndWait` via the `StartSection` parameter.

### Dash (No Animation)

Replace the montage with a `LaunchCharacter` or direct velocity override. Use a short `WaitDelay` task (0.3 seconds) to control the i-frame duration. This is faster to set up and works well for top-down games where you don't need a roll animation.

### Dodge with Stamina Regeneration Pause

Add a second Gameplay Effect -- `GE_StaminaRegenPause` -- with a duration of 1.5 seconds that applies a modifier blocking stamina regeneration. Apply it alongside the cost in `ActivateAbility` (after CommitAbility succeeds). This prevents the "dodge spam + regen" loop that trivializes stamina management.

## Related Pages

- [Melee Attack](melee-attack.md) -- montage-driven ability with damage application
- [Ranged Attack](ranged-attack.md) -- projectile spawning, passing GE specs to actors
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) -- PlayMontageAndWait details
- [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) -- cost/cooldown deep dive
- [Immunity](../gameplay-effects/immunity.md) -- the GE Component approach to i-frames
- [Damage Pipeline](../patterns/damage-pipeline.md) -- where invulnerability is checked
- [Tag Architecture](../patterns/tag-design.md) -- designing State.* and CrowdControl.* tags
