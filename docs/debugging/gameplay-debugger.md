---
title: Gameplay Debugger
description: Using UE's Gameplay Debugger Abilities category to inspect GAS state on any actor in the world.
---

# Gameplay Debugger

UE's Gameplay Debugger is a separate tool from `showdebug`. While `showdebug abilitysystem` only shows the debug target, the Gameplay Debugger lets you inspect any actor in the world by looking at it.

## Enabling

1. **Open the Gameplay Debugger:** Press the **apostrophe key** (`` ' ``) during PIE (default binding)
2. **Enable the Abilities category:** The debugger shows multiple categories (AI, Navigation, Perception, etc.). The Abilities category is provided by `UGameplayDebuggerCategory_Abilities`.

Navigate categories with the number keys shown at the top of the debugger overlay.

## What It Shows

The Abilities category displays for the selected actor:

### Owned Tags

All gameplay tags currently on the ASC, with their reference counts.

### Active Abilities

Granted abilities and their current state (active, inactive, blocked).

### Active Effects

Currently active Gameplay Effects with:

- Effect class name
- Remaining duration
- Stack count
- Granted tags
- Applied modifiers

### Attributes

Current attribute values from all attribute sets on the ASC.

## Navigating Between Actors

- **Look at an actor** and the debugger shows its data
- **Hold the key** to lock the debugger onto the current target
- This works for any actor with an ASC -- AI enemies, bosses, NPCs, other players

This is the key advantage over `showdebug`: you can inspect AI enemies and remote players without switching the debug target.

## Enabling in Packaged Builds

The Gameplay Debugger is available in Development builds by default but stripped from Shipping builds. If you need it in specific test builds:

1. Ensure `GameplayDebugger` module is included in your build target
2. The `UGameplayDebuggerCategory_Abilities` class is registered automatically when the GAS plugin is loaded

## Integration with Your Game

The Gameplay Debugger category for abilities (`UGameplayDebuggerCategory_Abilities`) is built into the GAS plugin. It automatically discovers ASCs on actors and displays their state. No additional setup is needed beyond having the GAS plugin enabled.

If you want to add custom debug information, you can create your own Gameplay Debugger category that reads from your custom components alongside the ASC.

## When to Use Which Tool

| Situation | Tool |
|:---|:---|
| Debugging the local player's state | `showdebug abilitysystem` |
| Inspecting an AI enemy's state | **Gameplay Debugger** |
| Comparing two actors' states | **Gameplay Debugger** (switch between them) |
| Detailed attribute modifier breakdown | `showdebug abilitysystem` (more detail) |
| Quick tag check on any actor | **Gameplay Debugger** (fastest) |
| Team debugging session / screenshots | `showdebug` (cleaner HUD layout) |

## Related Pages

- [ShowDebug AbilitySystem](showdebug-abilitysystem.md) -- the dedicated GAS debug HUD
- [Logging](logging.md) -- text-based diagnostics
- [Troubleshooting](troubleshooting.md) -- symptom-based problem solving
