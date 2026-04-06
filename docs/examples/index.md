
# Examples

Complete, end-to-end ability implementations. Each example walks through the full GAS loop — effects, ability blueprint, input wiring, and testing — so you can follow along from scratch or use them as reference for your own abilities.

Every example is **standalone**. You can jump to any one without reading the others.

!!! tip "Looking for a simpler starting point?"
    The [Getting Started](../getting-started/index.md) section includes a [melee attack tutorial](../getting-started/your-first-ability.md) — a simplified first ability that covers the full GAS loop. Start there if you're new, then come back here for production-ready versions.

## The Examples

### :material-circle:{ style="color: #4caf50" } Beginner

| Example | What It Demonstrates |
|:---|:---|
| **[Jump](jump.md)** | Stamina cost, cooldown, variable jump height via WaitInputRelease, tag-based CC blocking. |
| **[Sprint Toggle](sprint.md)** | Toggle/held ability, infinite-duration effects, periodic stamina drain, WaitInputRelease. |
| **[Stamina Regen](stamina-regen.md)** | Pure GE + tag system -- passive regen with delay-after-use, Ongoing Tag Requirements, no ability code. |

### :material-circle:{ style="color: #ffa726" } Intermediate

| Example | What It Demonstrates |
|:---|:---|
| **[Dodge Roll](dodge-roll.md)** | Stamina cost, cooldown, i-frames via Activation Owned Tags, root motion, ability blocking. |
| **[Melee Attack](melee-attack.md)** | Animation-driven hit timing, SetByCaller damage, montage lifecycle, gameplay events. |
| **[Ranged Attack](ranged-attack.md)** | Projectile spawning, passing GE specs to actors, mana cost, aim direction, travel-time damage. |
| **[Passive Aura](passive-aura.md)** | Passive activation, periodic area scanning, applying effects to other actors' ASCs, handle tracking. |

### :material-circle:{ style="color: #ef5350" } Advanced

| Example | What It Demonstrates |
|:---|:---|
| **[Custom Damage Calculation](exec-calc-damage.md)** | UGameplayEffectExecutionCalculation, attribute captures, elemental resistance, crit formula. C++ only. |
| **[Network-Predicted Ability](predicted-ability.md)** | Client-side prediction, predict-confirm-reject cycle, FScopedPredictionWindow, multiplayer. C++ only. |

## What You Need

All examples assume you've completed [Project Setup](../getting-started/project-setup.md) and have:

- A character with an **Ability System Component**
- An **AttributeSet** with at minimum Health, Stamina, and Mana
- A **base ability class** (`YourProjectGameplayAbility`) with an InputTag property
- An **input binding system** that routes Enhanced Input actions to abilities by tag

If any of that is missing, start with [Project Setup](../getting-started/project-setup.md).
