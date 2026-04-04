
# Gameplay Cues — The Big Picture

Gameplay Cues are how GAS handles visual and audio feedback — particle effects, sounds, screen shakes, damage numbers, buff auras. They're the polish layer that makes your gameplay *feel* good.

But here's the one rule you absolutely must internalize:

!!! danger "The Golden Rule"
    **No gameplay logic in Gameplay Cues. Ever.**

    Cues are cosmetic only. They can be skipped, batched, throttled, or not executed at all — the game must function identically without them. If removing every cue from your project would change gameplay outcomes, something is in the wrong place.

This page covers the concepts. For the full deep dive, see the [Gameplay Cues deep dive](../gameplay-cues/index.md).

## Why Cues Exist

You might wonder: why not just spawn particles and play sounds directly in your ability code? You *can*, but cues solve several problems that direct spawning doesn't:

**Decoupled feedback.** Your ability code says "a fireball hit something." The cue decides what that *looks like*. Designers can change VFX without touching ability logic. Different platforms can show different quality levels.

**Efficient replication.** Cues replicate through a lightweight, optimized path separate from the main GAS replication. Instead of replicating "spawn this particle at this location with these parameters," the server sends a compact tag-based message and the client handles the rest locally.

**Automatic lifecycle management.** For stateful effects (a burning aura while a DoT is active), cues tied to a Duration/Infinite effect are automatically added when the effect starts and removed when it ends. No manual cleanup.

**Batching and throttling.** The GameplayCueManager can batch multiple cues together and throttle how many fire per frame, which matters when 50 actors all take damage simultaneously.

## Execute vs Add/Remove

Gameplay Cues come in two paradigms, matching two different feedback patterns:

### Execute (Fire-and-Forget)

A one-shot event. Something happened, play feedback, done.

**Examples:**

- Hit impact sparks
- Damage number popup
- Ability cast sound
- Footstep dust puff

Execute cues fire once and are immediately forgotten. There's nothing to track, nothing to clean up.

### Add/Remove (Stateful)

Feedback that persists for a duration, then stops.

**Examples:**

- Burning aura while a fire DoT is active
- Shield bubble while a barrier buff is up
- Poison drip particles while poisoned
- Glowing eyes while enraged

Add fires when the effect starts (or when you manually trigger it). Remove fires when the effect ends. The cue is responsible for starting and stopping its visual/audio content in response to these events.

When a cue is tied to a Duration or Infinite Gameplay Effect, the add/remove lifecycle is handled automatically — the effect triggers "add" when it's applied and "remove" when it expires or is removed.

## The Four Notify Types

GAS provides four base classes for Gameplay Cues, each suited to different feedback needs:

| Type | Base Class | Spawned? | Stateful? | Use Case |
|---|---|---|---|---|
| **Static** | `UGameplayCueNotify_Static` | No instance | No | The lightest option — logic only, no actor overhead. Good for simple sounds or triggering existing FX. |
| **Burst** | `AGameplayCueNotify_Burst` | Spawned, then destroyed | No | One-shot effects with configurable VFX/SFX properties in the editor. Hit impacts, damage pops. |
| **Actor** | `AGameplayCueNotify_Actor` | Spawned, persists | Yes | Stateful effects that need an actor in the world. Burning aura, shield bubble. Handles Add/Remove. |
| **Looping** | `AGameplayCueNotify_Looping` | Spawned, persists | Yes | Duration-based looping effects. Similar to Actor but with built-in looping FX support. |

!!! tip "Choosing a type"
    - Need a quick one-shot with no configuration? **Static**
    - Need a one-shot with VFX/SFX configured in the editor? **Burst**
    - Need persistent visuals that start and stop? **Actor** or **Looping**
    - Not sure? **Burst** for fire-and-forget, **Actor** for stateful. You can always change later.

## Tag Matching

Gameplay Cues are routed by tag. Every cue is associated with a tag under the `GameplayCue.*` hierarchy, and the **GameplayCueManager** matches incoming cue events to the right notify asset.

For example:

- Effect grants tag `GameplayCue.Hit.Physical` → the GameplayCueManager finds the cue notify registered to that tag → fires its Execute/Add/Remove
- Effect grants tag `GameplayCue.Status.Burning` → the manager finds and triggers the burning cue

The tag hierarchy works here too. You can have cues at different levels of specificity:

```
GameplayCue.Hit                   → generic hit effect (fallback)
GameplayCue.Hit.Physical          → physical hit sparks
GameplayCue.Hit.Physical.Critical → critical hit with extra flourish
```

The manager looks for the most specific match first, then walks up the hierarchy.

## Triggering Cues

There are two ways cues get triggered:

### Automatic (From Effects)

When you add a `GameplayCue.*` tag to a Gameplay Effect's cue section (via the Gameplay Cue GE Component in 5.3+), the cue fires automatically when the effect is applied:

- **Instant** effects → Execute
- **Duration/Infinite** effects → Add on apply, Remove on expire/removal

This is the most common approach and requires zero code — just configure the tag on the effect.

### Manual (From Ability Code)

You can also fire cues directly from ability or gameplay code:

```cpp
// Execute a one-shot cue
ASC->ExecuteGameplayCue(CueTag, CueParameters);

// Add a stateful cue (you're responsible for removing it)
ASC->AddGameplayCue(CueTag, CueParameters);

// Remove a stateful cue
ASC->RemoveGameplayCue(CueTag);
```

Manual triggering is useful for feedback that isn't tied to an effect — like a UI flash when an ability is ready, or a sound cue when entering a zone.

## Local vs Replicated Cues

By default, cues triggered through the ASC are replicated — the server triggers them and clients see them. But sometimes you want a cue to play only locally:

```cpp
// Local only — no replication
ASC->ExecuteGameplayCueLocal(CueTag, CueParameters);
ASC->AddGameplayCueLocal(CueTag, CueParameters);
ASC->RemoveGameplayCueLocal(CueTag);
```

**Use local cues for:**

- First-person feedback that only the local player should see (screen flash, camera shake)
- Predicted cues on the owning client (play immediately, don't wait for server)
- Feedback for locally-simulated events

**Use replicated cues for:**

- Effects that all players should see (hit impacts, buff auras, status visuals)
- Anything that needs to be consistent across clients

## What's Next

This page covered the concepts. The [Gameplay Cues deep dive](../gameplay-cues/index.md) goes into:

- [Cue Notify Types](../gameplay-cues/cue-types.md) — detailed breakdown of Static, Burst, Actor, and Looping with implementation examples
- [Cue Parameters](../gameplay-cues/cue-parameters.md) — passing data to cues (location, magnitude, source, custom data)
- [Cue Manager](../gameplay-cues/cue-manager.md) — configuration, scan paths, and performance settings
- [AnimNotify and Sequencer](../gameplay-cues/anim-notify-and-sequencer.md) — triggering cues from animations and cinematics
