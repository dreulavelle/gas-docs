---
title: Lifecycle and Activation
description: Deep dive into the Gameplay Ability lifecycle — activation checks, commit, cancel, end, and the many ways abilities can be triggered.
---

# Lifecycle and Activation

Every Gameplay Ability follows the same fundamental lifecycle: something tries to activate it, the system runs a gauntlet of checks, and if everything passes, the ability enters its active state. From there it runs its logic, commits its costs, and eventually ends. Understanding this flow is essential — it is the backbone of everything else in this section.

## The Big Picture

Here is the full lifecycle at a glance. We will break down each step in detail below.

```
TryActivateAbility()
  └─ CanActivateAbility()           ← All the checks happen here
       ├─ Tag requirements
       ├─ Blocked ability tags
       ├─ Cooldown check
       ├─ Cost check
       ├─ Net role / ShouldActivateAbility
       └─ Blueprint K2_CanActivateAbility (if implemented)
  └─ CallActivateAbility()
       └─ PreActivate()              ← Boilerplate init (sets tags, blocks)
            └─ ActivateAbility()     ← YOUR CODE GOES HERE
                 ├─ CommitAbility()  ← Spend cost, start cooldown
                 ├─ ... do work ...
                 └─ EndAbility()     ← MUST be called when done
```

## CanActivateAbility — The Full Check Sequence { #the-full-check-sequence }

`CanActivateAbility` is a `const` function with no side effects. It is safe to call from UI, AI, or anywhere else you need to check whether an ability _could_ be activated without actually doing it. Here is the complete sequence of checks, in order, as they appear in the engine source:

### 1. Actor Info Validity

The `FGameplayAbilityActorInfo` must be valid. The owning actor must exist and the ASC must be initialized. If there is no valid avatar actor, activation fails.

### 2. ShouldActivateAbility (Net Role)

`ShouldActivateAbility` checks whether this network role should be activating this ability, based on the `NetExecutionPolicy`:

| Net Execution Policy | Who Activates |
|:---|:---|
| `LocalPredicted` | Client predicts, server confirms |
| `LocalOnly` | Only the locally controlled client/server |
| `ServerInitiated` | Server only (client receives via replication) |
| `ServerOnly` | Server only, never runs on client |

### 3. Tag Requirements

`DoesAbilitySatisfyTagRequirements` checks **six** tag containers against the owner's current tag state:

| Container | What It Does |
|:---|:---|
| `ActivationRequiredTags` | Owner **must have all** of these tags |
| `ActivationBlockedTags` | Owner **must not have any** of these tags |
| `SourceRequiredTags` | Source actor must have all of these |
| `SourceBlockedTags` | Source actor must not have any of these |
| `TargetRequiredTags` | Target actor must have all of these |
| `TargetBlockedTags` | Target actor must not have any of these |

It also checks whether this ability's tags are currently **blocked** by another active ability's `BlockAbilitiesWithTag` container.

!!! warning "Activation Failed Tags"
    When `CanActivateAbility` fails, it can write to an optional `FGameplayTagContainer* OptionalRelevantTags` output parameter. This tells the caller _why_ activation failed (e.g., `Ability.ActivationFail.Cooldown`, `Ability.ActivationFail.Cost`, `Ability.ActivationFail.TagsBlocked`). This is incredibly useful for UI feedback — "Ability on cooldown!", "Not enough mana!", etc.

### 4. Cooldown Check

`CheckCooldown` looks for any active Gameplay Effect whose granted tags match the ability's `CooldownGameplayEffectClass` tags. If such an effect is active, the ability is still on cooldown and cannot be activated.

See [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) for the full picture on how cooldown GEs work.

### 5. Cost Check

`CheckCost` evaluates the ability's `CostGameplayEffectClass` to see if the owner has enough resources (mana, stamina, etc.) to pay for activation. It applies the cost effect in a "dry run" mode — checking whether the attribute modifications would be valid without actually committing them.

### 6. Blueprint CanActivateAbility

If the Blueprint has implemented the `K2_CanActivateAbility` event, it is called last. This lets designers add custom checks (e.g., "can only activate while grounded", "can only activate when a specific weapon is equipped") without touching C++.

!!! tip "Checking from outside"
    Since `CanActivateAbility` is `const` and has no side effects, you can safely call it from a UI widget to gray out ability icons, from an AI behavior tree to decide whether to attempt activation, or from any other external code.

## ActivateAbility — Where Your Logic Lives

Once all checks pass, the system calls `CallActivateAbility`, which does some boilerplate setup (`PreActivate`) and then calls `ActivateAbility`. This is the function you override in your ability subclass. In Blueprint, this is the `ActivateAbility` or `ActivateAbilityFromEvent` event.

`PreActivate` handles:

- Incrementing the `ActiveCount` on the spec
- Applying `ActivationOwnedTags` to the owner
- Canceling other abilities that match `CancelAbilitiesWithTag`
- Setting up blocking for abilities that match `BlockAbilitiesWithTag`
- Registering the `OnGameplayAbilityEnded` delegate

Your `ActivateAbility` override should:

1. Do whatever the ability does (spawn tasks, play montages, apply effects)
2. Call `CommitAbility` at the right time
3. Call `EndAbility` when the ability is done

```cpp
void UMyFireballAbility::ActivateAbility(
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

    // Spawn a fireball, play a montage, apply effects, etc.
    // When the fireball logic completes, call EndAbility.
}
```

### ActivateAbilityFromEvent

If the ability was triggered by a gameplay event (via `AbilityTriggers`), the system calls `ActivateAbilityFromEvent` instead, passing the `FGameplayEventData` payload. In Blueprint, this is a separate event node that exposes the event data directly.

## CommitAbility — Spending Resources { #commitability }

`CommitAbility` is the point of no return. It does three things:

1. **CommitCheck** — runs `CanActivateAbility` one more time (the "last chance to fail")
2. **CommitExecute** — applies the cost GE and the cooldown GE
3. Returns `true` if successful, `false` if the checks failed

You **must** call `CommitAbility` (or its split variants) from your `ActivateAbility` implementation. If you forget, no cost is spent and no cooldown is applied.

### Splitting Commit: CommitCost and CommitCooldown

Sometimes you want to commit cost and cooldown at different times. Classic example: a channeled ability that starts the cooldown when channeling begins, but only deducts the resource cost when the channel completes.

```cpp
// Start channeling — cooldown starts now
if (!CommitAbilityCooldown(Handle, ActorInfo, ActivationInfo, false))
{
    EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
    return;
}

// ... later, when the channel completes ...

// Now pay the cost
if (!CommitAbilityCost(Handle, ActorInfo, ActivationInfo))
{
    // Channel finished but we can't pay — cancel the effect
    EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
    return;
}
```

`CommitAbilityCooldown` also has a `ForceCooldown` parameter. When `true`, it applies the cooldown even if `CheckCooldown` would fail (useful for forcing a cooldown onto an ability from external code).

The `BroadcastCommitEvent` parameter on both functions controls whether the ASC broadcasts the commit event that tasks like `WaitAbilityCommit` listen for. Usually you want this to be `false` when splitting commits, and only broadcast on the final commit call.

## CancelAbility — External Interruption { #cancelability }

`CancelAbility` is called when something _outside_ the ability wants to stop it. Stuns, death, another ability with `CancelAbilitiesWithTag` — these all trigger cancellation.

```cpp
virtual void CancelAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    bool bReplicateCancelAbility);
```

When an ability is cancelled:

1. The `OnGameplayAbilityCancelled` delegate fires
2. All active [Ability Tasks](ability-tasks.md) receive cancellation
3. `EndAbility` is called with `bWasCancelled = true`

You can prevent cancellation by calling `SetCanBeCanceled(false)` on an instanced ability. This is useful during critical sections (e.g., a finisher animation that should not be interrupted).

!!! note "Internal vs External Cancel"
    If your ability wants to cancel _itself_, call `K2_CancelAbility()` from Blueprint or `CancelAbility` in C++. From _outside_ the ability, use `AbilitySystemComponent->CancelAbilityHandle(Handle)` or `CancelAbilities` with a tag filter.

## EndAbility — Cleaning Up { #endability }

`EndAbility` is the final step. It cleans up the ability, removes `ActivationOwnedTags`, ends all active tasks, removes tracked gameplay cues, and broadcasts the ended delegate.

```cpp
virtual void EndAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    bool bReplicateEndAbility,
    bool bWasCancelled);
```

In Blueprint, call **End Ability** to replicate the end to the other side, or **End Ability Locally** to only end the local instance (useful for predicted abilities where you want the server to end naturally rather than being force-ended by the client).

!!! danger "The #1 GAS Bug: Forgetting to Call EndAbility"
    If you never call `EndAbility`, your ability stays "active" forever. This means:

    - `ActivationOwnedTags` are never removed (your character might be permanently "casting")
    - `BlockAbilitiesWithTag` stays in effect (other abilities remain blocked)
    - `ActiveCount` on the spec never decrements
    - The ability instance is never cleaned up (memory leak for InstancedPerExecution)
    - The ability cannot be re-activated (for InstancedPerActor with single activation)

    **Every single code path in your ability must eventually reach EndAbility.** If you have branching logic, make sure all branches end. If you use ability tasks, make sure every delegate output path ends the ability.

### The K2_OnEndAbility Callback

Both C++ and Blueprint can hook into the end of an ability. In Blueprint, implement the `OnEndAbility` event. It receives a `bWasCancelled` boolean so you can distinguish between normal completion and interruption.

