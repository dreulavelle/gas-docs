---
title: Melee Attack
description: Complete melee attack ability with montage-driven hit timing, SetByCaller damage, AnimNotify integration, stamina cost, and cooldown.
---

# Example: Melee Attack

<div class="example-badges" markdown>
  <span class="badge badge--intermediate">Intermediate</span>
</div>

## Overview

A melee attack ability that demonstrates the full GAS pipeline working together. The character swings a weapon, pays a stamina cost, respects a cooldown, and deals damage at a precise point in the attack animation using an AnimNotify and `WaitGameplayEvent`. This covers montage-driven timing, SetByCaller damage, the PendingDamage meta attribute pattern, and proper ability lifecycle management.

## What We're Building

- **Stamina cost** of 15 per swing
- **Cooldown** of 1 second between attacks
- **Montage-driven hit timing** -- damage fires at a specific animation frame, not on button press
- **SetByCaller damage** -- the damage amount is data-driven, not hardcoded in the effect
- **PendingDamage meta attribute** -- damage flows through a processing pipeline before reaching Health
- **AnimNotify integration** -- the montage tells the ability when the weapon connects
- **Tag-based blocking** -- can't attack while dead or hard-stunned

## Prerequisites

!!! tip "What you need before starting"
    This example assumes you have completed [Project Setup](../getting-started/project-setup.md) and have:

    - A character with an **Ability System Component**
    - An **AttributeSet** with Health, Stamina, and PendingDamage attributes
    - A **base ability class** (`YourProjectGameplayAbility`) with an `InputTag` property
    - An **input binding system** that routes Enhanced Input actions to abilities by tag
    - An **attack animation montage** (`AM_MeleeAttack`) set up for your character's skeleton

    If any of that is missing, start with [Project Setup](../getting-started/project-setup.md).

## Step 1: Create the Effects

We create the effects *first*, before the ability. This is intentional -- effects are the data that defines what an ability actually *does*. The ability is just the orchestrator that decides *when* and *how* to apply them. Think of effects as the nouns and the ability as the verb.

All three effects are created in the Unreal Editor. Right-click in the Content Browser and select **Blueprint Class > GameplayEffect** as the parent class.

### GE_Cost_MeleeAttack

This effect represents the stamina cost. GAS checks costs automatically before an ability activates -- if the character can't afford it, the ability simply won't fire. No manual checks needed.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] -- Attribute** | `YourProjectAttributeSet.Stamina` |
| **Modifiers[0] -- Modifier Op** | Add |
| **Modifiers[0] -- Magnitude** | Scalable Float: `-15.0` |

!!! tip "Why negative?"
    Costs are applied as instant effects that *add* a negative value. GAS doesn't have a "subtract" operation -- you add negative numbers. It feels odd at first, but it's consistent: every attribute modification is an additive operation with a signed value.

### GE_Cooldown_MeleeAttack

The cooldown effect uses GAS's built-in cooldown system. When an ability activates, it applies this effect. While the effect is active, the ability checks for the cooldown tag and blocks re-activation.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `1.0` (1 second) |
| **Granted Tags** | `Cooldown.Ability.BasicAttack` (via TargetTagsGameplayEffectComponent) |

The magic is in the tag. Your ability will be configured to check for `Cooldown.Ability.BasicAttack` -- while this effect is active and that tag is present on the ASC, the ability can't activate. When the 1-second duration expires, the effect is removed, the tag is removed, and the ability is available again. No timers, no manual cleanup.

!!! info "Tag hierarchy for cooldowns"
    We use `Cooldown.Ability.BasicAttack` rather than a flat name. This hierarchy matters -- if you later want a "reset all cooldowns" ability, you can check for any tag under `Cooldown` and remove them all. Plan your [tag architecture](../patterns/tag-design.md) early.

### GE_Damage_Melee

This is the damage effect. Instead of hardcoding a damage number, we use **SetByCaller** -- the ability sets the damage value at runtime when it creates the effect spec. This means a single damage effect class can be reused with different damage values, and those values can come from attributes, data tables, or calculations.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] -- Attribute** | `YourProjectAttributeSet.PendingDamage` |
| **Modifiers[0] -- Modifier Op** | Add |
| **Modifiers[0] -- Magnitude Type** | Set By Caller |
| **Modifiers[0] -- Set By Caller Tag** | `SetByCaller.Damage` |

Notice we target `PendingDamage`, not `Health`. This is the meta attribute pattern from [Project Setup](../getting-started/project-setup.md) -- damage flows into `PendingDamage`, gets processed in `PostGameplayEffectExecute` (armor, shields, etc.), and the final result is applied to Health. This gives you a single processing pipeline for all damage sources.

