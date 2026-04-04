---
title: Tags and Requirements
description: All tag-based configuration on Gameplay Effects — asset tags, granted tags, application requirements, ongoing requirements, removal requirements, and migration from monolithic properties.
---

# Tags and Requirements

Tags are the connective tissue of GAS. Gameplay Effects use tags in several distinct ways: to identify themselves, to grant tags to targets, to conditionally apply, to inhibit, and to trigger removal. Each of these roles is handled by a different [GE Component](ge-components.md), and understanding which component does what is key to configuring effects correctly.

## Tag Roles at a Glance

| Role | Component | What It Does |
|:---|:---|:---|
| **Asset Tags** | `UAssetTagsGameplayEffectComponent` | Tags the GE itself "has" — for matching and queries |
| **Granted Tags** | `UTargetTagsGameplayEffectComponent` | Tags given to the target actor while the GE is active |
| **Blocked Ability Tags** | `UBlockAbilityTagsGameplayEffectComponent` | Prevents abilities with matching tags from activating |
| **Application Requirements** | `UTargetTagRequirementsGameplayEffectComponent` | Tags the target must/must not have for the GE to apply |
| **Ongoing Requirements** | `UTargetTagRequirementsGameplayEffectComponent` | Tags the target must/must not have for the GE to remain uninhibited |
| **Removal Requirements** | `UTargetTagRequirementsGameplayEffectComponent` | Tags that, when matched, cause the GE to be removed |

## Asset Tags

Configured via `UAssetTagsGameplayEffectComponent`.

Asset tags describe **what the effect is**, not what it does to the target. They are metadata on the GE asset itself. The target actor doesn't gain these tags — they live on the effect.

**Uses:**

- Immunity matching: "Block all effects tagged `Damage.Fire`"
- Querying active effects: "Does the target have any active effect tagged `Buff`?"
- Removal matching: "Remove all active effects tagged `Debuff.Poison`"

**Inheritance:** Asset tags use `FInheritedTagContainer`, which supports tag inheritance from parent GE Blueprints. If a parent GE has `Damage`, a child GE automatically has `Damage` too (unless it explicitly removes it).

```
Parent GE Asset Tags: Damage, Damage.Physical
Child GE (inherits): Damage, Damage.Physical, Damage.Physical.Bleed  ← added by child
```

## Granted Tags (Target Tags)

Configured via `UTargetTagsGameplayEffectComponent`.

These tags are **applied to the target actor** while the GE is active. When the GE is removed, the tags are removed. If the GE is inhibited, the tags are also removed (and re-added when uninhibited).

**Uses:**

- Marking states: `State.Stunned`, `State.OnFire`, `State.Invisible`
- Enabling/disabling other systems: abilities that check `State.CanAttack`
- Triggering other effects' ongoing/removal requirements

**Example:** A freeze effect grants `State.Frozen`. Other effects can use this as an ongoing requirement — a "Shatter" bonus damage effect only works while the target has `State.Frozen`.

!!! warning "Only for Duration/Infinite Effects"
    Instant effects cannot grant tags because they don't persist. The editor will warn you if you add this component to an instant GE.

## Blocked Ability Tags

Configured via `UBlockAbilityTagsGameplayEffectComponent`.

While the GE is active, abilities on the target that have any of the specified tags are **prevented from activating**. Already-active abilities are not cancelled (use `UCancelAbilityTagsGameplayEffectComponent` for that).

**Example:** A silence effect with blocked ability tags `Ability.Magic` prevents all magic abilities from being activated.

## Application Tag Requirements

Configured via `UTargetTagRequirementsGameplayEffectComponent` (the `ApplicationTagRequirements` property).

These are **pass/fail at application time**. If the target's tags don't satisfy the requirements when the GE tries to apply, the application fails entirely — the GE is never added.

`FGameplayTagRequirements` has two containers:

- `RequireTags` — The target must have **all** of these tags
- `IgnoreTags` — The target must have **none** of these tags

```
ApplicationTagRequirements:
  RequireTags: State.Vulnerable
  IgnoreTags: State.Immune

Result: The GE only applies if the target has State.Vulnerable AND does NOT have State.Immune
```

## Ongoing Tag Requirements

Also on `UTargetTagRequirementsGameplayEffectComponent` (the `OngoingTagRequirements` property).

These are checked **continuously** while the effect is active. If the requirements stop being met, the effect is **inhibited** (not removed). If they are met again, the effect uninhibits.

