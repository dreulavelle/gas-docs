---
title: ShowDebug AbilitySystem
description: How to enable and read the ShowDebug AbilitySystem HUD overlay — attributes, abilities, effects, and console commands.
---

# ShowDebug AbilitySystem

The `showdebug abilitysystem` console command is your primary debugging tool for GAS. It renders a HUD overlay showing the complete ability system state for the targeted actor.

## Enabling

Open the console (~) and type:

```
showdebug abilitysystem
```

This toggles the debug HUD on the current debug target (usually the locally controlled pawn).

## Debug Pages

The HUD has multiple pages you cycle through:

### Attributes Page

Shows all attributes on the target's attribute sets with their current and base values:

```
--- Attributes ---
Health: 85.0 / Base: 100.0
MaxHealth: 100.0 / Base: 100.0
Stamina: 60.0 / Base: 100.0
MoveSpeed: 1.3 / Base: 1.0
Armor: 25.0 / Base: 25.0
```

If the current value differs from the base value, there are active modifiers affecting it.

### Abilities Page

Lists all granted abilities and their current state:

```
--- Abilities ---
[0] GA_LightAttack (Active) [InputTag.Attack]
[1] GA_DodgeRoll (Ready) [InputTag.Dodge]
[2] GA_Sprint (Ready) [InputTag.Sprint]
[3] GA_PassiveHealthRegen (Active) [None]
```

States: `Ready`, `Active`, `Cooldown`, `Blocked`.

### Effects Page

Shows all active Gameplay Effects on the target:

```
--- Active Effects ---
[0] GE_Buff_SpeedBoost (Duration: 3.2s remaining, Stacks: 1)
    Modifiers: MoveSpeed +0.3 (MultiplyAdditive)
    Tags Granted: Buff.SpeedBoost
[1] GE_Cooldown_DodgeRoll (Duration: 0.3s remaining)
    Tags Granted: Cooldown.Ability.DodgeRoll
[2] GE_PassiveRegen (Infinite, Period: 1.0s)
    Modifiers: Health +5 (Additive)
```

This page is especially useful for confirming effects are applied, checking remaining durations, and verifying stack counts.

### Tags Page

Shows all gameplay tags currently owned by the ASC:

```
--- Owned Tags ---
Ability.Movement.DodgeRoll (1)
Buff.SpeedBoost (1)
State.InCombat (1)
```

The number in parentheses is the tag count (how many sources are granting that tag).

## Console Commands

| Command | What It Does |
|:---|:---|
| `showdebug abilitysystem` | Toggle the HUD |
| `AbilitySystem.Debug.NextCategory` | Cycle to next debug page |
| `AbilitySystem.Debug.NextTarget` | Cycle to next debuggable actor |

### Debug HUD Extensions

UE 5.7 supports extensible debug HUD via `UAbilitySystemDebugHUDExtension` subclasses. Built-in extensions:

| Extension | Console Toggle | What It Shows |
|:---|:---|:---|
| **Tags** | `AbilitySystem.Debug.ToggleTags [TagFilter]` | Owned tags for all nearby actors |
| **Attributes** | `AbilitySystem.Debug.ToggleAttributes [AttributeFilter]` | Specific attributes for all nearby actors |
| **Blocked Ability Tags** | `AbilitySystem.Debug.ToggleBlockedAbilityTags [TagFilter]` | Tags currently blocking abilities |

These extensions render debug info in world space above actors, not just on the HUD. Very useful for seeing the ability state of multiple actors at once.

Additional commands:

| Command | What It Does |
|:---|:---|
| `AbilitySystem.Debug.ToggleAttributes Health MaxHealth` | Show Health and MaxHealth above all actors |
| `AbilitySystem.Debug.ToggleAttributeModifiers` | Show attribute modifiers alongside values |
| `AbilitySystem.Debug.ClearAttributes` | Clear displayed attributes |

## Reading the HUD Effectively

### "Why won't my ability activate?"

1. Check the **Abilities** page -- is the ability listed? (If not, it's not granted)
2. Check the **Tags** page -- any activation-blocking tags present?
3. Check the **Effects** page -- is the cooldown GE still active?
4. Check the **Attributes** page -- enough resources for the cost?

### "Why isn't my effect working?"

1. Check the **Effects** page -- is the effect listed as active?
2. Check the modifier details -- correct attribute and operation?
3. Compare **Attributes** page base vs current -- is the modifier being aggregated?

### "Something is wrong with my tags"

1. Check the **Tags** page -- unexpected tags? Missing tags?
2. Cross-reference with the **Effects** page -- which effect is granting each tag?
3. Check tag counts -- a tag with count 2 means two sources are granting it. Removing one source won't remove the tag.

## HUD Target Configuration

By default, the HUD targets the locally controlled pawn. You can change this:

- **Project Settings > Gameplay Abilities Settings > Use Debug Target From HUD**: If true, uses the HUD's standard debug target (which you can cycle with standard `showdebug` target controls)
- **AbilitySystem.Debug.NextTarget**: Cycle through actors with ASCs

## Custom Debug HUD

You can replace the default HUD with a custom subclass:

```cpp
AAbilitySystemDebugHUD::RegisterHUDClass(AMyCustomAbilityDebugHUD::StaticClass());
```

And create custom extensions by subclassing `UAbilitySystemDebugHUDExtension`:

```cpp
UCLASS()
class UMyDebugExtension : public UAbilitySystemDebugHUDExtension
{
    GENERATED_BODY()
    virtual bool IsEnabled() const override;
    virtual void GetDebugStrings(const AActor* Actor,
        const UAbilitySystemComponent* Comp,
        TArray<FString>& OutDebugStrings) const override;
};
```

## Related Pages

- [Gameplay Debugger](gameplay-debugger.md) -- alternative debugging tool
- [Logging](logging.md) -- text-based diagnostics
- [Troubleshooting](troubleshooting.md) -- specific symptoms and fixes
