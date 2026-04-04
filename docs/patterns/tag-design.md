---
title: Tag Architecture
description: Gameplay Tag namespace design, a 21-namespace starter preset, loading from .ini, and governance strategies for teams.
---

# Tag Architecture

Gameplay Tags are the connective tissue of GAS. A well-designed tag hierarchy makes your project navigable, your effects composable, and your team productive. A poorly designed one creates confusion, duplicates, and bugs that are hard to trace.

This page covers the principles and gives you a battle-tested starting point.

## Design Principles

### 1. Tags Are a Taxonomy, Not a Database

Tags describe *what something is*, not *what it does*. `Damage.Type.Fire` describes a damage type. `ApplyBurnEffectToTargetAndPlayFireParticles` describes an action -- that's not a tag, that's an ability.

### 2. Shallow Hierarchies with Meaningful Depth

Two to four levels deep is the sweet spot. One level is too flat (no grouping). Five or more is too deep (hard to navigate, easy to misplace).

```
Good:  Damage.Type.Fire
Good:  CrowdControl.Stun
Fine:  Ability.Combat.Melee.LightAttack
Bad:   Game.Combat.Player.Ability.Melee.Sword.LightAttack.Version2
```

### 3. Leaf Tags for Specifics, Parent Tags for Queries

Apply leaf tags to effects and abilities. Use parent tags in requirements and queries.

```
Apply:  CrowdControl.Stun         (specific)
Query:  CrowdControl              (matches Stun, Slow, Root, etc.)
```

This works because of GAS's tag hierarchy matching -- a tag query for `CrowdControl` matches anything under it.

### 4. Singular Nouns for Categories

`Damage.Type.Fire`, not `Damage.Types.Fire`. `Ability.Combat`, not `Abilities.Combat`. Singular reads better and is consistent.

### 5. No Verbs in Tag Names

Tags are state, not actions. `State.Sprinting` (yes), `Action.Sprint` (no). `CrowdControl.Stunned` or `CrowdControl.Stun` (both fine -- pick one convention and stick with it).

## The 21-Namespace Starter Preset

This is a starting point that covers most action/RPG games. Add or remove namespaces based on your game's needs.

| Namespace | Purpose | Example Tags |
|:---|:---|:---|
| `Ability` | Ability identification and types | `Ability.Combat.Melee`, `Ability.Utility.Heal` |
| `AI` | AI behavior states | `AI.State.Alerted`, `AI.Behavior.Patrol` |
| `Animation` | Animation state flags | `Animation.FullBody`, `Animation.UpperBody` |
| `Combat` | Combat states and phases | `Combat.InCombat`, `Combat.Combo.1` |
| `Combo` | Combo tracking | `Combo.Count.1`, `Combo.Window.Open` |
| `Cooldown` | Cooldown identification | `Cooldown.Ability.Fireball` |
| `CrowdControl` | CC types | `CrowdControl.Stun`, `CrowdControl.Slow` |
| `Damage` | Damage types and properties | `Damage.Type.Fire`, `Damage.Type.Physical` |
| `Enemy` | Enemy classification | `Enemy.Type.Boss`, `Enemy.Type.Minion` |
| `Equipment` | Equipment slots and types | `Equipment.Slot.Weapon`, `Equipment.Type.Sword` |
| `Event` | Gameplay events | `Event.Montage.ComboWindow`, `Event.Death` |
| `GameplayCue` | Cue routing | `GameplayCue.Hit.Physical`, `GameplayCue.Status.Burning` |
| `Immunity` | Immunity flags | `Immunity.CrowdControl`, `Immunity.Damage.Fire` |
| `InputTag` | Input action mapping | `InputTag.Attack`, `InputTag.Dodge` |
| `Movement` | Movement states | `Movement.Airborne`, `Movement.Swimming` |
| `Phase` | Game phase tracking | `Phase.Round.1`, `Phase.Intermission` |
| `Resistance` | Damage resistance types | `Resistance.Fire`, `Resistance.Physical` |
| `SetByCaller` | SetByCaller magnitude keys | `SetByCaller.Damage`, `SetByCaller.Duration` |
| `State` | Character states | `State.Sprinting`, `State.Dead`, `State.Invulnerable` |
| `Status` | Status effect types | `Status.Burning`, `Status.Poisoned`, `Status.Bleeding` |
| `Weapon` | Weapon identification | `Weapon.Type.Sword`, `Weapon.Slot.Primary` |