??? question "Why SetByCaller instead of a fixed value?"
    You *could* hardcode `25.0` as the damage. It would work. But the moment you want a second melee ability with different damage, or want damage to scale with a stat, you'd need a separate effect class. SetByCaller keeps the effect generic -- the ability (or an Execution Calculation) sets the actual number. One effect class, many damage values. See [SetByCaller](../gameplay-effects/set-by-caller.md) for the full picture.

## Step 2: Create the Ability

Create a new Blueprint class in the Content Browser. The parent class should be your **YourProjectGameplayAbility** (the base ability class from [Project Setup](../getting-started/project-setup.md)). Name it `GA_MeleeAttack`.

### Class Defaults

Open the Blueprint and set these in the **Class Defaults** panel:

| Property | Value | Why |
|:---|:---|:---|
| **Input Tag** | `InputTag.Combat.Primary` | Maps this ability to your primary attack input |
| **Ability Tags** | `Ability.Combat.MeleeAttack` | Identifies this ability for queries and blocking |
| **Activation Blocked Tags** | `State.Dead`, `CrowdControl.Hard` | Can't attack while dead or hard-stunned |
| **Cancel Abilities With Tag** | *(leave empty for now)* | Could cancel other abilities on activation |
| **Instancing Policy** | `InstancedPerActor` | Required when using Ability Tasks |
| **Net Execution Policy** | `LocalPredicted` | Feels responsive on the client |
| **Cost Gameplay Effect Class** | `GE_Cost_MeleeAttack` | GAS checks this automatically before activation |
| **Cooldown Gameplay Effect Class** | `GE_Cooldown_MeleeAttack` | GAS applies this automatically on activation |

!!! note "Tags you need to create"
    If these tags don't exist in your project yet, you'll need to create them. Go to **Project Settings > Gameplay Tags** or add them in a `GameplayTags.ini` file. The tag names shown here follow the [naming conventions](../patterns/naming-conventions.md) used throughout this guide.

### Event Graph

Here's the ability logic. This runs when the ability activates (after cost and cooldown checks pass):

=== "Blueprint"

    !!! abstract "Event Graph"

        1. **ActivateAbility** fires when GAS activates the ability
        2. Two [Ability Tasks](../gameplay-abilities/ability-tasks.md) run **concurrently**:

            - **PlayMontageAndWait** plays `AM_MeleeAttack` and gives you delegates for completion, interruption, and cancellation
            - **WaitGameplayEvent** listens for `Event.Montage.MeleeHit` -- sent by an **AnimNotify** at the exact frame the weapon connects

        3. When the hit event fires: **MakeOutgoingGESpec** creates a spec from `GE_Damage_Melee`
        4. **AssignSetByCallerMagnitude** sets `SetByCaller.Damage` to `25.0` (the damage number lives here)
        5. **ApplyGESpecToTarget** applies the configured damage spec to the hit actor (target comes from the event payload)
        6. When the montage completes or is interrupted/cancelled: **EndAbility**

    ```mermaid
    flowchart LR
        A["ActivateAbility"]:::event --> B["PlayMontageAndWait\nAM_MeleeAttack"]:::task
        A --> C["WaitGameplayEvent\nEvent.Montage.MeleeHit"]:::task
        C -->|Hit Event| D["MakeOutgoingGESpec\nGE_Damage_Melee"]:::func
        D --> E["SetByCaller\nDamage = 25"]:::func
        E --> F["ApplyGESpec\nto Target"]:::func
        B -->|Completed / BlendOut| G["EndAbility"]:::endpoint
        B -->|Interrupted / Cancelled| G

        classDef event fill:#5c1a1a,stroke:#ff6666,color:#fff
        classDef func fill:#2a2a4a,stroke:#9b89f5,color:#fff
        classDef task fill:#1a3a5c,stroke:#4a9eff,color:#fff
        classDef endpoint fill:#1a4a2d,stroke:#6bcb3a,color:#fff
    ```

