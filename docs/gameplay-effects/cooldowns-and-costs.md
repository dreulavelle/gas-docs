---
title: Cooldowns and Costs
description: Implementing ability cooldowns and resource costs using Gameplay Effects — cooldown GEs, cost GEs, dynamic values, shared cooldowns, and the commit pattern.
---

# Cooldowns and Costs

Cooldowns and costs are the two economic constraints on Gameplay Abilities. Both are implemented using Gameplay Effects, which makes them consistent with the rest of GAS — they interact with tags, stacking, and modifiers the same way everything else does.

## Cooldowns

A cooldown in GAS is a **Duration GE that grants a tag**. The ability checks for that tag before activating — if the tag is present, the ability is "on cooldown."

### Setting Up a Cooldown GE

A cooldown GE is typically:

- **Duration Policy:** Has Duration
- **Duration Magnitude:** The cooldown time (ScalableFloat, or SetByCaller for dynamic cooldowns)
- **Component:** `UTargetTagsGameplayEffectComponent` granting a tag like `Cooldown.Ability.Fireball`
- **No modifiers needed** — the GE exists purely to grant the cooldown tag for a duration

### The Ability's Cooldown Configuration

On your `UGameplayAbility`, override `GetCooldownGameplayEffect` to return the cooldown GE class:

```cpp
TSubclassOf<UGameplayEffect> UGA_Fireball::GetCooldownGameplayEffect() const
{
    return GE_Fireball_Cooldown;
}
```

Also override `GetCooldownTags` to tell the ability which tags represent its cooldown:

```cpp
const FGameplayTagContainer* UGA_Fireball::GetCooldownTags() const
{
    return &CooldownTags; // Contains Cooldown.Ability.Fireball
}
```

The ability system uses these to:

1. **Check cooldown** — Before activation, it checks if the owning ASC has any active effects granting the cooldown tags
2. **Apply cooldown** — When the ability commits, it applies the cooldown GE to the owner

### Querying Cooldown State

```cpp
// Check if on cooldown
float TimeRemaining = 0.f;
float Duration = 0.f;
Ability->GetCooldownTimeRemainingAndDuration(
    AbilitySpecHandle, ActorInfo, &TimeRemaining, &Duration);

if (TimeRemaining > 0.f)
{
    // On cooldown
}
```

In Blueprint, `GetCooldownTimeRemaining` and `GetCooldownTimeRemainingAndDuration` are both available.

### Listening for Cooldown Changes

To update a UI cooldown indicator, you can listen for tag changes:

```cpp
ASC->RegisterGameplayTagEvent(
    CooldownTag,
    EGameplayTagEventType::NewOrRemoved).AddUObject(
        this, &UMyCooldownWidget::OnCooldownTagChanged);
```

When the tag is added (cooldown starts), query the remaining time. When removed (cooldown ends), reset the UI.

Alternatively, listen for the active GE being added:

```cpp
ASC->OnActiveGameplayEffectAddedDelegateToSelf.AddUObject(
    this, &UMyCooldownWidget::OnActiveEffectAdded);
```

### Cooldown Prediction

Cooldowns are one of the most commonly predicted things in GAS. When a client activates an ability:

1. The client applies the cooldown GE locally as a predicted effect
2. The ability feels immediately responsive — the UI shows the cooldown starting
3. The server independently applies the cooldown
4. When the server's version replicates back, the predicted version is reconciled

If the server denies the ability activation, the predicted cooldown is removed.

!!! info "Prediction Key"
    The prediction system uses `FPredictionKey` to track which predicted effects should be reconciled with server-authoritative ones. You generally don't need to manage this manually — the ability system handles it.

## Costs

A cost in GAS is an **Instant GE with negative modifiers** that reduce a resource attribute (mana, stamina, energy, ammo, etc.).

### Setting Up a Cost GE

A cost GE is typically:

- **Duration Policy:** Instant
- **Modifier:** AddBase with a negative magnitude to the resource attribute

For a fireball that costs 30 mana:

- Modifier: Attribute = `Mana`, Operation = `AddBase`, Magnitude = `-30` (ScalableFloat)

### The Ability's Cost Configuration

On your `UGameplayAbility`, override `GetCostGameplayEffect`:

```cpp
TSubclassOf<UGameplayEffect> UGA_Fireball::GetCostGameplayEffect() const
{
    return GE_Fireball_Cost;
}
```

### The Commit Pattern

The standard flow for checking and applying costs (and cooldowns) is the **commit** pattern:

```cpp
void UGA_Fireball::ActivateAbility(...)
{
    // 1. Check if we can pay the cost
    if (!CheckCost(Handle, ActorInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 2. Do the ability logic (targeting, animation, etc.)
    // ...

    // 3. Commit — applies both cost and cooldown
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 4. Continue with the effect (spawn projectile, etc.)
}
```

`CommitAbility` calls both `CommitCost` and `CommitCooldown` internally. You can also call them separately if you need to apply them at different times:

```cpp
CommitCost(Handle, ActorInfo, ActivationInfo);
// Later...
CommitCooldown(Handle, ActorInfo, ActivationInfo);
```

