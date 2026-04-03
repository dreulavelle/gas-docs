---
icon: material/tag-multiple
---

# Gameplay Tags in GAS

If GAS has a lingua franca, it's Gameplay Tags. They're how abilities know whether they can activate, how effects decide whether to apply, how cues know what to show, and how you query an actor's current state. Understanding tags is understanding the *language* that every GAS component speaks.

## What Tags Are

A Gameplay Tag is a hierarchical, dot-separated label. That's it — just a name.

```
CrowdControl.Hard.Stun
Damage.Type.Fire
Ability.Skill.Fireball
State.Dead
Cooldown.Ability.Dash
```

Under the hood, each tag is registered in a global tag dictionary and compared by hash, which makes tag operations fast — comparable to integer comparisons, not string comparisons. You can use tags liberally without worrying about performance.

Tags are **not** booleans. They're not enums. They're a structured namespace that GAS can query hierarchically.

## Tag Hierarchy and Parent Matching

This is the feature that makes tags powerful: **checking for a parent tag matches all of its children**.

Given these tags on an actor:

```
CrowdControl.Hard.Stun
CrowdControl.Soft.Slow
```

These checks all return true:

| Query | Result | Why |
|---|---|---|
| `CrowdControl.Hard.Stun` | :material-check: Match | Exact match |
| `CrowdControl.Hard` | :material-check: Match | Parent of `CrowdControl.Hard.Stun` |
| `CrowdControl` | :material-check: Match | Parent of both tags |
| `CrowdControl.Soft.Slow` | :material-check: Match | Exact match |
| `CrowdControl.Hard.Freeze` | :material-close: No match | Not present |
| `Damage.Type.Fire` | :material-close: No match | Not present |

This is enormously useful. An ability that should be blocked by *any* hard crowd control just checks for `CrowdControl.Hard` — it automatically blocks on Stun, Freeze, Fear, or any future hard CC you add. No code changes needed.

## MatchesTag vs MatchesTagExact

GAS provides two matching functions, and choosing the right one matters:

```cpp
// MatchesTag — hierarchical matching (parent matches children)
Tag.MatchesTag(OtherTag);

// MatchesTagExact — exact match only (no hierarchy)
Tag.MatchesTagExact(OtherTag);
```

**Use `MatchesTag`** (the default) when you want hierarchical matching — which is *most of the time*. Checking for `CrowdControl.Hard` should match `CrowdControl.Hard.Stun`.

**Use `MatchesTagExact`** when you specifically need to distinguish between a parent and its children. For example, if you have logic that should only fire for `CrowdControl.Hard.Stun` and not for `CrowdControl.Hard.Freeze`, use exact matching.

!!! note "Direction matters"
    `A.MatchesTag(B)` checks if A is the same as or a *child* of B. So `CrowdControl.Hard.Stun.MatchesTag(CrowdControl.Hard)` is true, but `CrowdControl.Hard.MatchesTag(CrowdControl.Hard.Stun)` is false. The tag on the left is the one being tested; the tag on the right is the "filter."

## Tag Containers

In practice, actors don't hold a single tag — they hold a **set** of tags in an `FGameplayTagContainer`:

```cpp
FGameplayTagContainer TagContainer;
TagContainer.AddTag(FGameplayTag::RequestGameplayTag(FName("State.InCombat")));
TagContainer.AddTag(FGameplayTag::RequestGameplayTag(FName("CrowdControl.Soft.Slow")));
```

You can query containers with:

```cpp
// Does the container have ANY tag matching this one?
bool bHasCC = TagContainer.HasTag(CrowdControlTag);  // Hierarchical

// Does the container have ALL of these tags?
bool bHasAll = TagContainer.HasAll(RequiredTags);

// Does the container have ANY of these tags?
bool bHasAny = TagContainer.HasAny(BlockedTags);
```

GAS uses these container queries extensively — ability activation checks, effect application requirements, and tag-based filtering all use `HasTag`, `HasAll`, and `HasAny` on the ASC's current tag container.

## Tag Counts

Here's something that surprises people: GAS tracks tags by **count**, not by presence/absence.

When an effect grants the tag `CrowdControl.Hard.Stun`, the ASC increments the count for that tag. When the effect ends, it decrements the count. The tag is considered "present" as long as the count is greater than zero.

Why does this matter? Because **multiple sources can grant the same tag**:

```
Stun effect A applies → Stun count = 1 (tag is present)
Stun effect B applies → Stun count = 2 (tag is still present)
Stun effect A expires → Stun count = 1 (tag is STILL present!)
Stun effect B expires → Stun count = 0 (tag is gone)
```

