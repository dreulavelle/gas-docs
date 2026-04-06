
# Your First Ability — Melee Attack

Time to build something. In the next ten minutes, you'll create a simple melee attack ability with hit detection, damage, stamina cost, cooldown, and crowd control blocking -- all through data configuration and a handful of Blueprint nodes.

A melee attack is the simplest ability that demonstrates why GAS exists. It touches every part of the loop -- cost, cooldown, targeting, damage application, and animation -- which makes it perfect for seeing how the pieces connect without needing projectile spawning or complex state machines.

!!! tip "Prerequisites"
    This page assumes you've completed [Project Setup](project-setup.md) and have a working character with an ASC, an AttributeSet (at minimum: Health and Stamina), and a base ability class (`YourProjectGameplayAbility`).

## Before GAS: The Manual Way

Here's what a melee attack looks like without GAS:

```
void TryMeleeAttack()
{
    if (bIsStunned || bIsDead) return;              // CC checks
    if (Stamina < 15.f) return;                      // resource check
    Stamina -= 15.f;                                  // cost
    if (bAttackOnCooldown) return;                    // cooldown check
    GetWorldTimerManager().SetTimer(CooldownHandle,   // cooldown
        this, &AMyChar::ResetAttackCooldown, 1.0f);
    bAttackOnCooldown = true;

    // Hit detection
    FHitResult Hit;
    FVector Start = GetActorLocation();
    FVector End = Start + GetActorForwardVector() * 150.f;
    GetWorld()->SweepSingleByChannel(Hit, Start, End,
        FQuat::Identity, ECC_Pawn, FCollisionShape::MakeSphere(50.f));

    if (Hit.GetActor())
    {
        // Apply damage... somehow
        Hit.GetActor()->TakeDamage(25.f, FDamageEvent(), GetController(), this);
    }

    PlayAnimMontage(AttackMontage);
}
```

Ten lines of manual state management, and damage goes through `TakeDamage` instead of the attribute system. Now imagine adding a new CC type -- freeze. You'd need to find every action that freeze should block and add `|| bIsFrozen` to each one. That doesn't scale.

With GAS, you add one tag (`CrowdControl.Hard`) to the freeze effect. Every ability that blocks on `CrowdControl.Hard` -- attack, jump, dodge, sprint -- is automatically blocked. Zero code changes to any ability.

That's the pitch. Let's build it.

## Step 1: Create the Effects

Effects first, ability second. Effects define *what happens*; the ability decides *when*.

All three effects are created in the editor: right-click in the Content Browser, select **Blueprint Class**, and choose **GameplayEffect** as the parent.

### GE_Cost_MeleeAttack

The stamina cost. GAS checks this automatically before the ability activates -- if the character can't afford it, the ability won't fire.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.Stamina` |
| **Modifiers[0] — Modifier Op** | Add |
| **Modifiers[0] — Magnitude** | Scalable Float: `-15.0` |

!!! tip "Why negative?"
    GAS doesn't have a "subtract" operation -- costs are instant effects that *add* a negative value. Feels odd at first, but it's consistent. See [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) for the full picture.

### GE_Cooldown_MeleeAttack

A one-second cooldown to prevent attack spam. While this effect is active, the cooldown tag blocks re-activation.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `1.0` (seconds) |
| **Granted Tags** | `Cooldown.Ability.MeleeAttack` |

!!! info "Where to find Granted Tags in the editor"
    In UE 5.3+, tag granting moved from a top-level GE property to a **GE Component**. In your Gameplay Effect Blueprint, look for **Components → Target Tags (Granted to Actor)** → **Add Tags** → **Add to Inherited**.

When the 1-second duration expires, the effect is removed, the tag disappears, and the ability is available again. No timers, no manual cleanup.

### GE_Damage_MeleeAttack

The damage effect applied to whatever the sphere trace hits. This is a simple instant modifier -- no math pipeline, no meta attributes.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.Health` |
| **Modifiers[0] — Modifier Op** | Add |
| **Modifiers[0] — Magnitude** | Scalable Float: `-25.0` |

