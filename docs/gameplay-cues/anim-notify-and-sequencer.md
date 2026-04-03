---
title: AnimNotify and Sequencer
description: Triggering Gameplay Cues from Animation Montages via UAnimNotify_GameplayCue and from Level Sequences via the Gameplay Cue Track.
---

# AnimNotify and Sequencer

Gameplay Cues can be triggered from Animation Montages and Level Sequences, not just from Gameplay Effects or direct code calls. This is essential for tightly syncing VFX/SFX to specific animation frames or cinematic moments.

## Animation Notifies

UE 5.7 provides two anim notify classes for Gameplay Cues:

### UAnimNotify_GameplayCue (Burst)

**Editor display name:** GameplayCue (Burst)

A standard `UAnimNotify` that fires a one-shot (Execute) Gameplay Cue at a specific frame in an animation. Use this for instant feedback tied to animation events -- weapon swish sounds, footstep impacts, cast effects.

**Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `GameplayCue` | `FGameplayCueTag` | The `GameplayCue.*` tag to trigger |

**How it works:**

When the notify fires during animation playback, it:

1. Finds the owning actor's skeletal mesh component
2. Gets the actor from the mesh component
3. Calls `UGameplayCueManager::ExecuteGameplayCue_NonReplicated` on the actor

!!! info "Non-replicated by design"
    Animation-driven cues are **non-replicated**. Since animations play on all clients via AnimMontage replication, each client fires the cue locally from its own animation. There's no need to replicate the cue event separately -- the animation replication already ensures all clients see it.

**Setup:**

