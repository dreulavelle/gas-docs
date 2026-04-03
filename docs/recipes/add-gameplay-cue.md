---
title: "Recipe: Add a Gameplay Cue"
description: Step-by-step procedure for choosing a cue type, creating the Blueprint, setting the tag, and triggering it.
---

# Recipe: Add a Gameplay Cue

**Goal:** Create a Gameplay Cue that plays VFX/SFX in response to a gameplay event.

**Prerequisites:** A Gameplay Effect or Ability that will trigger the cue. See [Gameplay Cues Overview](../core-concepts/gameplay-cues-overview.md) for concepts.

---

## Steps

### 1. Choose the Cue Type

| Need | Cue Type | Parent Class |
|:---|:---|:---|
| One-shot effect (hit, cast, pop) | **Burst** | `GameplayCueNotify_Burst` |
| One-shot with timelines/delays | **BurstLatent** | `GameplayCueNotify_BurstLatent` |
| Duration effect (aura, trail, loop) | **Looping** | `GameplayCueNotify_Looping` |
| Fully custom persistent actor | **Actor** | `GameplayCueNotify_Actor` |
| Minimal code-only handler | **Static** | `GameplayCueNotify_Static` |

See [Cue Notify Types](../gameplay-cues/cue-types.md) for detailed guidance.

### 2. Create the Blueprint

1. In Content Browser, navigate to `GAS/Cues/`
2. Right-click > **Blueprint Class**
3. Search for your chosen parent class (e.g., `GameplayCueNotify_Burst`)
4. Name with `GC_` prefix: `GC_Hit_Physical`

### 3. Set the GameplayCue Tag

Open the Blueprint, go to **Class Defaults**:

- Set **GameplayCue Tag** to `GameplayCue.Hit.Physical`

!!! tip "Auto-derive from name"
    If you follow the naming convention (dots → underscores), the tag can be auto-derived. `GC_Hit_Physical` → `GameplayCue.Hit.Physical`. But it's safer to set it explicitly.

### 4. Implement the Events

=== "Burst"

    Configure effects in Class Defaults:

    1. **BurstEffects > Burst Particles**: Add Niagara systems
    2. **BurstEffects > Burst Sounds**: Add sound cues/waves
    3. **BurstEffects > Burst Camera Shakes**: Add camera shake classes
    4. **DefaultPlacementInfo**: Set to use hit location from parameters

    Optionally, override the **OnBurst** event for additional custom logic.

=== "Looping"

    Configure effects in Class Defaults:

    1. **ApplicationEffects**: One-shot effects when the buff starts
    2. **LoopingEffects**: Persistent particles/sounds while active
    3. **RecurringEffects**: Effects per periodic tick
    4. **RemovalEffects**: One-shot effects when the buff ends

    Override **OnApplication**, **OnLoopingStart**, **OnRecurring**, **OnRemoval** events as needed.

=== "Actor"

    1. Add components to the actor (Niagara, Audio, etc.)
    2. Override **On Become Relevant** (WhileActive): activate VFX
    3. Override **On Cease Relevant** (OnRemove): deactivate VFX
    4. Set **bAutoAttachToOwner** = true if it should follow the target
    5. Set **bAutoDestroyOnRemove** = true

### 5. Trigger the Cue

=== "From a Gameplay Effect"

    On your GE, add the **GameplayCue GE Component** and set the tag to `GameplayCue.Hit.Physical`.

    - Instant effects → `Execute` event
    - Duration/Infinite effects → `Add` on apply, `Remove` on end

=== "From Ability Code"

    ```cpp
    // One-shot
    ASC->ExecuteGameplayCue(CueTag, CueParams);

    // Duration (you manage the remove)
    ASC->AddGameplayCue(CueTag, CueParams);
    // Later:
    ASC->RemoveGameplayCue(CueTag);
    ```

=== "From Animation"

    Add a **GameplayCue (Burst)** or **GameplayCue (Looping)** anim notify to your montage. See [AnimNotify and Sequencer](../gameplay-cues/anim-notify-and-sequencer.md).

### 6. Verify Scan Path

Make sure your cue folder is in the GameplayCueManager's scan paths:

**Project Settings > Gameplay Abilities > GameplayCue Notify Paths:**
```
/Game/GAS/Cues
```

### 7. Test

1. PIE
2. Trigger the ability or effect that fires the cue
3. Verify VFX/SFX play at the correct location
4. For duration cues: verify they start and stop correctly
5. Check the log for any "missing gameplay cue" warnings

---

## Checklist

- [ ] Cue type chosen based on needs
- [ ] Blueprint created with `GC_` prefix
- [ ] GameplayCue tag set in Class Defaults
- [ ] Effects configured (particles, sounds, etc.)
- [ ] Events implemented (OnBurst / OnApplication / etc.)
- [ ] Trigger configured (GE cue component, ability code, or anim notify)
- [ ] Scan path includes the cue folder
- [ ] Tested in PIE

## Related

- [Cue Notify Types](../gameplay-cues/cue-types.md) -- detailed type comparison
- [Cue Parameters](../gameplay-cues/cue-parameters.md) -- passing data to your cue
- [Cue Manager](../gameplay-cues/cue-manager.md) -- scan paths and loading
