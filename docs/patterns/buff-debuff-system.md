---
title: Buff/Debuff System
description: Duration effects with tags, stacking policies, UI representation, cleansing, buff categories, and immunity patterns.
---

# Buff/Debuff System

Buffs and debuffs are Duration or Infinite Gameplay Effects that modify attributes, grant tags, or both. This page covers the patterns for building a robust buff/debuff system -- including stacking, categories, UI, and cleansing.

## Basic Buff Structure

A typical buff GE has:

1. **Duration:** Duration or Infinite (depending on whether it expires naturally)
2. **Modifiers:** Attribute modifications (e.g., +20% MoveSpeed)
3. **Tags granted to target:** `Buff.SpeedBoost` (via TargetTagsGameplayEffectComponent)
4. **Cue tag:** `GameplayCue.Buff.SpeedBoost` for visual feedback
5. **Stacking policy:** How multiple applications interact

A typical debuff is identical in structure, just with negative modifiers and debuff-category tags.

## Stacking Policies

Stacking determines what happens when the same effect is applied multiple times. Set via `StackingType` on the GE:

| Policy | Behavior | Use Case |
|:---|:---|:---|
| **No Stacking** | Each application creates a separate instance | Unique buffs, HoTs from different sources |
| **Stack Per Source** | Same source stacks up to `StackLimitCount`; different sources are separate | Poison applied by different enemies |
| **Stack Per Target** | All applications on the same target share one stack | Generic buff that intensifies regardless of source |

### Common Stack Settings

```
StackLimitCount = 5           // Max stacks
StackDurationRefreshPolicy = RefreshOnSuccessfulApplication
StackPeriodResetPolicy = ResetOnSuccessfulApplication
StackExpirationPolicy = ClearEntireStack  // or RemoveSingleStackAndRefreshDuration
```

**Refresh on application** is the standard for most buffs -- reapplying resets the duration so the buff doesn't fall off during sustained combat.

**Remove single stack on expiry** creates a "diminishing" pattern where stacks fall off one at a time. Good for poison stacks or combo counters.

### Scaling with Stacks

Modifiers can scale with stack count. The modifier magnitude is multiplied by the current stack count automatically. So a modifier of `+5 AttackPower` with 3 stacks gives `+15 AttackPower`.

For more complex scaling (e.g., stacks 1-3 give +5 each, stacks 4-5 give +10 each), use a [Custom Magnitude Calculation](../gameplay-effects/magnitude-calculations.md) that reads the stack count.

## UI Representation

Players need to see active buffs/debuffs. The typical approach:

### Listening for Active Effects

```cpp
// Register for changes
ASC->OnActiveGameplayEffectAddedDelegateToSelf.AddUObject(
    this, &UMyBuffWidget::OnEffectAdded);

ASC->OnAnyGameplayEffectRemovedDelegate().AddUObject(
    this, &UMyBuffWidget::OnEffectRemoved);
```

### Querying Active Effects

```cpp
// Get all active effects
TArray<FActiveGameplayEffectHandle> ActiveEffects =
    ASC->GetActiveEffects(FGameplayEffectQuery());

// Query by tag
FGameplayEffectQuery Query;
Query.EffectTagQuery = FGameplayTagQuery::MakeQuery_MatchTag(BuffTag);
TArray<FActiveGameplayEffectHandle> BuffEffects =
    ASC->GetActiveEffects(Query);

// Get remaining duration
float Remaining = ASC->GetActiveGameplayEffectRemainingDuration(Handle);

// Get stack count
int32 Stacks = ASC->GetActiveGameplayEffectStackCount(Handle);
```

### What to Show

For each active buff/debuff, display:

- **Icon:** Stored on the GE's `UIData` component (`UGameplayEffectUIData_TextOnly` or a custom subclass)
- **Duration:** Remaining time (update each frame or on a timer)
- **Stack count:** If > 1
- **Category color:** Green for buffs, red for debuffs, etc.

## Cleansing / Dispelling

Removing specific buffs or debuffs is a common game mechanic.

### By Tag

The most flexible approach -- remove all effects that grant a specific tag:

```cpp
// Remove all effects granting "Debuff.*" tags
FGameplayTagContainer DebuffTags;
DebuffTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Debuff")));

TArray<FActiveGameplayEffectHandle> ActiveEffects =
    ASC->GetActiveEffects(FGameplayEffectQuery());

for (const FActiveGameplayEffectHandle& Handle : ActiveEffects)
{
    const FActiveGameplayEffect* ActiveGE = ASC->GetActiveGameplayEffect(Handle);
    if (ActiveGE)
    {
        FGameplayTagContainer GrantedTags;
        ActiveGE->Spec.GetAllGrantedTags(GrantedTags);
        if (GrantedTags.HasAny(DebuffTags))
        {
            ASC->RemoveActiveGameplayEffect(Handle);
        }
    }
}
```

### By GE Class

Remove a specific effect type:

```cpp
ASC->RemoveActiveGameplayEffectBySourceEffect(
    SpecificDebuffClass, nullptr); // nullptr = any source
```

### Via RemoveOtherGameplayEffectComponent

The `URemoveOtherGameplayEffectComponent` on a "cleanse" GE can automatically remove effects matching tag queries when the cleanse is applied. This is entirely data-driven -- no code needed.

### Dispel Resistance

Some buffs shouldn't be removable by normal dispels. Options:

- Add a `Buff.Undispellable` tag and check for it before removing
- Use the Immunity system to make certain effects immune to removal GEs
- Track "dispel strength" vs "buff strength" in your cleanse logic

## Buff Categories

Organizing buffs/debuffs into categories enables game mechanics like "cleanse all curses" or "immunity to slows."

### Tag Hierarchy

```
Buff.
  Buff.Attack.PowerUp
  Buff.Defense.Shield
  Buff.Movement.Speed
  Buff.Healing.Regeneration

Debuff.
  Debuff.CrowdControl.Stun
  Debuff.CrowdControl.Slow
  Debuff.CrowdControl.Root
  Debuff.DamageOverTime.Poison
  Debuff.DamageOverTime.Burn
  Debuff.Curse.Weakness
```

### Category-Based Cleansing

"Remove all crowd control effects":

```cpp
FGameplayTag CCTag = FGameplayTag::RequestGameplayTag(TEXT("Debuff.CrowdControl"));
// Use HasTag with partial matching to catch all children
```

### Category-Based Immunity

"Immune to all damage-over-time effects" -- apply an Infinite GE with the [Immunity Component](../gameplay-effects/immunity.md) that blocks effects with `Debuff.DamageOverTime` tags.

## Immunity to Specific Debuff Types

### Pattern 1: Immunity GE Component

Create an Infinite GE (`GE_PoisonImmunity`) with:

- `ImmunityGameplayEffectComponent` configured to block effects with tag `Debuff.DamageOverTime.Poison`
- Grants tag `Immunity.Debuff.Poison`
- Optional cue: `GameplayCue.Immunity.Poison` (visual indicator)

### Pattern 2: Tag-Based Blocking

On the debuff GE, add **Application Tag Requirements**:

- **Ignore Tags:** `Immunity.Debuff.Poison`

If the target has this tag, the debuff won't apply.

### Pattern 3: Stacking with Immunity Threshold

For "immune after N applications":

1. Debuff stacks up to 5
2. At 5 stacks, the debuff GE (via an Additional Effect) applies an immunity GE
3. The immunity GE removes the debuff and grants a tag that blocks reapplication
4. Immunity expires after a duration, allowing the cycle to restart

## Common Buff Effect Setup

Here's a checklist for setting up a well-structured buff:

- [ ] Duration or Infinite (will it expire naturally?)
- [ ] Modifiers for attribute changes
- [ ] Tags granted: at minimum a category tag (e.g., `Buff.Defense.Shield`)
- [ ] Stacking: None, PerSource, or PerTarget
- [ ] Stack limit and duration refresh policy
- [ ] Cue tag for visual feedback
- [ ] UIData for buff icon and description
- [ ] Application requirements (can it apply while another version is active?)
- [ ] Immunity considerations (what can block this buff?)

## Related Pages

- [Stacking](../gameplay-effects/stacking.md) -- deep dive on stacking policies
- [Tags and Requirements](../gameplay-effects/tags-and-requirements.md) -- conditional application
- [Immunity](../gameplay-effects/immunity.md) -- the immunity system
- [GE Components](../gameplay-effects/ge-components.md) -- the modular GE architecture
- [Tag Architecture](tag-design.md) -- organizing your tag hierarchy
