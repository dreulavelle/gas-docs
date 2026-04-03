---
title: Troubleshooting
description: FAQ-format troubleshooting — specific symptoms, likely causes, and fixes for common GAS issues.
---

# Troubleshooting

Symptom → likely cause → fix. Jump to whatever matches your problem.

---

## Ability Won't Activate

**Symptom:** `TryActivateAbility` returns false. Nothing happens when the player presses the button.

**Check in order:**

1. **Is the ability granted?** Use `showdebug abilitysystem` -- the ability must appear in the ability list. If not, call `GiveAbility` first.

2. **Activation Blocked Tags?** The ASC might have a tag that blocks this ability. Check `ActivationBlockedTags` on the ability vs the ASC's current tags.

3. **Activation Required Tags?** The ASC might be missing a required tag.

4. **On cooldown?** Check Active Effects for the cooldown GE. If it's still active, the cooldown hasn't expired.

5. **Can't afford cost?** Check the attribute value vs the cost GE's modifier. If stamina is 15 and the cost is 20, the ability won't commit.

6. **Wrong Net Execution Policy?** `ServerOnly` abilities won't activate from client input. `LocalOnly` abilities won't trigger server-side effects.

7. **CanActivateAbility override?** If you've overridden `CanActivateAbility` in C++, add logging to see which check fails.

---

## Effect Doesn't Modify Attribute

**Symptom:** The Gameplay Effect is applied (it shows up in Active Effects) but the attribute doesn't change.

**Check:**

1. **Modifier targets the wrong attribute.** Double-check the attribute reference in the modifier. Typos in the attribute set class or attribute name are silent failures.

2. **Magnitude is zero.** If using SetByCaller, did you actually set the value? The warning `"SetByCaller Magnitude was not set"` appears in the log. If using a Curve Table, is the row name correct? If using an MMC, does it return non-zero?

3. **Tag requirements on the modifier.** Modifiers can have source/target tag requirements. If the tags don't match, the modifier is skipped during evaluation (it's still "active" but contributes 0).

4. **Wrong modifier operation.** `Multiply Additive` with a value of `20` means *+2000%*. You probably wanted `0.2` for +20%. See [Modifier Formula](../reference/modifier-formula.md).

5. **Attribute not on the target.** If the target's ASC doesn't have the AttributeSet containing that attribute, the modifier silently does nothing.

---

## SetByCaller Is Always 0

**Symptom:** The effect applies but the magnitude from SetByCaller is 0.

**Cause:** The tag used in `SetSetByCallerMagnitude` doesn't match the tag configured on the GE modifier.

**Fix:**

1. Check the tag on the GE modifier's magnitude: it should be something like `SetByCaller.Damage`
2. Check your code: `SpecHandle.Data->SetSetByCallerMagnitude(Tag, Value)` -- is the tag identical?
3. Enable verbose logging -- the system warns when a SetByCaller value is requested but wasn't set
4. Common mistake: setting the value on the wrong spec handle (e.g., creating a new spec after setting the value)

---

## Cooldown Not Working

**Symptom:** The ability can be activated repeatedly with no cooldown.

**Check:**

1. **Cooldown GE assigned?** On the ability's Class Defaults, is the `CooldownGameplayEffectClass` set?
2. **Cooldown GE has correct tag?** The cooldown GE must grant a tag that matches the ability's cooldown query. By default, the system checks for any tag granted by the cooldown GE.
3. **Duration set?** The cooldown GE must have `Duration Policy = Has Duration` with a non-zero duration.
4. **CommitAbility called?** If you're handling activation manually, you must call `CommitAbility` (or `CommitExecute`) to apply the cooldown.

---

## Duration Buff Doesn't Undo When It Expires

**Symptom:** A Duration GE expires, but the attribute modifier persists.

**Check:**

1. **Is it actually a Duration GE?** Check that `Duration Policy = Has Duration`, not `Instant`. Instant effects permanently change the base value -- they don't revert.
2. **Is something reapplying it?** Another effect or ability might be reapplying the same modifier.
3. **Stacking misconfiguration.** If stacking is set to `Remove Single Stack`, and there are multiple stacks, only one stack is removed on expiry.
4. **Clamping in PreAttributeChange.** If you're clamping in `PreAttributeChange`, it might set a "floor" that prevents the attribute from reverting to a lower base value.

---

## Tags Don't Block / Aren't Working

**Symptom:** Tag-based blocking or requirements seem to be ignored.

**Check:**

