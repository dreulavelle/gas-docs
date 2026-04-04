---
title: Dynamic Effects
description: Creating and modifying Gameplay Effects at runtime — MakeOutgoingGameplayEffectSpec, spec modification, runtime GE creation, the GE Container pattern, and FGameplayEffectSpecHandle.
---

# Dynamic Effects

Most of the time, Gameplay Effects are configured as data-only Blueprint assets and applied as-is. But GAS also supports creating and modifying effects at runtime — adjusting specs before application, adding dynamic tags, and in rare cases, constructing entire GEs from scratch in code.

## MakeOutgoingGameplayEffectSpec

The standard entry point for working with GE specs at runtime. This function creates an `FGameplayEffectSpec` from a GE class, capturing source attributes and tags:

```cpp
FGameplayEffectSpecHandle SpecHandle =
    ASC->MakeOutgoingGameplayEffectSpec(MyEffectClass, AbilityLevel);
```

The returned `FGameplayEffectSpecHandle` is a shared pointer wrapper around the spec. You can modify the spec before applying it.

### What Happens During Spec Creation

1. The GE definition is read (the CDO)
2. Source attributes are captured (snapshotted or linked, as configured)
3. Source actor tags are captured
4. The level is set
5. An `FGameplayEffectContext` is created (instigator, effect causer, etc.)
6. Modifier magnitudes are **not** computed yet — that happens at application time

## Modifying Specs Before Application

After creating a spec, you can modify it before applying. This is the primary way to make effects dynamic at runtime.

### SetByCaller Magnitudes

The most common modification. See [SetByCaller](set-by-caller.md) for full details:

```cpp
SpecHandle.Data->SetSetByCallerMagnitude(DamageTag, 150.f);
```

### Dynamic Asset Tags

Add tags to the spec that weren't on the original GE definition:

```cpp
SpecHandle.Data->AddDynamicAssetTag(
    FGameplayTag::RequestGameplayTag("Damage.Critical"));

// Or add multiple at once
FGameplayTagContainer ExtraTags;
ExtraTags.AddTag(FGameplayTag::RequestGameplayTag("Damage.Elemental"));
ExtraTags.AddTag(FGameplayTag::RequestGameplayTag("Source.Weapon.Sword"));
SpecHandle.Data->AppendDynamicAssetTags(ExtraTags);
```

Dynamic asset tags are replicated and show up in immunity/query matching alongside the GE's static asset tags.

### Dynamic Granted Tags

Add tags that will be granted to the target:

```cpp
SpecHandle.Data->DynamicGrantedTags.AddTag(
    FGameplayTag::RequestGameplayTag("State.OnFire"));
```

### Setting the Level

```cpp
SpecHandle.Data->SetLevel(NewLevel);
```

### Modifying the Duration

```cpp
SpecHandle.Data->SetDuration(5.0f, true); // true = lock duration (prevents further changes)
```

!!! warning "Duration Lock"
    Once `SetDuration` is called with `bLockDuration = true`, subsequent calls to `SetDuration` are ignored. This prevents accidental overrides during the application pipeline.

### Effect Context

The effect context carries information about who created the effect and how:

```cpp
FGameplayEffectContextHandle ContextHandle = SpecHandle.Data->GetEffectContext();
ContextHandle.AddHitResult(HitResult);
ContextHandle.AddActors(RelevantActors);
ContextHandle.AddOrigin(SpawnLocation);

// Or replace the entire context
FGameplayEffectContextHandle NewContext =
    ASC->MakeEffectContext();
NewContext.AddInstigator(Instigator, EffectCauser);
SpecHandle.Data->SetContext(NewContext);
```

For passing custom data, subclass `FGameplayEffectContext` (a common and well-supported pattern).

### Stack Count

```cpp
SpecHandle.Data->SetStackCount(3);
```

## Creating GEs Dynamically at Runtime

In rare cases, you might want to construct a `UGameplayEffect` entirely in code rather than using an asset. This is possible but comes with caveats.

```cpp
UGameplayEffect* DynamicGE = NewObject<UGameplayEffect>(GetTransientPackage(), FName("DynamicDamageGE"));
DynamicGE->DurationPolicy = EGameplayEffectDurationType::Instant;

// Add a modifier
FGameplayModifierInfo ModifierInfo;
ModifierInfo.Attribute = UMyAttributeSet::GetHealthAttribute();
ModifierInfo.ModifierOp = EGameplayModOp::Additive;
ModifierInfo.ModifierMagnitude = FGameplayEffectModifierMagnitude(FScalableFloat(-50.f));
DynamicGE->Modifiers.Add(ModifierInfo);

// Apply it
FGameplayEffectSpecHandle SpecHandle =
    ASC->MakeOutgoingGameplayEffectSpec(DynamicGE->GetClass());
ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
```