=== "C++"
    ```cpp
    void UGA_MeleeAttack::ActivateAbility(
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

        // --- Play the attack montage ---
        UAbilityTask_PlayMontageAndWait* MontageTask =
            UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
                this,
                NAME_None,
                AttackMontage,  // UPROPERTY: TObjectPtr<UAnimMontage>
                1.0f);

        MontageTask->OnCompleted.AddDynamic(
            this, &UGA_MeleeAttack::OnMontageCompleted);
        MontageTask->OnBlendOut.AddDynamic(
            this, &UGA_MeleeAttack::OnMontageCompleted);
        MontageTask->OnInterrupted.AddDynamic(
            this, &UGA_MeleeAttack::OnMontageCancelled);
        MontageTask->OnCancelled.AddDynamic(
            this, &UGA_MeleeAttack::OnMontageCancelled);
        MontageTask->ReadyForActivation();

        // --- Wait for the AnimNotify hit event ---
        UAbilityTask_WaitGameplayEvent* EventTask =
            UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
                this,
                FGameplayTag::RequestGameplayTag(
                    FName("Event.Montage.MeleeHit")),
                nullptr,    // no external target
                false);     // keep listening (for combo hits)

        EventTask->EventReceived.AddDynamic(
            this, &UGA_MeleeAttack::OnHitEventReceived);
        EventTask->ReadyForActivation();
    }

    void UGA_MeleeAttack::OnHitEventReceived(FGameplayEventData Payload)
    {
        // The payload's Target is populated by hit detection
        // (trace/overlap in the AnimNotify or weapon actor)
        if (!Payload.Target)
        {
            return;
        }

        UAbilitySystemComponent* TargetASC =
            UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(
                const_cast<AActor*>(Payload.Target.Get()));

        if (!TargetASC)
        {
            return;
        }

        // Create the damage spec with caster context
        FGameplayEffectSpecHandle DamageSpec =
            MakeOutgoingGameplayEffectSpec(
                DamageEffectClass,  // UPROPERTY: TSubclassOf<UGameplayEffect>
                GetAbilityLevel());

        DamageSpec.Data->SetSetByCallerMagnitude(
            FGameplayTag::RequestGameplayTag(
                FName("SetByCaller.Damage")),
            DamageAmount);  // UPROPERTY: float, default 25.0

        TargetASC->ApplyGameplayEffectSpecToSelf(
            *DamageSpec.Data.Get());
    }

    void UGA_MeleeAttack::OnMontageCompleted()
    {
        EndAbility(
            CurrentSpecHandle,
            CurrentActorInfo,
            CurrentActivationInfo,
            true, false);
    }

    void UGA_MeleeAttack::OnMontageCancelled()
    {
        EndAbility(
            CurrentSpecHandle,
            CurrentActorInfo,
            CurrentActivationInfo,
            true, true);
    }
    ```

    **Key points:**

    - Both tasks (`PlayMontageAndWait` and `WaitGameplayEvent`) run concurrently -- the montage plays while the event listener waits
    - `ReadyForActivation()` must be called on each task to start it
    - `OnHitEventReceived` can fire multiple times if `OnlyTriggerOnce` is false -- useful for combo attacks with multiple hit frames
    - The `DamageEffectClass` and `DamageAmount` are UPROPERTYs on the ability, editable in the Blueprint Class Defaults

!!! danger "Always End Ability"
    Every code path must call **End Ability**. If you forget, the ability stays "active" forever -- blocking re-activation, holding its slot, and leaking resources. This is the most common ability bug. Connect End Ability to every completion delegate from Play Montage and Wait.

### The AnimNotify

Open your attack montage (`AM_MeleeAttack`) in the Montage Editor. At the frame where the weapon should connect with a target, add an **AnimNotify** that sends a gameplay event:

1. Right-click the Notifies track at the desired frame
2. Add Notify > **AN_SendGameplayEvent** (or your custom notify class)
3. Set the Event Tag to `Event.Montage.MeleeHit`
4. The event payload should include the target actor (populated by your hit detection logic -- a trace, overlap, etc.)

??? question "Why WaitGameplayEvent instead of a delay or timer?"
    Timing damage to a specific animation frame is critical for game feel. A delay node is fragile -- if you change the animation speed or swap montages, the timing breaks. `WaitGameplayEvent` decouples the ability from specific frame timing. The montage itself declares when the hit happens (via the AnimNotify), and the ability reacts to that event. Change the montage, change the timing -- the ability code doesn't need to change at all.

??? info "Hit detection in the AnimNotify"
    The AnimNotify is responsible for two things: running a hit trace (or checking overlaps on a weapon collision volume) and sending the gameplay event with the target in the payload. A typical approach is a custom `UAnimNotify` subclass that:

    1. Gets the owning actor's weapon component
    2. Runs a sphere trace or overlap check along the weapon's arc
    3. Calls `UAbilitySystemBlueprintLibrary::SendGameplayEventToActor()` with the owner as the target actor, the `Event.Montage.MeleeHit` tag, and `FGameplayEventData` populated with each hit target

    The ability never needs to know *how* the hit was detected -- it just reacts to the event.

