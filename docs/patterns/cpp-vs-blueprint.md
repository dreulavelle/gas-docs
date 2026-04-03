---
title: C++ vs Blueprint
description: What must be C++, what should be C++, what works in Blueprint, and the recommended split for GAS projects at different scales.
---

# C++ vs Blueprint

One of the most common questions in GAS development: "Should this be C++ or Blueprint?" The answer depends on the asset type, your team composition, and your project scale. This page provides clear guidance.

## The General Rule

**C++ for structure, Blueprint for content.**

C++ defines the *framework*: base classes, attribute sets, execution calculations, the ASC setup, input binding infrastructure. Blueprint creates the *content*: individual abilities, effects, cues, and wires them together.

## What Must Be C++

These cannot be done in Blueprint at all, or are severely limited:

| Asset | Why C++ |
|:---|:---|
| **Attribute Sets** | `UAttributeSet` subclasses with `UPROPERTY` attributes, `ATTRIBUTE_ACCESSORS`, replication, and `PostGameplayEffectExecute` |
| **Execution Calculations** | `UGameplayEffectExecutionCalculation` requires C++ for attribute capture declarations and the calculation logic |
| **AbilitySystemGlobals subclass** | Overriding `AllocGameplayEffectContext`, `AllocAbilityActorInfo`, or configuring global behavior |
| **Custom GameplayEffectContext** | `FGameplayEffectContext` subclass with `NetSerialize`, `GetScriptStruct`, `Duplicate` |
| **AbilitySystemComponent setup** | Initial configuration, replication mode, ASC ownership model |
| **IAbilitySystemInterface** | Must be implemented on your character/player state in C++ |
| **Custom Ability Tasks** | New `UAbilityTask` subclasses for project-specific async operations |
| **Replication Proxy** | `IAbilitySystemReplicationProxyInterface` implementation |
| **Custom Magnitude Calculations** | `UGameplayModMagnitudeCalculation` subclasses (though simple ones can be Blueprint) |

## What Should Be C++ (But Could Be Blueprint)

These technically work in Blueprint but are significantly better in C++:

| Asset | Why C++ Is Better |
|:---|:---|
| **Base Ability class** | Your project's `UGameplayAbility` subclass with shared logic (input handling, targeting helpers, common hooks). Individual abilities then inherit from this in Blueprint. |
| **Magnitude Calculations** | Complex MMCs with multiple captured attributes and conditional logic are cleaner in C++ |
| **Custom GE Components** | `UGameplayEffectComponent` subclasses for project-specific GE behavior |
| **Tag constants** | Native gameplay tags (`UE_DEFINE_GAMEPLAY_TAG`) avoid string-based lookups and typos |
| **Input binding setup** | The `InputTag`-to-ability mapping infrastructure |
| **GameplayCueManager override** | Custom routing, suppression, or loading behavior |

## What Works Best in Blueprint

These are where Blueprint shines -- fast iteration, designer-friendly, no compilation needed:

| Asset | Why Blueprint |
|:---|:---|
| **Individual Gameplay Abilities** | Each ability's activation logic, montage selection, targeting flow. Designers iterate on these constantly. |
| **Gameplay Effects** | Duration, modifiers, tags, stacking -- all configured in the editor with no code |
| **Gameplay Cue Notifies** | VFX/SFX configuration, particle system references, sound cues |
| **Ability activation logic** | The "what happens when this ability fires" flow |
| **Targeting actors** | Blueprint subclasses of targeting actors for custom reticles |
| **UI for buff/debuff display** | UMG widgets reading ASC state |

## The Recommended Split

### Per Asset Type

| GAS Asset Type | Created In | Why |
|:---|:---|:---|
| `UAttributeSet` subclass | C++ | Required |
| `UAbilitySystemComponent` subclass | C++ | Configuration and setup |
| Base `UGameplayAbility` | C++ | Shared infrastructure |
| Individual abilities (GA_*) | Blueprint (child of C++ base) | Designer iteration |
| `UGameplayEffect` (GE_*) | Blueprint | Purely data-driven |
| `UGameplayEffectExecutionCalculation` | C++ | Required |
| `UGameplayModMagnitudeCalculation` | C++ or Blueprint | Complexity-dependent |
| Cue Notify (GC_*) | Blueprint | VFX/SFX configuration |
| `UAbilityTask` custom tasks | C++ | Required |
| Async Ability Tasks | C++ | Required |
| Input Actions / Mappings | Editor / Blueprint | Data assets |
| Gameplay Tags | .ini + C++ (native tags) | Version control and type safety |
| DataTables for scaling | Editor | Data-driven content |

### The Layered Architecture

```
Layer 4: Content (Blueprint)
  ├── GA_Fireball, GA_DodgeRoll, GA_Sprint
  ├── GE_FireDamage, GE_SpeedBuff, GE_Heal
  └── GC_FireImpact, GC_HealBurst

Layer 3: Game Framework (C++ + Blueprint)
  ├── UGA_BaseAbility (C++ base with helpers)
  ├── UGA_BaseMeleeAbility (C++ base for melee)
  └── Custom Ability Tasks

Layer 2: GAS Infrastructure (C++)
  ├── UMyAttributeSet
  ├── UMyAbilitySystemGlobals
  ├── UExecCalc_Damage
  ├── FMyGameplayEffectContext
  └── Input binding setup

Layer 1: Engine (Epic's Code)
  └── GAS Plugin source
```

## Project Scale Considerations

### Solo Developer / Prototype

**Lean into Blueprint.** Write the minimum C++ (attribute set, ASC setup, base ability class) and do everything else in Blueprint. You can always move things to C++ later.

```
C++: ~5 files (AttributeSet, ASC subclass, base ability, globals, interface)
Blueprint: Everything else
```

### Small Team (2-5)

**C++ framework, Blueprint content.** One programmer maintains the C++ infrastructure. Designers create abilities and effects in Blueprint.

```
C++: ~10-15 files (above + ExecCalc, custom tasks, MMCs, context)
Blueprint: All abilities, effects, cues
```

### Large Team (5+)

**Strong C++ foundation with extensive Blueprint support.** Multiple base ability classes for different categories (melee, ranged, utility). Blueprint function libraries for common operations. Strict naming conventions and tag governance.

```
C++: ~20-30 files (multiple base classes, multiple ExecCalcs, helper libraries)
Blueprint: Content pipeline for abilities, effects, cues
```

## Common Mistakes

### Too Much Blueprint

Putting the damage formula in Blueprint makes it hard to debug, impossible to profile precisely, and painful to refactor. Complex math belongs in C++.

### Too Much C++

Creating every individual ability in C++ when they're just "play montage, apply effect, end" is wasted programmer time. If the ability follows a pattern, make the pattern in C++ and the instances in Blueprint.

### No Base Class

Creating abilities directly from `UGameplayAbility` instead of a project-specific base class. You'll want shared helpers (commit/cancel input, targeting shortcuts, common tag checks) that live on a base class.

### Hardcoded References in C++

Referencing specific Gameplay Effects or Cues by class in C++ instead of using `TSubclassOf` properties that designers can set. Keep C++ generic; let Blueprint plug in the specifics.

## Related Pages

- [Project Setup](../getting-started/project-setup.md) -- initial C++ setup
- [Naming Conventions](naming-conventions.md) -- naming for both C++ and Blueprint assets
- [Common Abilities](common-abilities.md) -- patterns that demonstrate the C++/Blueprint split
