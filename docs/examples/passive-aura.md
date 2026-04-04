---
title: "Example: Passive Aura"
description: Passive ability that automatically heals nearby allies, demonstrating passive activation, periodic area scanning, and applying effects to other actors' ASCs.
---

# Example: Passive Aura

<div class="example-badges" markdown>
  <span class="badge badge--intermediate">Intermediate</span>
</div>

A passive ability that automatically activates when granted and periodically heals allies within range. This demonstrates passive abilities (activate on grant, never manually triggered), periodic area scanning with `AbilityTask_Repeat`, and applying effects to other actors' Ability System Components — a pattern you'll use for auras, proximity buffs, and area-of-effect support abilities.

## What We're Building

- Passive ability that activates automatically when granted (no input)
- Every 2 seconds, scans for allies within 500 units
- Applies a heal-over-time effect to allies in range
- Removes the effect when allies leave range
- No input, no cost, no cooldown

This is more advanced than the other examples because:

- It's a **passive** — it activates on grant, not via player input
- It interacts with **multiple ASCs** (not just the owner's)
- It manages **effect handles on other actors** (must track who has the buff)
- The scanning logic is best implemented in **C++** for performance and clean handle management

## Prerequisites

Everything from [Project Setup](../getting-started/project-setup.md), plus:

- A **Health** attribute on allied characters (the heal target)
- A way to identify allies — either a `Team.Ally` gameplay tag on friendly actors, or a team/faction system you can query
- Characters that can receive effects (they have their own ASC and AttributeSet)

---

## Step 1: Create the Effects

### GE_Aura_HealOverTime

This is the buff applied to allies within the aura's range. It heals periodically and has a duration slightly longer than the scan interval, so the effect persists between scans even if there's minor timing jitter.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `3.0` (slightly longer than the 2s scan interval) |
| **Period** | `1.0` seconds |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.Health` |
| **Modifiers[0] — Modifier Op** | `Add (Base)` |
| **Modifiers[0] — Magnitude** | Scalable Float: `5.0` |

**Tags (via GE Components):**

| GE Component | Tag | Purpose |
|:---|:---|:---|
| **TargetTagsGameplayEffectComponent** — Granted Tags | `Status.Buff.Regeneration` | Marks the target as having a regen buff (for UI display) |
| **AssetTagsGameplayEffectComponent** — Asset Tags | `Effect.Aura.Heal` | Identifies this effect (useful for stacking rules or removal) |

!!! info "Why 3-second duration instead of Infinite?"
    Using a finite duration that's slightly longer than the scan interval gives us a natural cleanup mechanism. If an ally walks out of range, their buff simply expires after 3 seconds without the aura needing to explicitly track and remove it. This is much simpler than managing Infinite effects that require explicit removal. The aura re-applies the effect every 2 seconds to allies still in range, refreshing the duration.

---

## Step 2: Create the Ability

**Asset:** `GA_PassiveAura_Heal`

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Ability Tags** | `Ability.Passive.Aura.Heal` | Identifies this passive |
| **Activation Blocked Tags** | `State.Dead` | Don't run the aura while dead |
| **Instancing Policy** | `InstancedPerActor` | Required — we store state (handles, timers) |
| **Net Execution Policy** | `ServerOnly` | Aura scanning and effect application should only happen on the server |

!!! warning "ServerOnly for auras"
    Auras that affect other actors should use `ServerOnly` net execution. The server is the authority on who's in range and who gets the buff. Clients will see the results through normal GE replication. Trying to predict aura effects on the client would be a nightmare of synchronization issues.

### Passive Activation

There are several ways to make an ability activate automatically on grant:

1. **`bActivateAbilityOnGranted`** — Set this to `true` in the ability's Class Defaults. This is the simplest approach: the ability activates immediately when granted to the ASC.

2. **Gameplay Event Trigger** — Configure the ability to activate from a specific gameplay event tag. Then send that event when the character spawns or when you want the aura to start.

For this example, we'll use option 1 — `bActivateAbilityOnGranted = true`.

### Event Graph

=== "Blueprint"
    ```
    Event ActivateAbility
        |
        +-- Repeat (TimeBetweenActions = 2.0, TotalActionCount = 9999)
              |
              +-- On Perform Action
              |     |
              |     +-- Sphere Overlap Actors (Location = Owner Location,
              |     |     Radius = 500, Object Types = Pawn)
              |     |
              |     +-- For Each actor in Overlapping Actors:
              |     |     +-- Get Ability System Component (from actor)
              |     |     +-- [Branch: has ASC AND has tag Team.Ally]
              |     |           |
              |     |           +-- Make Outgoing GE Spec (GE_Aura_HealOverTime)
              |     |           +-- Apply GE Spec to Target (target = ally's ASC)
              |     |
              |     \-- (done with loop)
              |
              +-- On Finished -> End Ability
    ```

    The Blueprint approach is simpler: it re-applies the effect every 2 seconds to all allies in range. Because the effect has a 3-second duration, allies who leave range will lose the buff naturally when it expires. No explicit handle tracking needed.

    !!! tip "Stacking behavior"
        By default, re-applying an effect to a target that already has it either stacks or refreshes depending on the effect's stacking settings. For this aura, set the stacking policy on `GE_Aura_HealOverTime` to **Aggregate by Source** with a stack limit of 1 and **Refresh Duration** on new application. This way, re-applying just resets the timer instead of stacking multiple heal effects.

=== "C++"
    The C++ version adds explicit handle tracking, which is more efficient (avoids re-applying effects to actors that already have the buff) and gives you precise control over removal.

    ```cpp
    // GA_PassiveAura_Heal.h
    #pragma once

    #include "CoreMinimal.h"
    #include "YourProjectGameplayAbility.h"
    #include "ActiveGameplayEffectHandle.h"
    #include "GA_PassiveAura_Heal.generated.h"

    UCLASS()
    class UGA_PassiveAura_Heal : public UYourProjectGameplayAbility
    {
        GENERATED_BODY()

    public:
        UGA_PassiveAura_Heal();

        virtual void ActivateAbility(
            const FGameplayAbilitySpecHandle Handle,
            const FGameplayAbilityActorInfo* ActorInfo,
            const FGameplayAbilityActivationInfo ActivationInfo,
            const FGameplayEventData* TriggerEventData) override;

        virtual void EndAbility(
            const FGameplayAbilitySpecHandle Handle,
            const FGameplayAbilityActorInfo* ActorInfo,
            const FGameplayAbilityActivationInfo ActivationInfo,
            bool bReplicateEndAbility,
            bool bWasCancelled) override;

    protected:
        /** The heal effect to apply to allies. */
        UPROPERTY(EditDefaultsOnly, Category = "Aura")
        TSubclassOf<UGameplayEffect> HealEffectClass;

        /** Radius of the aura in Unreal units. */
        UPROPERTY(EditDefaultsOnly, Category = "Aura")
        float AuraRadius = 500.0f;

        /** Tag that allies must have to receive the buff. */
        UPROPERTY(EditDefaultsOnly, Category = "Aura")
        FGameplayTag AllyTag;

    private:
        UFUNCTION()
        void OnScanTick(int32 ActionNumber);

        UFUNCTION()
        void OnScanFinished(int32 ActionNumber);

        void RemoveAllAuraEffects();

        /** Maps each affected actor to the active effect handle on their ASC. */
        TMap<TWeakObjectPtr<AActor>, FActiveGameplayEffectHandle> ActiveAuraHandles;
    };
    ```

    ```cpp
    // GA_PassiveAura_Heal.cpp
    #include "GA_PassiveAura_Heal.h"
    #include "AbilitySystemComponent.h"
    #include "AbilitySystemInterface.h"
    #include "Abilities/Tasks/AbilityTask_Repeat.h"
    #include "Kismet/KismetSystemLibrary.h"

    UGA_PassiveAura_Heal::UGA_PassiveAura_Heal()
    {
        InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
        NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerOnly;

        // Activate automatically when granted
        bActivateAbilityOnGranted = true;
    }

    void UGA_PassiveAura_Heal::ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData)
    {
        if (!HasAuthorityOrPredictionKey(ActorInfo, &ActivationInfo))
        {
            return;
        }

        // Start repeating scan: every 2 seconds, up to 9999 times
        // (effectively infinite for gameplay purposes)
        UAbilityTask_Repeat* RepeatTask =
            UAbilityTask_Repeat::RepeatAction(this, 2.0f, 9999);
        RepeatTask->OnPerformAction.AddDynamic(
            this, &UGA_PassiveAura_Heal::OnScanTick);
        RepeatTask->OnFinished.AddDynamic(
            this, &UGA_PassiveAura_Heal::OnScanFinished);
        RepeatTask->ReadyForActivation();
    }

    void UGA_PassiveAura_Heal::OnScanTick(int32 ActionNumber)
    {
        AActor* OwnerActor = GetAvatarActorFromActorInfo();
        if (!OwnerActor)
        {
            return;
        }

        // Sphere overlap to find nearby pawns
        TArray<FOverlapResult> Overlaps;
        FCollisionQueryParams QueryParams;
        QueryParams.AddIgnoredActor(OwnerActor);

        GetWorld()->OverlapMultiByObjectType(
            Overlaps,
            OwnerActor->GetActorLocation(),
            FQuat::Identity,
            FCollisionObjectQueryParams(ECC_Pawn),
            FCollisionShape::MakeSphere(AuraRadius),
            QueryParams);

        // Collect actors currently in range
        TSet<AActor*> ActorsInRange;
        for (const FOverlapResult& Overlap : Overlaps)
        {
            AActor* OverlapActor = Overlap.GetActor();
            if (!OverlapActor)
            {
                continue;
            }

            // Check if the actor has an ASC and the ally tag
            if (IAbilitySystemInterface* ASCInterface =
                    Cast<IAbilitySystemInterface>(OverlapActor))
            {
                UAbilitySystemComponent* TargetASC =
                    ASCInterface->GetAbilitySystemComponent();
                if (TargetASC && TargetASC->HasMatchingGameplayTag(AllyTag))
                {
                    ActorsInRange.Add(OverlapActor);

                    // Apply the effect if this actor doesn't already have it
                    if (!ActiveAuraHandles.Contains(OverlapActor))
                    {
                        FGameplayEffectSpecHandle SpecHandle =
                            MakeOutgoingGameplayEffectSpec(
                                HealEffectClass, GetAbilityLevel());

                        if (SpecHandle.IsValid())
                        {
                            FActiveGameplayEffectHandle GEHandle =
                                TargetASC->ApplyGameplayEffectSpecToSelf(
                                    *SpecHandle.Data.Get());

                            if (GEHandle.IsValid())
                            {
                                ActiveAuraHandles.Add(
                                    OverlapActor, GEHandle);
                            }
                        }
                    }
                    else
                    {
                        // Actor already has the effect — refresh duration
                        // by removing and re-applying
                        UAbilitySystemComponent* ExistingASC =
                            ASCInterface->GetAbilitySystemComponent();
                        FActiveGameplayEffectHandle& ExistingHandle =
                            ActiveAuraHandles[OverlapActor];

                        if (ExistingASC && ExistingHandle.IsValid())
                        {
                            ExistingASC->RemoveActiveGameplayEffect(
                                ExistingHandle);
                        }

                        FGameplayEffectSpecHandle SpecHandle =
                            MakeOutgoingGameplayEffectSpec(
                                HealEffectClass, GetAbilityLevel());

                        if (SpecHandle.IsValid())
                        {
                            FActiveGameplayEffectHandle NewHandle =
                                ExistingASC->ApplyGameplayEffectSpecToSelf(
                                    *SpecHandle.Data.Get());
                            ExistingHandle = NewHandle;
                        }
                    }
                }
            }
        }

        // Remove effects from actors who left the range
        TArray<TWeakObjectPtr<AActor>> ToRemove;
        for (auto& Pair : ActiveAuraHandles)
        {
            AActor* TrackedActor = Pair.Key.Get();
            if (!TrackedActor || !ActorsInRange.Contains(TrackedActor))
            {
                // Actor left range or was destroyed — remove effect
                if (TrackedActor)
                {
                    if (IAbilitySystemInterface* ASCInterface =
                            Cast<IAbilitySystemInterface>(TrackedActor))
                    {
                        UAbilitySystemComponent* TargetASC =
                            ASCInterface->GetAbilitySystemComponent();
                        if (TargetASC && Pair.Value.IsValid())
                        {
                            TargetASC->RemoveActiveGameplayEffect(
                                Pair.Value);
                        }
                    }
                }
                ToRemove.Add(Pair.Key);
            }
        }

        for (const TWeakObjectPtr<AActor>& Actor : ToRemove)
        {
            ActiveAuraHandles.Remove(Actor);
        }
    }

    void UGA_PassiveAura_Heal::OnScanFinished(int32 ActionNumber)
    {
        // If the repeat task finishes (hit 9999), just end the ability.
        // In practice this won't happen during normal gameplay.
        RemoveAllAuraEffects();
        K2_EndAbility();
    }

    void UGA_PassiveAura_Heal::RemoveAllAuraEffects()
    {
        for (auto& Pair : ActiveAuraHandles)
        {
            AActor* TrackedActor = Pair.Key.Get();
            if (TrackedActor)
            {
                if (IAbilitySystemInterface* ASCInterface =
                        Cast<IAbilitySystemInterface>(TrackedActor))
                {
                    UAbilitySystemComponent* TargetASC =
                        ASCInterface->GetAbilitySystemComponent();
                    if (TargetASC && Pair.Value.IsValid())
                    {
                        TargetASC->RemoveActiveGameplayEffect(Pair.Value);
                    }
                }
            }
        }
        ActiveAuraHandles.Empty();
    }

    void UGA_PassiveAura_Heal::EndAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility,
        bool bWasCancelled)
    {
        RemoveAllAuraEffects();

        Super::EndAbility(Handle, ActorInfo, ActivationInfo,
            bReplicateEndAbility, bWasCancelled);
    }
    ```

    !!! info "TWeakObjectPtr for safety"
        We use `TWeakObjectPtr<AActor>` as the map key because actors can be destroyed at any time (killed, despawned, level streaming). A weak pointer lets us detect stale entries safely — `Pair.Key.Get()` returns `nullptr` if the actor was destroyed, and we clean it up on the next scan tick.

---

## Step 3: Wire Input

There is no input to wire. This is a **passive ability** — it activates automatically when granted via `bActivateAbilityOnGranted = true`.

To grant the ability, add `GA_PassiveAura_Heal` to your character's startup abilities array, or grant it at runtime:

```cpp
// Grant the passive aura when equipping an item, leveling up, etc.
AbilitySystemComponent->GiveAbility(
    FGameplayAbilitySpec(GA_PassiveAura_HealClass, Level, INDEX_NONE, this));
