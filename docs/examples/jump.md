---
title: Jump
description: Simple jump ability with stamina cost, cooldown, variable jump height via WaitInputRelease, and tag-based CC blocking.
---

# Example: Jump

<div class="example-badges" markdown>
  <span class="badge badge--beginner">Beginner</span>
</div>

A jump ability that costs stamina, enforces a short cooldown, and supports variable jump height -- tap for a short hop, hold for a full jump. The actual jumping is a single `Character->Jump()` call, so the focus stays on GAS concepts: costs, cooldowns, ability tasks, and tag-based blocking.

## What We're Building

- **Stamina cost** of 5 per jump
- **Short cooldown** of 0.3 seconds to prevent spam
- **Variable jump height** -- tap for short hop, hold for full jump (via `WaitInputRelease`)
- **Tag-based blocking** -- can't jump while dead, stunned, or rooted
- **EndAbility** on input release after calling `StopJumping()`

## Prerequisites

!!! tip "What you need before starting"
    This example assumes you have completed [Project Setup](../getting-started/project-setup.md) and have:

    - A character with an **Ability System Component**
    - An **AttributeSet** with at minimum a Stamina attribute
    - A **base ability class** (`YourProjectGameplayAbility`) with an InputTag property
    - An **input binding system** that routes Enhanced Input actions to abilities by tag

    If any of that is missing, start with [Project Setup](../getting-started/project-setup.md).

!!! info "Built-in alternative"
    The engine ships with `UGameplayAbility_CharacterJump` -- a minimal jump ability you can subclass or use as reference. See [Built-in Abilities](../gameplay-abilities/built-in-abilities.md) for details. This example builds it from scratch so you can see every GAS concept involved.

---

## Step 1: Create the Effects

We need two effects: a cost effect that deducts stamina and a cooldown effect that prevents spam.

### GE_Cost_Jump

An Instant effect that deducts stamina when the ability commits. This is assigned as the ability's **Cost Gameplay Effect** -- GAS applies it automatically during `CommitAbility`.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] -- Attribute** | `YourProjectAttributeSet.Stamina` |
| **Modifiers[0] -- Modifier Op** | `Add (Base)` |
| **Modifiers[0] -- Magnitude** | Scalable Float: `-5.0` |

!!! info "Why -5.0?"
    Cost effects use negative values because they're applied via the standard modifier pipeline. `Add (Base)` with `-5.0` means "subtract 5 from the current Stamina value." GAS checks `CanApplyCost` before committing -- if the character has less than 5 Stamina, `CommitAbility` fails and the jump doesn't happen. See [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) for the full breakdown of how cost checking works.

### GE_Cooldown_Jump

A short-duration effect that grants a cooldown tag. GAS checks for this tag before allowing reactivation.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `0.3` |

**Tags (via GE Components):**

| GE Component | Tag | Purpose |
|:---|:---|:---|
| **TargetTagsGameplayEffectComponent** -- Target Tags (Granted to Actor) | `Cooldown.Movement.Jump` | Blocks reactivation while present |

!!! note "How cooldowns work"
    The cooldown GE doesn't block the ability directly -- the ability's **Cooldown Gameplay Effect** property tells GAS to check for the granted tag before activation. While `Cooldown.Movement.Jump` exists on the character (for 0.3 seconds), `CheckCooldown` returns false and `CommitAbility` fails. The tag expires when the effect's duration ends.

---

## Step 2: Create the Ability Blueprint

**Asset:** `GA_Jump`

