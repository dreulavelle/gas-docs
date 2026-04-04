---
title: Debugging GAS
description: Triage guides and debugging tools overview for Gameplay Ability System issues.
---

# Debugging GAS

Something's not working. Let's figure out what.

## Triage: What's Going Wrong?

Pick the symptom that matches your situation:

### :material-lightning-bolt-circle:{ .lg } My ability won't activate

??? note " Is the ability granted?"
    Open the console and run `showdebug abilitysystem`. Look at the **Granted Abilities** list. Is your ability there?

    - **Not listed** — the ability was never granted. Check that it's in your character's `StartupAbilities` array, or that `GiveAbility()` is being called at runtime.
    - **Listed** — move to Step 2.

??? note " Is it blocked by tags?"
    Check your ability's **Activation Blocked Tags** in its Class Defaults. Then check `showdebug` for the owner's current tags. If the owner has *any* of the blocked tags, the ability can't activate.

    Common culprits: `State.Dead`, `CrowdControl.Hard.Stun`, a leftover tag from a previous ability that didn't call `EndAbility`.

??? note " Is it on cooldown?"
    Check `showdebug` → **Active Effects**. If you see the cooldown GE still active, the ability is still on cooldown. Check the cooldown duration — is it longer than expected?

    Also check: does the cooldown GE actually grant a `Cooldown.*` tag? Without the tag, GAS doesn't know the ability is on cooldown.

??? note " Can the owner afford the cost?"
    Check the relevant attribute value in `showdebug` → **Attributes**. If Stamina is 3 and the cost is 5, the ability fails silently.

??? note " Is the Net Execution Policy correct?"
    - `ServerOnly` abilities won't activate on clients
    - `LocalOnly` abilities won't activate on the server
    - `LocalPredicted` is the most common choice for player abilities

??? note " Check the output log"
    Filter for `LogAbilitySystem` in the Output Log. GAS logs activation failures with reasons. Common messages:

    - `"Can't activate LocalOnly or LocalPredicted ability when not local"` — wrong net role
    - `"Can't activate ability because tags are blocked"` — tag check failed
    - `"Cost check failed"` — not enough resources

---

### :material-diamond-stone:{ .lg } My effect doesn't modify the attribute

??? note " Is the effect actually applied?"
    Check `showdebug` → **Active Effects**. For Instant effects, they won't appear here (they execute and disappear). For Duration/Infinite effects, they should be listed.

    If it's not listed, the effect was never applied — check the ability's `ApplyGameplayEffectSpec` call.

??? note " Is the modifier targeting the right attribute?"
    Open the GE Blueprint and verify the modifier's **Attribute** field matches your actual AttributeSet class name *exactly*. `MyAttributeSet.Health` is different from `OtherAttributeSet.Health`.

??? note " Is the magnitude non-zero?"
    - **SetByCaller**: Check the Output Log for `"SetByCaller tag not found"`. This means the ability didn't call `SetSetByCallerMagnitude` with the matching tag.
    - **Scalable Float**: Is the value actually set? A default of 0 does nothing.
    - **Attribute-Based or MMC**: Add logging to your calculation class.

??? note " Are tag requirements filtering the modifier?"
    Individual modifiers can have source/target tag requirements. If the source ASC or target ASC doesn't have the required tags, the modifier is skipped silently.

??? note " Is the attribute on the target's AttributeSet?"
    If the target doesn't have the attribute you're modifying, nothing happens — no error, no crash, just silence. Verify the target has an ASC with the right AttributeSet.

---

### :material-volume-off:{ .lg } My cue doesn't play

??? note " Does the tag match?"
    The cue Blueprint's **Gameplay Cue Tag** must exactly match the `GameplayCue.*` tag on the effect or the tag passed to `ExecuteGameplayCue`. Case-sensitive, hierarchy matters.

??? note " Is the cue discovered?"
    Check the Output Log for `"missing gameplay cue"` warnings. The GameplayCueManager scans specific paths — your cue Blueprint might be in a folder it doesn't scan. Check **Project Settings → Gameplay Cues → Paths**.

??? note " Is the cue loaded?"
    Cues can be lazy-loaded. If the cue hasn't been loaded yet when the effect fires, it won't play on the first trigger. Check [Lazy Loading](../optimization/lazy-loading.md) settings.

??? note " Are cues suppressed?"
    `ShouldSuppressGameplayCues` can be overridden to conditionally suppress cues. Also check if this is a local-only cue firing on a non-local client — replicated cues go through the server.

---

## Debugging Tools

| Tool | Best For | Page |
|---|---|---|
| **ShowDebug AbilitySystem** | Real-time HUD overlay of abilities, effects, attributes, tags | [Details](showdebug-abilitysystem.md) |
| **Gameplay Debugger** | Inspecting any actor in the world (not just the player) | [Details](gameplay-debugger.md) |
| **Output Log** | Activation failures, missing SetByCaller, tag warnings | [Details](logging.md) |
| **Troubleshooting FAQ** | Specific symptoms with known causes and fixes | [Details](troubleshooting.md) |

## Quick Console Commands

| Command | What It Does |
|---|---|
| `showdebug abilitysystem` | Toggle the ability system debug HUD |
| `AbilitySystem.Debug.NextCategory` | Cycle through debug pages (attributes, abilities, effects) |
| `AbilitySystem.Debug.NextTarget` | Cycle through debug targets |
| `AbilitySystem.GameplayCue.PrintGameplayCueNotifyMap` | Dump all registered cue-to-class mappings |
| `Log LogAbilitySystem Verbose` | Enable verbose ability logging |
| `Log LogGameplayEffects Verbose` | Enable verbose effect logging |
