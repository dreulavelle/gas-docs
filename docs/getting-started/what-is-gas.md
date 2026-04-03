---
icon: material/help-circle-outline
---

# What Is the Gameplay Ability System?

The **Gameplay Ability System** (GAS) is a framework built into Unreal Engine for handling anything a character can *do* or *have done to them*. Attacks, spells, buffs, debuffs, cooldowns, status effects, resource costs, crowd control — GAS provides a unified pipeline for all of it.

It was originally built by Epic for **Paragon** (their competitive MOBA) and later shipped as a plugin available to all UE projects. Because it was designed for a fast-paced multiplayer game, it comes with built-in support for networking, client-side prediction, and efficient replication — things that are brutally hard to build from scratch.

## The Problem GAS Solves

Without GAS, building a combat system means hand-wiring every interaction. Consider a game where a character can:

- Attack (dealing damage based on weapon stats)
- Get buffed (+50 MaxHP for 10 seconds)
- Be stunned (can't act for 2 seconds)
- Dodge (costs stamina, grants brief invulnerability)

Without a framework, every one of these needs custom logic, and worse, they all need to know about each other. The stun needs to block the attack. The dodge needs to check if you're stunned. The buff needs to undo itself when it expires. The attack needs to check if the target is invulnerable. The more mechanics you add, the more everything touches everything else.

GAS replaces this spaghetti with a **pipeline**. You define your mechanics as data-driven assets (effects, abilities, tags), and GAS handles the interactions, timing, networking, and cleanup for you.

## When to Use GAS

GAS is a good fit when your project has:

- **Multiple abilities** that interact with each other (blocking, cancelling, comboing)
- **Stats that change temporarily** (buffs, debuffs, auras, equipment bonuses)
- **Status effects with duration** (stun for 2 seconds, burn for 10 damage per second)
- **Resource costs and cooldowns** (mana costs, stamina costs, ability cooldowns)
- **Multiplayer requirements** (ability prediction, effect replication)
- **Complex damage formulas** (base damage + modifiers + defense - armor + crits)

If your project has two or more of these, GAS will likely save you time compared to building it yourself.

## When Not to Use GAS

GAS adds complexity. It's not the right tool for every project:

- **Simple prototypes** — If you just need "press button, thing happens" with no stats or interactions, a direct function call is simpler.
- **Non-combat games** — A puzzle game or walking simulator probably doesn't need GAS. (Though some non-combat games use it for complex item/status systems.)
- **Projects that avoid C++** — GAS requires a small C++ foundation. If your project is 100% Blueprint with no willingness to touch C++, GAS will fight you. That said, the C++ portion is minimal and templated — the majority of GAS work is Blueprint and editor configuration.

!!! note "The C++ question"
    A common concern: "I don't know C++, can I use GAS?" The honest answer is: you need *some* C++, but it's a small, well-defined set of boilerplate. The [Project Setup](project-setup.md) page walks through every line. After that initial setup, you can do 90%+ of your GAS work in Blueprint.

## GAS vs. Building It Yourself

Here's what GAS gives you for free that you'd otherwise have to build and maintain:

| Feature | GAS | Hand-Rolled |
|---|---|---|
| Buff/debuff with auto-expiry | Built-in (Duration effects) | Manual timers + cleanup |
| Stat modification with undo | Built-in (CurrentValue vs BaseValue) | Manual tracking of every modifier |
| Cooldowns | Built-in (tag-based) | Manual timers per ability |
| Resource costs with pre-check | Built-in (Cost GE) | Manual check + subtract |
| Ability blocking/cancelling | Built-in (tag queries) | Manual if-checks everywhere |
| Stacking rules | Built-in (stack policies) | Manual per-effect |
| Client-side prediction | Built-in | Extremely hard to build |
| Effect replication | Built-in | Manual per-effect type |
| Damage pipeline | ExecCalc + meta attributes | Manual calculation chains |

The tradeoff is learning curve. GAS has a lot of concepts to absorb upfront. But once you understand the mental model, adding new mechanics becomes mostly configuration rather than code.

## What's Next

Ready to learn how GAS actually works? Head to [The Mental Model](mental-model.md) — it's the most important page in this entire guide.
