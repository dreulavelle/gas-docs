---
title: Debugging GAS
description: Triage flowchart and debugging tools overview for Gameplay Ability System issues.
---

# Debugging GAS

Something's not working. Let's figure out what.

## "My Ability Isn't Working" Decision Tree

```
Ability won't activate?
├─ Is the ability granted?
│  └─ No → Grant it (GiveAbility). Check showdebug.
├─ Check showdebug abilitysystem → is it in the ability list?
│  └─ No → It's not granted. Fix granting logic.
├─ Does the ability have Activation Blocked Tags?
│  └─ Check if the owning ASC has any of those tags right now.
├─ Does the ability have Activation Required Tags?
│  └─ Check if the owning ASC is MISSING any of those tags.
├─ Is the ability on cooldown?
│  └─ Check Active Effects for the cooldown GE.
├─ Can the owner afford the cost?
│  └─ Check the relevant attribute value vs the cost GE.
├─ Is the Net Execution Policy correct?
│  └─ ServerOnly won't run on clients. LocalOnly won't run on server.
├─ Does CanActivateAbility return true?
│  └─ Override it, add logging, check what fails.
└─ Check the output log for "AbilitySystem" warnings.

Effect doesn't modify attribute?
├─ Is the effect actually applied?
│  └─ Check showdebug → Active Effects list.
├─ Is the modifier targeting the right attribute?
│  └─ Check the GE's modifier setup.
├─ Is the modifier magnitude non-zero?
│  └─ Check SetByCaller values, curve lookups, MMC output.
├─ Are there tag requirements filtering the modifier?
│  └─ Check source/target tag requirements on the modifier.
└─ Is the attribute on the target's AttributeSet?
    └─ Missing attributes cause silent failures.

Cue doesn't play?
├─ Is the cue tag set correctly on both the GE and the cue notify?
├─ Is the cue in a scanned path?
│  └─ Check GameplayCueNotifyPaths in project settings.
├─ Is the cue loaded?
│  └─ Check for "missing gameplay cue" log warnings.
├─ Are cues suppressed?
│  └─ Check ShouldSuppressGameplayCues.
└─ Is this a local-only cue on a non-local client?
    └─ Replicated cues need to go through the server.
```

## Debugging Tools

### [ShowDebug AbilitySystem](showdebug-abilitysystem.md)

The built-in HUD overlay that shows attributes, active abilities, active effects, and tags for the debugged actor. This is your first stop for any GAS debugging.

### [Gameplay Debugger](gameplay-debugger.md)

UE's Gameplay Debugger has an Abilities category that shows GAS state for any actor in the world, not just the player.

### [Logging](logging.md)

The `LogAbilitySystem`, `VLogAbilitySystem`, and `LogGameplayEffects` log categories, plus the Visual Logger for attribute change tracking.

### [Troubleshooting](troubleshooting.md)

FAQ-format page: specific symptom → likely cause → fix. Covers all the common gotchas.

## Quick Console Commands

| Command | What It Does |
|:---|:---|
| `showdebug abilitysystem` | Toggle the ability system debug HUD |
| `AbilitySystem.Debug.NextCategory` | Cycle through debug pages (attributes, abilities, effects) |
| `AbilitySystem.Debug.NextTarget` | Cycle through debug targets |
| `AbilitySystem.GameplayCue.PrintGameplayCueNotifyMap` | Dump all registered cue-to-class mappings |
| `Log LogAbilitySystem Verbose` | Enable verbose ability logging |
| `Log LogGameplayEffects Verbose` | Enable verbose effect logging |
