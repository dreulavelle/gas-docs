---
title: "Example: Network-Predicted Ability"
icon: material/access-point-network
description: Melee attack with full client-side prediction, demonstrating the predict-confirm-reject cycle, FScopedPredictionWindow, and multiplayer best practices.
---

# Example: Network-Predicted Ability

<div class="example-badges" markdown>
  <span class="badge badge--advanced">Advanced</span>
</div>

A melee attack implemented with proper client-side prediction, showing how GAS handles the predict-confirm-reject cycle. This walks through what happens on the client vs server, how mispredictions are handled, and best practices for responsive multiplayer abilities. If you're building a networked game and want abilities to feel instant without waiting for server round-trips, this is the page.

This is a **C++-only** example focused on networking concepts. Blueprint ability logic works the same way at a high level, but understanding prediction requires seeing what happens under the hood.

## What We're Building

- A melee attack with `LocalPredicted` net execution policy
- Client **predicts**: ability activation, cost deduction, cooldown application, montage playback
- Server **confirms or rejects** the prediction
- On rejection: client rolls back all predicted effects
- Proper use of `FScopedPredictionWindow` for effects applied outside the initial activation window
- Proper use of `CommitAbility` (which is prediction-aware)

## Prerequisites

Everything from [Project Setup](../getting-started/project-setup.md), plus:

- A multiplayer project (dedicated server or listen server)
- Familiarity with the [Melee Attack](melee-attack.md) example (this builds on it)
- Basic understanding of Unreal's client-server replication model

---

## How Prediction Works in GAS

Before looking at code, let's understand the mechanism. GAS prediction is built around **prediction keys** — unique IDs generated on the client that the server uses to match predicted actions with authoritative results.

### The Prediction Key (`FPredictionKey`)

Every predicted action gets a `FPredictionKey`. This key:

- Is generated on the **client** when an ability attempts to activate
- Is sent to the **server** alongside the activation request
- Is stored on any **predicted effects** (GEs, cues, attribute changes)
- Is used to **match** client predictions with server-confirmed results
- Replicates client-to-server, but when replicating server-to-clients, it **only goes back to the originating client** (other clients receive an invalid key)

### The Prediction Window

A prediction key is only valid during a **prediction window** — essentially the initial callstack of `ActivateAbility`. Once `ActivateAbility` returns (or any latent action starts), the prediction window closes. This is why you cannot predict across multiple frames or after a timer fires.

!!! warning "Prediction does not cross frame boundaries"
    Anything that happens in the initial `ActivateAbility` call (before it returns) is inside the prediction window. Anything after — timers, delays, montage callbacks, ability task delegates — is **outside** the window and requires a new `FScopedPredictionWindow` if you need to predict those actions.

### What GAS Predicts Automatically

From the UE 5.7 engine source comments on `FPredictionKey`:

| Predicted | Not Predicted |
|:---|:---|
| Ability activation | GameplayEffect removal |
| Triggered events | Periodic effects (DoT ticks) |
| GE application (attribute mods, tag changes) | Execution Calculations |
| Gameplay Cues (from predicted GEs or standalone) | |
| Montage playback | |
| Movement (via UCharacterMovement) | |

---

## Step 1: The Predicted Ability Class

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Net Execution Policy** | `LocalPredicted` | Client predicts, server confirms |
| **Instancing Policy** | `InstancedPerActor` | Required for storing state across the prediction lifecycle |
| **Net Security Policy** | `ClientOrServer` | Both can attempt activation |
| **Replication Policy** | `ReplicateYes` | Ability state replicates to owner |

The key setting is `NetExecutionPolicy = LocalPredicted`. This tells GAS:

1. When the **owning client** wants to activate, predict locally AND send a request to the server
2. The **server** receives the request, validates it, and confirms or rejects
3. If **confirmed**: the client's predictions were correct, clean up prediction state
4. If **rejected**: roll back all predicted side effects

### Header

