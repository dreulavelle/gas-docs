---
title: Common Abilities
description: Implementation patterns for stun, sprint, ADS, lifesteal, crits, interaction systems, passive abilities, and toggles in GAS.
---

# Common Abilities

These are abilities that appear in nearly every action game. For each pattern, we cover the key setup steps and design decisions. These aren't full tutorials -- they're architectural guidance to get you moving in the right direction.

## Stun / Crowd Control

**Goal:** Apply a stun that prevents the target from using abilities and moving.

### Key Setup

1. **Create a tag:** `CrowdControl.Stun`
2. **Create a Duration GE** (`GE_Stun`) that:
    - Grants the tag `CrowdControl.Stun` to the target (via TargetTagsGameplayEffectComponent)
    - Duration set via SetByCaller so different abilities can stun for different lengths
3. **On your abilities**, add `CrowdControl.Stun` to the **Activation Blocked Tags** -- any ability with this will refuse to activate while stunned
4. **For movement blocking**, check for the stun tag in your character movement component:

```cpp
bool AMyCharacter::CanMove() const
{
    return !AbilitySystemComponent->HasMatchingGameplayTag(
        FGameplayTag::RequestGameplayTag(TEXT("CrowdControl.Stun")));
}
```

### Variations

- **Slow:** Instead of blocking abilities, apply a movement speed modifier via attribute
- **Silence:** Block only abilities tagged with `Ability.Type.Spell`, not physical attacks
- **Root:** Block movement but allow abilities
- **Knockback:** Pair the stun GE with a root motion ability task

### Stun Immunity

Create an Infinite GE that grants `Immunity.CrowdControl.Stun`. Use the [Immunity GE Component](../gameplay-effects/immunity.md) on the stun GE to check for this tag.

---

## Sprint

**Goal:** Hold a button to move faster, draining stamina.

### Key Setup

1. **Create a passive Gameplay Ability** (`GA_Sprint`) with `InputTag.Sprint`
2. On activation, apply an Infinite GE (`GE_Sprint`) that:
    - Modifies `MoveSpeed` attribute (Multiply Additive, e.g., +0.5 for 50% faster)
    - Applies a periodic stamina cost (Period = 0.2s, modifying `Stamina` with negative value)
    - Grants tag `State.Sprinting`
3. **End condition:** End the ability when input is released (use `WaitInputRelease` task) or when stamina reaches 0
4. On ability end, the GE is automatically removed, reverting move speed

### Input Binding

Use the [input binding system](../gameplay-abilities/input-binding.md) with `InputTag.Sprint`. The ability starts on press and you wire the end to release.

### Integration with Character Movement

Read `MoveSpeed` attribute in `GetMaxSpeed()`:

```cpp
float UMyCharacterMovement::GetMaxSpeed() const
{
    float Base = Super::GetMaxSpeed();
    if (const UMyAttributeSet* Attrs = GetOwnerAttributes())
    {
        return Base * Attrs->GetMoveSpeed();
    }
    return Base;
}
```

---

## Aim Down Sights (ADS)

**Goal:** Hold a button to enter an aiming state with different camera, sensitivity, and movement speed.

### Key Setup

1. **Create `GA_AimDownSights`** -- toggle or hold-based ability
2. On activation, apply an Infinite GE that:
    - Grants `State.Aiming` tag
    - Reduces `MoveSpeed` (Multiply Additive, e.g., -0.3 for 30% slower)
3. Use the `State.Aiming` tag in your camera system to switch to ADS camera
4. Use the tag in your input handling to adjust sensitivity

### Camera Integration

```cpp
void AMyCharacter::UpdateCamera(float DeltaTime)
{
    bool bAiming = ASC->HasMatchingGameplayTag(
        FGameplayTag::RequestGameplayTag(TEXT("State.Aiming")));
    TargetFOV = bAiming ? ADSFieldOfView : DefaultFieldOfView;
    CurrentFOV = FMath::FInterpTo(CurrentFOV, TargetFOV, DeltaTime, FOVInterpSpeed);
}
```

---

## Lifesteal

**Goal:** A portion of damage dealt heals the attacker.

### Key Setup

The cleanest approach is to handle this in the damage [Execution Calculation](../gameplay-effects/execution-calculations.md):

1. In your damage ExecCalc, after calculating final damage:
2. Check if the source has a `Stat.LifestealPercent` attribute (or a tag like `Buff.Lifesteal`)
3. Calculate heal amount: `FinalDamage * LifestealPercent`
4. Output the heal as an additional modifier to the *source's* Health attribute

```cpp
// In your ExecCalc, after computing FinalDamage:
float LifestealPct = 0.f;
if (SourceTags->HasTag(LifestealTag))
{
    LifestealPct = GetCapturedAttributeMagnitude(
        LifestealPercentDef, EvaluationParameters);
}

if (LifestealPct > 0.f)
{
    float HealAmount = FinalDamage * LifestealPct;
    // Apply heal to source (not target)
    OutExecutionOutput.AddOutputModifier(
        FGameplayModifierEvaluatedData(
            UMyAttributeSet::GetHealthAttribute(),
            EGameplayModOp::Additive,
            HealAmount));
}
```

