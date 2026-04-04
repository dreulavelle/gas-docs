---
title: "Example: Sprint Toggle"
icon: material/run
description: Toggle ability that increases movement speed while held, drains stamina over time, and demonstrates infinite-duration effects applied and removed on activate/deactivate.
---

# Example: Sprint Toggle

<div class="example-badges" markdown>
  <span class="badge badge--beginner">Beginner</span>
</div>

A toggle ability that increases movement speed while the input is held and drains stamina continuously. When you release the button (or run out of stamina), the sprint deactivates and movement returns to normal. This demonstrates toggle/held abilities, infinite-duration effects managed by the ability lifecycle, and periodic attribute modification.

## What We're Building

- Press to start sprinting, release to stop
- Increases movement speed by 50% while active
- Drains stamina continuously (3 per 0.5 seconds)
- Blocked by root, stun, and death
- Grants `State.Sprinting` tag while active
- Automatically ends if stamina reaches 0

## Prerequisites

This example assumes you've completed [Project Setup](../getting-started/project-setup.md) and have:

- A character with an **Ability System Component**
- An **AttributeSet** with at minimum Health, Stamina, and a MoveSpeed attribute (or you use a `MoveSpeedMultiplier` attribute your movement component reads)
- A **base ability class** (`YourProjectGameplayAbility`) with an InputTag property
- An **input binding system** that routes Enhanced Input actions to abilities by tag

If any of that is missing, start with [Project Setup](../getting-started/project-setup.md).

---

## Step 1: Create the Effects

We need two effects: one for the speed buff and one for the stamina drain. Both are **Infinite** duration (no natural expiration) because the ability itself manages their lifetime — applying them on activation and removing them on deactivation.

### GE_Sprint_SpeedBuff

This effect boosts movement speed while sprinting. It also grants tags that mark the character's state.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Infinite |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.MoveSpeed` |
| **Modifiers[0] — Modifier Op** | `Multiply (Additive)` |
| **Modifiers[0] — Magnitude** | Scalable Float: `0.5` |

!!! info "Why Multiply (Additive) with 0.5?"
    `Multiply (Additive)` adds to a multiplier pool before applying. A value of `0.5` means "+50% move speed." If another effect also adds `0.3`, the total multiplier becomes `1 + 0.5 + 0.3 = 1.8` (80% increase). This is different from `Multiply (Compound)`, which chains multiplications (`1.5 * 1.3 = 1.95`). Additive is usually what you want for stacking buffs. See [Modifiers](../gameplay-effects/modifiers.md) for the full breakdown of `EGameplayModOp` types.

**Tags (via GE Components):**

| GE Component | Tag | Purpose |
|:---|:---|:---|
| **TargetTagsGameplayEffectComponent** — Granted Tags | `State.Sprinting` | Marks the character as sprinting (other systems can check this) |
| **TargetTagsGameplayEffectComponent** — Granted Tags | `Status.Buff.Haste` | Categorizes this as a haste buff (for UI, cleanse abilities, etc.) |

### GE_Sprint_StaminaDrain