!!! note "Direct Health vs. damage pipeline"
    This targets Health directly for simplicity. Production games use SetByCaller magnitudes and a PendingDamage meta attribute so damage flows through a processing pipeline (armor, shields). See [Melee Attack](../examples/melee-attack.md) for that approach.

## Step 2: Create the Ability Blueprint

Create a new Blueprint class with **YourProjectGameplayAbility** as the parent. Name it `GA_MeleeAttack`.

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Input Tag** | `InputTag.Combat.Primary` | Maps this ability to your primary attack input |
| **Ability Tags** | `Ability.Combat.MeleeAttack` | Identifies this ability for queries and interactions |
| **Activation Blocked Tags** | `State.Dead`, `CrowdControl.Hard` | Can't attack while dead or hard-CC'd |
| **Cost Gameplay Effect Class** | `GE_Cost_MeleeAttack` | GAS checks this automatically before activation |
| **Cooldown Gameplay Effect Class** | `GE_Cooldown_MeleeAttack` | GAS applies this automatically on commit |

!!! note "Tags you need to create"
    If these tags don't exist yet, add them in **Project Settings > Gameplay Tags** or in a `GameplayTags.ini` file. The naming follows the conventions from [Tag Architecture](../patterns/tag-design.md).

### Event Graph

The melee attack ability needs to handle five things: committing resources, detecting hits, applying damage, playing the attack animation, and ending cleanly.