Create a new Blueprint class with your `YourProjectGameplayAbility` as the parent.

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Input Tag** | `InputTag.Movement.Jump` | Maps to your jump input action |
| **Ability Tags** | `Ability.Movement.Jump` | Identifies this ability for queries and tag matching |
| **Activation Blocked Tags** | `State.Dead`, `CrowdControl.Hard`, `CrowdControl.Soft.Root` | Can't jump while dead, stunned/knocked down, or rooted |
| **Instancing Policy** | `InstancedPerActor` | Required for `WaitInputRelease` (ability tasks need an instanced ability) |
| **Cost Gameplay Effect** | `GE_Cost_Jump` | Deducts 5 Stamina on commit |
| **Cooldown Gameplay Effect** | `GE_Cooldown_Jump` | Enforces 0.3s cooldown between jumps |

!!! tip "Why InstancedPerActor?"
    `WaitInputRelease` (and most ability tasks) require an instanced ability because they bind delegates to the ability instance. With `NonInstanced`, there's no instance to bind to and the task silently fails. If your jump doesn't respond to input release, check the instancing policy first. See [Instancing Policy](../gameplay-abilities/instancing-policy.md) for a full comparison.

!!! info "Activation Blocked Tags design"
    `CrowdControl.Hard` covers stun, knockdown, and similar full-body CC. `CrowdControl.Soft.Root` specifically prevents movement abilities while still allowing attacks and casting. This follows the tag hierarchy from [Tag Architecture](../patterns/tag-design.md). If your project uses a different CC taxonomy, adjust the blocked tags accordingly.

### Event Graph

=== "Blueprint"

    !!! abstract "Event Graph"

        1. **ActivateAbility** fires
        2. **CommitAbility** -- applies cost and cooldown. If it fails (not enough stamina, still on cooldown), branch to **EndAbility**
        3. **Get Avatar Actor** -> **Cast to Character**
        4. **Character->Jump()** -- launches the character
        5. **WaitInputRelease** -- waits for the player to release the jump key
        6. **Character->StopJumping()** -- tells the movement component to stop adding jump velocity (enables variable jump height)
        7. **EndAbility** (success)

    ```mermaid
    flowchart LR
        A["ActivateAbility"]:::event --> B["CommitAbility"]:::func
        B -->|Fail| Z["EndAbility\n(cancel)"]:::endpoint
        B -->|Success| C["Get Avatar Actor\nCast to Character"]:::func
        C --> D["Character->Jump()"]:::func
        D --> E["WaitInputRelease"]:::task
        E -->|Released| F["Character->StopJumping()"]:::func
        F --> G["EndAbility\n(success)"]:::endpoint

        classDef event fill:#5c1a1a,stroke:#ff6666,color:#fff
        classDef func fill:#2a2a4a,stroke:#9b89f5,color:#fff
        classDef task fill:#1a3a5c,stroke:#4a9eff,color:#fff
        classDef endpoint fill:#1a4a2d,stroke:#6bcb3a,color:#fff
    ```

    !!! tip "Two tasks, one outcome"
        `WaitInputRelease` runs asynchronously -- the ability stays active while the character is in the air. When the player releases the jump key, the task fires its delegate, we call `StopJumping()`, and then end the ability. This is why instancing matters: the task needs a live ability instance to call back into.