1. Open your Animation Montage in the editor
2. Right-click on the Notifies track at the desired frame
3. Select **Add Notify > GameplayCue (Burst)**
4. In the Details panel, set the `GameplayCue` tag
5. Create a matching cue notify asset ([Burst](cue-types.md#ugameplaycuenotify_burst) is the natural fit)

### UAnimNotify_GameplayCueState (Looping)

**Editor display name:** GameplayCue (Looping)

A `UAnimNotifyState` that triggers a duration-based Gameplay Cue across a range of animation frames. It fires Add at the start of the notify range and Remove at the end.

**Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `GameplayCue` | `FGameplayCueTag` | The `GameplayCue.*` tag to trigger |

**Lifecycle mapping:**

| Notify Event | Cue Action |
|:---|:---|
| `NotifyBegin` | `AddGameplayCue_NonReplicated` |
| `NotifyTick` | (no cue action by default) |
| `NotifyEnd` | `RemoveGameplayCue_NonReplicated` |

**Setup:**

1. Open your Animation Montage
2. Right-click on the Notifies track
3. Select **Add Notify State > GameplayCue (Looping)**
4. Drag the ends to set the duration range
5. Set the `GameplayCue` tag
6. Create a matching [Looping](cue-types.md#agameplaycuenotify_looping) or [Actor](cue-types.md#agameplaycuenotify_actor) cue notify

**Use cases:**

- Weapon trail effects during the active swing frames
- Charge-up particles while a heavy attack winds up
- Ground pound dust cloud that persists during the landing recovery

### Combining with Abilities

A common pattern is to have an ability play a montage (via `PlayMontageAndWait` ability task), and the montage's anim notifies trigger the cues:

```
Ability activates
  → PlayMontageAndWait starts montage
    → Frame 12: AnimNotify_GameplayCue fires "GameplayCue.Ability.Slash.Swish"
    → Frame 18: AnimNotify_GameplayCue fires "GameplayCue.Ability.Slash.Impact"
    → Montage ends
  → Ability ends
```

This keeps VFX timing tightly coupled to animation without any manual delay management in the ability itself.

!!! tip "Don't double-trigger"
    If your ability also applies a Gameplay Effect with a cue tag, and the montage has a notify for the same cue, you'll get the cue fired twice. Pick one triggering mechanism -- usually the anim notify for frame-precise effects, and the GE for gameplay-state-driven effects.

## Sequencer Integration

Level Sequences (cinematics, in-game cutscenes) can trigger Gameplay Cues through a dedicated track type.

### UMovieSceneGameplayCueTrack

The Gameplay Cue track can be added to any actor binding in Sequencer. It supports multiple rows and two section types.

**To add:**

1. In Sequencer, select your actor binding
2. Click **+ Track > Gameplay Cue**
3. Right-click on the track to add sections

### Section Types

#### Trigger Section (UMovieSceneGameplayCueTriggerSection)

Fires one-shot (Execute) cues at specific keyframe times within the section. Each key is a `FMovieSceneGameplayCueKey` with full cue parameter configuration.

Use for: hit impacts at specific cinematic moments, ability cast flashes, explosion effects.

#### Range Section (UMovieSceneGameplayCueSection)

A duration section that fires Add at the start and Remove at the end. Contains a single `FMovieSceneGameplayCueKey` for configuration.

Use for: persistent effects during a cinematic sequence (aura while a character powers up, fire while a building burns).

### FMovieSceneGameplayCueKey

Each key/section configures the cue through this struct:

| Field | Type | Description |
|:---|:---|:---|
| `Cue` | `FGameplayCueTag` | The GameplayCue tag to trigger |
| `Location` | `FVector` | Location (relative to attached component if applicable) |
| `Normal` | `FVector` | Impact normal |
| `AttachSocketName` | `FName` | Socket on skeletal mesh to trigger at |
| `NormalizedMagnitude` | `float` | 0-1 magnitude |
| `Instigator` | `FMovieSceneObjectBindingID` | Instigator actor binding in sequencer |
| `EffectCauser` | `FMovieSceneObjectBindingID` | Effect causer actor binding |
| `PhysicalMaterial` | `const UPhysicalMaterial*` | Surface material |
| `GameplayEffectLevel` | `int32` | GE level (default 1) |
| `AbilityLevel` | `int32` | Ability level (default 1) |
| `bAttachToBinding` | `bool` | Attach cue to the track's bound actor |

### Custom Track Handler

You can override how sequencer cues are invoked by setting a custom handler:

```cpp
UMovieSceneGameplayCueTrack::SetSequencerTrackHandler(MyCustomHandler);
```

The delegate signature is:

```cpp
DECLARE_DYNAMIC_DELEGATE_FourParams(FMovieSceneGameplayCueEvent,
    AActor*, Target,
    FGameplayTag, GameplayTag,
    const FGameplayCueParameters&, Parameters,
    EGameplayCueEvent::Type, Event);
```

This is useful if you need custom routing for cinematic cues (e.g., routing through a different manager, or adding game-specific context).

## Practical Patterns

### Animation-Driven Combat VFX

```
AttackMontage
├── Section: WindUp
│   └── NotifyState: GameplayCue.Weapon.Trail (Looping)
│       → Weapon trail particles active during swing
├── Section: Strike
│   ├── Notify: GameplayCue.Weapon.Swish (Burst)
│   │   → Whoosh sound at swing frame
│   └── [Hit detection happens in ability code]
│       → Ability applies DamageGE with GameplayCue.Hit.Physical
│       → Hit impact cue fires from the GE
└── Section: Recovery
```

### Cinematic Ability Showcase

```
LevelSequence: BossIntro
├── Camera Track: cinematic camera
├── Boss Actor Binding:
│   ├── Animation Track: PowerUp montage
│   └── GameplayCue Track:
│       ├── [0.0 - 3.0] Range: GameplayCue.Boss.PowerUpAura
│       ├── [1.5] Trigger: GameplayCue.Boss.EnergyBurst
│       └── [3.0] Trigger: GameplayCue.Boss.Roar
└── Environment Binding:
    └── GameplayCue Track:
        └── [2.5] Trigger: GameplayCue.Environment.ScreenShake
```

## Related Pages

- [Cue Notify Types](cue-types.md) -- which cue class to pair with your notify
- [Cue Parameters](cue-parameters.md) -- the data that flows to your cue
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) -- `PlayMontageAndWait` for ability-driven montages
- [Add a Gameplay Cue](../recipes/add-gameplay-cue.md) -- step-by-step creation recipe
