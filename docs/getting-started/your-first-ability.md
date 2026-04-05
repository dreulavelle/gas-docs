
# Your First Ability — Jump

Time to build something. In the next ten minutes, you'll take a basic "press space to jump" and turn it into a GAS-managed ability with stamina cost, cooldown, and crowd control blocking — all through data configuration, not code changes.

Jump is the simplest ability that demonstrates why GAS exists. It's one function call (`Jump()`), which makes it perfect for seeing the GAS loop without getting lost in animation timing or damage pipelines.

!!! tip "Prerequisites"
    This page assumes you've completed [Project Setup](project-setup.md) and have a working character with an ASC, an AttributeSet (at minimum: Stamina), and a base ability class (`YourProjectGameplayAbility`).

## Before GAS: The Manual Way

Here's what jump looks like without GAS:

```
void TryJump()
{
    if (bIsStunned || bIsRooted || bIsDead) return;  // CC checks
    if (Stamina < 5.f) return;                        // resource check
    Stamina -= 5.f;                                    // cost
    GetWorldTimerManager().SetTimer(CooldownHandle,    // cooldown
        this, &AMyChar::ResetJumpCooldown, 0.3f);
    bJumpOnCooldown = true;
    Character->Jump();
}
```

Five lines of manual state management. Now imagine adding a new CC type — freeze. You'd need to find every action that freeze should block and add `|| bIsFrozen` to each one. That doesn't scale.

With GAS, you add one tag (`CrowdControl.Hard`) to the freeze effect. Every ability that blocks on `CrowdControl.Hard` — jump, attack, dodge, sprint — is automatically blocked. Zero code changes to any ability.

That's the pitch. Let's build it.

## Step 1: Create Two Effects

Effects first, ability second. Effects define *what happens*; the ability decides *when*.

Both effects are created in the editor: right-click in the Content Browser, select **Blueprint Class**, and choose **GameplayEffect** as the parent.

### GE_Cost_Jump