```cpp
// GA_PredictedMeleeAttack.h
#pragma once

#include "CoreMinimal.h"
#include "YourProjectGameplayAbility.h"
#include "GA_PredictedMeleeAttack.generated.h"

UCLASS()
class UGA_PredictedMeleeAttack : public UYourProjectGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_PredictedMeleeAttack();

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
    UPROPERTY(EditDefaultsOnly, Category = "Damage")
    TSubclassOf<UGameplayEffect> DamageEffectClass;

    UPROPERTY(EditDefaultsOnly, Category = "Damage")
    float BaseDamageAmount = 25.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Animation")
    TObjectPtr<UAnimMontage> AttackMontage;

private:
    UFUNCTION()
    void OnMontageCompleted();

    UFUNCTION()
    void OnMontageInterrupted();

    UFUNCTION()
    void OnDamageEventReceived(FGameplayEventData Payload);
};
```

### Source

```cpp
// GA_PredictedMeleeAttack.cpp
#include "GA_PredictedMeleeAttack.h"
#include "AbilitySystemComponent.h"
#include "Abilities/Tasks/AbilityTask_PlayMontageAndWait.h"
#include "Abilities/Tasks/AbilityTask_WaitGameplayEvent.h"

UGA_PredictedMeleeAttack::UGA_PredictedMeleeAttack()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

void UGA_PredictedMeleeAttack::ActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData)
{
    // -------------------------------------------------------
    // PREDICTION WINDOW IS OPEN HERE
    // Everything in this callstack (before return) is predicted.
    // -------------------------------------------------------

    if (!HasAuthorityOrPredictionKey(ActorInfo, &ActivationInfo))
    {
        return;
    }

    // CommitAbility is prediction-aware:
    // - On the CLIENT: predicts cost deduction and cooldown application
    // - On the SERVER: actually applies cost and cooldown
    // - If the server rejects, the client's predicted cost/cooldown
    //   are automatically rolled back
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Play montage — this is also predicted.
    // The client starts the montage immediately (feels responsive).
    // The server also plays it. If the server rejects the ability,
    // the client's predicted montage is cancelled.
    UAbilityTask_PlayMontageAndWait* MontageTask =
        UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
            this,
            NAME_None,
            AttackMontage,
            1.0f);

    MontageTask->OnCompleted.AddDynamic(
        this, &UGA_PredictedMeleeAttack::OnMontageCompleted);
    MontageTask->OnInterrupted.AddDynamic(
        this, &UGA_PredictedMeleeAttack::OnMontageInterrupted);
    MontageTask->OnCancelled.AddDynamic(
        this, &UGA_PredictedMeleeAttack::OnMontageInterrupted);
    MontageTask->ReadyForActivation();

    // Wait for the hit event from the anim notify.
    // This fires OUTSIDE the prediction window (it's a callback
    // that happens later in the montage). That's fine — we'll
    // handle the damage application on the server only.
    UAbilityTask_WaitGameplayEvent* EventTask =
        UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
            this,
            FGameplayTag::RequestGameplayTag(
                FName("Event.Montage.MeleeHit")));

    EventTask->EventReceived.AddDynamic(
        this, &UGA_PredictedMeleeAttack::OnDamageEventReceived);
    EventTask->ReadyForActivation();

    // -------------------------------------------------------
    // PREDICTION WINDOW CLOSES when this function returns.
    // After this point, anything we do is NOT automatically
    // predicted unless we open a new FScopedPredictionWindow.
    // -------------------------------------------------------
}

void UGA_PredictedMeleeAttack::OnDamageEventReceived(
    FGameplayEventData Payload)
{
    // This callback fires when the AnimNotify sends the hit event.
    // We are OUTSIDE the original prediction window.
    //
    // IMPORTANT: We apply damage on the SERVER ONLY.
    // The client should NOT predict damage application because:
    // 1. Execution Calculations don't support prediction
    // 2. The target's health is server-authoritative
    // 3. Predicting damage creates complex rollback scenarios

    if (!GetActorInfo().IsNetAuthority())
    {
        // Client: don't apply damage, let the server handle it.
        // The target's health change will replicate down normally.
        return;
    }

    // Server: apply the damage effect
    if (Payload.Target && DamageEffectClass)
    {
        FGameplayEffectSpecHandle SpecHandle =
            MakeOutgoingGameplayEffectSpec(
                DamageEffectClass, GetAbilityLevel());

        if (SpecHandle.IsValid())
        {
            SpecHandle.Data->SetSetByCallerMagnitude(
                FGameplayTag::RequestGameplayTag(
                    FName("SetByCaller.Damage")),
                BaseDamageAmount);

            UAbilitySystemComponent* TargetASC =
                UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(
                    const_cast<AActor*>(
                        Cast<AActor>(Payload.Target.Get())));

            if (TargetASC)
            {
                ApplyGameplayEffectSpecToTarget(
                    GetCurrentAbilitySpecHandle(),
                    GetCurrentActorInfo(),
                    GetCurrentActivationInfo(),
                    SpecHandle,
                    TargetASC);
            }
        }
    }
}

void UGA_PredictedMeleeAttack::OnMontageCompleted()
{
    EndAbility(
        GetCurrentAbilitySpecHandle(),
        GetCurrentActorInfo(),
        GetCurrentActivationInfo(),
        true, false);
}

void UGA_PredictedMeleeAttack::OnMontageInterrupted()
{
    EndAbility(
        GetCurrentAbilitySpecHandle(),
        GetCurrentActorInfo(),
        GetCurrentActivationInfo(),
        true, true);
}

void UGA_PredictedMeleeAttack::EndAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    bool bReplicateEndAbility,
    bool bWasCancelled)
{
    Super::EndAbility(Handle, ActorInfo, ActivationInfo,
        bReplicateEndAbility, bWasCancelled);
}
```

