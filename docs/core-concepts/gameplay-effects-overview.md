
# Gameplay Effects — The Big Picture

Gameplay Effects (GEs) are the workhorses of GAS. Every stat change, every buff, every debuff, every damage instance, every cooldown — they all flow through Gameplay Effects. If abilities are "what your characters do," effects are "what actually happens to the numbers."

This page covers the concepts. For the full deep dive into every property, modifier type, and advanced feature, see the [Gameplay Effects deep dive](../gameplay-effects/index.md).

## What a Gameplay Effect Is

A Gameplay Effect is a **data asset** that describes a modification to an actor's attributes and/or tags. You create them in the editor as Blueprint assets (subclasses of `UGameplayEffect`), configure their properties in the Details panel, and apply them at runtime through C++ or Blueprint.

The key word is *data*. A Gameplay Effect is not code — it's configuration. You don't write logic inside a GE. You fill out fields: which attribute to modify, by how much, for how long, what tags to grant, what requirements must be met. GAS reads this data and does the rest.

!!! tip "Effects are data, not code"
    You can build an entire buff/debuff system, a damage pipeline, resource costs, and cooldowns without writing a single line of C++ inside a Gameplay Effect. Almost all GE work is editor configuration.

## The Three Duration Types

Every Gameplay Effect has a duration policy that fundamentally determines how it behaves:

### Instant

Applies once and is done. Permanently modifies the **BaseValue** of the target attribute.

**Example:** A healing potion restores 50 HP. The effect fires, BaseValue of Health increases by 50, and the effect is gone. There's nothing to track, nothing to undo.

Instant effects don't have an "active" lifetime — they execute and vanish. This means they can't grant tags, can't tick, and can't be removed (they already happened).

### Duration

Applies for a specified time, then automatically reverses itself. Adds a modifier to the **CurrentValue** — the BaseValue is untouched.

**Example:** A strength potion gives +20 AttackPower for 30 seconds. While active, CurrentValue of AttackPower includes the +20 modifier. When the 30 seconds are up, the modifier is removed and CurrentValue drops back to BaseValue. No cleanup code needed.

Duration effects can grant tags (active while the effect is active), trigger periodic ticks, and be manually removed early.

### Infinite

Like Duration, but with no expiration time. The modifier persists until you explicitly remove it.

**Example:** An equipment bonus of +10 Armor. It stays active as long as the equipment is worn. When unequipped, you remove the effect by its handle, and the +10 modifier vanishes.

Infinite effects are commonly used for equipment bonuses, passive skills, auras, and derived attributes.

### Quick Decision Table

| I need to... | Duration Type |
|---|---|
| Deal damage or heal (permanent change) | **Instant** |
| Apply a timed buff/debuff (auto-expires) | **Duration** |
| Apply a persistent modifier (manual removal) | **Infinite** |
| Apply a cooldown | **Duration** (with cooldown tag) |
| Set starting attribute values | **Instant** |
| Create a derived attribute (HealthRegen = Stamina * 0.1) | **Infinite** |
| Apply a periodic effect (DoT, HoT) | **Duration** or **Infinite** with period |

## GE Class vs GE Spec

There's a distinction that trips people up at first: the **Gameplay Effect class** and the **Gameplay Effect Spec** are different things.

### The GE Class (Template)

The `UGameplayEffect` subclass you create in the editor. It defines the *shape* of the effect: which attributes, what modifier operations, what tags, what duration type. It's a template — a cookie cutter.

### The GE Spec (Runtime Instance)

The `FGameplayEffectSpec` is what actually gets applied at runtime. It's the filled-out version of the template, with concrete values:

```cpp
// Create a spec from the GE class
FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingSpec(
    DamageEffectClass,  // The template
    Level,              // Effect level
    ASC->MakeEffectContext()  // Who's applying it, and how
);

// Fill in runtime values
SpecHandle.Data->SetSetByCallerMagnitude(DamageTag, 75.0f);

// Apply it
ASC->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
```

The spec captures everything about *this particular application*: the source actor, the target, the level, any SetByCaller values, the context. The same GE class can produce specs with completely different damage numbers depending on who's applying it and what values are set.

## FActiveGameplayEffectHandle

When you apply a Duration or Infinite effect, you get back a handle:

```cpp
FActiveGameplayEffectHandle Handle = ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
```

This handle is your reference to the active effect. You need it to:

- **Remove the effect early:** `ASC->RemoveActiveGameplayEffect(Handle)`
- **Query the effect:** check remaining duration, stack count, etc.
- **Register callbacks:** be notified when the effect expires or is removed

!!! note "Instant effects don't return useful handles"
    Since Instant effects execute and vanish, the handle they return isn't meaningful. You only need to track handles for Duration and Infinite effects.

## GameplayEffectContext

Every effect spec carries a `FGameplayEffectContext` — metadata about the application:

```cpp
FGameplayEffectContextHandle Context = ASC->MakeEffectContext();
Context.AddSourceObject(this);
Context.AddInstigator(InstigatorActor, InstigatorActor);
```

The context tells you:

- **Who** applied the effect (instigator, source object)
- **How** it was applied (hit result, origin location)
- **What** ability triggered it

This context flows through the entire pipeline — your `PostGameplayEffectExecute` callback can read it to determine who dealt the damage, where the hit landed, and what ability caused it. Many projects create a custom context subclass to carry additional data (damage type enum, critical hit flag, etc.).

## The Modular GE Architecture (5.3+)

In UE 5.3, Epic refactored Gameplay Effects from a monolithic class into a **component-based architecture**. Instead of one massive class with dozens of properties, GEs are now composed of `UGameplayEffectComponent` subobjects.

What this means in practice:

- Features like immunity, tag requirements, and custom application logic are now **components** you add to a GE
- You can create custom GE Components to extend effect behavior without subclassing `UGameplayEffect`
- The Details panel organizes properties by component instead of one long list

If you're working in UE 5.3 or later (including 5.7), this is how GEs work. For the full reference of built-in components and how to create custom ones, see [GE Components](../gameplay-effects/ge-components.md).

!!! info "Older tutorials"
    Many GAS tutorials were written before 5.3 and reference properties directly on the GE class. The functionality is the same — it's just organized differently now. If a tutorial says "set the Immunity property on the GE," look for the corresponding GE Component instead.

## What's Next

This page gave you the conceptual foundation. The [Gameplay Effects deep dive](../gameplay-effects/index.md) covers everything in detail:

- [Modifiers](../gameplay-effects/modifiers.md) — Add, Multiply, Override, and how they stack
- [Magnitude Calculations](../gameplay-effects/magnitude-calculations.md) — ScalableFloat, Attribute Based, Custom Calculation, SetByCaller
- [Execution Calculations](../gameplay-effects/execution-calculations.md) — custom damage formulas with full access to source and target attributes
- [Stacking](../gameplay-effects/stacking.md) — what happens when the same effect is applied multiple times
- [Tags and Requirements](../gameplay-effects/tags-and-requirements.md) — conditional application and removal
- [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) — how abilities use effects for resource management
