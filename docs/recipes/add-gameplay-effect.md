---
title: "Recipe: Add a Gameplay Effect"
description: Step-by-step procedure for creating a Gameplay Effect with modifiers, tags, and requirements.
---

# Recipe: Add a Gameplay Effect

**Goal:** Create a Gameplay Effect that modifies attributes and applies tags.

**Prerequisites:** An Attribute Set with the target attributes. See [Add an Attribute](add-attribute.md).

---

## Steps

### 1. Create the Asset

1. In Content Browser, right-click in your `GAS/Effects/` folder
2. Select **Gameplay > Gameplay Effect** (or **Blueprint Class > GameplayEffect**)
3. Name it with the `GE_` prefix: `GE_Buff_SpeedBoost`

### 2. Set Duration Policy

Open the effect and find **Duration Policy**:

| Policy | When to Use |
|:---|:---|
| **Instant** | One-time attribute change (heal, damage via modifier) |
| **Has Duration** | Temporary buff/debuff that expires |
| **Infinite** | Persists until explicitly removed |

For a speed buff: **Has Duration**, then set **Duration Magnitude** to your desired seconds (e.g., 5.0).

### 3. Add Modifiers

In the **Modifiers** section, click **+** to add:

| Field | Example Value | Notes |
|:---|:---|:---|
| **Attribute** | `MyAttributeSet.MoveSpeed` | The attribute to modify |
| **Modifier Op** | `Multiply Additive` | See [Modifier Formula](../reference/modifier-formula.md) |
| **Modifier Magnitude > Type** | `Scalable Float` | Simplest option |
| **Scalable Float Value** | `0.3` | +30% speed (for Multiply Additive) |

Common modifier ops:

- **Add (Base):** Flat addition to base value
- **Multiply Additive:** Additive percentage (0.3 = +30%)
- **Multiply Compound:** Multiplicative stacking (1.3 = 130%)

### 4. Configure Tags (via GE Components)

In 5.3+, tags are managed through **GE Components**. The key ones:

**TargetTagsGameplayEffectComponent** -- tags granted to the target while the effect is active:

- Add `Buff.SpeedBoost` (identifies this buff)
- Add `State.Sprinting` (if appropriate)

**AssetTagsGameplayEffectComponent** -- tags on the effect itself (for querying):

- Add `Effect.Type.Buff`

**TargetTagRequirementsGameplayEffectComponent** -- requirements for the effect to apply/stay active:

- **Application Tag Requirements:** Tags the target must/must-not have for the effect to apply
- **Ongoing Tag Requirements:** Tags the target must have for the effect to remain active (removed if requirements fail)

### 5. (Optional) Set Stacking

If the buff can stack:

| Field | Example |
|:---|:---|
| **Stacking Type** | `Stack Per Target` |
| **Stack Limit Count** | `3` |
| **Stack Duration Refresh Policy** | `Refresh On Successful Application` |
| **Stack Expiration Policy** | `Clear Entire Stack` |

See [Stacking](../gameplay-effects/stacking.md) for all options.

### 6. (Optional) Add a Cue

Add the **GameplayCue GE Component** and set the cue tag:

- `GameplayCue.Buff.SpeedBoost`

For Duration/Infinite effects, this triggers Add when applied and Remove when expired/removed.

### 7. Apply from an Ability

=== "Blueprint"

    In your ability's event graph:
    ```
    Make Outgoing Gameplay Effect Spec (class: GE_Buff_SpeedBoost)
      → Apply Gameplay Effect Spec to Owner
    ```

=== "C++"

    ```cpp
    FGameplayEffectSpecHandle Spec = MakeOutgoingGameplayEffectSpec(
        SpeedBuffClass, GetAbilityLevel());
    ApplyGameplayEffectSpecToOwner(CurrentActivationInfo, Spec);
    ```

### 8. Test

1. PIE
2. Activate the ability that applies this effect
3. `showdebug abilitysystem` -- confirm the effect appears under Active Effects
4. Verify attribute changes (check the attribute values)
5. Wait for duration to expire and confirm the effect is removed

---

## Checklist

- [ ] Asset created with `GE_` prefix
- [ ] Duration policy set
- [ ] Modifiers configured with correct attribute and operation
- [ ] Tags granted via TargetTagsGameplayEffectComponent
- [ ] Application requirements set (if needed)
- [ ] Stacking configured (if applicable)
- [ ] Cue tag set (if visual feedback needed)
- [ ] Applied from ability code
- [ ] Tested in PIE with ShowDebug

## Related

- [Gameplay Effects Overview](../core-concepts/gameplay-effects-overview.md) -- concepts
- [Modifiers](../gameplay-effects/modifiers.md) -- modifier deep dive
- [GE Components](../gameplay-effects/ge-components.md) -- the component architecture
- [Blueprint Function Library](../reference/blueprint-library.md) -- all available Blueprint nodes for GAS
