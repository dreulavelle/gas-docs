---
title: Starter Attributes
description: A complete attribute schema with 50+ attributes across 13 categories for action RPG projects. Includes base values, ranges, derived totals, and copyable C++ declarations.
---

# Starter Attribute Schema

A complete attribute set design for action RPG projects built on GAS. This schema covers health, stamina, magic, offense, defense, evasion, poise/stagger, riposte, last stand, movement, cooldowns, and progression -- everything you need for a souls-like or action RPG combat system.

Use it as a blueprint for your C++ `UAttributeSet` class, as a DataTable import source, or just as a reference when deciding what attributes your game needs.

!!! tip "Pair with the tag preset"
    This schema is designed to work alongside the [Starter Tag Preset](starter-tags.md). The damage types, resistance tags, and status effect tags map directly to the offensive and defensive attributes listed here.

---

## How to Use

Each attribute below should become an `FGameplayAttributeData` property in your `UAttributeSet` subclass. Use the `ATTRIBUTE_ACCESSORS` macro to generate the boilerplate getter/setter/init functions.

```cpp
// In your AttributeSet header
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Vital")
FGameplayAttributeData Health;
ATTRIBUTE_ACCESSORS(UYourProjectAttributeSet, Health)
```

For a full walkthrough of creating an AttributeSet from scratch, see the [Add an Attribute](../recipes/add-attribute.md) recipe. For the minimal starter setup, see [Project Setup](../getting-started/project-setup.md#4-the-attribute-set).

!!! info "Meta attributes are not replicated"
    Attributes in the **Meta** category are server-only transient values. They hold intermediate calculations during `PostGameplayEffectExecute` and are zeroed out immediately after. They should not have `ReplicatedUsing` or appear in `GetLifetimeReplicatedProps`.

---

## Before You Add Everything

!!! warning "Only add attributes your game actually uses"
    Unlike tags, attributes have **real cost**: each one takes memory per actor, requires replication setup (if networked), needs `OnRep` callbacks, and adds to your `GetLifetimeReplicatedProps`. A character with 50 attributes uses measurably more bandwidth than one with 10. Only add what your game needs.

!!! tip "Start small, expand later"
    Begin with the essentials: **Health**, **MaxHealth**, **Stamina**, **MaxStamina**, and **PendingDamage**. That's enough to build abilities with costs and a damage pipeline. Add magic, poise, crits, and other systems as your game requires them -- each category below is independent.

!!! info "The Add/Percent pattern"
    You'll notice attributes like `MaxHealth`, `MaxHealthAdd`, and `MaxHealthPercent`. This pattern lets flat bonuses and percent bonuses **stack independently**:

    `MaxHealthTotal = (MaxHealth + MaxHealthAdd) * (1 + MaxHealthPercent)`

    A piece of gear adds +50 to `MaxHealthAdd`. A buff adds +0.10 to `MaxHealthPercent`. They don't conflict, and removing either cleanly undoes just its portion. This is the industry-standard approach used in games like Diablo, Path of Exile, and most MMOs.

!!! danger "Meta attributes are NOT player-facing stats"
    `PendingDamage`, `PendingHeal`, and `PendingMitigationCost` are **processing channels**, not real stats. They should never be shown to the player, never saved, never replicated. They exist only to pass values through `PostGameplayEffectExecute` and are reset to zero immediately after processing.

---

## Attributes by Category

### Meta (Transient)

Server-only holding values for the damage/heal pipeline. These are written by incoming Gameplay Effects, processed in `PostGameplayEffectExecute`, and immediately zeroed out.

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `PendingDamage` | 0 | 0 | -- | Raw damage before mitigation. Processed in PostGameplayEffectExecute. |
| `PendingHeal` | 0 | 0 | -- | Raw heal amount before modifiers. Processed in PostGameplayEffectExecute. |
| `PendingMitigationCost` | 0 | 0 | -- | Stamina/resource cost from blocking or absorbing a hit. |

### Health

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `Health` | 100 | 0 | MaxHealth | Current health. Death occurs at 0. |
| `MaxHealth` | 100 | 1 | -- | Base maximum health before modifiers. |
| `MaxHealthAdd` | 0 | -- | -- | Flat bonus to max health (from gear, buffs). |
| `MaxHealthPercent` | 0 | -- | -- | Percentage bonus to max health (0.1 = +10%). |
| `HealthRegenRate` | 1.0 | 0 | -- | Health regenerated per second (when out of combat or always, your call). |

### Stamina

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `Stamina` | 100 | 0 | MaxStamina | Current stamina. Consumed by attacks, dodges, blocking. |
| `MaxStamina` | 100 | 1 | -- | Base maximum stamina. |
| `MaxStaminaAdd` | 0 | -- | -- | Flat bonus to max stamina. |
| `MaxStaminaPercent` | 0 | -- | -- | Percentage bonus to max stamina. |
| `StaminaCost` | 20 | 0 | -- | Default stamina cost per action (overridden per ability via SetByCaller). |
| `StaminaRegenRate` | 12 | 0 | -- | Stamina regenerated per second. |
| `StaminaRegenDelay` | 1.25 | 0 | -- | Seconds after stamina use before regen resumes. |

### Magic

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `Magic` | 100 | 0 | MaxMagic | Current magic/mana resource. |
| `MaxMagic` | 100 | 1 | -- | Base maximum magic. |
| `MaxMagicAdd` | 0 | -- | -- | Flat bonus to max magic. |
| `MaxMagicPercent` | 0 | -- | -- | Percentage bonus to max magic. |
| `MagicCost` | 20 | 0 | -- | Default magic cost per spell (overridden per ability). |
| `MagicRegenRate` | 3.5 | 0 | -- | Magic regenerated per second. |
| `MagicRegenDelay` | 1.0 | 0 | -- | Seconds after magic use before regen resumes. |

### Offense

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `BaseDamage` | 10 | 0 | -- | Base damage before weapon and ability scaling. |
| `BaseDamageBonus` | 0 | -- | -- | Flat additive bonus to all damage dealt. |
| `CriticalHitChance` | 0.05 | 0 | 1.0 | Probability of a critical hit (0.05 = 5%). |
| `CriticalHitMultiplier` | 1.5 | 1.0 | -- | Damage multiplier on critical hit. |
| `AttackSpeed` | 1.0 | 0.1 | 5.0 | Montage play rate multiplier. 1.0 = normal speed. |
| `LifeSteal` | 0 | 0 | 1.0 | Fraction of damage dealt returned as health (0.1 = 10%). |

### Defense

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `BaseDamageReduction` | 0 | 0 | 0.9 | Flat percentage damage reduction (0.1 = 10% less damage taken). |
| `ArmorReduction` | 0 | 0 | 0.9 | Armor-based damage reduction, applied to physical damage types. |
| `BlockChance` | 0.10 | 0 | 1.0 | Probability of blocking an incoming attack (when actively blocking). |
| `BlockMitigation` | 0.50 | 0 | 1.0 | Fraction of damage mitigated on block (0.5 = half damage). |
| `BlockAngle` | 120 | 0 | 360 | Arc in degrees within which incoming attacks can be blocked. |
| `BlockStaminaCostRate` | 0.10 | 0 | 1.0 | Fraction of blocked damage subtracted from stamina. |
| `DefenseChance` | 0 | 0 | 1.0 | Generic defense/dodge chance (passive, separate from evade input). |
| `DefenseMitigation` | 0 | 0 | 1.0 | Damage reduction when DefenseChance procs. |
| `TenacityPercent` | 0 | 0 | 1.0 | Crowd control duration reduction (0.3 = 30% shorter CC). |

### Evade

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `EvadeStaminaCost` | 20 | 0 | -- | Stamina consumed per dodge/evade. |
| `EvadeCooldown` | 0.25 | 0 | -- | Minimum seconds between evades. |

### Poise and Stagger

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `Poise` | 100 | 0 | MaxPoise | Current poise. Reaching 0 triggers stagger. |
| `MaxPoise` | 100 | 1 | -- | Maximum poise value. |
| `PoiseDamage` | 0 | 0 | -- | Poise damage dealt per hit (set per ability or weapon). |
| `PoiseRecovery` | 100 | 0 | -- | Poise recovered per interval when not recently hit. |
| `PoiseRecoveryInterval` | 10 | 0 | -- | Seconds after last poise damage before recovery begins. |
| `StaggerDuration` | 1.25 | 0 | -- | Duration of stagger when poise breaks (modified by TenacityPercent). |

### Riposte

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `RiposteWindowDuration` | 0.75 | 0 | -- | Seconds after a successful parry during which a riposte can execute. |
| `RiposteDamageBonusMultiplier` | 2.0 | 1.0 | -- | Damage multiplier applied to riposte attacks. |

### Last Stand

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `LastStandCount` | 0 | 0 | -- | Number of last stand procs remaining. Decremented on use. |
| `LastStandHealthPercent` | 0.25 | 0 | 1.0 | Health percentage restored when last stand triggers. |

### Movement

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `MoveSpeed` | 600 | 0 | -- | Base movement speed in Unreal units per second. |
| `MoveSpeedMultiplier` | 1.0 | 0 | -- | Movement speed multiplier (from buffs/debuffs). Applied on top of base. |

### Cooldown Reduction

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `CooldownReduction` | 0 | 0 | 0.5 | Fraction of cooldown time removed (0.2 = 20% shorter cooldowns). Capped at 50%. |

### Progression

| Name | Base | Min | Max | Description |
|---|---|---|---|---|
| `Level` | 1 | 1 | -- | Current character level. Used for scaling curves and unlock gating. |
| `Experience` | 0 | 0 | -- | Current experience points toward next level. |
| `ExperienceBonus` | 0 | 0 | -- | Flat bonus XP added to each XP gain (from gear, buffs, events). |

---

## Derived Totals

Some values are **computed at runtime**, not stored as attributes. The canonical pattern for "base + flat + percent" stacking:

```
MaxHealthTotal = (MaxHealth + MaxHealthAdd) * (1 + MaxHealthPercent)
MaxStaminaTotal = (MaxStamina + MaxStaminaAdd) * (1 + MaxStaminaPercent)
MaxMagicTotal = (MaxMagic + MaxMagicAdd) * (1 + MaxMagicPercent)
MoveSpeedTotal = MoveSpeed * MoveSpeedMultiplier
```

These derived totals are typically computed in one of two places:

1. **Execution Calculations** -- when resolving damage, your `UGameplayEffectExecutionCalculation` captures the raw attributes and computes derived values inline.
2. **PreAttributeChange / PostGameplayEffectExecute** -- when clamping current values against their maximums, compute the derived max and clamp against it.

!!! warning "Do not store derived totals as attributes"
    It's tempting to add a `MaxHealthTotal` attribute, but this creates a sync problem -- you'd need to recalculate it every time any of its inputs change. Instead, compute it on demand from the source attributes. GAS modifiers and execution calculations are designed for exactly this.

---

## Quick Copy

A single block with all attributes as C++ `UPROPERTY` declarations. Paste this into your `UAttributeSet` header and add `ATTRIBUTE_ACCESSORS` and replication boilerplate as needed.

```cpp
// ── Meta (transient, not replicated) ─────────────────────────────────
UPROPERTY(BlueprintReadOnly, Category = "Meta")
FGameplayAttributeData PendingDamage;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, PendingDamage)

UPROPERTY(BlueprintReadOnly, Category = "Meta")
FGameplayAttributeData PendingHeal;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, PendingHeal)

UPROPERTY(BlueprintReadOnly, Category = "Meta")
FGameplayAttributeData PendingMitigationCost;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, PendingMitigationCost)

// ── Health ───────────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Health")
FGameplayAttributeData Health;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, Health)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealth, Category = "Health")
FGameplayAttributeData MaxHealth;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxHealth)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealthAdd, Category = "Health")
FGameplayAttributeData MaxHealthAdd;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxHealthAdd)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealthPercent, Category = "Health")
FGameplayAttributeData MaxHealthPercent;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxHealthPercent)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_HealthRegenRate, Category = "Health")
FGameplayAttributeData HealthRegenRate;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, HealthRegenRate)

// ── Stamina ──────────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Stamina, Category = "Stamina")
FGameplayAttributeData Stamina;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, Stamina)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxStamina, Category = "Stamina")
FGameplayAttributeData MaxStamina;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxStamina)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxStaminaAdd, Category = "Stamina")
FGameplayAttributeData MaxStaminaAdd;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxStaminaAdd)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxStaminaPercent, Category = "Stamina")
FGameplayAttributeData MaxStaminaPercent;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxStaminaPercent)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_StaminaCost, Category = "Stamina")
FGameplayAttributeData StaminaCost;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, StaminaCost)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_StaminaRegenRate, Category = "Stamina")
FGameplayAttributeData StaminaRegenRate;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, StaminaRegenRate)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_StaminaRegenDelay, Category = "Stamina")
FGameplayAttributeData StaminaRegenDelay;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, StaminaRegenDelay)

// ── Magic ────────────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Magic, Category = "Magic")
FGameplayAttributeData Magic;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, Magic)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxMagic, Category = "Magic")
FGameplayAttributeData MaxMagic;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxMagic)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxMagicAdd, Category = "Magic")
FGameplayAttributeData MaxMagicAdd;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxMagicAdd)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxMagicPercent, Category = "Magic")
FGameplayAttributeData MaxMagicPercent;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxMagicPercent)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MagicCost, Category = "Magic")
FGameplayAttributeData MagicCost;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MagicCost)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MagicRegenRate, Category = "Magic")
FGameplayAttributeData MagicRegenRate;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MagicRegenRate)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MagicRegenDelay, Category = "Magic")
FGameplayAttributeData MagicRegenDelay;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MagicRegenDelay)

// ── Offense ──────────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BaseDamage, Category = "Offense")
FGameplayAttributeData BaseDamage;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, BaseDamage)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BaseDamageBonus, Category = "Offense")
FGameplayAttributeData BaseDamageBonus;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, BaseDamageBonus)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_CriticalHitChance, Category = "Offense")
FGameplayAttributeData CriticalHitChance;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, CriticalHitChance)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_CriticalHitMultiplier, Category = "Offense")
FGameplayAttributeData CriticalHitMultiplier;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, CriticalHitMultiplier)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_AttackSpeed, Category = "Offense")
FGameplayAttributeData AttackSpeed;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, AttackSpeed)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_LifeSteal, Category = "Offense")
FGameplayAttributeData LifeSteal;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, LifeSteal)

// ── Defense ──────────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BaseDamageReduction, Category = "Defense")
FGameplayAttributeData BaseDamageReduction;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, BaseDamageReduction)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_ArmorReduction, Category = "Defense")
FGameplayAttributeData ArmorReduction;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, ArmorReduction)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BlockChance, Category = "Defense")
FGameplayAttributeData BlockChance;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, BlockChance)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BlockMitigation, Category = "Defense")
FGameplayAttributeData BlockMitigation;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, BlockMitigation)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BlockAngle, Category = "Defense")
FGameplayAttributeData BlockAngle;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, BlockAngle)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BlockStaminaCostRate, Category = "Defense")
FGameplayAttributeData BlockStaminaCostRate;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, BlockStaminaCostRate)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_DefenseChance, Category = "Defense")
FGameplayAttributeData DefenseChance;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, DefenseChance)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_DefenseMitigation, Category = "Defense")
FGameplayAttributeData DefenseMitigation;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, DefenseMitigation)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_TenacityPercent, Category = "Defense")
FGameplayAttributeData TenacityPercent;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, TenacityPercent)

// ── Evade ────────────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_EvadeStaminaCost, Category = "Evade")
FGameplayAttributeData EvadeStaminaCost;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, EvadeStaminaCost)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_EvadeCooldown, Category = "Evade")
FGameplayAttributeData EvadeCooldown;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, EvadeCooldown)

// ── Poise / Stagger ─────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Poise, Category = "Poise")
FGameplayAttributeData Poise;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, Poise)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxPoise, Category = "Poise")
FGameplayAttributeData MaxPoise;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MaxPoise)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_PoiseDamage, Category = "Poise")
FGameplayAttributeData PoiseDamage;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, PoiseDamage)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_PoiseRecovery, Category = "Poise")
FGameplayAttributeData PoiseRecovery;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, PoiseRecovery)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_PoiseRecoveryInterval, Category = "Poise")
FGameplayAttributeData PoiseRecoveryInterval;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, PoiseRecoveryInterval)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_StaggerDuration, Category = "Poise")
FGameplayAttributeData StaggerDuration;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, StaggerDuration)

// ── Riposte ──────────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_RiposteWindowDuration, Category = "Riposte")
FGameplayAttributeData RiposteWindowDuration;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, RiposteWindowDuration)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_RiposteDamageBonusMultiplier, Category = "Riposte")
FGameplayAttributeData RiposteDamageBonusMultiplier;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, RiposteDamageBonusMultiplier)

// ── Last Stand ───────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_LastStandCount, Category = "LastStand")
FGameplayAttributeData LastStandCount;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, LastStandCount)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_LastStandHealthPercent, Category = "LastStand")
FGameplayAttributeData LastStandHealthPercent;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, LastStandHealthPercent)

// ── Movement ─────────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MoveSpeed, Category = "Movement")
FGameplayAttributeData MoveSpeed;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MoveSpeed)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MoveSpeedMultiplier, Category = "Movement")
FGameplayAttributeData MoveSpeedMultiplier;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, MoveSpeedMultiplier)

// ── Cooldown ─────────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_CooldownReduction, Category = "Cooldown")
FGameplayAttributeData CooldownReduction;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, CooldownReduction)

// ── Progression ──────────────────────────────────────────────────────
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Level, Category = "Progression")
FGameplayAttributeData Level;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, Level)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Experience, Category = "Progression")
FGameplayAttributeData Experience;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, Experience)

UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_ExperienceBonus, Category = "Progression")
FGameplayAttributeData ExperienceBonus;
ATTRIBUTE_ACCESSORS(UYourAttributeSet, ExperienceBonus)
```

!!! note "You'll need OnRep functions too"
    Each `ReplicatedUsing` attribute needs a matching `OnRep_*` function. The pattern is always the same -- see [Project Setup](../getting-started/project-setup.md#4-the-attribute-set) for the boilerplate. Meta attributes (`PendingDamage`, `PendingHeal`, `PendingMitigationCost`) don't need OnRep functions since they're not replicated.