!!! tip "This is a starting point"
    Don't add all 21 on day one if your game doesn't need them. Start with the 5-6 you'll use immediately (Ability, Damage, State, GameplayCue, InputTag, SetByCaller) and add more as needed.

## Loading Tags from .ini Files

Tags can be defined in `DefaultGameplayTags.ini` or a custom `.ini` file. This is the recommended approach for teams because it's version-controllable and diff-friendly.

### DefaultGameplayTags.ini

```ini
[/Script/GameplayTags.GameplayTagsSettings]
+GameplayTagList=(Tag="Ability.Combat.Melee",DevComment="Melee combat abilities")
+GameplayTagList=(Tag="Ability.Combat.Ranged",DevComment="Ranged combat abilities")
+GameplayTagList=(Tag="Damage.Type.Physical",DevComment="Physical damage")
+GameplayTagList=(Tag="Damage.Type.Fire",DevComment="Fire damage")
+GameplayTagList=(Tag="State.Dead",DevComment="Character is dead")
+GameplayTagList=(Tag="CrowdControl.Stun",DevComment="Stunned, cannot act")
```

### Custom Tag .ini Files

You can split tags across multiple files using the `GameplayTagRedirects` and `GameplayTagTableList` settings:

```ini
[/Script/GameplayTags.GameplayTagsSettings]
+GameplayTagTableList=/Game/Data/Tags/CombatTags
+GameplayTagTableList=/Game/Data/Tags/AbilityTags
```

These reference `DataTable` assets with tag definitions.

### C++ Native Tags

For tags that C++ code references frequently, define them as native tags to avoid string lookups:

```cpp
// In a header
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_State_Dead);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_CrowdControl_Stun);

// In a cpp file
UE_DEFINE_GAMEPLAY_TAG(TAG_State_Dead, "State.Dead");
UE_DEFINE_GAMEPLAY_TAG(TAG_CrowdControl_Stun, "CrowdControl.Stun");
```

These are resolved at startup and avoid the overhead of `FGameplayTag::RequestGameplayTag()` at runtime.

## Tag Governance for Teams

As your team grows, tag sprawl becomes a real problem. Two people independently creating `Damage.Fire` and `DamageType.Fire` is the classic failure mode.

### Strategies

**Single owner.** One person (or a small group) owns the tag hierarchy and reviews all additions. Works for teams up to ~10.

**Namespace ownership.** Different people own different namespaces. The combat designer owns `Damage.*` and `CrowdControl.*`. The ability designer owns `Ability.*`. Works for larger teams.

**PR-based governance.** Tags are defined in `.ini` files, and changes go through code review like any other code change. The diff is clear and reviewable.

**Documentation table.** Maintain a living document (or use the `DevComment` field in the .ini) that explains each tag's purpose. Review it periodically and prune unused tags.

### Tag Redirects

When you need to rename a tag (it happens), use redirects instead of find-and-replace:

```ini
[/Script/GameplayTags.GameplayTagsSettings]
+GameplayTagRedirects=(OldTag="DamageType.Fire",NewTag="Damage.Type.Fire")
```

This ensures existing assets continue to work while you migrate.

### Rules of Thumb

- **No tag without a consumer.** If nothing checks for it, don't create it.
- **No duplicate semantics.** If `State.Dead` exists, don't also create `Status.Dead`.
- **Prefer specific over generic.** `CrowdControl.Stun` over `CC.1`.
- **DevComment everything.** Future you (and your teammates) will thank you.

## Related Pages

- [Gameplay Tags](../core-concepts/gameplay-tags.md) -- how tags work in GAS
- [Tags and Requirements](../gameplay-effects/tags-and-requirements.md) -- tag-based GE requirements
- [Naming Conventions](naming-conventions.md) -- broader naming patterns
- [Buff/Debuff System](buff-debuff-system.md) -- tags for buff categories
- [Starter Tag Preset](starter-tags.md) -- 135+ ready-to-use tags across 21 namespaces