=== "Blueprint"

    !!! abstract "Event Graph"

        1. **ActivateAbility** fires after GAS confirms the ability *can* activate (tag checks pass)
        2. **CommitAbility** deducts the cost and applies the cooldown. If it fails (not enough stamina, still on cooldown), **EndAbility** immediately
        3. **Get Avatar Actor from Actor Info** -- returns the pawn/character this ability is running on
        4. **SphereTraceByChannel** -- trace forward from the avatar actor (Start = actor location, End = actor location + forward * 150, Radius = 50) to detect hits
        5. **Check hit actor for an ASC** -- call `GetAbilitySystemComponent` on the hit actor via `IAbilitySystemInterface`. If no ASC, skip damage
        6. **Apply GE_Damage_MeleeAttack to target** -- `ApplyGameplayEffectToTarget` on the target's ASC
        7. **PlayMontageAndWait** -- an [Ability Task](../gameplay-abilities/ability-tasks.md) that plays the attack animation and waits for it to finish
        8. **EndAbility** -- on montage Completed, Interrupted, or Cancelled

    ```mermaid
    flowchart LR
        A["ActivateAbility"]:::event --> B["CommitAbility"]:::func
        B -->|Failed| C["EndAbility"]:::endpoint
        B -->|Success| D["Sphere Trace\nForward"]:::func
        D -->|Hit + Has ASC| E["Apply\nGE_Damage_MeleeAttack"]:::func
        D -->|Miss / No ASC| F["PlayMontageAndWait"]:::task
        E --> F
        F -->|Completed| G["EndAbility"]:::endpoint
        F -->|Interrupted| G
        F -->|Cancelled| G

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

        AActor* AvatarActor = ActorInfo->AvatarActor.Get();
        if (!AvatarActor)
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }

        // --- Sphere trace forward from the character ---
        const FVector Start = AvatarActor->GetActorLocation();
        const FVector End = Start
            + AvatarActor->GetActorForwardVector() * TraceDistance;

        FHitResult HitResult;
        FCollisionQueryParams Params;
        Params.AddIgnoredActor(AvatarActor);

        const bool bHit = GetWorld()->SweepSingleByChannel(
            HitResult,
            Start,
            End,
            FQuat::Identity,
            ECC_Pawn,
            FCollisionShape::MakeSphere(TraceRadius),
            Params);

        // --- Apply damage if we hit something with an ASC ---
        if (bHit && HitResult.GetActor())
        {
            if (UAbilitySystemComponent* TargetASC =
                    UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(
                        HitResult.GetActor()))
            {
                FGameplayEffectSpecHandle DamageSpec =
                    MakeOutgoingGameplayEffectSpec(
                        DamageEffectClass, GetAbilityLevel());

                if (DamageSpec.IsValid())
                {
                    ApplyGameplayEffectSpecToTarget(
                        Handle, ActorInfo, ActivationInfo,
                        DamageSpec, TargetASC);
                }
            }
        }

        // --- Play attack montage (gates EndAbility) ---
        UAbilityTask_PlayMontageAndWait* MontageTask =
            UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
                this, NAME_None, AttackMontage, 1.0f);

        MontageTask->OnCompleted.AddDynamic(
            this, &UGA_MeleeAttack::OnMontageFinished);
        MontageTask->OnInterrupted.AddDynamic(
            this, &UGA_MeleeAttack::OnMontageFinished);
        MontageTask->OnCancelled.AddDynamic(
            this, &UGA_MeleeAttack::OnMontageFinished);

        MontageTask->ReadyForActivation();
    }

    void UGA_MeleeAttack::OnMontageFinished()
    {
        K2_EndAbility();
    }
    ```

    The header declares the configurable properties:

    ```cpp
    UCLASS()
    class UGA_MeleeAttack : public UYourProjectGameplayAbility
    {
        GENERATED_BODY()

    public:
        /** The damage effect to apply on hit. */
        UPROPERTY(EditDefaultsOnly, Category = "Damage")
        TSubclassOf<UGameplayEffect> DamageEffectClass;

        /** How far forward the sphere trace reaches (units). */
        UPROPERTY(EditDefaultsOnly, Category = "Trace")
        float TraceDistance = 150.f;

        /** Radius of the sphere trace (units). */
        UPROPERTY(EditDefaultsOnly, Category = "Trace")
        float TraceRadius = 50.f;

        /** The attack animation montage. */
        UPROPERTY(EditDefaultsOnly, Category = "Animation")
        TObjectPtr<UAnimMontage> AttackMontage;

    protected:
        virtual void ActivateAbility(
            const FGameplayAbilitySpecHandle Handle,
            const FGameplayAbilityActorInfo* ActorInfo,
            const FGameplayAbilityActivationInfo ActivationInfo,
            const FGameplayEventData* TriggerEventData) override;

    private:
        UFUNCTION()
        void OnMontageFinished();
    };
    ```

!!! note "Trace timing"
    We fire the trace on activation for simplicity. In a production ability, you'd time the hit to a specific animation frame using AnimNotify + WaitGameplayEvent -- see [Melee Attack](../examples/melee-attack.md).

!!! danger "Always End Ability"
    Every code path must call **End Ability**. If you forget, the ability stays "active" forever -- blocking re-activation, holding its slot, and leaking resources. This is the single most common GAS bug.

## Step 3: Wire Input

Three pieces connect a key press to your ability:

### 1. Input Action

Create an Input Action asset: right-click > **Input > Input Action**. Name it `IA_PrimaryAttack`. Set the Value Type to **Digital (Bool)**.

### 2. Input Mapping Context

Open your `IMC_Default` (or create one). Add a mapping:

- **Input Action:** `IA_PrimaryAttack`
- **Key:** Left Mouse Button

### 3. Route Input to the ASC

Your character class needs C++ code that bridges Enhanced Input to the ASC. When `IA_PrimaryAttack` fires, the character tells the ASC to activate the ability bound to that input -- either by matching an integer InputID or by matching the `InputTag` you set in Class Defaults.

There are two approaches to this routing, and both require a `SetupPlayerInputComponent` override in C++:

- **InputID** -- simpler, maps each input to an enum value. The ASC handles the rest.
- **Tag-based routing** -- more flexible, maps each input to a gameplay tag. Scales to dynamic ability sets.

