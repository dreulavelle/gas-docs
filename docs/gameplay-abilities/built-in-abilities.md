---
title: Built-in Abilities
description: Walkthrough of the engine's built-in ability examples — UGameplayAbility_CharacterJump and UGameplayAbility_Montage.
---

# Built-in Abilities

The engine ships with two concrete `UGameplayAbility` subclasses. They are intentionally simple — not production-ready frameworks, but instructive examples that show how to wrap common gameplay patterns into the GAS lifecycle.

## UGameplayAbility_CharacterJump

This ability wraps `ACharacter::Jump()` into GAS. It is a clean example of how to take an existing character action and make it GAS-aware, gaining access to cooldowns, costs, tags, and the full activation check system.

### How It Works

The class overrides four functions:

```cpp
UCLASS()
class UGameplayAbility_CharacterJump : public UGameplayAbility
{
    virtual bool CanActivateAbility(...) const override;
    virtual void ActivateAbility(...) override;
    virtual void InputReleased(...) override;
    virtual void CancelAbility(...) override;
};
```

**CanActivateAbility** — runs the standard checks (via `Super::CanActivateAbility`), then adds a character-specific check: is the character able to jump right now? This calls into `ACharacter::CanJump()`, which checks movement mode, physics, and other character-level constraints.

```cpp
bool UGameplayAbility_CharacterJump::CanActivateAbility(...) const
{
    if (!Super::CanActivateAbility(...))
    {
        return false;
    }

    const ACharacter* Character = CastChecked<ACharacter>(
        ActorInfo->AvatarActor.Get());
    return Character->CanJump();
}
```

**ActivateAbility** — gets the character from actor info, calls `Character->Jump()`, and that's it. The ability stays active until the input is released or it is cancelled.

```cpp
void UGameplayAbility_CharacterJump::ActivateAbility(...)
{
    if (HasAuthorityOrPredictionKey(ActorInfo, &ActivationInfo))
    {
        if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }

        ACharacter* Character = CastChecked<ACharacter>(
            ActorInfo->AvatarActor.Get());
        Character->Jump();
    }
}
```

**InputReleased** — when the jump button is released, calls `Character->StopJumping()` and ends the ability.

```cpp
void UGameplayAbility_CharacterJump::InputReleased(...)
{
    if (ActorInfo && ActorInfo->AvatarActor.IsValid())
    {
        ACharacter* Character = CastChecked<ACharacter>(
            ActorInfo->AvatarActor.Get());
        Character->StopJumping();
    }

    // End the ability
    CancelAbility(Handle, ActorInfo, ActivationInfo, true);
}
```

**CancelAbility** — stops jumping if the ability is externally cancelled (e.g., stun while jumping).

### What You Can Learn From It

1. **The pattern of wrapping existing character actions in GAS.** You do not rewrite the jump logic — you delegate to `ACharacter::Jump()` and let GAS handle the meta-game (costs, cooldowns, tags).

2. **Custom CanActivateAbility checks.** It adds game-specific checks on top of the standard tag/cooldown/cost checks.

3. **Input forwarding.** It uses `InputReleased` to react to the player letting go of the button while the ability is active.

4. **HasAuthorityOrPredictionKey.** The check before committing ensures the ability only does work when it has network authority or a valid prediction key. This is a pattern worth following in your own abilities.

!!! tip "Should you use this class directly?"
    You could, but most projects create their own jump ability (often as a Blueprint) that adds things like double-jump logic, anim montages, landing effects, and coyote time. This class is more useful as a reference than a base class.

## UGameplayAbility_Montage

This ability plays a montage and applies Gameplay Effects while the montage is active. It was originally designed as an example of a **NonInstanced** ability (operating on the CDO with no per-instance state).

### How It Works

```cpp
UCLASS()
class UGameplayAbility_Montage : public UGameplayAbility
{
    virtual void ActivateAbility(...) override;

    UPROPERTY(EditDefaultsOnly)
    UAnimMontage* MontageToPlay;

    UPROPERTY(EditDefaultsOnly)
    float PlayRate;

    UPROPERTY(EditDefaultsOnly)
    FName SectionName;

    UPROPERTY(EditDefaultsOnly)
    TArray<TSubclassOf<UGameplayEffect>> GameplayEffectClassesWhileAnimating;

    void OnMontageEnded(UAnimMontage* Montage, bool bInterrupted,
        TWeakObjectPtr<UAbilitySystemComponent> ASC,
        TArray<FActiveGameplayEffectHandle> AppliedEffects);
};
```

**ActivateAbility** — plays the montage on the avatar's anim instance and applies all `GameplayEffectClassesWhileAnimating` effects to the owner. It binds `OnMontageEnded` to clean up when the montage finishes.

**OnMontageEnded** — removes all the effects that were applied when the montage started. This creates a "effects are active only while the animation plays" pattern.

### Design Notes

This class was designed around the NonInstanced pattern — it does not use ability tasks (which require instancing) and instead manually interacts with the anim instance and ASC. The montage playback and callback binding is done directly rather than through `PlayMontageAndWait`.

!!! warning "NonInstanced is deprecated"
    As noted in [Instancing Policy](instancing-policy.md), `NonInstanced` is deprecated as of UE 5.5. For new code, use `InstancedPerActor` and the `PlayMontageAndWait` task, which handles the montage lifecycle much more cleanly.

### What You Can Learn From It

1. **The "apply effects while animating" pattern.** This is useful for buffs/debuffs that should only be active during an animation (e.g., a "parry window" effect that grants immunity only during the parry frames).

2. **Manual montage management.** If you ever need to play montages without tasks (rare, but possible in specialized contexts), this shows the direct approach.

3. **How NonInstanced abilities work.** Even though the approach is deprecated, understanding it helps when reading older code.

### A Modern Alternative

Here is how you would implement the same "montage with temporary effects" pattern using modern GAS conventions:

```cpp
void UGA_MontageWithEffects::ActivateAbility(...)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Apply temporary effects
    for (const auto& EffectClass : EffectsWhileAnimating)
    {
        FActiveGameplayEffectHandle EffectHandle =
            BP_ApplyGameplayEffectToOwner(EffectClass);
        ActiveEffectHandles.Add(EffectHandle);
    }

    // Play montage via task
    auto* MontageTask = UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
        this, NAME_None, MontageToPlay, PlayRate, SectionName);

    MontageTask->OnCompleted.AddDynamic(this, &ThisClass::OnFinished);
    MontageTask->OnBlendOut.AddDynamic(this, &ThisClass::OnFinished);
    MontageTask->OnInterrupted.AddDynamic(this, &ThisClass::OnFinished);
    MontageTask->OnCancelled.AddDynamic(this, &ThisClass::OnFinished);
    MontageTask->ReadyForActivation();
}

void UGA_MontageWithEffects::OnFinished()
{
    // Remove temporary effects
    for (const auto& EffectHandle : ActiveEffectHandles)
    {
        BP_RemoveGameplayEffectFromOwnerWithHandle(EffectHandle);
    }
    ActiveEffectHandles.Reset();

    K2_EndAbility();
}
```

This approach uses `InstancedPerActor`, the `PlayMontageAndWait` task, and member variables — all things that the original NonInstanced version could not do.
