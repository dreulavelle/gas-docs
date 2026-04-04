---
title: "Example: Jump"
icon: material/arrow-up-bold
description: Complete jump ability with stamina cost, cooldown, CC blocking, airborne state tracking, and landing detection.
---

# Example: Jump

The simplest GAS ability — and a surprisingly good demonstration of the system's value. We'll build a jump that costs stamina, has a cooldown, respects crowd control, and tracks airborne state through tags.

!!! tip "Simpler version available"
    If you just want the 10-minute version without airborne tracking, see [Your First Ability](../getting-started/your-first-ability.md) in the Getting Started section. This page builds on that with additional features.

## What We're Building

A jump ability that:

- **Costs stamina** (5 per jump)
- **Has a cooldown** (0.3 seconds — prevents spam)
- **Blocked by CC** — can't jump while stunned, rooted, or dead
- **Tracks airborne state** — grants `State.Airborne` while in the air
- **Detects landing** — ends the ability when the character touches ground

## Step 1: Create the Effects

### GE_Cost_Jump

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.Stamina` |
| **Modifiers[0] — Modifier Op** | Add |
| **Modifiers[0] — Magnitude** | Scalable Float: `-5.0` |

### GE_Cooldown_Jump

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `0.3` |
| **GrantedTags** | `Cooldown.Movement.Jump` |

No damage effect needed — jump doesn't hurt anyone (unless you're building a Mario game, in which case see [Variations](#variations)).

## Step 2: Create the Ability Blueprint

Create `GA_Jump` with **YourProjectGameplayAbility** as the parent.

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Input Tag** | `InputTag.Movement.Jump` | Maps to jump input |
| **Ability Tags** | `Ability.Movement.Jump` | Identifies this ability |
| **Activation Blocked Tags** | `State.Dead`, `CrowdControl.Hard`, `CrowdControl.Soft.Root` | Can't jump while dead, hard-CC'd, or rooted |
| **Activation Owned Tags** | `State.Airborne` | Granted while the ability is active |
| **Cost Gameplay Effect Class** | `GE_Cost_Jump` | Stamina check and deduction |
| **Cooldown Gameplay Effect Class** | `GE_Cooldown_Jump` | 0.3s re-activation delay |
| **Instancing Policy** | `InstancedPerActor` | Required because we use ability tasks |
| **Net Execution Policy** | `LocalPredicted` | Jump should feel instant on the client |

!!! info "Activation Owned Tags"
    `State.Airborne` is granted automatically when the ability activates and removed when it ends. Other systems can query this tag — your UI could show a "midair" indicator, or other abilities could check for it (e.g., an air-dash that requires `State.Airborne`).

### Event Graph

This version tracks the full airborne lifecycle:

=== "Blueprint"
    ```
    Event ActivateAbility
        │
        ▼
    Commit Ability ──► [Failed] ──► End Ability (Cancelled = true)
        │
        ▼ [Success]
    Get Avatar Actor ──► Cast to Character ──► Jump()
        │
        ▼
    Wait Movement Mode Change (NewMode = Walking)
        │
        ▼ [On Change]
    End Ability (Cancelled = false)
    ```

=== "Step-by-Step"
    1. **ActivateAbility** — GAS has confirmed tag checks pass
    2. **Commit Ability** — deducts stamina and applies cooldown. If it fails, bail
    3. **Jump()** — triggers the character's jump. `State.Airborne` is already granted via Activation Owned Tags
    4. **WaitMovementModeChange** — an ability task that listens for the character's movement mode to change to `Walking` (i.e., landing)
    5. **End Ability** — called when the character lands. This removes `State.Airborne` automatically

=== "C++"
    ```cpp
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

        ACharacter* Character = CastChecked<ACharacter>(
            ActorInfo->AvatarActor.Get());
        Character->Jump();

        // Wait for landing
        UAbilityTask_WaitMovementModeChange* WaitTask =
            UAbilityTask_WaitMovementModeChange::CreateWaitMovementModeChange(
                this, EMovementMode::MOVE_Walking);

        WaitTask->OnChange.AddDynamic(
            this, &UGA_Jump::OnLanded);
        WaitTask->ReadyForActivation();
    }

    void UGA_Jump::OnLanded(EMovementMode NewMovementMode)
    {
        EndAbility(
            CurrentSpecHandle,
            CurrentActorInfo,
            CurrentActivationInfo,
            true, false);
    }
    ```

!!! warning "Instancing Policy matters here"
    The fire-and-forget version from Getting Started can use `NonInstanced` because it doesn't use ability tasks. This version uses `WaitMovementModeChange`, which requires `InstancedPerActor` (or `InstancedPerExecution`). Ability tasks need a persistent ability instance to bind their delegates to. If you use `NonInstanced` with tasks, it will crash.

## Step 3: Wire Input

1. **Input Action:** Create `IA_Jump` (Digital/Bool)
2. **Input Mapping Context:** Map `IA_Jump` to Spacebar in `IMC_Default`
3. **Input-to-Tag mapping:** Route `IA_Jump` to `InputTag.Movement.Jump`
4. **Grant the ability:** Add `GA_Jump` to your character's Startup Abilities array

See [Input Binding](../gameplay-abilities/input-binding.md) for the full architecture.

## Step 4: Test

1. **Play** — press Spacebar, character jumps
2. **Stamina** drops by 5 (check with `showdebug abilitysystem`)
3. **Cooldown** — spam Spacebar, jumps are throttled to every 0.3s
4. **Airborne tag** — watch `State.Airborne` appear in the tags list during the jump and disappear on landing
5. **CC blocking** — give the character `CrowdControl.Hard` (via a debug GE) and confirm jump is blocked

??? example "ShowDebug checklist"
    | Scenario | Expected in ShowDebug |
    |:---|:---|
    | Idle on ground | `GA_Jump` in granted abilities, no active effects |
    | Mid-jump | `State.Airborne` in active tags, `GE_Cooldown_Jump` in active effects |
    | After landing | `State.Airborne` gone, cooldown ticking down |
    | Stamina at 0 | Spacebar does nothing, ability doesn't appear in active list |

## Comparison: UGameplayAbility_CharacterJump

The engine ships with a built-in jump ability (`UGameplayAbility_CharacterJump`). Here's how it compares to ours:

| Aspect | Built-in | Our GA_Jump |
|:---|:---|:---|
| **Instancing** | `NonInstanced` | `InstancedPerActor` |
| **Net Policy** | `LocalPredicted` | `LocalPredicted` |
| **Airborne tracking** | None | `State.Airborne` via Activation Owned Tags |
| **Landing detection** | None — cancels on input release | `WaitMovementModeChange` task |
| **Cost/Cooldown** | None configured (but supports it) | Stamina cost + 0.3s cooldown |
| **CanActivateAbility** | Calls `ACharacter::CanJump()` | Relies on GAS tag checks |

The built-in version is deliberately minimal. It cancels the ability when you *release* the jump key (via `InputReleased`), which calls `StopJumping()` — this controls variable jump height (hold longer = jump higher). Our version doesn't do this because we end on landing instead.

!!! tip "Combining approaches"
    You can get the best of both worlds: use `WaitMovementModeChange` for landing detection *and* override `InputReleased` to call `StopJumping()` for variable jump height. The ability stays active until landing either way. See [Built-in Abilities](../gameplay-abilities/built-in-abilities.md) for the full source walkthrough.

## Variations

### Double Jump

Add a counter variable to the ability. On activation, increment it. In `CanActivateAbility`, check if the counter is less than your max jump count. Reset the counter in `EndAbility` when landing. The `State.Airborne` tag lets you differentiate between first jump (from ground) and subsequent jumps (midair).

### Jump Pad

Create a gameplay event trigger. When the character overlaps a jump pad volume, send a gameplay event with tag `Event.Movement.JumpPad`. Configure a second jump ability (`GA_JumpPad`) that activates from event instead of input — set its **Activation Event** to that tag and have it call `LaunchCharacter()` instead of `Jump()`. No cost, no cooldown.

### Stomp Attack

Check for `State.Airborne` on activation. If the character is airborne, play a stomp animation, apply downward velocity, and on landing, do an area damage sweep. This combines the jump's airborne tracking with the damage pipeline from [Melee Attack](melee-attack.md).

## Related

- [Your First Ability](../getting-started/your-first-ability.md) — the simpler 10-minute version
- [Built-in Abilities](../gameplay-abilities/built-in-abilities.md) — full source walkthrough of `UGameplayAbility_CharacterJump`
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) — details on `WaitMovementModeChange` and other tasks
- [Input Binding](../gameplay-abilities/input-binding.md) — how to wire Enhanced Input to GAS
- [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) — deep dive on the cost/cooldown system