This is the mechanism behind [inhibition](duration-and-lifecycle.md#inhibition).

**Example:** A "Berserker Rage" buff with ongoing requirement `RequireTags: State.LowHealth`. The buff is only active when the character is at low health. It automatically inhibits when they heal up and uninhibits when their health drops again.

## Removal Tag Requirements

Also on `UTargetTagRequirementsGameplayEffectComponent` (the `RemovalTagRequirements` property).

If the target's tags come to satisfy these requirements, the effect is **removed** permanently (not just inhibited). Removal requirements are also checked at application time — if already satisfied, the GE won't apply.

**Example:** A regeneration buff with removal requirement `RequireTags: State.Dead`. If the character dies, the regen is removed.

!!! note "Inhibition vs. Removal"
    - **Ongoing requirements not met** → Effect inhibits (dormant, can come back)
    - **Removal requirements met** → Effect is removed (gone forever)

    Use ongoing requirements for toggleable behavior. Use removal requirements for permanent invalidation.

## Remove Other Effects

Configured via `URemoveOtherGameplayEffectComponent`.

When this GE is applied, it removes other active GEs that match the configured `FGameplayEffectQuery` queries. This runs on application (including restacking).

**Example:** A "Cleanse" spell that removes all active effects whose asset tags match `Debuff`. Or more precisely — the query matches against granted tags, asset tags, source tags, modifying attributes, or even specific GE classes.

## Cancel Abilities

Configured via `UCancelAbilityTagsGameplayEffectComponent`.

Cancels abilities on the target that match (or don't match) specified tags. Supports two modes:

- **OnApplication** — Cancels once when the GE is applied
- **OnExecution** — Cancels on each periodic execution

## Modifier-Level Tag Requirements

Separate from GE-level tag requirements, individual [modifiers](modifiers.md) can have their own `SourceTags` and `TargetTags` requirements. These don't prevent the GE from applying — they just make individual modifiers conditional within an already-applied effect.

See [Modifiers: Modifier Tag Requirements](modifiers.md#modifier-tag-requirements) for details.

## Migration Reference

If you are upgrading from pre-5.3 or reading older tutorials, here is the mapping from old monolithic properties to new components:

| Old Property | New Location |
|:---|:---|
| `InheritableGameplayEffectTags` | `UAssetTagsGameplayEffectComponent` → `InheritableAssetTags` |
| `InheritableOwnedTagsContainer` | `UTargetTagsGameplayEffectComponent` → `InheritableGrantedTagsContainer` |
| `InheritableBlockedAbilityTagsContainer` | `UBlockAbilityTagsGameplayEffectComponent` → `InheritableBlockedAbilityTagsContainer` |
| `ApplicationTagRequirements` | `UTargetTagRequirementsGameplayEffectComponent` → `ApplicationTagRequirements` |
| `OngoingTagRequirements` | `UTargetTagRequirementsGameplayEffectComponent` → `OngoingTagRequirements` |
| `RemovalTagRequirements` | `UTargetTagRequirementsGameplayEffectComponent` → `RemovalTagRequirements` |
| `RemoveGameplayEffectsWithTags` | `URemoveOtherGameplayEffectComponent` → `RemoveGameplayEffectQueries` |

The engine handles this migration automatically when old assets are loaded. See [GE Components: Migration](ge-components.md#the-gameplayeffectversion-enum-and-migration) for details.

## Common Patterns

### Mutually Exclusive Buffs

Want "Fire Shield" and "Ice Shield" to be mutually exclusive?

1. Give both GEs an asset tag of `Buff.ElementalShield`
2. Add a `URemoveOtherGameplayEffectComponent` to each with a query matching `Buff.ElementalShield`

Applying Fire Shield removes any active Ice Shield (and vice versa).

### State-Dependent Abilities

Want abilities that only work in certain states?

1. Your state effects grant tags like `State.Airborne`, `State.Grounded`
2. Your abilities check those tags via their activation requirements (on the ability side)
3. Optionally, use `UBlockAbilityTagsGameplayEffectComponent` on CC effects to block ability categories

### Layered CC

Want a stun to override a slow, but not vice versa?

1. Stun grants `CC.Stun` and `CC.Any`
2. Slow has ongoing requirement: `IgnoreTags: CC.Stun` — it inhibits when stunned
3. Both grant `CC.Any` so abilities can check for "any CC active"

## Related Pages

- [Immunity](immunity.md) -- blocking effects based on the target's active immunity queries
- [Custom Application Requirements](custom-application.md) -- C++ logic for application checks beyond tag matching
- [GE Components](ge-components.md) -- the component classes that implement each tag role described here
