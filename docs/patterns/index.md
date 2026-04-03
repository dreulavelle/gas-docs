---
title: Patterns
description: Battle-tested design patterns for GAS — common abilities, damage pipelines, buff systems, tag architecture, C++ vs Blueprint decisions, and naming conventions.
---

# Patterns

This section covers the design patterns that experienced GAS developers have landed on after shipping games. These aren't the only way to do things, but they represent well-tested approaches that work across a range of project types.

!!! info "Patterns vs Recipes"
    **Patterns** explain the *design* -- the why, the trade-offs, and the architecture. **[Recipes](../recipes/index.md)** are the step-by-step *procedures* for implementing specific things. Patterns link to recipes where applicable.

## In This Section

### [Common Abilities](common-abilities.md)

Implementation patterns for abilities that show up in nearly every action game: stun/CC, sprint, aim-down-sights, lifesteal, critical hits, interaction systems, passive abilities, and toggle abilities.

### [Damage Pipeline](damage-pipeline.md)

The end-to-end flow of damage from source to target: GE spec creation, ExecCalc with armor/resistance/crits, meta attributes, `PostGameplayEffectExecute` for health subtraction and death checks. Full code example included.

### [Buff/Debuff System](buff-debuff-system.md)

Duration effects with tags, stacking policies, UI representation, cleansing/dispelling, buff categories, and immunity to specific debuff types.

### [Tag Architecture](tag-design.md)

Namespace design principles, a 21-namespace starter preset, loading tags from `.ini` files, and governance strategies for teams.

### [C++ vs Blueprint](cpp-vs-blueprint.md)

What must be C++, what should be C++, what works best in Blueprint, and how the split changes with project scale. Includes a table mapping every GAS asset type to its recommended creation location.

### [Naming Conventions](naming-conventions.md)

Prefix conventions for assets (GA\_, GE\_, GC\_), class naming (A/U prefixes), tag naming patterns, and Content Browser folder structure.