!!! warning "Why You Usually Shouldn't Do This"
    Dynamically created GEs have several drawbacks:

    - They don't benefit from editor-based configuration and iteration
    - They bypass the component system (no GE Components are set up)
    - They can be harder to debug (no asset to inspect)
    - They may not garbage collect cleanly if you're not careful about their outer/lifetime
    - They don't show up in the GE debugger the same way assets do

    The overwhelmingly common pattern is: define GEs as assets, create specs at runtime, and modify specs before application. Only create GEs dynamically when you have a compelling reason (e.g., procedurally generated items with novel stat combinations that can't be expressed with SetByCaller).

## The GE Container Pattern

For effects that logically group together — like all the stat bonuses from a piece of equipment, or all the effects of a complex ability — a common pattern is to use a **container GE** that applies additional effects:

```
GE_EquipSword (Infinite, no modifiers)
  ├── Component: AdditionalEffectsGEC
  │   ├── OnApplication: GE_SwordDamageBonus
  │   ├── OnApplication: GE_SwordSpeedBonus
  │   └── OnApplication: GE_SwordPassiveAbility
  └── Component: TargetTagsGEC
      └── Grants: Equipment.Sword
```

Removing `GE_EquipSword` triggers the "on premature removal" path, which can clean up the sub-effects. Or better yet, design the sub-effects to have ongoing requirements tied to the `Equipment.Sword` tag — they auto-inhibit when the sword is unequipped.

The advantage: one handle to track, one removal call to clean everything up.

## FGameplayEffectSpecHandle

The spec handle is a thin wrapper around `TSharedPtr<FGameplayEffectSpec>`:

```cpp
FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingGameplayEffectSpec(GEClass);

// Check validity
if (SpecHandle.IsValid())
{
    // Access the spec
    FGameplayEffectSpec* Spec = SpecHandle.Data.Get();

    // Modify it
    Spec->SetSetByCallerMagnitude(Tag, Value);

    // Apply it
    FActiveGameplayEffectHandle ActiveHandle =
        ASC->ApplyGameplayEffectSpecToTarget(*Spec, TargetASC);
}
```

The handle uses shared pointer semantics — the spec stays alive as long as something holds a reference to the handle. Once all handles go out of scope, the spec is destroyed (unless it was applied and is now part of an `FActiveGameplayEffect`).

### Copying and Forwarding Specs

```cpp
// Copy SetByCaller magnitudes from one spec to another
NewSpecHandle.Data->CopySetByCallerMagnitudes(*OriginalSpecHandle.Data);

// Merge (only adds values that don't already exist)
NewSpecHandle.Data->MergeSetByCallerMagnitudes(OriginalSpec.SetByCallerTagMagnitudes);

// Initialize as a linked spec (preserves context, strips original asset tags)
NewSpecHandle.Data->InitializeFromLinkedSpec(NewGE, *OriginalSpecHandle.Data);
```

## Applying to Self vs. Target

```cpp
// Apply to yourself
FActiveGameplayEffectHandle Handle =
    ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());

// Apply to another target
FActiveGameplayEffectHandle Handle =
    ASC->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
```

Both return an `FActiveGameplayEffectHandle` for duration/infinite effects (or an invalid handle for instant effects, since they don't persist).

## Runtime Workflow Summary

Here's the typical runtime flow for dynamic effects:

```
1. MakeOutgoingGameplayEffectSpec(GEClass, Level)
   ↓
2. Modify the spec:
   - SetSetByCallerMagnitude()
   - AddDynamicAssetTag()
   - SetContext() / AddHitResult()
   - SetDuration()
   ↓
3. ApplyGameplayEffectSpecToTarget(*Spec, TargetASC)
   ↓
4. (Optional) Store the ActiveGE handle for later removal
```

This pattern covers the vast majority of dynamic effect needs. Start here, and only reach for runtime GE construction if spec modification isn't enough.

## Tips

!!! tip "Spec Reuse"
    You can apply the same spec to multiple targets. However, be careful — applying a spec captures target attributes, so applying to a second target after the first may use stale target captures. Create a fresh spec per target if target attributes matter.

!!! note "Duplicate Context"
    If you need to modify the effect context after initial creation without affecting the original, call `SpecHandle.Data->DuplicateEffectContext()` to create a deep copy.

!!! info "Dynamic Tags Are Replicated"
    Both `DynamicAssetTags` and `DynamicGrantedTags` are replicated as part of the spec. Changes made on the server will propagate to clients.

## Related Pages

- [SetByCaller](set-by-caller.md) -- the most common spec modification for passing runtime magnitudes
- [Execution Calculations](execution-calculations.md) -- reading dynamic spec data inside multi-attribute formulas
- [Gameplay Abilities Overview](../core-concepts/gameplay-abilities-overview.md) -- where MakeOutgoingGameplayEffectSpec is typically called from