```cpp
// C++ — override EndAbility for cleanup
void UMyAbility::EndAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    bool bReplicateEndAbility,
    bool bWasCancelled)
{
    // Your cleanup here (before calling Super)

    Super::EndAbility(Handle, ActorInfo, ActivationInfo,
                      bReplicateEndAbility, bWasCancelled);
}
```

## Activation Methods

There are several ways to trigger an ability. Each suits different use cases.

### By Class

The most direct approach. You have the ability class and you tell the ASC to activate it:

```cpp
AbilitySystemComponent->TryActivateAbilityByClass(UMyFireballAbility::StaticClass());
```

This finds the granted `FGameplayAbilitySpec` that matches the class and attempts activation.

### By Spec Handle

If you already have a handle to a specific spec (e.g., from when you granted the ability), you can activate it directly:

```cpp
AbilitySystemComponent->TryActivateAbility(SpecHandle);
```

### By Input

When abilities are bound to input (either through `InputID` or Enhanced Input tag mapping), pressing the bound input calls `TryActivateAbility` on the matching spec. See [Input Binding](input-binding.md) for the full setup.

### By Gameplay Event

You can send a `FGameplayEventData` payload to the ASC, which will activate any ability whose `AbilityTriggers` match the event tag:

```cpp
FGameplayEventData EventData;
EventData.EventTag = FGameplayTag::RequestGameplayTag(FName("Event.Combo.Hit"));
EventData.Instigator = GetOwner();
EventData.Target = HitActor;
EventData.EventMagnitude = DamageAmount;

UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(
    GetOwner(), EventData.EventTag, EventData);
```

The triggered ability receives this data through its `ActivateAbilityFromEvent` override (or the Blueprint `ActivateAbilityFromEvent` event).

### By Tag Trigger

Abilities can be configured to auto-activate when certain tags are added to or present on the owner. This is set up through the `AbilityTriggers` array on the ability CDO:

| Trigger Source | Behavior |
|:---|:---|
| `GameplayEvent` | Activates when a gameplay event with the matching tag is received |
| `OwnedTagAdded` | Activates when the trigger tag is **added** to the owner (does not cancel on removal) |
| `OwnedTagPresent` | Activates when the trigger tag is **present** on the owner, and **cancels** if the tag is removed |

`OwnedTagPresent` is useful for reactive abilities like "while stunned, play a stun animation" — the ability auto-activates when the stun tag appears and auto-cancels when it is removed.

## Ability State Booleans

The ability instance tracks several state flags that are useful for runtime queries:

| Property | Meaning |
|:---|:---|
| `bIsActive` | The ability is currently executing |
| `bIsAbilityEnding` | `EndAbility` has been called but hasn't finished yet |
| `bIsCancelable` | Whether the ability can be cancelled right now |
| `bIsBlockingOtherAbilities` | Whether the ability's block tags are currently in effect |
| `RemoteInstanceEnded` | The remote (server/client) instance has ended but the local one is still running |

## Other Callbacks Worth Knowing

Beyond the main lifecycle, `UGameplayAbility` provides several notification callbacks:

| Callback | When It Fires |
|:---|:---|
| `OnGiveAbility` | When the ability is first granted to an ASC |
| `OnRemoveAbility` | When the ability is removed from the ASC |
| `OnAvatarSet` | When the avatar actor changes (e.g., possess a new pawn) |
| `ConfirmActivateSucceed` | On a predictive client, when the server confirms the activation |
| `NotifyAvatarDestroyed` | When the avatar actor is destroyed while the ability is active |
| `InputPressed` / `InputReleased` | Input events forwarded to an already-active ability |

These are virtual and overridable. `OnGiveAbility` and `OnAvatarSet` are great places for one-time setup (e.g., binding to attribute change delegates, caching component references).

## Putting It All Together

Here is a minimal but complete ability that demonstrates the full lifecycle:

```cpp
UCLASS()
class UGA_Fireball : public UGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Fireball();

    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData) override;

    void OnMontageCompleted();
    void OnMontageCancelled();

protected:
    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UAnimMontage> CastMontage;

    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<AActor> ProjectileClass;
};
```

```cpp
UGA_Fireball::UGA_Fireball()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

void UGA_Fireball::ActivateAbility(
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
            this, NAME_None, CastMontage);

    MontageTask->OnCompleted.AddDynamic(this, &UGA_Fireball::OnMontageCompleted);
    MontageTask->OnCancelled.AddDynamic(this, &UGA_Fireball::OnMontageCancelled);
    MontageTask->OnInterrupted.AddDynamic(this, &UGA_Fireball::OnMontageCancelled);
    MontageTask->ReadyForActivation();
}

void UGA_Fireball::OnMontageCompleted()
{
    // Spawn projectile, apply effects, etc.
    K2_EndAbility();
}

void UGA_Fireball::OnMontageCancelled()
{
    K2_EndAbility();
}
```

Note how every delegate output path from the montage task leads to `EndAbility`. This is the pattern you should follow.