### Alternative: Separate Heal GE

If you prefer not to mix it into the ExecCalc, apply a separate instant heal GE to the source in `PostGameplayEffectExecute` after damage is applied. This is slightly less efficient but more modular.

---

## Critical Hits

**Goal:** Attacks have a chance to deal bonus damage.

### Key Setup

Handle crits in the damage [Execution Calculation](../gameplay-effects/execution-calculations.md):

1. Add `Stat.CritChance` and `Stat.CritMultiplier` attributes
2. In the ExecCalc, roll for crit:

```cpp
float CritChance = GetCapturedAttributeMagnitude(CritChanceDef, Params);
float CritMultiplier = GetCapturedAttributeMagnitude(CritMultiplierDef, Params);

bool bCrit = FMath::FRand() < CritChance;
float FinalDamage = BaseDamage * (bCrit ? CritMultiplier : 1.0f);
```

3. Pass the crit flag to the cue via a [custom GameplayEffectContext](../gameplay-cues/cue-parameters.md#passing-custom-data-via-effect-context):

```cpp
FMyGameplayEffectContext* Context = static_cast<FMyGameplayEffectContext*>(
    Spec.GetContext().Get());
Context->bIsCriticalHit = bCrit;
```

4. The damage number cue checks `bIsCriticalHit` to show a bigger/flashier number.

---

## One-Button Interaction System

**Goal:** A single "interact" button that contextually does different things (open door, pick up item, talk to NPC).

### Key Setup

1. **Create `GA_Interact`** -- triggered by `InputTag.Interact`
2. On activation, the ability does a trace or overlap check to find the nearest interactable
3. Interactables implement an interface (`IInteractable`) with `GetInteractionAbility()` that returns a specific ability class
4. The interact ability **grants and activates** the specific interaction ability, then ends itself

```cpp
void UGA_Interact::ActivateAbility(...)
{
    AActor* Target = FindBestInteractable();
    if (!Target) { EndAbility(...); return; }

    if (IInteractable* Interactable = Cast<IInteractable>(Target))
    {
        TSubclassOf<UGameplayAbility> InteractAbility =
            Interactable->GetInteractionAbility();
        FGameplayAbilitySpecHandle Handle =
            ASC->GiveAbility(FGameplayAbilitySpec(InteractAbility));
        ASC->TryActivateAbility(Handle);
    }
    EndAbility(...);
}
```

### Why This Pattern?

Each interactable type (door, chest, NPC) gets its own ability class with its own costs, cooldowns, tags, and logic. The interaction system is just a router.

---

## Passive Abilities

**Goal:** Abilities that are always active and provide ongoing effects.

### Key Setup

1. **Create the ability** with `ActivationPolicy` set to activate on granted (or activate manually at game start)
2. On activation, apply an Infinite GE that provides the passive bonuses
3. The ability never calls `EndAbility` -- it stays active indefinitely
4. If the ability is removed (ungranted), it ends and the Infinite GE is cleaned up

```cpp
void UGA_PassiveHealthRegen::ActivateAbility(...)
{
    if (HasAuthority(&CurrentActivationInfo))
    {
        FGameplayEffectSpecHandle Spec = MakeOutgoingGameplayEffectSpec(
            RegenEffectClass, GetAbilityLevel());
        ApplyGameplayEffectSpecToOwner(CurrentActivationInfo, Spec);
    }
}
```

### Tag-Based Passives

Some passives don't even need an ability. An Infinite GE that grants tags and modifies attributes, applied at character initialization, works fine for simple stat bonuses.

---

## Toggle Abilities

**Goal:** Press once to activate, press again to deactivate.

### Key Setup

1. **Create the ability** with `InputTag.ToggleAbility`
2. Track toggle state using tags:

```cpp
void UGA_ToggleShield::ActivateAbility(...)
{
    if (ASC->HasMatchingGameplayTag(ShieldActiveTag))
    {
        // Already active -- deactivate
        ASC->RemoveActiveGameplayEffectBySourceEffect(
            ShieldEffectClass, ASC);
        EndAbility(...);
    }
    else
    {
        // Activate
        ApplyGameplayEffectSpecToOwner(...);
        // Don't end -- stay active, wait for next input
        WaitInputPress(); // Ability task to listen for next press
    }
}
```

### Alternative: Two Abilities

Some projects use separate activate/deactivate abilities. The activate ability grants a tag that blocks its own re-activation but allows the deactivate ability. The deactivate ability removes the effect and the blocking tag. This is more verbose but clearer in the ability list.

## Related Pages

- [Damage Pipeline](damage-pipeline.md) -- the full damage flow
- [Buff/Debuff System](buff-debuff-system.md) -- duration effect patterns
- [Add an Ability](../recipes/add-ability.md) -- step-by-step ability creation
- [Dodge Roll Example](../examples/dodge-roll.md) -- complete worked example