```

The ability will activate immediately upon being granted and keep running until the character dies or the ability is removed.

---

## Step 4: Test

### Setup

1. Place your aura-granting character in a level
2. Place one or more ally characters nearby (within 500 units) — they need ASCs with a Health attribute and the `Team.Ally` tag
3. Place one ally far away (>500 units) as a control

### What to Check

| Check | Expected Result |
|:---|:---|
| `showdebug abilitysystem` on the aura owner | `GA_PassiveAura_Heal` is active |
| `showdebug abilitysystem` on a nearby ally | `GE_Aura_HealOverTime` is active, `Status.Buff.Regeneration` tag present |
| Ally Health attribute | Increases by 5 every 1 second |
| Far-away ally | No heal effect, no regen tag |
| Move ally out of range | Effect expires after ~3 seconds |
| Move ally back into range | Effect re-applied on next scan tick |
| Kill the aura owner | All ally effects removed (EndAbility cleanup) |

### Edge Cases

- **Ally dies while buffed** — the effect should handle this gracefully (GAS removes effects when an ASC is destroyed)
- **Many allies in range** — verify no performance issues; each scan is one sphere overlap + N effect applications
- **Aura owner destroyed** — `EndAbility` fires, `RemoveAllAuraEffects` cleans up all handles

---

## The Full Flow

```
Character granted GA_PassiveAura_Heal
  |
  v  (bActivateAbilityOnGranted = true)
  |