---

## Step 2: Understanding the Timeline

Here's what happens frame-by-frame when a client activates this ability with 100ms network latency:

```
CLIENT (t=0ms)                          SERVER
  |                                       |
  | TryActivateAbility()                  |
  | -> Generate PredictionKey #42         |
  | -> Call ActivateAbility()             |
  |    -> CommitAbility():                |
  |       PREDICT: Stamina 100 -> 85     |
  |       PREDICT: Cooldown tag applied   |
  |    -> Play montage (predicted)        |
  |                                       |
  | Send activation request ----------->  |  (t=50ms, half RTT)
  |                                       |
  | [Client sees montage playing,         |
  |  stamina at 85, on cooldown]          |
  |                                       |  ServerTryActivateAbility()
  |                                       |  -> Validate: can this activate?
  |                                       |  -> CommitAbility():
  |                                       |     AUTHORITY: Stamina 100 -> 85
  |                                       |     AUTHORITY: Cooldown applied
  |                                       |  -> Play montage (authority)
  |                                       |
  |  <----------- Confirm key #42        |  (t=100ms)
  |                                       |
  | OnRep: PredictionKey #42 confirmed   |
  | -> Remove predicted GEs              |
  | -> Server's replicated GEs take over  |
  | [Seamless transition, player          |
  |  noticed nothing]                     |
```

### What Happens on Rejection

```
CLIENT (t=0ms)                          SERVER
  |                                       |
  | TryActivateAbility()                  |
  | -> PredictionKey #42                  |
  | -> PREDICT: Stamina 100 -> 85        |
  | -> PREDICT: Cooldown applied          |
  | -> PREDICT: Montage plays             |
  |                                       |
  | Send activation request ----------->  |
  |                                       |
  |                                       |  ServerTryActivateAbility()
  |                                       |  -> FAILS (stunned, silenced, etc.)
  |                                       |
  |  <----------- REJECT key #42         |
  |                                       |
  | ClientActivateAbilityFailed()         |
  | -> ROLLBACK: Stamina 85 -> 100       |
  | -> ROLLBACK: Cooldown tag removed     |
  | -> ROLLBACK: Montage cancelled        |
  | [Player sees a brief animation        |
  |  that snaps back — "misprediction"]   |
```