1. **Typo in tag name.** Tags are case-sensitive and hierarchy-sensitive. `CrowdControl.Stun` is not the same as `Crowdcontrol.Stun` or `CC.Stun`.
2. **Tag not registered.** If the tag doesn't exist in the project's tag table, it's invalid. Check Project Settings > Gameplay Tags.
3. **Tag query direction.** Are you checking for the tag on the right actor? `ActivationBlockedTags` checks the *owning* ASC's tags. `TargetTagRequirements` on a GE checks the *target's* tags.
4. **Tag count mismatch.** If two effects grant the same tag, removing one effect doesn't remove the tag -- it decrements the count. The tag is only fully removed when the count reaches 0.
5. **Hierarchy mismatch.** Checking for `CrowdControl` matches `CrowdControl.Stun` (child matches parent). But checking for `CrowdControl.Stun` does NOT match `CrowdControl` (parent doesn't match child).

---

## Cue Doesn't Play

**Symptom:** The effect applies, the attribute changes, but no VFX/SFX.

**Check:**

1. **Cue tag matches?** The tag on the GE's GameplayCue component must match the tag on the cue notify asset. Exactly.
2. **Cue in scan path?** The cue asset's folder must be in `GameplayCueNotifyPaths`. Check Project Settings > Gameplay Abilities.
3. **Cue loaded?** Check the log for `"Missing gameplay cue handler"`. If the cue isn't loaded, it won't play.
4. **Suppressed?** If `ShouldSuppressGameplayCues` returns true for the target actor, all cues are silently dropped.
5. **Local vs replicated?** A replicated cue on a simulated proxy won't play if the proxy doesn't have the ASC configured for cue handling.
6. **Effect is Instant?** Instant effects trigger `Execute` cues. Duration effects trigger `Add`/`Remove` cues. Make sure the cue handles the right event type.

---

## Compile Errors

### `"cannot open include file: *.generated.h"`

**Cause:** The `.generated.h` file is auto-generated by Unreal Header Tool (UHT). It doesn't exist until you compile.

**Fix:** Build the project. If the error persists, check for syntax errors in the header file *above* the `GENERATED_BODY()` macro -- UHT will fail silently and not generate the file.

### `"unresolved external symbol"`

**Cause:** You're using a GAS class but haven't linked the module.

**Fix:** Add `"GameplayAbilities"`, `"GameplayTags"`, and `"GameplayTasks"` to your module's `Build.cs`:

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "GameplayAbilities",
    "GameplayTags",
    "GameplayTasks"
});
```

### `"ATTRIBUTE_ACCESSORS undeclared identifier"`

**Cause:** Missing include.

**Fix:** Include `"AttributeSet.h"` in your attribute set header. The `ATTRIBUTE_ACCESSORS` macro is defined there.

---

## AttributeSet Is Null

**Symptom:** `GetSet<UMyAttributeSet>()` returns null. Attribute modifications crash.

**Check:**

1. **Is the AttributeSet a subobject of the ASC's owner?** AttributeSets must be created as subobjects (usually `CreateDefaultSubobject` in the character/player state constructor) of the actor that owns the ASC, or registered manually.
2. **Is the ASC initialized?** `InitAbilityActorInfo` must be called before accessing attribute sets.
3. **Wrong ASC?** If the ASC is on the PlayerState but you're checking the Pawn's ASC, you'll get null. Make sure `IAbilitySystemInterface::GetAbilitySystemComponent()` returns the right ASC.
4. **Timing issue?** In multiplayer, the ASC and its attribute sets might not be ready immediately on `BeginPlay`. Use `OnAbilitySystemInitialized` or a similar delegate.

---

## Prediction Feels Wrong

**Symptom:** Ability activates but there's a visible delay, rubber-banding, or double-triggering of effects.

**Check:**

1. **Net Execution Policy:** Must be `LocalPredicted` for predicted abilities
2. **Replication Mode:** `Mixed` or `Full` for predicted abilities (not `Minimal` without extra work)
3. **Effects outside prediction window:** Any effect applied after `ActivateAbility` returns (timers, delays) won't be predicted
4. **ExecCalc prediction:** Execution Calculations are server-only. The client can't predict the calculated value.
5. **Double cues:** If you see a cue play twice (once predicted, once from server), the prediction key matching might be failing. Check that the GE has a valid prediction key.

## Related Pages

- [ShowDebug AbilitySystem](showdebug-abilitysystem.md) -- visual state inspection
- [Logging](logging.md) -- enabling verbose logs
- [Prediction](../networking/prediction.md) -- understanding prediction behavior
- [Modifier Formula](../reference/modifier-formula.md) -- verifying modifier math
