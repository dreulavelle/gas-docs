---
title: Naming Conventions
description: Asset prefixes, class naming, tag naming, and Content Browser folder structure for GAS projects.
---

# Naming Conventions

Consistent naming makes your GAS project navigable at scale. These conventions are widely adopted across the GAS community and align with Unreal Engine standards.

## Asset Prefixes

| Prefix | Asset Type | Example |
|:---|:---|:---|
| `GA_` | Gameplay Ability (Blueprint) | `GA_Fireball`, `GA_DodgeRoll` |
| `GE_` | Gameplay Effect | `GE_Damage_Fire`, `GE_Buff_SpeedBoost` |
| `GC_` | Gameplay Cue Notify | `GC_Hit_Physical`, `GC_Status_Burning` |
| `AT_` | Ability Task (custom) | `AT_WaitTargetConfirm` |
| `IA_` | Input Action | `IA_Attack`, `IA_Dodge`, `IA_Interact` |
| `IMC_` | Input Mapping Context | `IMC_Default`, `IMC_Vehicle` |
| `DT_` | DataTable | `DT_DamageScaling`, `DT_AttributeDefaults` |
| `CT_` | Curve Table | `CT_LevelScaling` |
| `AS_` | Ability Set (data asset) | `AS_DefaultHero`, `AS_WarriorClass` |

### Why Prefixes Matter

Prefixes let you:

- **Filter in Content Browser:** Type `GE_` to see all effects at a glance
- **Identify type from references:** When you see `GE_Damage` in an ability graph, you immediately know it's an effect
- **Avoid name collisions:** `GA_Sprint` (the ability) vs `GE_Sprint` (the speed buff effect) are clearly different assets

## C++ Class Naming

Follow UE conventions:

| Prefix | Class Type | Example |
|:---|:---|:---|
| `U` | UObject subclass | `UMyAttributeSet`, `UMyAbilitySystemGlobals` |
| `A` | AActor subclass | `AMyCharacter`, `AMyProjectile` |
| `F` | Struct or non-UObject | `FMyGameplayEffectContext`, `FMyDamageInfo` |
| `I` | Interface | `IInteractable`, `IAbilitySystemInterface` |
| `E` | Enum | `EMyDamageType`, `EMyAbilityState` |

### GAS-Specific C++ Naming

| Convention | Example | Notes |
|:---|:---|:---|
| Attribute Set | `UMyProjectAttributeSet` or `UCombatAttributeSet` | Descriptive, not generic |
| ExecCalc | `UExecCalc_Damage` or `UDamageExecution` | Prefix or suffix both work |
| MMC | `UMMC_CooldownReduction` | Prefix makes it filterable |
| Base Ability | `UMyGameplayAbility` | Your project-wide base class |
| Ability Task | `UAbilityTask_MyCustomTask` | Follows engine convention |
| Custom Context | `FMyGameplayEffectContext` | Matches engine naming pattern |

## Tag Naming

Tags use a dot-separated hierarchy. See [Tag Architecture](tag-design.md) for the full design guide.

### Rules

- **PascalCase** for each segment: `Damage.Type.Fire`, not `damage.type.fire`
- **Singular nouns** for categories: `Damage.Type`, not `Damage.Types`
- **No verbs**: `State.Sprinting`, not `Action.Sprint`
- **No abbreviations** (mostly): `CrowdControl.Stun`, not `CC.Stun`. Exception: well-known abbreviations like `AI`, `UI`, `HP`

### Tag-to-Cue Asset Mapping

GameplayCue tags map to cue notify assets by name. The convention:

| Tag | Asset Name |
|:---|:---|
| `GameplayCue.Hit.Physical` | `GC_Hit_Physical` |
| `GameplayCue.Status.Burning` | `GC_Status_Burning` |
| `GameplayCue.Ability.Fireball.Impact` | `GC_Ability_Fireball_Impact` |

Dots become underscores, prefixed with `GC_`. The [GameplayCueManager](../gameplay-cues/cue-manager.md) can auto-derive tags from asset names if you follow this pattern.

## Content Browser Folder Structure

A recommended structure that scales from small to large projects:

```
Content/
├── GAS/
│   ├── Abilities/
│   │   ├── Combat/
│   │   │   ├── GA_LightAttack
│   │   │   ├── GA_HeavyAttack
│   │   │   └── GA_DodgeRoll
│   │   ├── Movement/
│   │   │   ├── GA_Sprint
│   │   │   └── GA_Dash
│   │   └── Utility/
│   │       ├── GA_Heal
│   │       └── GA_Interact
│   ├── Effects/
│   │   ├── Damage/
│   │   │   ├── GE_Damage_Physical
│   │   │   └── GE_Damage_Fire
│   │   ├── Buffs/
│   │   │   ├── GE_Buff_SpeedBoost
│   │   │   └── GE_Buff_AttackUp
│   │   ├── Debuffs/
│   │   │   ├── GE_Debuff_Slow
│   │   │   └── GE_Debuff_Poison
│   │   └── Costs/
│   │       ├── GE_Cost_Mana_20
│   │       └── GE_Cost_Stamina_10
│   ├── Cues/
│   │   ├── Combat/
│   │   │   ├── GC_Hit_Physical
│   │   │   └── GC_Hit_Fire
│   │   └── Status/
│   │       ├── GC_Status_Burning
│   │       └── GC_Status_Poisoned
│   ├── AbilitySets/
│   │   ├── AS_DefaultHero
│   │   └── AS_WarriorClass
│   └── Data/
│       ├── DT_AttributeDefaults
│       └── CT_LevelScaling
├── Input/
│   ├── IA_Attack
│   ├── IA_Dodge
│   └── IMC_Default
```

### Alternative: Per-Character

For games with many distinct characters:

```
Content/
├── Characters/
│   ├── Warrior/
│   │   ├── Abilities/
│   │   ├── Effects/
│   │   └── Cues/
│   └── Mage/
│       ├── Abilities/
│       ├── Effects/
│       └── Cues/
├── Shared/
│   ├── Effects/
│   │   └── (shared damage, cooldowns, costs)
│   └── Cues/
│       └── (shared hit impacts, status effects)
```

### GameplayCue Scan Paths

Whatever folder structure you choose, make sure your `Cues/` folder is listed in the [GameplayCueManager's scan paths](../gameplay-cues/cue-manager.md#scan-paths-and-configuration). Only cues in scanned paths are discoverable at runtime.

## Effect Naming Patterns

### Costs and Cooldowns

```
GE_Cost_Mana_20           (flat 20 mana cost)
GE_Cost_Stamina_Percent_10 (10% stamina cost)
GE_Cooldown_2s             (2 second cooldown)
GE_Cooldown_Fireball       (ability-specific cooldown)
```

### Damage

```
GE_Damage                  (generic, uses ExecCalc + SetByCaller)
GE_Damage_Fire             (fire-typed variant, if needed)
GE_Damage_Poison_DoT       (periodic damage)
```

### Buffs and Debuffs

```
GE_Buff_SpeedBoost_20pct   (20% speed increase)
GE_Buff_Shield_500          (500 HP shield)
GE_Debuff_Slow_30pct        (30% slow)
GE_Debuff_Stun_2s           (2 second stun)
```

## Related Pages

- [Tag Architecture](tag-design.md) -- detailed tag naming guidance
- [C++ vs Blueprint](cpp-vs-blueprint.md) -- where each asset type is created
- [Cue Manager](../gameplay-cues/cue-manager.md) -- cue asset discovery
- [Add an Ability](../recipes/add-ability.md) -- naming in practice