This effect periodically drains stamina while sprinting.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `999.0` (effectively infinite, but allows Period) |
| **Period** | `0.5` seconds |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.Stamina` |
| **Modifiers[0] — Modifier Op** | `Add (Base)` |
| **Modifiers[0] — Magnitude** | Scalable Float: `-3.0` |

!!! warning "Why Has Duration instead of Infinite?"
    Infinite-duration effects do not support periodic execution. If you set Duration Policy to Infinite and add a Period, the periodic tick won't fire. Use Has Duration with a very large value (like 999 seconds) when you need periodic execution that the ability will manually remove. The ability removes this effect long before the 999-second duration expires, so the actual number doesn't matter — it just needs to be non-infinite.

---

## Step 2: Create the Ability

**Asset:** `GA_Sprint`

Create a new Blueprint class with your `YourProjectGameplayAbility` as the parent.

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Input Tag** | `InputTag.Movement.Sprint` | Maps to your sprint input |
| **Ability Tags** | `Ability.Movement.Sprint` | Identifies this ability |
| **Activation Blocked Tags** | `CrowdControl.Root`, `CrowdControl.Stun`, `State.Dead` | Can't sprint while rooted, stunned, or dead |
| **Instancing Policy** | `InstancedPerActor` | Required — we store effect handles as member variables |
| **Net Execution Policy** | `LocalPredicted` | Sprint should feel instant on the client |
| **Cost Gameplay Effect** | *(leave empty)* | We handle stamina drain manually via the periodic effect |
| **Cooldown Gameplay Effect** | *(leave empty)* | No cooldown on sprint |

!!! note "No Cost GE?"
    Unlike most abilities, sprint doesn't have a one-time cost. Instead, it drains stamina continuously via `GE_Sprint_StaminaDrain`. If we used a Cost GE, it would deduct stamina once on activation and then the periodic drain would be separate — doubling the initial cost. The periodic effect handles everything.

### Event Graph

=== "Blueprint"
    ```
    Event ActivateAbility
        |
        +-- Apply GE to Self (GE_Sprint_SpeedBuff)
        |     \-- Store Handle -> SpeedBuffHandle (variable)
        |
        +-- Apply GE to Self (GE_Sprint_StaminaDrain)
        |     \-- Store Handle -> StaminaDrainHandle (variable)
        |
        +-- Wait Input Release (bTestAlreadyReleased = false)
        |     \-- On Release -> [Go to Deactivate]
        |
        +-- Wait for Attribute Change Threshold
        |     +-- Attribute: YourProjectAttributeSet.Stamina
        |     +-- Comparison: LessThanOrEqual
        |     +-- Value: 0.0
        |     +-- bTriggerOnce: true
        |     \-- On Change (bMatchesComparison = true) -> [Go to Deactivate]
        |
        [Deactivate]:
        +-- Remove Active GE (SpeedBuffHandle)
        +-- Remove Active GE (StaminaDrainHandle)
        \-- End Ability (Cancelled = false)
    ```

    **Variables to add:**

    | Variable | Type |
    |:---|:---|
    | `SpeedBuffHandle` | `FActiveGameplayEffectHandle` |
    | `StaminaDrainHandle` | `FActiveGameplayEffectHandle` |

    !!! tip "Two tasks, one outcome"
        Both `Wait Input Release` and `Wait for Attribute Change Threshold` run simultaneously. Whichever fires first triggers the deactivation. This is the "race" pattern — two ability tasks listening in parallel, both leading to the same cleanup logic.

=== "C++"
    ```cpp
    // GA_Sprint.h
    #pragma once

    #include "CoreMinimal.h"
    #include "YourProjectGameplayAbility.h"
    #include "ActiveGameplayEffectHandle.h"
    #include "GA_Sprint.generated.h"

    UCLASS()
    class UGA_Sprint : public UYourProjectGameplayAbility
    {
        GENERATED_BODY()

    public:
        UGA_Sprint();

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
        UPROPERTY(EditDefaultsOnly, Category = "Effects")
        TSubclassOf<UGameplayEffect> SpeedBuffEffectClass;

        UPROPERTY(EditDefaultsOnly, Category = "Effects")
        TSubclassOf<UGameplayEffect> StaminaDrainEffectClass;

    private:
        UFUNCTION()
        void OnInputReleased(float TimeHeld);

        UFUNCTION()
        void OnStaminaDepleted(bool bMatches, float CurrentValue);

        void RemoveEffectsAndEnd();

        FActiveGameplayEffectHandle SpeedBuffHandle;
        FActiveGameplayEffectHandle StaminaDrainHandle;
    };
    ```

    ```cpp
    // GA_Sprint.cpp
    #include "GA_Sprint.h"
    #include "AbilitySystemComponent.h"
    #include "Abilities/Tasks/AbilityTask_WaitInputRelease.h"
    #include "Abilities/Tasks/AbilityTask_WaitAttributeChangeThreshold.h"

    UGA_Sprint::UGA_Sprint()
    {
        InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
        NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    }

    void UGA_Sprint::ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData)
    {
        if (!HasAuthorityOrPredictionKey(ActorInfo, &ActivationInfo))
        {
            return;
        }

        UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();
        if (!ASC)
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }

        // Apply the speed buff (infinite duration)
        if (SpeedBuffEffectClass)
        {
            FGameplayEffectSpecHandle SpecHandle =
                MakeOutgoingGameplayEffectSpec(SpeedBuffEffectClass, GetAbilityLevel());
            SpeedBuffHandle = ApplyGameplayEffectSpecToOwner(
                Handle, ActorInfo, ActivationInfo, SpecHandle);
        }

        // Apply the stamina drain (periodic)
        if (StaminaDrainEffectClass)
        {
            FGameplayEffectSpecHandle SpecHandle =
                MakeOutgoingGameplayEffectSpec(StaminaDrainEffectClass, GetAbilityLevel());
            StaminaDrainHandle = ApplyGameplayEffectSpecToOwner(
                Handle, ActorInfo, ActivationInfo, SpecHandle);
        }

        // Wait for input release
        UAbilityTask_WaitInputRelease* WaitRelease =
            UAbilityTask_WaitInputRelease::WaitInputRelease(this);
        WaitRelease->OnRelease.AddDynamic(this, &UGA_Sprint::OnInputReleased);
        WaitRelease->ReadyForActivation();

        // Wait for stamina to hit 0
        UAbilityTask_WaitAttributeChangeThreshold* WaitStamina =
            UAbilityTask_WaitAttributeChangeThreshold::WaitForAttributeChangeThreshold(
                this,
                UYourProjectAttributeSet::GetStaminaAttribute(),
                EWaitAttributeChangeComparison::LessThanOrEqual,
                0.0f,
                /*bTriggerOnce=*/ true);
        WaitStamina->OnChange.AddDynamic(this, &UGA_Sprint::OnStaminaDepleted);
        WaitStamina->ReadyForActivation();
    }

    void UGA_Sprint::OnInputReleased(float TimeHeld)
    {
        RemoveEffectsAndEnd();
    }

    void UGA_Sprint::OnStaminaDepleted(bool bMatches, float CurrentValue)
    {
        if (bMatches)
        {
            RemoveEffectsAndEnd();
        }
    }

    void UGA_Sprint::RemoveEffectsAndEnd()
    {
        UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();
        if (ASC)
        {
            if (SpeedBuffHandle.IsValid())
            {
                ASC->RemoveActiveGameplayEffect(SpeedBuffHandle);
            }
            if (StaminaDrainHandle.IsValid())
            {
                ASC->RemoveActiveGameplayEffect(StaminaDrainHandle);
            }
        }

        K2_EndAbility();
    }

    void UGA_Sprint::EndAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility,
        bool bWasCancelled)
    {
        // Safety net: if EndAbility is called externally (e.g., ability cancelled),
        // make sure we clean up our effects
        UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();
        if (ASC)
        {
            if (SpeedBuffHandle.IsValid())
            {
                ASC->RemoveActiveGameplayEffect(SpeedBuffHandle);
                SpeedBuffHandle.Invalidate();
            }
            if (StaminaDrainHandle.IsValid())
            {
                ASC->RemoveActiveGameplayEffect(StaminaDrainHandle);
                StaminaDrainHandle.Invalidate();
            }
        }

        Super::EndAbility(Handle, ActorInfo, ActivationInfo,
            bReplicateEndAbility, bWasCancelled);
    }
    ```

!!! danger "Always clean up in EndAbility"
    The `EndAbility` override is a safety net. If something external cancels the ability (stun, death, manual cancellation), the effects still get removed. Without this, a cancelled sprint would leave the speed buff running forever.

---

## Step 3: Wire Input

Sprint uses a **held input** pattern — the ability activates when the button is pressed and uses `WaitInputRelease` to detect when the button is released.

1. **Input Action:** `IA_Sprint` (Value Type: `Digital (Bool)`, Trigger: `Pressed`)
2. **Input Mapping Context:** Map to your sprint key (e.g., Left Shift, left stick click)
3. **InputTag:** `InputTag.Movement.Sprint`
4. **On the ability:** Set InputTag to `InputTag.Movement.Sprint`

!!! info "Pressed, not Held"
    The Input Action trigger should be **Pressed** (fires once on key down), not Held. The ability itself handles the "held" behavior through the `WaitInputRelease` task. If you use the Held trigger, the input system would keep re-firing the activation, which isn't what we want for a toggle.

---

## Step 4: Test

### Basic Sprint Test

1. PIE
2. Press and hold sprint key — character should move faster
3. Check `showdebug abilitysystem`:
    - `State.Sprinting` tag should be present
    - `GE_Sprint_SpeedBuff` should appear in active effects
    - Stamina should be ticking down (3 every 0.5s)
4. Release sprint key — speed returns to normal, tags removed, effects removed
5. Sprint until stamina hits 0 — sprint should auto-cancel

### Edge Cases

| Scenario | Expected Result |
|:---|:---|
| Sprint at 0 stamina | Activates briefly, stamina threshold fires almost immediately, ends |
| Sprint while rooted | Doesn't activate (Activation Blocked Tags) |
| Sprint interrupted by stun | Ability cancelled externally, EndAbility cleans up effects |
| Sprint key tapped quickly | Activates and deactivates rapidly — verify no leaked effects |
| Network test with `net pktlag=100` | Sprint should feel instant on client (LocalPredicted) |

---

!!! tip "Connecting to UI"
    The stamina drain during sprint is a great candidate for a real-time UI bar. Use `UAbilityAsync_WaitAttributeChanged` to drive a stamina progress bar that updates reactively — no Tick polling needed. See [Connecting GAS to UI](../patterns/gas-to-ui.md) for the full pattern.

## The Full Flow

```
Player presses Sprint key
  |
  v