## Step 3: Wire Input

You need three things to connect a keyboard/gamepad input to your ability:

### 1. InputAction Asset

Create an Input Action asset in the editor (right-click > **Input > Input Action**). Name it `IA_PrimaryAttack`. Set the Value Type to `Digital (Bool)` -- it's a press, not an axis.

### 2. InputMappingContext

Open (or create) your Input Mapping Context (`IMC_Default` or similar). Add a mapping:

- **Input Action:** `IA_PrimaryAttack`
- **Key:** Left Mouse Button (or whatever you prefer)

### 3. Route Input to the ASC

The connection between Enhanced Input and GAS happens in your character. The exact implementation depends on your input binding approach, but the core idea is: when `IA_PrimaryAttack` fires, find all granted abilities whose `InputTag` matches `InputTag.Combat.Primary` and try to activate them.

=== "Blueprint"
    In your Character Blueprint's Event Graph:

    1. Add an **Enhanced Input Action** event node for `IA_PrimaryAttack`
    2. From the exec pin, call **Get Ability System Component** on Self
    3. Call a custom function that iterates activatable abilities and tries to activate any whose Input Tag matches `InputTag.Combat.Primary`

=== "C++"
    ```cpp
    void AYourCharacter::SetupPlayerInputComponent(
        UInputComponent* PlayerInputComponent)
    {
        Super::SetupPlayerInputComponent(PlayerInputComponent);

        if (UEnhancedInputComponent* EnhancedInput =
            Cast<UEnhancedInputComponent>(PlayerInputComponent))
        {
            EnhancedInput->BindAction(
                PrimaryAttackAction,  // UPROPERTY: TObjectPtr<UInputAction>
                ETriggerEvent::Started,
                this, &AYourCharacter::OnPrimaryAttackInput);
        }
    }

    void AYourCharacter::OnPrimaryAttackInput()
    {
        if (!AbilitySystemComponent) return;

        for (FGameplayAbilitySpec& Spec :
             AbilitySystemComponent->GetActivatableAbilities())
        {
            if (const UYourProjectGameplayAbility* GA =
                Cast<UYourProjectGameplayAbility>(Spec.Ability))
            {
                if (GA->InputTag.MatchesTagExact(
                    FGameplayTag::RequestGameplayTag(
                        FName("InputTag.Combat.Primary"))))
                {
                    AbilitySystemComponent->TryActivateAbility(
                        Spec.Handle);
                }
            }
        }
    }
    ```

!!! tip "Production input systems"
    The approach above is simplified for clarity. Production projects often use a more scalable pattern -- an InputAction-to-Tag mapping table that automatically routes all inputs to abilities by tag, without per-action binding functions. See [Input Binding](../gameplay-abilities/input-binding.md) for the full architecture.

### 4. Grant the Ability

Open your Character Blueprint. In Class Defaults, find the **Startup Abilities** array and add `GA_MeleeAttack`. When the character spawns and `InitializeAbilities` runs, this ability will be granted to the ASC and ready to activate.

!!! info "Runtime granting"
    Startup arrays are for abilities the character always has. For abilities gained later (from equipment, level-ups, pickups), you call `AbilitySystemComponent->GiveAbility()` at runtime. See [Ability Sets](../gameplay-abilities/ability-sets.md) for a scalable approach.

## Step 4: Test

### Quick Smoke Test

1. Place your character Blueprint in a level
2. Hit Play
3. Press your attack button -- the character should play the attack montage
4. If you have a second character in range, they should take 25 damage

### Debugging with ShowDebug

Open the console (++grave++) and type:

```
showdebug abilitysystem
```

This overlay shows you everything happening in the ASC in real time:

- **Granted Abilities** -- you should see `GA_MeleeAttack` listed
- **Active Abilities** -- lights up while your attack is playing
- **Active Effects** -- you'll see the cooldown effect appear for 1 second after attacking
- **Attributes** -- watch Stamina drop by 15 each attack, Health drop on the target
- **Tags** -- the cooldown tag appears and disappears

### What to Verify

| Scenario | Expected Result |
|:---|:---|
| Stamina >= 15, not on cooldown | Montage plays, stamina drops by 15, cooldown tag appears |
| Stamina < 15 | Nothing happens -- ability fails cost check |
| On cooldown | Nothing happens -- ability blocked by cooldown tag |
| While stunned (`CrowdControl.Hard` present) | Nothing happens -- blocked by Activation Blocked Tags |
| Hit lands on target | Target's PendingDamage receives 25, `PostGameplayEffectExecute` processes it, Health drops |

