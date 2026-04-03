---
title: Logging
description: GAS log categories, enabling verbose logging, the Visual Logger for attribute tracking, and key log messages to watch for.
---

# Logging

When the visual debuggers aren't enough, the GAS log output tells you exactly what the system is doing internally.

## Log Categories

GAS defines three log categories in `AbilitySystemLog.h`:

| Category | Purpose | Default Level |
|:---|:---|:---|
| `LogAbilitySystem` | General ability system operations (activation, granting, prediction) | Display |
| `VLogAbilitySystem` | Visual Logger integration for ability system events | Display |
| `LogGameplayEffects` | Gameplay Effect application, removal, and modifier evaluation | Display |

Plus the cue-specific category from `GameplayCueNotifyTypes.h`:

| Category | Purpose |
|:---|:---|
| `LogGameplayCueNotify` | Cue notify spawning, recycling, and execution |

## Enabling Verbose Logging

### At Runtime (Console)

```
Log LogAbilitySystem Verbose
Log LogGameplayEffects Verbose
Log LogGameplayCueNotify Verbose
```

To go even deeper:

```
Log LogAbilitySystem VeryVerbose
```

The three verbosity levels for GAS:

| Level | What You Get |
|:---|:---|
| **Log** (default) | "This happened" -- major events |
| **Verbose** | "This is why" -- internal decision logic |
| **VeryVerbose** | "This is what DIDN'T happen and why" -- exhaustive tracing |

### Via Command Line

```
-LogCmds="LogAbilitySystem Verbose, LogGameplayEffects Verbose"
```

### In DefaultEngine.ini

```ini
[Core.Log]
LogAbilitySystem=Verbose
LogGameplayEffects=Verbose
```

## Key Log Messages to Watch For

These are the messages that most often point to the root cause of common issues:

### Ability Activation

```
LogAbilitySystem: Warning: Can't activate ability [GA_MyAbility] because it's blocked by tags
LogAbilitySystem: Warning: Can't activate ability [GA_MyAbility] - failed cost check
LogAbilitySystem: Warning: Can't activate ability [GA_MyAbility] - on cooldown
LogAbilitySystem: Warning: InternalTryActivateAbility failed: [GA_MyAbility] returned false from CanActivateAbility
```

### Effect Application

```
LogAbilitySystem: Verbose: Applied [GE_Damage] to [TargetActor], handle [42]
LogAbilitySystem: Warning: SetByCaller Magnitude for tag [SetByCaller.Damage] was not set. Using 0.
LogAbilitySystem: Warning: Attempted to apply GE with execution but no execution class set
```

### Cue Handling

```
LogAbilitySystem: Warning: Missing gameplay cue handler for tag [GameplayCue.Hit.Physical]
LogGameplayCueNotify: Warning: GameplayCueNotify [GC_Hit_Physical] not found in any GameplayCueSet
```

### Prediction

```
LogAbilitySystem: Verbose: Prediction key [42] confirmed by server
LogAbilitySystem: Warning: Prediction key [42] rejected - rolling back effects
```

## The ABILITY_LOG Macro

GAS uses a custom macro for internal logging:

```cpp
ABILITY_LOG(Verbosity, Format, ...)
```

This maps to `UE_LOG(LogAbilitySystem, ...)`. If you're adding logging to your own ability code, you can use either this macro or standard `UE_LOG`.

## Visual Logger

The Visual Logger (`VLogAbilitySystem` category) provides a timeline-based view of ability system events, particularly useful for attribute changes.

### Enabling

1. **Window > Developer Tools > Visual Logger** (or `vislog` console command)
2. Start recording (the red button)
3. Play through your scenario
4. Stop recording and browse the timeline

### Attribute Graphs

The `ABILITY_VLOG_ATTRIBUTE_GRAPH` macro (in `AbilitySystemLog.h`) logs attribute changes as plotted data points in the Visual Logger. When enabled, you get a graph showing attribute values over time -- very useful for debugging gradual changes, regen, DoTs, and modifier stacking.

### What to Look For

- **Sudden drops:** Instant damage or cost application
- **Gradual decline:** Periodic effects (DoTs, stamina drain)
- **Plateaus:** Attribute not changing when it should (missing modifier?)
- **Spikes:** Unexpected values (modifier misconfiguration, stack overflow)

## Filtering Log Output

The ability system can be noisy at verbose levels. Filter effectively:

### By Category

Only show GAS-related logs:

```
Log LogAbilitySystem Verbose
Log LogGameplayEffects Verbose
Log LogNet Off           // Silence networking noise
Log LogPlayerController Off  // Silence input noise
```

### By Actor

In C++, many log lines include the actor name. Use the output log filter in the editor to search for your specific actor.

### By Keyword

Search the output log for:

- `"CanActivate"` -- activation failures
- `"SetByCaller"` -- magnitude issues
- `"PredictionKey"` -- prediction flow
- `"GameplayCue"` -- cue routing
- `"ApplyGameplayEffect"` -- effect application

## Related Pages

- [ShowDebug AbilitySystem](showdebug-abilitysystem.md) -- visual debugging
- [Gameplay Debugger](gameplay-debugger.md) -- per-actor inspection
- [Troubleshooting](troubleshooting.md) -- symptom-based solutions
