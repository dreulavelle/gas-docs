---
title: "Recipe: Dodge Roll Walkthrough"
description: Complete walkthrough building a dodge roll ability with stamina cost, cooldown, i-frames, montage, and input.
---

# Recipe: Dodge Roll Walkthrough

**Goal:** Build a complete dodge roll ability with:

- Stamina cost (-20)
- Cooldown (0.5 seconds)
- I-frames (invulnerability during the roll)
- Animation montage
- Input binding

This walks through the full process end-to-end.

---

## Step 1: Create the Cost GE

**Asset:** `GE_Cost_Stamina_20`

1. Right-click > Gameplay Effect
2. **Duration Policy:** Instant
3. **Modifiers:** Add one modifier:
    - Attribute: `MyAttributeSet.Stamina`
    - Modifier Op: `Add (Base)`
    - Magnitude: `-20` (negative = costs stamina)

That's it for the cost. No tags, no cue.

## Step 2: Create the Cooldown GE

**Asset:** `GE_Cooldown_DodgeRoll`

1. Right-click > Gameplay Effect
2. **Duration Policy:** Has Duration
3. **Duration Magnitude:** `0.5` (seconds)
4. **Tags granted to target** (via TargetTagsGameplayEffectComponent):
    - `Cooldown.Ability.DodgeRoll`
5. **Asset Tags** (via AssetTagsGameplayEffectComponent):
    - `Cooldown.Ability.DodgeRoll`

The cooldown tag is what GAS checks -- while this tag is present, the ability can't activate.

## Step 3: Create the Ability Blueprint

**Asset:** `GA_DodgeRoll`

1. Right-click > Blueprint Class > your base ability class
2. Name: `GA_DodgeRoll`

### Class Defaults

| Property | Value |
|:---|:---|
| **Ability Tags** | `Ability.Movement.DodgeRoll` |
| **Activation Blocked Tags** | `CrowdControl.Stun`, `State.Dead`, `State.DodgeRolling` |
| **Activation Owned Tags** | `State.DodgeRolling`, `State.Invulnerable` |
| **Block Abilities with Tag** | `Ability.Combat` (can't attack while rolling) |
| **Instancing Policy** | `InstancedPerActor` |
| **Net Execution Policy** | `LocalPredicted` |
| **Cost Gameplay Effect** | `GE_Cost_Stamina_20` |
| **Cooldown Gameplay Effect** | `GE_Cooldown_DodgeRoll` |

!!! info "Activation Owned Tags for I-frames"
    `State.Invulnerable` is granted when the ability activates and removed when it ends. Your damage pipeline checks for this tag and skips damage if present. This is the i-frame mechanism. `State.DodgeRolling` blocks re-activation during the roll.

### Event Graph

```
Event ActivateAbility
  │
  ├─ Commit Ability
  │   └─ [Branch: Failed] → End Ability (Cancelled = true)
  │
  └─ [Branch: Success]
      │
      ├─ Play Montage And Wait
      │   ├─ Montage: DodgeRoll_Montage
      │   ├─ Rate: 1.0
      │   └─ Section: (leave empty for full montage)
      │
      ├─ On Completed → End Ability (Cancelled = false)
      ├─ On Blend Out → End Ability (Cancelled = false)
      ├─ On Interrupted → End Ability (Cancelled = true)
      └─ On Cancelled → End Ability (Cancelled = true)
```

**Commit Ability** does two things:

1. Checks if the cost can be paid and the cooldown is ready
2. Deducts the cost and applies the cooldown

If it fails (not enough stamina, still on cooldown), the ability ends immediately.

## Step 4: Set Up the Montage

1. Create or use an existing dodge roll animation montage
2. Make sure it's set up to play on the appropriate slot (usually FullBody)
3. Duration should be around 0.5-0.8 seconds for a responsive feel

Optional: Add anim notifies for:

- A **GameplayCue (Burst)** notify at the moment feet leave the ground for a dust puff
- A notify to trigger root motion (or use the animation's root motion)

## Step 5: Wire Input

Follow the [Bind Ability to Input](bind-ability-to-input.md) recipe:

1. **Input Action:** `IA_Dodge` (bool, Pressed trigger)
2. **Input Mapping Context:** Map to your dodge key (e.g., Space bar, B button)
3. **InputTag:** `InputTag.Dodge`
4. **On the ability:** Set InputTag to `InputTag.Dodge`

## Step 6: I-Frame Integration

In your [damage pipeline](../patterns/damage-pipeline.md), check for invulnerability before applying damage:

```cpp
// In PostGameplayEffectExecute or ExecCalc
UAbilitySystemComponent* TargetASC = Data.Target.GetAbilitySystemComponent();
if (TargetASC && TargetASC->HasMatchingGameplayTag(
    FGameplayTag::RequestGameplayTag(TEXT("State.Invulnerable"))))
{
    // Skip damage
    SetIncomingDamage(0.f);
    return;
}
```

Or use the **Immunity system**: create an Infinite GE that the dodge ability applies, which uses the `ImmunityGameplayEffectComponent` to block damage effects. This is cleaner but more setup.

## Step 7: Test

### Basic Test

1. PIE
2. Press dodge key → character should play roll animation
3. Stamina should decrease by 20
4. Press dodge again immediately → should not activate (cooldown)
5. Wait 0.5s → dodge should work again

### I-Frame Test

1. Set up a damage source (enemy attack or debug command)
2. Time the damage to hit during the roll
3. Verify no damage is taken during the roll
4. Verify damage works normally outside the roll

### Edge Cases

- Dodge at 0 stamina → should fail (Commit Ability returns false)
- Dodge while stunned → should fail (Activation Blocked Tags)
- Dodge while already dodging → should fail (`State.DodgeRolling` in blocked tags)
- Network test: dodge on client, verify prediction feels responsive

## Full Assembly Diagram

```
Player presses Dodge key
  │
  ▼
Input System: IA_Dodge → InputTag.Dodge → ASC looks for matching ability
  │
  ▼
GA_DodgeRoll.ActivateAbility()
  ├─ Check: Activation Blocked Tags (stun? dead? already rolling?)
  ├─ Commit: GE_Cost_Stamina_20 applied (instant, -20 stamina)
  ├─ Commit: GE_Cooldown_DodgeRoll applied (0.5s duration)
  ├─ Tags granted: State.DodgeRolling, State.Invulnerable
  ├─ Play DodgeRoll_Montage
  │   ├─ [Frame 5] AnimNotify: GameplayCue.Movement.DodgeDust
  │   └─ [Montage ends]
  ├─ Tags removed: State.DodgeRolling, State.Invulnerable
  └─ End Ability
```

## Related

- [Common Abilities](../patterns/common-abilities.md) -- more ability patterns
- [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) -- cost/cooldown deep dive
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) -- PlayMontageAndWait details
- [Damage Pipeline](../patterns/damage-pipeline.md) -- where i-frames are checked