Input System: IA_Sprint -> InputTag.Movement.Sprint -> ASC finds GA_Sprint
  |
  v
GA_Sprint.ActivateAbility()
  +-- Check: Activation Blocked Tags (rooted? stunned? dead?)
  +-- Apply GE_Sprint_SpeedBuff (infinite, +50% MoveSpeed)
  |     +-- Tags granted: State.Sprinting, Status.Buff.Haste
  +-- Apply GE_Sprint_StaminaDrain (periodic, -3 Stamina every 0.5s)
  +-- Start WaitInputRelease task
  +-- Start WaitForAttributeChangeThreshold task (Stamina <= 0)
  |
  v  (some time passes, stamina drains...)
  |
  v  [Player releases key OR stamina hits 0]
  |
  v
RemoveEffectsAndEnd()
  +-- Remove GE_Sprint_SpeedBuff (speed returns to normal)
  +-- Remove GE_Sprint_StaminaDrain (drain stops)
  +-- Tags removed: State.Sprinting, Status.Buff.Haste
  +-- End Ability
```

---

## Variations

??? example "Sprint with minimum stamina threshold"
    Instead of letting sprint activate at 0 stamina (and immediately cancel), add a `CanActivateAbility` override that checks `Stamina > 10` before allowing activation. This prevents the awkward "activate then immediately deactivate" pattern.

??? example "Sprint with acceleration curve"
    Instead of an instant 50% boost, use a second periodic effect that gradually increases `MoveSpeed` over 1-2 seconds. Apply the ramp-up effect on activate and replace it with the full-speed effect after the ramp completes. Or use a curve table with the speed buff modifier for a smooth acceleration.

??? example "Sprint that drains stamina proportionally"
    Use a `Modifier Magnitude Calculation` on the stamina drain instead of a flat -3. The custom MMC can read the character's current MoveSpeed and drain proportionally — sprinting while buffed costs more stamina. See [Magnitude Calculations](../gameplay-effects/magnitude-calculations.md).

---

## Related Pages

- [Gameplay Effects Duration and Lifecycle](../gameplay-effects/duration-and-lifecycle.md) — how Infinite and Has Duration effects work
- [Modifiers](../gameplay-effects/modifiers.md) — `EGameplayModOp` types explained (Additive vs Compound multiply)
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) — `WaitInputRelease` and `WaitForAttributeChangeThreshold` details
- [Input Binding](../gameplay-abilities/input-binding.md) — full input architecture
- [Common Abilities](../patterns/common-abilities.md) — more ability patterns including toggle abilities