=== "C++"

    ```cpp
    // GA_Jump.h
    #pragma once

    #include "CoreMinimal.h"
    #include "YourProjectGameplayAbility.h"
    #include "GA_Jump.generated.h"

    class ACharacter;

    UCLASS()
    class UGA_Jump : public UYourProjectGameplayAbility
    {
        GENERATED_BODY()

    public:
        UGA_Jump();

        virtual void ActivateAbility(
            const FGameplayAbilitySpecHandle Handle,
            const FGameplayAbilityActorInfo* ActorInfo,
            const FGameplayAbilityActivationInfo ActivationInfo,
            const FGameplayEventData* TriggerEventData) override;

    private:
        UFUNCTION()
        void OnJumpReleased(float TimeHeld);
    };
    ```

    ```cpp
    // GA_Jump.cpp
    #include "GA_Jump.h"
    #include "GameFramework/Character.h"
    #include "AbilitySystemComponent.h"
    #include "Abilities/Tasks/AbilityTask_WaitInputRelease.h"

    UGA_Jump::UGA_Jump()
    {
        InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
        NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    }

    void UGA_Jump::ActivateAbility(
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

        ACharacter* Character = Cast<ACharacter>(GetAvatarActorFromActorInfo());
        if (!Character)
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }

        // Launch the character
        Character->Jump();

        // Wait for the player to release the jump key
        UAbilityTask_WaitInputRelease* WaitRelease =
            UAbilityTask_WaitInputRelease::WaitInputRelease(this);
        WaitRelease->OnRelease.AddDynamic(this, &UGA_Jump::OnJumpReleased);
        WaitRelease->ReadyForActivation();
    }

    void UGA_Jump::OnJumpReleased(float TimeHeld)
    {
        ACharacter* Character = Cast<ACharacter>(GetAvatarActorFromActorInfo());
        if (Character)
        {
            // Stop adding jump velocity -- enables variable jump height
            Character->StopJumping();
        }

        K2_EndAbility();
    }
    ```

    !!! note "CommitAbility does cost + cooldown"
        `CommitAbility` is a single call that checks and applies both the Cost GE and Cooldown GE. If the character doesn't have enough stamina or the cooldown tag is still active, it returns `false` and the ability bails. You don't need to apply them separately.

!!! danger "Always End Ability"
    Every code path must call `EndAbility`. If you forget to end the ability on the `CommitAbility` failure path, the ability stays "active" forever -- blocking reactivation, holding the ability slot, and potentially leaking tasks. This is the single most common GAS bug. When in doubt, add a breakpoint to `EndAbility` and verify it fires for every activation.

---

## Step 3: Wire Input

This example uses the tag-based input routing described in [Input Binding](../gameplay-abilities/input-binding.md), but the ability works with either tag-based or direct-binding approaches.

1. **Input Action:** `IA_Jump` (Value Type: `Digital (Bool)`, Trigger: `Pressed`)
2. **Input Mapping Context:** Map to your jump key (e.g., Spacebar, bottom face button on gamepad)
3. **InputTag:** `InputTag.Movement.Jump`
4. **On the ability:** Set InputTag to `InputTag.Movement.Jump`

!!! info "Pressed, not Held"
    The Input Action trigger should be **Pressed** (fires once on key down), not Held. The ability itself handles the "held" behavior through the `WaitInputRelease` task. If you use the Held trigger, the input system would keep re-firing the activation attempt, which conflicts with the cooldown and wastes `CommitAbility` checks.

---

## Step 4: Test

### Basic Jump Test

1. PIE
2. Press and release the jump key quickly -- character should do a short hop
3. Press and hold the jump key -- character should jump higher than the tap
4. Check `showdebug abilitysystem`:
    - `Cooldown.Movement.Jump` tag should appear briefly after each jump
    - Stamina should decrease by 5 per jump
    - `GE_Cooldown_Jump` should appear in active effects for 0.3 seconds
5. Spam the jump key -- cooldown should prevent jumps faster than every 0.3 seconds

### Edge Cases

| Scenario | Expected Result |
|:---|:---|
| Jump at 0 stamina | `CommitAbility` fails, ability ends immediately, no jump |
| Jump with 3 stamina (less than cost) | Same as above -- cost check fails |
| Jump while rooted | Doesn't activate (Activation Blocked Tags) |
| Jump while dead | Doesn't activate (Activation Blocked Tags) |
| Jump during cooldown | `CommitAbility` fails (cooldown tag still active) |
| Rapid tap-tap-tap | First jump fires, second blocked by cooldown, third fires after 0.3s |
| Hold jump key for 5 seconds | Character reaches max jump height, `StopJumping` fires on release |
| Jump interrupted by stun mid-air | Ability cancelled externally -- verify `EndAbility` fires |