---

## Step 3: FScopedPredictionWindow

If you need to predict effects **outside** the initial activation window (like applying a buff mid-montage), use `FScopedPredictionWindow`:

```cpp
void UGA_SomeAbility::OnSomeDelayedCallback()
{
    // We're outside the original prediction window.
    // Open a new one to predict this effect application.
    UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();
    if (!ASC)
    {
        return;
    }

    // This creates a new dependent prediction key
    FScopedPredictionWindow ScopedPrediction(ASC, true);

    // Now we can apply effects that will be predicted
    FGameplayEffectSpecHandle SpecHandle =
        MakeOutgoingGameplayEffectSpec(BuffEffectClass, GetAbilityLevel());
    ApplyGameplayEffectSpecToOwner(
        GetCurrentAbilitySpecHandle(),
        GetCurrentActorInfo(),
        GetCurrentActivationInfo(),
        SpecHandle);

    // When ScopedPrediction goes out of scope, the window closes.
    // The new prediction key is sent to the server as a dependent key.
}
```

!!! danger "Use FScopedPredictionWindow sparingly"
    Every new prediction window creates a new prediction key that must be synchronized with the server. Overusing them increases network traffic and rollback complexity. The best practice is to only predict things that directly affect the player's perceived responsiveness (movement, stamina, cooldowns) and let the server handle everything else (damage to other players, spawned actors).

---

## Step 4: Test

### Simulate Latency

Open the console and add artificial network latency:

```
net pktlag=200
```

This adds 200ms round-trip latency, making prediction behavior visible.

### What to Test