The [Input Binding](../gameplay-abilities/input-binding.md) page walks through both approaches with complete working code. The [Bind Ability to Input](../recipes/bind-ability-to-input.md) recipe has a condensed checklist.

!!! info "Why can't this be Blueprint-only?"
    `SetupPlayerInputComponent` is a C++ virtual function on `ACharacter`. The binding loop that maps Enhanced Input actions to ASC calls lives there. Your abilities, effects, and gameplay logic can still be entirely Blueprint -- only this input bridge needs C++.

### 4. Grant the Ability

Open your Character Blueprint. In Class Defaults, find the **Startup Abilities** array and add `GA_MeleeAttack`.

When the character spawns and `InitializeAbilities` runs (from [Project Setup](project-setup.md)), the ability is granted to the ASC and ready to go.

## Step 4: Test

1. Hit **Play**
2. Click **Left Mouse Button** -- the character should play the attack montage
3. Watch Stamina drop by 15 in the debug overlay
4. Spam LMB -- you can only attack every 1.0 second

### Debugging with ShowDebug

Open the console (++grave++) and type:

```
showdebug abilitysystem
```

You should see:

| What to Check | Expected |
|:---|:---|
| **Granted Abilities** | `GA_MeleeAttack` listed |
| **Active Effects** (after attacking) | `GE_Cooldown_MeleeAttack` appears for 1.0s |
| **Attributes** | Stamina decreases by 15 per attack; target Health decreases by 25 on hit |
| **Tags** | `Cooldown.Ability.MeleeAttack` appears and disappears |

??? example "Edge case testing"
    | Scenario | Expected Result |
    |:---|:---|
    | Stamina >= 15, not on cooldown, target in range | Attack fires, stamina drops by 15, target Health drops by 25 |
    | Stamina < 15 | Nothing happens -- cost check fails |
    | On cooldown (< 1.0s since last attack) | Nothing happens -- cooldown tag blocks |
    | No target in range | Attack fires (montage plays, stamina deducted), but no damage applied |
    | Character has `CrowdControl.Hard` | Nothing happens -- blocked by Activation Blocked Tags |
    | Character has `State.Dead` | Nothing happens -- blocked by Activation Blocked Tags |

!!! warning "Nothing happening?"
    Common issues:

    1. **Ability not granted** -- check `GA_MeleeAttack` is in the StartupAbilities array
    2. **Input not firing** -- verify your IMC is added to the Enhanced Input subsystem on the local player
    3. **Cost check fails immediately** -- make sure your Stamina attribute is initialized to a value >= 15
    4. **Tags not matching** -- double-check the tag names are identical between the effect and the ability's blocked tags
    5. **No damage on hit** -- verify the target actor implements `IAbilitySystemInterface` and has an ASC with a Health attribute

    See [Troubleshooting](../debugging/troubleshooting.md) for the full debugging checklist.

## The GAS Advantage

Here's the payoff. Say your designer adds a new CC type: **Freeze**. Freeze should prevent all actions -- attacking, jumping, dodging, sprinting.

Without GAS, you'd search every action function and add `if (bIsFrozen) return;`. Miss one, and frozen characters can still attack.

With GAS, the freeze effect grants the tag `CrowdControl.Hard`. Every ability that has `CrowdControl.Hard` in its Activation Blocked Tags -- attack, jump, dodge, sprint -- is automatically blocked. You add the tag to *one* effect. Zero changes to any ability. That's the entire point of the system.

## What's Next

You've built a working GAS ability. But should *everything* be an ability? A melee attack makes sense -- but what about opening a door? Crouching? Swimming?

Head to [Should It Be an Ability?](should-it-be-an-ability.md) for the decision framework you'll use on every feature going forward.

After that, check out the [Examples](../examples/index.md) section for production-ready versions -- [melee attacks](../examples/melee-attack.md) with animation-driven hit timing and a proper damage pipeline, [ranged attacks](../examples/ranged-attack.md) with projectile spawning, and [dodge rolls](../examples/dodge-roll.md) with i-frames.
