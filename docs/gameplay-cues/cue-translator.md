---
title: Cue Translator
description: How the GameplayCueTranslator remaps cue tags at runtime for dynamic VFX selection without creating separate cue assets per variant.
---

# Cue Translator

The Gameplay Cue Translator is a system that intercepts a cue tag *before* it reaches the notify and remaps it to a different tag at runtime. It sits between "a cue was fired" and "a cue notify was spawned," giving you a hook to dynamically swap which VFX, sound, or animation plays based on context -- without needing separate cue assets for every possible variant.

## Why It Exists

Imagine you have a hit effect system. Without the translator, to get different VFX for different weapon types, you'd need one of these approaches:

**Approach A: Monolithic handler.** One cue notify for `GameplayCue.Combat.Hit` that internally checks the weapon type and branches to different effects. This notify now has to load and reference *every* possible variant, which defeats the purpose of the cue system's async loading. It also becomes a maintenance nightmare as you add weapons.

**Approach B: Manual override data.** Store the override cue/asset on a character blueprint or weapon data asset, and have the cue handler look it up. Now every cue that wants variant support needs custom override properties somewhere, and you can't just drop in a self-contained cue notify class anymore.

**Approach C: Translator.** Fire `GameplayCue.Combat.Hit`. The translator checks the context (equipped weapon, surface material, character class) and remaps it to `GameplayCue.Combat.Hit.Sword` or `GameplayCue.Combat.Hit.Axe`. Each of those is a separate, self-contained cue notify that just plays its VFX. No monolith. No override data plumbing.

The translator gives you Approach C. The key advantages are:

- **Atomic cue notifies.** Each variant is its own asset that "just plays sounds and FX."
- **No loading bloat.** Only the translated variant gets loaded, not every possible variant.
- **Context-driven.** The translation decision can use any runtime information: the target actor, the effect context, gameplay tags, equipped items, physics materials.
- **Designer workflow.** The GameplayCue editor has built-in support for translators -- it can auto-create the derived tags and notify assets.

## How It Works

The translation happens inside the `FGameplayCueTranslationManager`, which lives on the `UGameplayCueManager`. When a cue event fires, the flow is:

```
Cue event fired with tag: GameplayCue.Combat.Hit
    │
    ├─ FGameplayCueTranslationManager::TranslateTag()
    │   │
    │   ├─ Look up tag in TranslationLUT
    │   │
    │   ├─ For each translator registered on this node:
    │   │   │
    │   │   ├─ Call translator->GameplayCueToTranslationIndex(Tag, Actor, Params)
    │   │   │   → Returns an index, or INDEX_NONE if no translation
    │   │   │
    │   │   ├─ If valid index: remap to the target node
    │   │   │   → Tag is now GameplayCue.Combat.Hit.Sword
    │   │   │
    │   │   └─ Recurse: check if the NEW tag also has translations
    │   │       (with cycle detection to prevent infinite loops)
    │   │
    │   └─ Final tag is used for cue dispatch
    │
    └─ Cue manager routes the (possibly translated) tag to its notify
```

The translation table (LUT) is built at startup by calling `GetTranslationNameSpawns` on every registered `UGameplayCueTranslator` CDO. This is a one-time cost -- at runtime, the translation is a lookup + one virtual function call per translator.

### Recursion and Cycle Detection

Translations can chain. If Translator A remaps `Hero` to `Steel`, and Translator B remaps `Steel` to `Steel.Legendary`, the system will apply both. The `UsedTranslators` set on each node prevents infinite loops by tracking which translators have already been applied in the current chain.

## UGameplayCueTranslator

This is the base class you extend. It's abstract, not instantiated -- the engine calls virtual functions on the CDO (Class Default Object).

```cpp
UCLASS(Abstract)
class UGameplayCueTranslator : public UObject
{
    GENERATED_BODY()

public:
    // Define your translation rules (called once at startup)
    virtual void GetTranslationNameSpawns(
        TArray<FGameplayCueTranslationNameSwap>& SwapList) const { }

    // At runtime, decide which translation to apply
    virtual int32 GameplayCueToTranslationIndex(
        const FName& TagName,
        AActor* TargetActor,
        const FGameplayCueParameters& Parameters) const { return INDEX_NONE; }

    // Higher priority = first chance to translate
    virtual int32 GetPriority() const { return 0; }

    // Toggle this translator on/off
    virtual bool IsEnabled() const { return true; }

    // Whether to show in the top-level filter list in the GC editor
    virtual bool ShouldShowInTopLevelFilterList() const { return true; }
};
```

### GetTranslationNameSpawns

Called once at startup. You return a list of `FGameplayCueTranslationNameSwap` entries, each defining a "from name" to "to name" substitution:

```cpp
struct FGameplayCueTranslationNameSwap
{
    FName FromName;                              // e.g. "Hero"
    TArray<FName, TInlineAllocator<4>> ToNames;  // e.g. {"Steel"}
};
```

This says: "wherever `FromName` appears as a segment in a cue tag, I *can* replace it with `ToNames`." The `ToNames` array supports multi-segment replacements (e.g., replacing `Hero` with `Steel.Legendary`).

!!! important "Order Matters"
    The index of each swap in the array returned by `GetTranslationNameSpawns` is what `GameplayCueToTranslationIndex` returns at runtime. Swap at index 0 is "translate to the first option," index 1 is the second, and so on. Keep this mapping stable.

### GameplayCueToTranslationIndex

Called at runtime for every cue event that matches one of your translation rules. You receive:

- **TagName** -- the full tag name being translated
- **TargetActor** -- the actor the cue is targeting
- **Parameters** -- the full `FGameplayCueParameters` (effect context, instigator, location, etc.)

Return the index into your swap list for which translation to apply, or `INDEX_NONE` to skip translation.

### GetPriority

If multiple translators can translate the same tag, higher priority goes first. This matters when translators are competing -- the first one to return a valid index wins.

### IsEnabled

Return `false` to disable the translator entirely. Useful for WIP translators that shouldn't run in shipping builds.

## Built-in Translators

The engine ships with `UGameplayCueTranslator_Test`, a disabled example translator that demonstrates the pattern. It translates `Hero` into one of three character names (`Steel`, `Rampage`, `Kurohane`) based on the target actor. It's disabled by default (`IsEnabled` returns `false`) and uses dummy pointer comparisons for the runtime selection -- it's purely educational.

There are no production-ready translators shipping with the engine. The system is designed to be extended by your game.

## Writing a Custom Translator

Here's a practical example: translating `GameplayCue.Combat.Hit` to weapon-specific variants based on a gameplay tag in the effect context.

### Step 1: Define the Tag Hierarchy

First, create the tags:

```
GameplayCue.Combat.Hit          (base)
GameplayCue.Combat.Hit.Sword    (sword variant)
GameplayCue.Combat.Hit.Axe      (axe variant)
GameplayCue.Combat.Hit.Mace     (mace variant)
```

Create a cue notify for each variant. The base `GameplayCue.Combat.Hit` serves as the fallback.

### Step 2: Implement the Translator

```cpp
UCLASS()
class UGameplayCueTranslator_WeaponType : public UGameplayCueTranslator
{
    GENERATED_BODY()

public:
    virtual void GetTranslationNameSpawns(
        TArray<FGameplayCueTranslationNameSwap>& SwapList) const override
    {
        // Index 0: Hit -> Hit.Sword
        {
            FGameplayCueTranslationNameSwap Swap;
            Swap.FromName = FName(TEXT("Hit"));
            Swap.ToNames.Add(FName(TEXT("Hit")));
            Swap.ToNames.Add(FName(TEXT("Sword")));
            SwapList.Add(Swap);
        }

        // Index 1: Hit -> Hit.Axe
        {
            FGameplayCueTranslationNameSwap Swap;
            Swap.FromName = FName(TEXT("Hit"));
            Swap.ToNames.Add(FName(TEXT("Hit")));
            Swap.ToNames.Add(FName(TEXT("Axe")));
            SwapList.Add(Swap);
        }

        // Index 2: Hit -> Hit.Mace
        {
            FGameplayCueTranslationNameSwap Swap;
            Swap.FromName = FName(TEXT("Hit"));
            Swap.ToNames.Add(FName(TEXT("Hit")));
            Swap.ToNames.Add(FName(TEXT("Mace")));
            SwapList.Add(Swap);
        }
    }

    virtual int32 GameplayCueToTranslationIndex(
        const FName& TagName,
        AActor* TargetActor,
        const FGameplayCueParameters& Parameters) const override
    {
        // Extract weapon type from the effect context's source tags
        if (Parameters.EffectContext.IsValid())
        {
            const FGameplayTagContainer* SourceTags =
                Parameters.AggregatedSourceTags.Num() > 0
                ? &Parameters.AggregatedSourceTags : nullptr;

            if (SourceTags)
            {
                if (SourceTags->HasTag(
                    FGameplayTag::RequestGameplayTag(TEXT("Weapon.Sword"))))
                    return 0;

                if (SourceTags->HasTag(
                    FGameplayTag::RequestGameplayTag(TEXT("Weapon.Axe"))))
                    return 1;

                if (SourceTags->HasTag(
                    FGameplayTag::RequestGameplayTag(TEXT("Weapon.Mace"))))
                    return 2;
            }
        }

        // No match -- don't translate, use the base cue
        return INDEX_NONE;
    }

    virtual int32 GetPriority() const override { return 10; }
};
```