| Test | Expected Behavior |
|:---|:---|
| Attack with `net pktlag=0` | Feels identical to single-player |
| Attack with `net pktlag=200` | Montage starts instantly (predicted), stamina drops instantly, cooldown applies instantly |
| Attack while stunned (server sees stun, client doesn't yet) | Client predicts attack, server rejects, client snaps back |
| Attack with insufficient stamina | CommitAbility fails on both client and server — no misprediction |
| Rapidly attack near cooldown boundary | May see occasional mispredictions if client thinks cooldown expired but server disagrees |

### Observing Prediction in Action

1. Run a listen server with one client
2. On the client, open `showdebug abilitysystem`
3. With `net pktlag=200`, attack and watch:
    - Stamina drops **instantly** (predicted)
    - Cooldown tag appears **instantly** (predicted)
    - After ~200ms, predicted GEs are silently replaced by server-replicated GEs
    - The player sees no visual difference — that's correct prediction working

### Forcing a Misprediction

To deliberately trigger a rollback:

1. On the server console, apply a stun tag to the player: `AbilitySystem.Debug.ApplyTag CrowdControl.Stun`
2. Immediately on the client, try to attack (before the stun tag replicates)
3. The client predicts the attack (montage starts, stamina drops)
4. ~100ms later, the server rejects (player is stunned)
5. The client rolls back: montage stops, stamina returns, cooldown removed

---

## The Full Flow

```
[CLIENT]                                    [SERVER]
   |                                           |
   | Player presses attack                     |
   | TryActivateAbility()                      |
   | -> Generate FPredictionKey                |
   | -> ActivateAbility()                      |
   |    +-- CommitAbility() [PREDICTED]        |
   |    |   +-- Cost GE applied (predicted)    |
   |    |   +-- Cooldown GE applied (predicted)|
   |    +-- PlayMontage (predicted)             |
   |    +-- WaitGameplayEvent started           |
   |                                           |
   | --- RPC: ServerTryActivateAbility ------> |
   |                                           | Validate activation
   |                                           | CommitAbility() [AUTHORITY]
   |                                           | PlayMontage [AUTHORITY]
   |                                           |
   | <---- Confirm / Reject ------------------ |
   |                                           |
   | [IF CONFIRMED]                            |
   | OnRep catches up                          |
   | Predicted GEs silently removed            |
   | Server GEs take over via replication      |
   |                                           |
   | [IF REJECTED]                             |
   | Rollback all predicted GEs                |
   | Cancel predicted montage                  |
   | Ability ends with bWasCancelled=true      |
   |                                           |
   | [LATER: AnimNotify fires]                 |
   | OnDamageEventReceived()                   |
   | -> Client: skip (not authority)           | -> Server: apply damage GE
   |                                           |    to target
   |                                           |
   | [Montage completes]                       |
   | EndAbility() on both sides                |
```

---

## Common Pitfalls

!!! danger "Don't spawn actors in a predicted context"
    If you spawn a projectile inside the prediction window, both the client and server will try to spawn it. You'll get duplicate actors. Instead, either:

    - Spawn cosmetic-only projectiles on the client (no gameplay logic) and real projectiles on the server
    - Use `FScopedPredictionWindow` carefully with `SpawnActorDeferred` and track by prediction key
    - Use the `AbilityTask_SpawnActor` task, which handles this correctly

!!! danger "Don't predict effects that shouldn't roll back"
    If you apply a GE that modifies another player's attributes inside a prediction window, that change will be rolled back on misprediction — but the other player might have already reacted to it. Only predict effects on the **owning player's** ASC.

!!! warning "Execution Calculations are not predicted"
    From the engine source: "Executions do not currently predict, only attribute modifiers." If your damage GE uses an ExecCalc, apply it on the server only (check `IsNetAuthority()`). This is why our example applies damage only on the server in `OnDamageEventReceived`.

!!! warning "Periodic effects are not predicted"
    DoT effects won't tick predictively on the client. The client sees the effect applied (tag granted) but individual ticks only execute on the server. The resulting attribute changes replicate down normally.

---

## Variations

??? example "Predictive projectile spawn"
    For projectiles, use a split approach: the client spawns a cosmetic-only projectile (visual mesh, particles, sound) immediately for responsiveness. The server spawns the real projectile with collision and damage. When the server projectile's position replicates, smoothly blend the cosmetic one to match. On misprediction, destroy the cosmetic projectile.

??? example "WaitNetSync for hit confirmation"
    Use `UAbilityTask_NetworkSyncPoint::WaitNetSync` with `EAbilityTaskNetSyncType::OnlyServerWait` to synchronize at the point where damage should be applied. The server waits for the client's "I hit something" signal before processing damage. The client signals immediately and continues. This ensures the server has the client's targeting data before processing.

    ```cpp
    UAbilityTask_NetworkSyncPoint* SyncTask =
        UAbilityTask_NetworkSyncPoint::WaitNetSync(
            this, EAbilityTaskNetSyncType::OnlyServerWait);
    SyncTask->OnSync.AddDynamic(this, &ThisClass::OnHitSynced);
    SyncTask->ReadyForActivation();
    ```

??? example "LocalOnly abilities"
    For abilities that are purely cosmetic (emotes, inspect weapon), use `NetExecutionPolicy = LocalOnly`. These run only on the owning client and never involve the server. No prediction needed, no confirmation needed.

---

## Related Pages

- [Prediction](../networking/prediction.md) — full deep dive on the prediction system
- [Net Execution Policies](../networking/net-execution-policies.md) — LocalPredicted vs ServerOnly vs LocalOnly vs ServerInitiated
- [Replication Modes](../networking/replication-modes.md) — how ASC state replicates
- [Melee Attack](melee-attack.md) — simpler version without prediction focus
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) — `WaitNetSync` and other network-aware tasks
- [Lifecycle and Activation](../gameplay-abilities/lifecycle-and-activation.md) — CommitAbility and the activation flow
