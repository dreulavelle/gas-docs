---
title: Immunity
description: How to grant immunity to Gameplay Effects using UImmunityGameplayEffectComponent — immunity queries, the ImmunityTrigger delegate, and practical patterns.
---

# Immunity

Immunity in GAS means **blocking the application of other Gameplay Effects**. When an actor has an active effect with immunity configured, incoming effects that match the immunity criteria are rejected before they ever apply. The incoming effect is simply denied — it never enters the Active GE Container, never modifies attributes, never grants tags.

## UImmunityGameplayEffectComponent

Immunity is configured through the `UImmunityGameplayEffectComponent`. You add this component to a GE, configure the immunity queries, and as long as that GE is active on the target, the target is immune to matching effects.

```cpp
UCLASS(DisplayName="Immunity to Other Effects", MinimalAPI)
class UImmunityGameplayEffectComponent : public UGameplayEffectComponent
{
    // ...
public:
    /** Grants immunity to GameplayEffects that match this query. */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = None)
    TArray<FGameplayEffectQuery> ImmunityQueries;
};
```

### How It Works

When the GE with this component is added to the Active GE Container, the component registers a global callback on the ASC via `OnActiveGameplayEffectAdded`. From that point on, every incoming `FGameplayEffectSpec` is checked against the `ImmunityQueries`. If any query matches the incoming spec, the application is blocked.

The check happens in `AllowGameplayEffectApplication`, which is called during the application pipeline — before the effect would be added or executed.

### FGameplayEffectQuery

The queries use `FGameplayEffectQuery`, which is a powerful matching struct. You can match on:

| Query Field | What It Matches Against |
|:---|:---|
| `EffectTagQuery` | The incoming effect's **asset tags** |
| `OwningTagQuery` | The incoming effect's **granted (owning) tags** |
| `SourceTagQuery` | The incoming effect's **source spec tags** |
| `SourceAggregateTagQuery` | All aggregated source tags |
| `ModifyingAttribute` | Effects that modify a specific attribute |
| `EffectSource` | Effects from a specific source object |
| `EffectDefinition` | A specific `UGameplayEffect` class |
| `CustomMatchDelegate` | Custom native matching logic |

All conditions within a single query are ANDed — every specified field must match for the query to match. The `ImmunityQueries` array is ORed — if any query matches, immunity blocks the effect.

!!! info "Instant GEs Cannot Grant Immunity"
    Since immunity requires the effect to be _active_ (present in the container), instant effects cannot grant immunity. The editor will warn you if you add this component to an instant GE.

## The ImmunityTrigger Delegate

When an effect is blocked by immunity, the `OnImmunityBlockGameplayEffectDelegate` on the ASC fires. This lets you react to blocked effects — for example, to play a "immune!" visual effect or log that something was blocked.

```cpp
// Somewhere in your setup
ASC->OnImmunityBlockGameplayEffectDelegate.AddUObject(
    this, &AMyCharacter::OnImmunityTriggered);

// Your handler
void AMyCharacter::OnImmunityTriggered(
    const FGameplayEffectSpec& BlockedSpec,
    const FActiveGameplayEffect* ImmunityGE)
{
    // Play "immune" gameplay cue, show floating text, etc.
    UE_LOG(LogGame, Log, TEXT("Blocked effect: %s"),
        *BlockedSpec.Def->GetName());
}
```

This is purely informational — by the time the delegate fires, the incoming effect has already been rejected.

## Practical Patterns

### Invulnerability Frames (I-Frames)

A dodge roll grants brief total immunity to all damage:

**GE_DodgeIFrames** (Duration = 0.3s):

- Component: `UImmunityGameplayEffectComponent`
    - Query: `EffectTagQuery` matches any effect with tag `Damage`

During the 0.3s window, all incoming effects tagged `Damage` are blocked. Other effects (buffs, debuffs unrelated to damage) still apply.

### Elemental Immunity

A fire resistance potion grants immunity to fire damage:

**GE_FireResistancePotion** (Duration = 30s):

- Component: `UImmunityGameplayEffectComponent`
    - Query: `EffectTagQuery` matches effects with tag `Damage.Fire`
- Component: `UTargetTagsGameplayEffectComponent`
    - Grants tag: `Buff.FireResistance`

Fire damage effects are blocked. Non-fire damage still applies. The granted tag can be used by other systems (UI showing "Fire Resistant", etc.).

### CC Immunity After Stun

After a stun ends, grant a brief window of CC immunity:

**GE_Stun** (Duration = 2s):

- Component: `UAdditionalEffectsGameplayEffectComponent`
    - `OnCompleteNormal`: `GE_CCImmunity`

**GE_CCImmunity** (Duration = 3s):

- Component: `UImmunityGameplayEffectComponent`
    - Query: `EffectTagQuery` matches effects with tag `CC`
- Component: `UTargetTagsGameplayEffectComponent`
    - Grants tag: `State.CCImmune`

When the stun expires naturally, the immunity kicks in. If the stun is cleansed early (premature removal), you could choose not to grant the immunity by using `OnCompleteNormal` instead of `OnCompleteAlways`.

### Immunity to Specific Sources

Block effects from a specific class:

- Query: `EffectDefinition` = `UGE_PoisonDamage`

Or block effects from a specific source object:

- Query: `EffectSource` = some specific actor or weapon

### Stacking Immunity Considerations

If multiple active effects on a target each have immunity components, they all contribute. An incoming effect is blocked if _any_ active immunity query matches it.

When an immunity-granting effect is removed, its immunity contribution is removed too. This is handled automatically by the component's callback cleanup.

## Immunity vs. Other Rejection Mechanisms

There are several ways an effect can be prevented from applying. Here's how immunity compares:

| Mechanism | Where Configured | When Checked | What Happens |
|:---|:---|:---|:---|
| **Immunity** | `UImmunityGameplayEffectComponent` on the _target's_ active effects | Before application | Rejected. ImmunityTrigger delegate fires. |
| **Application Tag Requirements** | `UTargetTagRequirementsGameplayEffectComponent` on the _incoming_ effect | Before application | Rejected silently. |
| **Custom Application Requirements** | `UCustomCanApplyGameplayEffectComponent` on the _incoming_ effect | Before application | Rejected silently. |
| **Chance to Apply** | `UChanceToApplyGameplayEffectComponent` on the _incoming_ effect | Before application | Rejected by random chance. |

The key distinction: **immunity lives on the target's existing effects**, while application requirements live on the incoming effect. Immunity says "I won't accept this." Application requirements say "I won't apply unless conditions are right."

## Tips

!!! tip "Tag Your Effects"
    Immunity is only as good as your tagging. If your damage effects don't have `Damage.Fire` tags, a fire immunity query can't match them. Establish a consistent tag hierarchy early — see [Tags and Requirements](tags-and-requirements.md).

!!! warning "Immunity Doesn't Remove Existing Effects"
    Immunity only blocks new applications. If a poison effect is already active and you then apply a poison immunity, the existing poison keeps ticking. To remove existing effects on immunity application, use `URemoveOtherGameplayEffectComponent` alongside the immunity component.

## Related Pages

- [Tags and Requirements](tags-and-requirements.md) -- application tag requirements and other tag-based rejection mechanisms
- [Custom Application Requirements](custom-application.md) -- custom C++ logic for controlling when effects can apply
- [Dodge Roll Example](../examples/dodge-roll.md) -- a practical i-frames implementation using immunity
