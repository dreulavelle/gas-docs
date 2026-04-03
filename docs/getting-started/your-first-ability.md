---
icon: material/sword
---

# Your First Ability

Time to build something real. We're going to create a complete melee attack ability that touches every piece of GAS — effects, tags, abilities, input, montages, and the damage pipeline. The goal isn't to build a production-ready combat system (we'll get there in [Patterns](../patterns/index.md)); it's to see the full loop end-to-end so you understand how the pieces connect before deep-diving into any one of them.

## What We're Building

A basic melee attack that:

- **Costs stamina** (15 per swing)
- **Has a cooldown** (1 second)
- **Plays an animation** (attack montage)
- **Deals damage** at a specific point in the animation (not on button press)
- **Uses SetByCaller** for the damage amount (data-driven, not hardcoded)

By the end of this page, you'll press a button, watch your character swing, see a hit register at the right moment in the animation, and watch the target's health drop.

## Step 1: Create the Gameplay Effects

We create the effects *first*, before the ability. This is intentional — effects are the data that defines what an ability actually *does*. The ability is just the orchestrator that decides *when* and *how* to apply them. Think of effects as the nouns and the ability as the verb.

All three effects are created in the Unreal Editor. Right-click in the Content Browser and go to **Blueprint Class**, then select **GameplayEffect** as the parent class. You can also right-click and search "Gameplay Effect" directly.

### GE_Cost_MeleeAttack

This effect represents the stamina cost. GAS checks costs automatically before an ability activates — if the character can't afford it, the ability simply won't fire. No manual checks needed.

| Setting | Value |
|---|---|
| **Duration Policy** | Instant |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.Stamina` |
| **Modifiers[0] — Modifier Op** | Add |
| **Modifiers[0] — Magnitude** | Scalable Float: `-15.0` |

!!! tip "Why negative?"
    Costs are applied as instant effects that *add* a negative value. GAS doesn't have a "subtract" operation — you add negative numbers. It feels odd at first, but it's consistent: every attribute modification is an additive operation with a signed value.

### GE_Cooldown_MeleeAttack

The cooldown effect uses GAS's built-in cooldown system. When an ability activates, it applies this effect. While the effect is active, the ability checks for the cooldown tag and blocks re-activation.

| Setting | Value |
|---|---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `1.0` (1 second) |
| **GrantedTags** | `Cooldown.Ability.BasicAttack` |

The magic is in the tag. Your ability will be configured to check for `Cooldown.Ability.BasicAttack` — while this effect is active and that tag is present on the ASC, the ability can't activate. When the 1-second duration expires, the effect is removed, the tag is removed, and the ability is available again. No timers, no manual cleanup.

!!! info "Tag hierarchy for cooldowns"
    We use `Cooldown.Ability.BasicAttack` rather than a flat name. This hierarchy matters — if you later want a "reset all cooldowns" ability, you can check for any tag under `Cooldown` and remove them all. Plan your [tag architecture](../patterns/tag-design.md) early.

### GE_Damage_Melee

This is the damage effect. Instead of hardcoding a damage number, we use **SetByCaller** — the ability sets the damage value at runtime when it creates the effect spec. This means a single damage effect class can be reused with different damage values, and those values can come from attributes, data tables, or calculations.

| Setting | Value |
|---|---|
| **Duration Policy** | Instant |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.PendingDamage` |
| **Modifiers[0] — Modifier Op** | Add |
| **Modifiers[0] — Magnitude Type** | Set By Caller |
| **Modifiers[0] — Set By Caller Tag** | `SetByCaller.Damage` |

Notice we target `PendingDamage`, not `Health`. This is the meta attribute pattern from [Project Setup](project-setup.md) — damage flows into `PendingDamage`, gets processed in `PostGameplayEffectExecute` (armor, shields, etc.), and the final result is applied to Health. This gives you a single processing pipeline for all damage sources.

??? question "Why SetByCaller instead of a fixed value?"
    You *could* hardcode `25.0` as the damage. It would work. But the moment you want a second melee ability with different damage, or want damage to scale with a stat, you'd need a separate effect class. SetByCaller keeps the effect generic — the ability (or an Execution Calculation) sets the actual number. One effect class, many damage values. See [SetByCaller](../gameplay-effects/set-by-caller.md) for the full picture.

## Step 2: Create the Ability Blueprint

Create a new Blueprint class in the Content Browser. The parent class should be your **YourProjectGameplayAbility** (the base ability class from [Project Setup](project-setup.md)).

Name it something like `GA_MeleeAttack`.

### Class Defaults

Open the Blueprint and set these in the **Class Defaults** panel:

| Property | Value | Why |
|---|---|---|
| **Input Tag** | `InputTag.Combat.Primary` | Maps this ability to your primary attack input |
| **Ability Tags** | `Ability.Combat.MeleeAttack` | Identifies this ability — used for queries and blocking |
| **Activation Blocked Tags** | `State.Dead`, `CrowdControl.Hard` | Can't attack while dead or stunned |
| **Cancel Abilities With Tag** | *(leave empty for now)* | Could cancel other abilities on activation |
| **Cost Gameplay Effect Class** | `GE_Cost_MeleeAttack` | GAS checks this automatically before activation |
| **Cooldown Gameplay Effect Class** | `GE_Cooldown_MeleeAttack` | GAS applies this automatically on activation |

!!! note "Tags you need to create"
    If these tags don't exist in your project yet, you'll need to create them. Go to **Project Settings > Gameplay Tags** or add them in a `GameplayTags.ini` file. The tag names shown here follow the [naming conventions](../patterns/naming-conventions.md) used throughout this guide.

### Event Graph

Here's the ability logic. This runs when the ability activates (after cost and cooldown checks pass):

=== "Blueprint"
    ```
    Event ActivateAbility
        │
        ▼
    Play Montage and Wait (AM_MeleeAttack, rate 1.0)
        ├── On Completed ──────────► End Ability (End on all paths!)
        ├── On Interrupted ─────────► End Ability
        ├── On Cancelled ───────────► End Ability
        └── On Blend Out ───────────► (optional: End Ability)

    Wait Gameplay Event (Tag: Event.Montage.MeleeHit)
        │
        ▼
    Make Outgoing Gameplay Effect Spec (GE_Damage_Melee)
        │
        ▼
    Assign Set By Caller Magnitude
        ├── Spec Handle: (from above)
        ├── Data Tag: SetByCaller.Damage
        └── Magnitude: 25.0
        │
        ▼
    Apply Gameplay Effect Spec to Target
        ├── Spec: (from above)
        └── Target: Event Data → Target (from Wait Gameplay Event)
    ```

=== "Step-by-Step"
    1. **ActivateAbility** fires when GAS activates the ability
    2. **Play Montage and Wait** — an [Ability Task](../gameplay-abilities/ability-tasks.md) that plays `AM_MeleeAttack` and gives you delegates for completion, interruption, and cancellation
    3. **Wait Gameplay Event** — another Ability Task that listens for a gameplay event with tag `Event.Montage.MeleeHit`. This event is sent by an **AnimNotify** in your montage at the exact frame the weapon should deal damage
    4. **Make Outgoing GE Spec** — creates a spec from `GE_Damage_Melee`, ready to be configured
    5. **Assign Set By Caller Magnitude** — sets the `SetByCaller.Damage` value to 25.0 (this is where the damage number lives)
    6. **Apply GE Spec to Target** — applies the configured damage spec to whoever was hit. The target comes from the gameplay event's payload

!!! danger "Always End Ability"
    Every code path must call **End Ability**. If you forget, the ability stays "active" forever — blocking re-activation, holding its slot, and leaking resources. This is the most common ability bug. Connect End Ability to every completion delegate from Play Montage and Wait.

### The AnimNotify

Open your attack montage (`AM_MeleeAttack`) in the Montage Editor. At the frame where the weapon should connect with a target, add an **AnimNotify** that sends a gameplay event:

1. Right-click the Notifies track at the desired frame
2. Add Notify > **AN_SendGameplayEvent** (or your custom notify class)
3. Set the Event Tag to `Event.Montage.MeleeHit`
4. The event payload should include the target actor (populated by your hit detection logic — a trace, overlap, etc.)

??? question "Why Wait Gameplay Event instead of a delay or timer?"
    Timing damage to a specific animation frame is critical for game feel. A delay node is fragile — if you change the animation speed or swap montages, the timing breaks. `Wait Gameplay Event` decouples the ability from specific frame timing. The montage itself declares when the hit happens (via the AnimNotify), and the ability reacts to that event. Change the montage, change the timing — the ability code doesn't need to change at all.

## Step 3: Wire Up Input

You need three things to connect a keyboard/gamepad input to your ability:

### 1. InputAction Asset

Create an Input Action asset in the editor (right-click > **Input > Input Action**). Name it `IA_PrimaryAttack`. Set the Value Type to `Digital (Bool)` — it's a press, not an axis.

### 2. InputMappingContext

Open (or create) your Input Mapping Context (`IMC_Default` or similar). Add a mapping:

- **Input Action:** `IA_PrimaryAttack`
- **Key:** Left Mouse Button (or whatever you prefer)

### 3. Route Input to the ASC

The connection between Enhanced Input and GAS happens in your character's `SetupPlayerInputComponent`. The exact implementation depends on your input binding approach, but the core idea is:

When `IA_PrimaryAttack` fires, you find all granted abilities whose `InputTag` matches `InputTag.Combat.Primary` and try to activate them.

=== "Simplified Approach"
    ```cpp
    // In your character's SetupPlayerInputComponent:
    void AYourProjectCharacterBase::SetupPlayerInputComponent(
        UInputComponent* PlayerInputComponent)
    {
        Super::SetupPlayerInputComponent(PlayerInputComponent);

        if (UEnhancedInputComponent* EnhancedInput =
            Cast<UEnhancedInputComponent>(PlayerInputComponent))
        {
            // Bind the primary attack input action
            EnhancedInput->BindAction(
                PrimaryAttackAction, // UPROPERTY pointing to IA_PrimaryAttack
                ETriggerEvent::Started,
                this, &AYourProjectCharacterBase::OnPrimaryAttackInput);
        }
    }

    void AYourProjectCharacterBase::OnPrimaryAttackInput()
    {
        if (!AbilitySystemComponent) return;

        // Find and activate abilities with matching InputTag
        for (FGameplayAbilitySpec& Spec :
             AbilitySystemComponent->GetActivatableAbilities())
        {
            if (const UYourProjectGameplayAbility* Ability =
                Cast<UYourProjectGameplayAbility>(Spec.Ability))
            {
                if (Ability->InputTag.MatchesTagExact(
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

=== "Blueprint"
    In your Character Blueprint's Event Graph:

    1. Add an **Enhanced Input Action** event node for `IA_PrimaryAttack`
    2. From the exec pin, call **Get Ability System Component** on Self
    3. Call a custom function that iterates activatable abilities and tries to activate any whose Input Tag matches `InputTag.Combat.Primary`

!!! tip "Production input systems"
    The approach above is simplified for clarity. Production projects often use a more scalable pattern — an InputAction-to-Tag mapping table that automatically routes all inputs to abilities by tag, without per-action binding functions. See [Input Binding](../gameplay-abilities/input-binding.md) for the full architecture.

## Step 4: Grant the Ability to Your Character

Open your Character Blueprint (the one based on `YourProjectCharacterBase`). In Class Defaults, find the **Startup Abilities** array and add your `GA_MeleeAttack` class.

That's it. When the character spawns and `InitializeAbilities` runs (from [Project Setup](project-setup.md)), this ability will be granted to the ASC and ready to activate.

!!! info "Runtime granting"
    Startup arrays are for abilities the character always has. For abilities gained later (from equipment, level-ups, pickups), you call `AbilitySystemComponent->GiveAbility()` at runtime. See [Ability Sets](../gameplay-abilities/ability-sets.md) for a scalable approach to runtime ability management.

## Step 5: Test It

### Quick Smoke Test

1. Place your character Blueprint in a level
2. Hit Play
3. Press your attack button — the character should play the attack montage
4. If you have a second character in range, they should take 25 damage

### Debugging with ShowDebug

Open the console (++grave++) and type:

```
showdebug abilitysystem
```

This overlay shows you everything happening in the ASC in real time:

- **Granted Abilities** — you should see `GA_MeleeAttack` listed
- **Active Abilities** — lights up while your attack is playing
- **Active Effects** — you'll see the cooldown effect appear for 1 second after attacking
- **Attributes** — watch Stamina drop by 15 each attack, Health drop on the target
- **Tags** — the cooldown tag appears and disappears

??? example "What to look for at each step"

    | You Press Attack | What Should Happen |
    |---|---|
    | **Stamina >= 15, not on cooldown** | Montage plays, stamina drops by 15, cooldown tag appears |
    | **Stamina < 15** | Nothing happens — ability fails cost check |
    | **On cooldown** | Nothing happens — ability blocked by cooldown tag |
    | **While stunned (has CrowdControl.Hard)** | Nothing happens — blocked by Activation Blocked Tags |
    | **Hit lands on target** | Target's PendingDamage receives 25, PostGameplayEffectExecute processes it, Health drops |

!!! warning "Nothing happening?"
    The most common issues at this stage:

    1. **Ability not granted** — Check that `GA_MeleeAttack` is in the StartupAbilities array
    2. **Input not firing** — Check your InputMappingContext is added to the local player's Enhanced Input subsystem
    3. **Montage doesn't play** — Make sure the montage's skeleton matches your character's skeleton
    4. **Hit doesn't register** — Check that the AnimNotify fires `Event.Montage.MeleeHit` and that your hit detection populates the event payload with a target
    5. **Damage doesn't apply** — Verify the target has an ASC (the target also needs to be a GAS character)

    See [Troubleshooting](../debugging/troubleshooting.md) for a complete debugging checklist.

## What Just Happened?

Let's trace the full flow one more time, now that you've seen every piece:

1. **Input** fires `IA_PrimaryAttack`
2. Your input handler finds abilities with `InputTag.Combat.Primary` and calls `TryActivateAbility`
3. The ASC checks: Can this ability activate? It evaluates:
    - Cost: Do we have 15+ Stamina? (checks `GE_Cost_MeleeAttack`)
    - Cooldown: Is `Cooldown.Ability.BasicAttack` tag present? (checks `GE_Cooldown_MeleeAttack`)
    - Blocked tags: Does the owner have `State.Dead` or `CrowdControl.Hard`?
4. If all checks pass, the ability **activates**:
    - Cost effect is applied (Stamina -15)
    - Cooldown effect is applied (tag granted for 1 second)
    - `ActivateAbility` fires in the Blueprint
5. The ability plays a montage and waits for the `Event.Montage.MeleeHit` gameplay event
6. An AnimNotify in the montage sends the event at the right frame
7. The ability creates a damage GE spec, sets the damage amount via SetByCaller, and applies it to the target
8. The target's `PostGameplayEffectExecute` processes `PendingDamage` and subtracts from Health
9. The montage completes, the ability calls End Ability

Every piece of GAS was involved. The ASC managed it. Tags controlled flow. Effects modified attributes. The ability orchestrated the sequence. This is the GAS loop.

## What's Next

You've built an ability that works. But should *everything* be an ability? Jumping? Sprinting? Opening a door? Head to [Should It Be an Ability?](should-it-be-an-ability.md) for the decision framework you'll use on every feature going forward.
