---
icon: material/book-open-page-variant
---

# Examples

Complete, end-to-end ability implementations. Each example walks through the full GAS loop — effects, ability blueprint, input wiring, and testing — so you can follow along from scratch or use them as reference for your own abilities.

Every example is **standalone**. You can jump to any one without reading the others.

## The Examples

| Example | What It Demonstrates | Complexity |
|:---|:---|:---|
| **[Jump](jump.md)** | The simplest GAS ability. Stamina cost, cooldown, CC blocking, airborne state tracking. | :material-star: |
| **[Melee Attack](melee-attack.md)** | Animation-driven hit timing, SetByCaller damage, montage lifecycle, gameplay events. | :material-star::material-star: |
| **[Dodge Roll](dodge-roll.md)** | Stamina cost, cooldown, i-frames via Activation Owned Tags, root motion, ability blocking. | :material-star::material-star::material-star: |
| **[Ranged Attack](ranged-attack.md)** | Projectile spawning, passing GE specs to actors, mana cost, aim direction, travel-time damage. | :material-star::material-star::material-star: |

!!! tip "Reading order"
    The examples are listed in order of increasing complexity — jump is the simplest, ranged attack is the most involved. But they're written to be self-contained, so feel free to jump (pun intended) to whichever one matches what you're building.

## How These Relate to Getting Started

The [Getting Started](../getting-started/index.md) section includes a [simplified jump ability](../getting-started/your-first-ability.md) as part of the guided tutorial. The [Jump example here](jump.md) is the expanded version — it adds airborne state tracking, landing detection, and a comparison to the engine's built-in `UGameplayAbility_CharacterJump`.

If you've already done the Getting Started tutorial, the Jump example here will feel like a natural extension.

## What You Need

All examples assume you've completed [Project Setup](../getting-started/project-setup.md) and have:

- A character with an **Ability System Component**
- An **AttributeSet** with at minimum Health, Stamina, and Mana
- A **base ability class** (`YourProjectGameplayAbility`) with an InputTag property
- An **input binding system** that routes Enhanced Input actions to abilities by tag

If any of that is missing, start with [Project Setup](../getting-started/project-setup.md).