### Edge Cases

- **Interrupted mid-swing** -- another montage or stun interrupts the attack. Ability should end cleanly, no damage applied (the AnimNotify never fires)
- **Target dies before hit frame** -- the event still fires, but the target's ASC may reject the effect. No crash, no damage
- **Spam click during cooldown** -- all attempts silently fail. No animation, no cost deduction

!!! warning "Nothing happening?"
    The most common issues:

    1. **Ability not granted** -- check that `GA_MeleeAttack` is in the StartupAbilities array
    2. **Input not firing** -- check your InputMappingContext is added to the local player's Enhanced Input subsystem
    3. **Montage doesn't play** -- make sure the montage's skeleton matches your character's skeleton
    4. **Hit doesn't register** -- check that the AnimNotify fires `Event.Montage.MeleeHit` and that your hit detection populates the event payload with a target
    5. **Damage doesn't apply** -- verify the target has an ASC and PendingDamage attribute

    See [Troubleshooting](../debugging/troubleshooting.md) for a complete debugging checklist.

!!! tip "Connecting to UI"
    Want to show cooldown remaining on an action bar? Display damage numbers when the hit lands? See [Connecting GAS to UI](../patterns/gas-to-ui.md) for best practices on driving widgets from GAS data — attribute listeners, cooldown displays, and floating combat text.

## The Full Flow

Let's trace the complete sequence from button press to health drop:

1. **Input** fires `IA_PrimaryAttack`
2. Your input handler finds abilities with `InputTag.Combat.Primary` and calls `TryActivateAbility`
3. The ASC checks: Can this ability activate? It evaluates:
    - Cost: Do we have 15+ Stamina? (checks `GE_Cost_MeleeAttack`)
    - Cooldown: Is `Cooldown.Ability.BasicAttack` tag present? (checks `GE_Cooldown_MeleeAttack`)
    - Blocked tags: Does the owner have `State.Dead` or `CrowdControl.Hard`?
4. If all checks pass, the ability **activates**:
    - Cost effect is applied (Stamina -15)
    - Cooldown effect is applied (tag granted for 1 second)
    - `ActivateAbility` fires in the Blueprint / C++
5. The ability plays a montage and waits for the `Event.Montage.MeleeHit` gameplay event
6. An AnimNotify in the montage fires at the exact hit frame, runs hit detection, and sends the event with the target in the payload
7. The ability receives the event, creates a damage GE spec, sets the damage amount via SetByCaller (25.0), and applies it to the target's ASC
8. The target's `PostGameplayEffectExecute` processes `PendingDamage` -- applies armor, shields, and other mitigation -- then subtracts the final result from Health
9. The montage completes, the ability calls End Ability, all tasks are cleaned up

Every piece of GAS was involved. The ASC managed it. Tags controlled flow. Effects modified attributes. The ability orchestrated the sequence. This is the GAS loop.

## Variations

### Heavy Attack

Same structure, but with a longer montage, higher stamina cost (30), higher damage (50), and a longer cooldown (2 seconds). You can add a `State.HeavyAttack` Activation Owned Tag that grants super armor (prevents interruption from light hits) during the swing. Use the same `GE_Damage_Melee` effect -- just set a higher SetByCaller magnitude.

### Combo Chain

Use multiple montage sections (`Swing1`, `Swing2`, `Swing3`) and track which section to play next with a combo counter on the ability instance. After each hit, start a short `WaitDelay` task -- if the player presses attack again within the window, play the next section. If the window expires, reset to `Swing1`. Each section can have its own AnimNotify with different damage values.

### AoE Sweep

Instead of a single target from the AnimNotify, run a multi-hit trace that returns all actors in the weapon's arc. Loop through the results and apply the damage spec to each target's ASC. The gameplay event payload supports `TargetData` for multi-target scenarios, or you can send multiple individual events.

## Related Pages

- [Dodge Roll](dodge-roll.md) -- stamina cost, i-frames, root motion montage
- [Ranged Attack](ranged-attack.md) -- projectile spawning, passing GE specs to actors
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) -- PlayMontageAndWait, WaitGameplayEvent details
- [SetByCaller](../gameplay-effects/set-by-caller.md) -- how the damage magnitude system works
- [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) -- cost/cooldown deep dive
- [Damage Pipeline](../patterns/damage-pipeline.md) -- how PendingDamage flows through to Health
- [Tag Architecture](../patterns/tag-design.md) -- designing your tag hierarchy