!!! tip "Debugging with showdebug"
    `showdebug abilitysystem` is your primary diagnostic tool. It shows active effects, granted tags, and ability states in real time. If jumps aren't working, check whether the cooldown tag is stuck, whether the cost is being applied, or whether a blocking tag is present. See [ShowDebug AbilitySystem](../debugging/showdebug-abilitysystem.md) for a full walkthrough of the debug HUD.

---

## The Full Flow

Here's the complete trace from button press to landing:

```
Player presses Jump key
  |
  v
Input System: IA_Jump -> InputTag.Movement.Jump -> ASC finds GA_Jump
  |
  v
GA_Jump.ActivateAbility()
  +-- Check: Activation Blocked Tags (dead? stunned? rooted?)
  +-- CommitAbility()
  |     +-- CheckCooldown: is Cooldown.Movement.Jump tag present? (no -> pass)
  |     +-- CheckCost: Stamina >= 5? (yes -> pass)
  |     +-- Apply GE_Cost_Jump (Stamina -= 5)
  |     +-- Apply GE_Cooldown_Jump (grants Cooldown.Movement.Jump for 0.3s)
  |
  +-- Character->Jump() (launches character upward)
  +-- WaitInputRelease task starts listening
  |
  v  (character is in the air, jump velocity still being applied...)
  |
  v  Player releases Jump key
  |
  v
OnJumpReleased()
  +-- Character->StopJumping() (stops adding jump velocity)
  +-- EndAbility (success)
  |
  v  (character falls, 0.3s cooldown expires, ready to jump again)
```

---

## Variations

??? example "Double Jump"
    Track a `JumpsRemaining` counter as a member variable (requires `InstancedPerActor`). On `ActivateAbility`, increment a counter instead of ending after the first jump. Use `WaitGameplayEvent` listening for a custom `GameplayEvent.Movement.Jump` tag -- fire that event from your input handler on the second press. When the counter hits the max (e.g., 2), end the ability normally. You'll want to skip the cooldown on the second jump by calling `CommitAbilityCost` and `CommitAbilityCooldown` separately, or by using a different cost/cooldown for the second activation.

??? example "Coyote Time"
    Coyote time lets the player jump for a short window after walking off a ledge. This is typically handled outside GAS -- in the Character Movement Component, extend `IsFalling()` to return `false` for a brief grace period after leaving ground. The jump ability itself doesn't need to change; it just calls `Character->Jump()`, which respects the movement component's falling state. If you want to track coyote time via GAS, grant a `State.CoyoteTime` tag on the `OnMovementModeChanged` event and block jump only when falling *without* that tag.

??? example "Jump Pad / Forced Jump"
    To trigger a jump from a world object (jump pad, launch zone), activate the ability via `GameplayEvent` instead of input. Add an Activation Trigger with tag `GameplayEvent.Movement.Launch` and trigger type `GameplayEventTriggered`. The jump pad sends the event with magnitude data for the launch force. In the ability, read the event magnitude and pass it to `LaunchCharacter()` instead of `Jump()`. Skip the `WaitInputRelease` task since there's no player input to release.

---

## Related Pages

- [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) -- how cost checking and cooldown tags work
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) -- `WaitInputRelease` and other async ability tasks
- [Input Binding](../gameplay-abilities/input-binding.md) -- full input architecture and tag-based routing
- [Built-in Abilities](../gameplay-abilities/built-in-abilities.md) -- `UGameplayAbility_CharacterJump` and other engine-provided abilities
- [Tag Architecture](../patterns/tag-design.md) -- CC tag hierarchy and blocking tag design
- [Instancing Policy](../gameplay-abilities/instancing-policy.md) -- why ability tasks require instanced abilities
- [Troubleshooting](../debugging/troubleshooting.md) -- common GAS issues and fixes
- [Sprint Toggle](sprint.md) -- another beginner example demonstrating held input and toggle abilities
- [Dodge Roll](dodge-roll.md) -- intermediate example with stamina cost, cooldown, and i-frames