ActivateAbility()
  +-- Start AbilityTask_Repeat (every 2s, 9999 iterations)
  |
  v  [Every 2 seconds]
  |
OnScanTick()
  +-- Sphere overlap at owner location, radius 500
  +-- For each overlapping pawn:
  |     +-- Has ASC? Has Team.Ally tag?
  |     +-- Not already tracked? -> Apply GE_Aura_HealOverTime, store handle
  |     +-- Already tracked? -> Refresh effect duration
  +-- For each tracked actor NOT in overlap results:
  |     +-- Remove effect by handle
  |     +-- Remove from tracking map
  |
  v  [Aura owner dies or ability removed]
  |
EndAbility()
  +-- Remove all active aura effects from all tracked actors
  +-- Clear tracking map
```

---

## Variations

??? example "Damage aura (enemy debuff)"
    Flip the logic: instead of `Team.Ally`, check for `Team.Enemy`. Apply a damage-over-time effect instead of healing. Add a `GameplayCue` for a visible damaging aura (burning ground, poison cloud, etc.).

??? example "Aura with stacking intensity"
    Make the heal effect stack by source. If multiple aura-holders are near the same ally, the ally gets multiple heal stacks. Configure `GE_Aura_HealOverTime` with **Aggregate by Source** stacking and no stack limit.

??? example "Aura with visual radius indicator"
    Spawn a `UDecalComponent` or `UNiagaraComponent` on the aura owner showing the 500-unit radius. Attach it in `ActivateAbility` and destroy it in `EndAbility`. This is cosmetic-only and doesn't need to be replicated if you use a `GameplayCue` instead.

??? example "Conditional aura (only while above 50% health)"
    Add a `WaitForAttributeChangeThreshold` task that monitors the owner's Health. When health drops below 50%, pause the aura (stop applying new effects, let existing ones expire). When health rises above 50%, resume scanning. This gives you a passive that rewards staying healthy.

---

## Related Pages

- [Lifecycle and Activation](../gameplay-abilities/lifecycle-and-activation.md) — passive ability activation patterns
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) — `AbilityTask_Repeat` details
- [Instancing Policy](../gameplay-abilities/instancing-policy.md) — why passives need `InstancedPerActor`
- [Duration and Lifecycle](../gameplay-effects/duration-and-lifecycle.md) — periodic effects and duration management
- [Net Execution Policies](../networking/net-execution-policies.md) — why auras use `ServerOnly`
- [Common Abilities](../patterns/common-abilities.md) — more passive ability patterns