### Step 3: Create the Cue Notify Assets

For each translated tag, create a cue notify blueprint or C++ class:

- `GCN_Hit_Sword` handling `GameplayCue.Combat.Hit.Sword`
- `GCN_Hit_Axe` handling `GameplayCue.Combat.Hit.Axe`
- `GCN_Hit_Mace` handling `GameplayCue.Combat.Hit.Mace`

And optionally keep a `GCN_Hit_Default` on the base tag as a fallback.

!!! tip "GameplayCue Editor Integration"
    The GameplayCue editor in UE has built-in awareness of translators. When you select a tag, it shows which translators apply and can auto-generate the derived tags and notify stubs for you. Use it.

## Registering With the Manager

You don't need to register your translator manually. The `FGameplayCueTranslationManager` discovers all `UGameplayCueTranslator` subclasses automatically via `TObjectIterator` at startup (during `BuildTagTranslationTable`). As long as your translator class is compiled into a loaded module:

1. It will be found automatically
2. `GetTranslationNameSpawns` will be called on its CDO
3. The translation rules will be added to the LUT

The only thing you need to ensure is that `IsEnabled()` returns `true`.

### Debugging

Two useful commands:

| Command | What It Does |
|:---|:---|
| `Log LogGameplayCueTranslator Verbose` | Enables detailed logging of tag translation at runtime |
| `GameplayCue.PrintGameplayCueTranslator` | Prints the entire translation LUT to the log |
| `GameplayCue.BuildGameplayCueTranslator` | Force-rebuilds the translation table (useful during development) |

## The Translation Data Structures

If you need to understand how the translation system works internally for advanced use cases or debugging:

### FGameplayCueTranslationNameSwap

The atomic unit of translation -- "replace `FromName` with `ToNames`":

```cpp
struct FGameplayCueTranslationNameSwap
{
    FName FromName;
    TArray<FName, TInlineAllocator<4>> ToNames;
};
```

### FGameplayCueTranslatorNode

Each node in the translation graph represents a tag (or potential tag):

```cpp
struct FGameplayCueTranslatorNode
{
    TArray<FGameplayCueTranslationLink> Links;    // Translations from this node
    FGameplayCueTranslatorNodeIndex CachedIndex;  // Index in the LUT
    FGameplayTag CachedGameplayTag;               // The actual tag (if it exists)
    FName CachedGameplayTagName;                  // Always valid
    TSet<const UGameplayCueTranslator*> UsedTranslators;  // Cycle detection
};
```

### FGameplayCueTranslationLink

Links a node to a translator and its possible translations:

```cpp
struct FGameplayCueTranslationLink
{
    const UGameplayCueTranslator* RulesCDO;       // Which translator
    TArray<FGameplayCueTranslatorNodeIndex> NodeLookup;  // Index -> target node
};
```

### FGameplayCueTranslationManager

The manager that owns the LUT and performs runtime translation:

```cpp
struct FGameplayCueTranslationManager
{
    void TranslateTag(FGameplayTag& Tag, AActor* TargetActor,
                      const FGameplayCueParameters& Parameters);
    void BuildTagTranslationTable();
    // ...

private:
    TArray<FGameplayCueTranslatorNode> TranslationLUT;
    TMap<FName, FGameplayCueTranslatorNodeIndex> TranslationNameToIndexMap;
};
```

The `TranslateTag` function is what the cue manager calls. It modifies the tag in-place if a translation applies.

## When to Use vs. When to Just Make Separate Cues

The translator adds indirection. That indirection is valuable when you have **combinatorial variants driven by runtime context**, but it's unnecessary complexity when simpler approaches work.

**Use a translator when:**

- You have a generic cue tag that needs to resolve to different variants based on who/what triggered it (character class, weapon type, element type)
- The number of variants is large or growing -- you don't want the trigger site to know about all variants
- The variant selection depends on target-side context (the target's character, the hit surface material)
- You want designers to add new variants by just creating new cue notify assets, without touching any code that fires the cue

**Just make separate cues when:**

- The caller already knows which variant to use and can fire the specific tag directly
- You have a small, fixed number of variants (fire `GameplayCue.Combat.Hit.Fire` directly when you know it's a fire attack)
- The variant is determined at design time, not runtime
- You don't need the indirection -- YAGNI applies here

**A good heuristic:** if the *caller* knows the variant, fire the specific tag. If only the *target* or *context* knows the variant, use a translator.

## Related Pages

- [Cue Manager](cue-manager.md) -- the routing hub that calls the translator
- [Cue Parameters](cue-parameters.md) -- the context data available to `GameplayCueToTranslationIndex`
- [Cue Notify Types](cue-types.md) -- the actual notify classes your translated tags resolve to