!!! tip "Check Before Commit"
    Always call `CheckCost` (and `CheckCooldown`) before `CommitAbility`. The commit functions will also check, but separating the check lets you provide better feedback to the player (e.g., "Not enough mana!" vs. just failing silently).

### CheckCost Implementation

Under the hood, `CheckCost` creates a temporary spec from the cost GE and checks if applying the modifiers would bring any attribute below its minimum. It does **not** actually apply the cost — that only happens in `CommitCost`.

## Dynamic Cooldowns with SetByCaller

Often you want cooldown durations that change at runtime — reduced by stats, modified by talents, or different per ability rank.

The pattern: use [SetByCaller](set-by-caller.md) for the cooldown GE's duration.

**Step 1:** Create the cooldown GE with `DurationMagnitude` set to `SetByCaller` with a tag like `SetByCaller.Cooldown`.

**Step 2:** In your ability, override `ApplyCooldown` to set the value:

```cpp
void UGA_Fireball::ApplyCooldown(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo) const
{
    // GetCooldownGameplayEffect() returns a TSubclassOf<UGameplayEffect>
    TSubclassOf<UGameplayEffect> CooldownGEClass = GetCooldownGameplayEffect();
    if (CooldownGEClass)
    {
        FGameplayEffectSpecHandle SpecHandle =
            MakeOutgoingGameplayEffectSpec(CooldownGEClass, GetAbilityLevel());

        // Set the cooldown duration dynamically
        float CooldownDuration = CalculateCooldownDuration(); // Your logic
        SpecHandle.Data->SetSetByCallerMagnitude(
            FGameplayTag::RequestGameplayTag("SetByCaller.Cooldown"),
            CooldownDuration);

        ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, SpecHandle);
    }
}
```

## Dynamic Costs with SetByCaller

Same pattern for costs that vary at runtime:

```cpp
void UGA_Fireball::ApplyCost(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo) const
{
    // GetCostGameplayEffect() returns a TSubclassOf<UGameplayEffect>
    TSubclassOf<UGameplayEffect> CostGEClass = GetCostGameplayEffect();
    if (CostGEClass)
    {
        FGameplayEffectSpecHandle SpecHandle =
            MakeOutgoingGameplayEffectSpec(CostGEClass, GetAbilityLevel());

        float ManaCost = CalculateManaCost(); // Your logic
        SpecHandle.Data->SetSetByCallerMagnitude(
            FGameplayTag::RequestGameplayTag("SetByCaller.ManaCost"),
            -ManaCost); // Negative because we're reducing the resource

        ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, SpecHandle);
    }
}
```

!!! warning "CheckCost and SetByCaller"
    If your cost GE uses SetByCaller, `CheckCost` may not work correctly out of the box because it doesn't know the SetByCaller value at check time. You may need to override `CheckCost` to handle this, or use a `CustomCalculationClass` for the magnitude that reads the data it needs from the spec or attributes.

## Shared Cooldowns

Multiple abilities can share a cooldown by using the **same cooldown tag**.

If `GA_Fireball` and `GA_FireBlast` both check for `Cooldown.Ability.Fire`, using either one puts both on cooldown. The cooldown GEs can even have different durations — the longer-lasting one will keep both abilities locked until it expires.

This is entirely tag-driven. No special C++ is needed — just configure both abilities to use the same cooldown tag.

## Unique Cooldowns

Conversely, if every ability needs its own independent cooldown, give each one a unique cooldown tag:

- `Cooldown.Ability.Fireball`
- `Cooldown.Ability.FireBlast`
- `Cooldown.Ability.IceBarrier`

Each ability only checks for its own tag, so they cool down independently.

## Cooldown Reduction

To reduce cooldowns, you can:

1. **Modify the duration via magnitude calculations** — Use AttributeBased or CustomCalculationClass for the duration magnitude, reading a "CooldownReduction" attribute
2. **Modify after creation** — Override `ApplyCooldown`, create the spec, compute the reduced duration, and set it via SetByCaller
3. **Remove the cooldown early** — Find and remove the cooldown GE by handle or by tag query

Option 2 (SetByCaller with computed duration) is the most common approach for cooldown reduction.

## Tips

!!! tip "One Cooldown GE Per Ability (Usually)"
    While you _can_ share cooldown GE classes across abilities, it's simpler to give each ability its own cooldown GE. The overhead of an extra Blueprint asset is negligible, and it keeps things clear.

!!! note "Cooldowns Are Gameplay Effects"
    Because cooldowns are just GEs, they interact with the rest of GAS naturally. An effect can grant immunity to cooldown GEs, a "reset cooldowns" ability can remove all active effects with the `Cooldown` tag, and the debug views show cooldowns alongside all other active effects.

## Related Pages

- [Lifecycle and Activation](../gameplay-abilities/lifecycle-and-activation.md) -- the commit pattern that applies costs and cooldowns
- [SetByCaller](set-by-caller.md) -- passing dynamic cooldown durations and cost amounts at runtime
- [Common Abilities](../patterns/common-abilities.md) -- cooldown and cost patterns in typical ability implementations