The stamina cost. GAS checks this automatically before the ability activates — if the character can't afford it, the ability won't fire.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.Stamina` |
| **Modifiers[0] — Modifier Op** | Add |
| **Modifiers[0] — Magnitude** | Scalable Float: `-5.0` |

!!! tip "Why negative?"
    GAS doesn't have a "subtract" operation — costs are instant effects that *add* a negative value. Feels odd at first, but it's consistent. See [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) for the full picture.

### GE_Cooldown_Jump

A short cooldown to prevent spam-jumping. While this effect is active, the cooldown tag blocks re-activation.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `0.3` (seconds) |
| **Granted Tags** | `Cooldown.Movement.Jump` |

!!! info "Where to find Granted Tags in the editor"
    In UE 5.3+, tag granting moved from a top-level GE property to a **GE Component**. In your Gameplay Effect Blueprint, look for **Components → Target Tags (Granted to Actor)** → **Add Tags** → **Add to Inherited**.

When the 0.3-second duration expires, the effect is removed, the tag disappears, and the ability is available again. No timers, no manual cleanup.

## Step 2: Create the Ability Blueprint

Create a new Blueprint class with **YourProjectGameplayAbility** as the parent. Name it `GA_Jump`.

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Input Tag** | `InputTag.Movement.Jump` | Maps this ability to your jump input |
| **Ability Tags** | `Ability.Movement.Jump` | Identifies this ability for queries and interactions |
| **Activation Blocked Tags** | `State.Dead`, `CrowdControl.Hard`, `CrowdControl.Soft.Root` | Can't jump while dead, stunned, or rooted |
| **Cost Gameplay Effect Class** | `GE_Cost_Jump` | GAS checks this automatically before activation |
| **Cooldown Gameplay Effect Class** | `GE_Cooldown_Jump` | GAS applies this automatically on commit |

!!! note "Tags you need to create"
    If these tags don't exist yet, add them in **Project Settings > Gameplay Tags** or in a `GameplayTags.ini` file. The naming follows the conventions from [Tag Architecture](../patterns/tag-design.md).

### Event Graph

The jump ability needs to handle two things: starting the jump and stopping it when the player releases the key. This gives you variable jump height (tap for short hop, hold for full jump).

=== "Blueprint"

    !!! abstract "Event Graph"

        1. **ActivateAbility** fires after GAS confirms the ability *can* activate (tag checks pass)
        2. **CommitAbility** deducts the cost and applies the cooldown. If it fails (not enough stamina, still on cooldown), **EndAbility** immediately
        3. **Get Avatar Actor from Actor Info** -- returns the pawn/character this ability is running on
        4. **Cast to Character** -- needed to access `Jump()` and `StopJumping()`. If cast fails, **EndAbility**
        5. **Jump()** -- starts the jump. The character's movement component handles the physics from here
        6. **WaitInputRelease** -- an [Ability Task](../gameplay-abilities/ability-tasks.md) that pauses until the player releases the jump key
        7. **StopJumping()** on the character -- tells the movement component to stop applying jump force (enables variable jump height)
        8. **EndAbility** -- clean up

    ```mermaid
    flowchart LR
        A["ActivateAbility"]:::event --> B["CommitAbility"]:::func
        B -->|Failed| C["EndAbility"]:::endpoint
        B -->|Success| D["Get Avatar Actor\nCast to Character"]:::func
        D -->|Cast Failed| C
        D -->|Success| E["Jump()"]:::func
        E --> F["WaitInputRelease"]:::task
        F --> G["StopJumping()"]:::func
        G --> H["EndAbility"]:::endpoint

        classDef event fill:#5c1a1a,stroke:#ff6666,color:#fff
        classDef func fill:#2a2a4a,stroke:#9b89f5,color:#fff
        classDef task fill:#1a3a5c,stroke:#4a9eff,color:#fff
        classDef endpoint fill:#1a4a2d,stroke:#6bcb3a,color:#fff
    ```

=== "C++"
    This follows the same pattern as Epic's built-in `UGameplayAbility_CharacterJump`. Note: because we use the `WaitInputRelease` ability task, this requires `InstancedPerActor` instancing. Epic's built-in version uses `NonInstanced` and handles input release differently (via the `InputReleased` override) — see [Built-in Abilities](../gameplay-abilities/built-in-abilities.md) for that approach.

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

        ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
        if (!Character)
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }

        Character->Jump();

        // Wait for input release to call StopJumping
        UAbilityTask_WaitInputRelease* WaitRelease =
            UAbilityTask_WaitInputRelease::WaitInputRelease(this);
        WaitRelease->OnRelease.AddDynamic(this, &UGA_Jump::OnJumpReleased);
        WaitRelease->ReadyForActivation();
    }

    void UGA_Jump::OnJumpReleased(float TimeHeld)
    {
        ACharacter* Character = Cast<ACharacter>(
            GetAvatarActorFromActorInfo());
        if (Character)
        {
            Character->StopJumping();
        }

        K2_EndAbility();
    }
    ```

!!! tip "Why StopJumping?"
    `ACharacter::Jump()` starts applying upward velocity. `StopJumping()` tells the movement component to stop — this is what gives you **variable jump height** (tap for a short hop, hold for a full jump). Without it, every jump is the same height regardless of how long you hold the button. This is the same pattern Epic's built-in `UGameplayAbility_CharacterJump` uses.

!!! info "The Cast to Character"
    `GetAvatarActorFromActorInfo()` returns the actor the ability is running on, but as an `AActor*`. We need `ACharacter*` to call `Jump()` and `StopJumping()`, so the cast is required. If your character base class has custom jump logic, cast to that class instead.

!!! danger "Always End Ability"
    Every code path must call **End Ability**. If you forget, the ability stays "active" forever — blocking re-activation, holding its slot, and leaking resources. This is the single most common GAS bug.

## Step 3: Wire Input

Three pieces connect a key press to your ability:

### 1. Input Action

Create an Input Action asset: right-click > **Input > Input Action**. Name it `IA_Jump`. Set the Value Type to **Digital (Bool)**.

### 2. Input Mapping Context

Open your `IMC_Default` (or create one). Add a mapping:

- **Input Action:** `IA_Jump`
- **Key:** Spacebar