If tags were simple booleans, removing one stun would un-stun the character even though another stun is still active. Tag counts prevent this.

### Loose Tags

You can also add tags manually (outside of effects) using "loose" tags:

```cpp
// Add a loose tag (increments count)
ASC->AddLooseGameplayTag(MyTag);

// Remove a loose tag (decrements count)
ASC->RemoveLooseGameplayTag(MyTag);

// Check current count
int32 Count = ASC->GetTagCount(MyTag);
```

Loose tags follow the same counting rules. If an effect granted a tag and you also added it as a loose tag, the count is 2 — both need to be removed for the tag to disappear.

!!! warning "Balance your adds and removes"
    Every `AddLooseGameplayTag` must have a corresponding `RemoveLooseGameplayTag`. If you add a tag in `BeginPlay` but forget to remove it, it will persist forever. Effect-granted tags handle this automatically — loose tags are your responsibility.

## Responding to Tag Changes

Often you need to react when a tag is added or removed. GAS provides two mechanisms:

### Tag Change Delegates

Register a delegate on the ASC to be notified when a specific tag's count changes:

```cpp
ASC->RegisterGameplayTagEvent(
    StunTag,
    EGameplayTagEventType::NewOrRemoved  // Only fires when count goes 0→1 or 1→0
).AddUObject(this, &AMyCharacter::OnStunTagChanged);

void AMyCharacter::OnStunTagChanged(const FGameplayTag Tag, int32 NewCount)
{
    if (NewCount > 0)
    {
        // Stun started — disable input, play stun animation
    }
    else
    {
        // Stun ended — re-enable input
    }
}
```

The `EGameplayTagEventType` controls when the delegate fires:

- `NewOrRemoved` — fires only on transitions: tag goes from absent to present (0 to 1+) or present to absent (1+ to 0). **This is usually what you want.**
- `AnyCountChange` — fires on every increment/decrement. Use this if you need to track stacks.

### GameplayTagResponseTable

For AI, there's also `UGameplayTagResponseTable` — a data asset that maps tag events to GameplayEffects. When a tag is added or removed, it can automatically apply or remove an effect. This is useful for AI state machines that respond to status effects.

## How Tags Are Registered

Tags must be registered before they can be used. There are several ways:

### Project Settings (Editor Tag Manager)

The most common method. Open **Project Settings > GameplayTags** and add tags directly. These are stored in your project's `DefaultGameplayTags.ini`.

### DataTables

Create a DataTable with row type `GameplayTagTableRow` and populate it with tags. Reference the DataTable in Project Settings under **GameplayTags > Gameplay Tag Table List**.

### Native Code

Declare tags in C++ using `UE_DECLARE_GAMEPLAY_TAG_EXTERN` / `UE_DEFINE_GAMEPLAY_TAG`:

```cpp
// Header
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_State_Dead);

// Source
UE_DEFINE_GAMEPLAY_TAG(TAG_State_Dead, "State.Dead");
```

Native tags are available immediately at startup with zero lookup cost. Use this for tags you reference frequently in C++ — it avoids the `RequestGameplayTag(FName("..."))` pattern that can fail silently if you typo the string.

### Plugin .ini Files

Plugins can register their own tags by including a `Config/Tags/` directory with `.ini` files in the standard GameplayTags format:

```ini
[/Script/GameplayTags.GameplayTagsSettings]
+GameplayTagList=(Tag="MyPlugin.Status.Burning",DevComment="Applied when the actor is on fire")
+GameplayTagList=(Tag="MyPlugin.Status.Frozen",DevComment="Applied when the actor is frozen")
```

These are loaded automatically when the plugin loads. This is how you ship reusable GAS functionality in plugins without requiring users to manually add your tags.

## Tag Design

Designing a good tag hierarchy is one of the most impactful decisions in a GAS project. A well-designed hierarchy lets you write broad queries (`CrowdControl.Hard` matches any hard CC) and keeps your tag namespace manageable as the project grows.

The key principles:

- **Design for queries, not just labels** — think about what you'll check for, not just what you'll grant
- **Use hierarchy for "is-a" relationships** — `Damage.Type.Fire` *is a* `Damage.Type`
- **Keep it shallow enough to be useful** — 3-4 levels is usually the sweet spot
- **Separate concerns** — `Ability.*` for ability identity, `State.*` for actor state, `CrowdControl.*` for CC types

For the complete tag design guide with naming conventions, common hierarchies, and anti-patterns, see [Tag Architecture](../patterns/tag-design.md).