### 3. Route Input to the ASC

In your input binding setup, add an entry that maps `IA_Jump` to the tag `InputTag.Movement.Jump`. When the input fires, your system finds all granted abilities whose InputTag matches and calls `TryActivateAbility`.

!!! tip "Production input systems"
    The exact wiring depends on your input binding approach. See [Input Binding](../gameplay-abilities/input-binding.md) for the full architecture — most production projects use a data-driven InputAction-to-Tag mapping table rather than per-action handler functions.

### 4. Grant the Ability

Open your Character Blueprint. In Class Defaults, find the **Startup Abilities** array and add `GA_Jump`.

When the character spawns and `InitializeAbilities` runs (from [Project Setup](project-setup.md)), the ability is granted to the ASC and ready to go.

## Step 4: Test

1. Hit **Play**
2. Press **Spacebar** — the character should jump
3. Watch Stamina drop by 5 in the debug overlay
4. Spam Spacebar — you can only jump every 0.3 seconds

### Debugging with ShowDebug

Open the console (++grave++) and type:

```
showdebug abilitysystem
```

You should see:

| What to Check | Expected |
|:---|:---|
| **Granted Abilities** | `GA_Jump` listed |
| **Active Effects** (after jumping) | `GE_Cooldown_Jump` appears for 0.3s |
| **Attributes** | Stamina decreases by 5 per jump |
| **Tags** | `Cooldown.Movement.Jump` appears and disappears |

??? example "Edge case testing"
    | Scenario | Expected Result |
    |:---|:---|
    | Stamina >= 5, not on cooldown | Jump fires, stamina drops by 5 |
    | Stamina < 5 | Nothing happens — cost check fails |
    | On cooldown (< 0.3s since last jump) | Nothing happens — cooldown tag blocks |
    | Character has `CrowdControl.Hard` | Nothing happens — blocked by Activation Blocked Tags |
    | Character has `CrowdControl.Soft.Root` | Nothing happens — rooted characters can't jump |

!!! warning "Nothing happening?"
    Common issues:

    1. **Ability not granted** — check `GA_Jump` is in the StartupAbilities array
    2. **Input not firing** — verify your IMC is added to the Enhanced Input subsystem on the local player
    3. **Cost check fails immediately** — make sure your Stamina attribute is initialized to a value >= 5
    4. **Tags not matching** — double-check the tag names are identical between the effect and the ability's blocked tags

    See [Troubleshooting](../debugging/troubleshooting.md) for the full debugging checklist.

## The GAS Advantage

Here's the payoff. Say your designer adds a new CC type: **Freeze**. Freeze should prevent all movement — jumping, dodging, sprinting.

Without GAS, you'd search every movement function and add `if (bIsFrozen) return;`. Miss one, and frozen characters can still dodge.

With GAS, the freeze effect grants the tag `CrowdControl.Hard`. Every ability that has `CrowdControl.Hard` in its Activation Blocked Tags — jump, dodge, sprint — is automatically blocked. You add the tag to *one* effect. Zero changes to any ability. That's the entire point of the system.

!!! info "Built-in jump ability"
    The engine ships with `UGameplayAbility_CharacterJump` — a minimal C++ ability that wraps `ACharacter::Jump()` into GAS. It's `NonInstanced`, `LocalPredicted`, and calls `CommitAbility` + `Jump()` in `ActivateAbility`. It's a useful reference, but most projects create their own jump ability for more control. See [Built-in Abilities](../gameplay-abilities/built-in-abilities.md) for a detailed walkthrough.

## What's Next

You've built a working GAS ability. But should *everything* be an ability? Jumping makes sense — but what about opening a door? Crouching? Swimming?

Head to [Should It Be an Ability?](should-it-be-an-ability.md) for the decision framework you'll use on every feature going forward.

After that, check out the [Examples](../examples/index.md) section for more complex abilities — [melee attacks](../examples/melee-attack.md) with animation-driven hit timing, [dodge rolls](../examples/dodge-roll.md) with i-frames, and [ranged attacks](../examples/ranged-attack.md) with projectile spawning.
