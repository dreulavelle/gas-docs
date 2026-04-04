---
title: Starter Tags
description: A ready-to-use gameplay tag library with 21 namespaces and 135+ tags for action RPG projects. Copy the .ini files into your project or browse and pick what you need.
---

# Starter Tag Preset

A ready-to-use gameplay tag library for action RPG projects built on GAS. It covers **21 namespaces** and roughly **135 tags** -- ability types, AI behavior, combat results, damage classification, crowd control, equipment slots, input mapping, and more.

You can use the whole set as-is, cherry-pick the namespaces you need, or treat it as a starting point for your own tag architecture. Every tag includes a `DevComment` so the purpose is visible right inside the Unreal editor's tag picker.

!!! tip "Designed for the GAS Bible"
    These tags align with the patterns used throughout this guide -- the [damage pipeline](damage-pipeline.md), [buff/debuff system](buff-debuff-system.md), [common abilities](common-abilities.md), and the [tag architecture](tag-design.md) page. If you're following along, this preset gives you a running start.

---

## How to Use

There are two ways to get these tags into your project.

### Option A: Copy the .ini files (recommended)

Each collapsible section below contains a **complete `.ini` file** ready to save. Copy the block and save it to your project's `Config/Tags/` directory with the filename shown.

```
YourProject/
  Config/
    Tags/
      Tags_Ability.ini
      Tags_AI.ini
      Tags_Animation.ini
      ...
    DefaultGameplayTags.ini
```

Unreal reads all `.ini` files in `Config/Tags/` automatically when `ImportTagsFromConfig=True` is set in your `DefaultGameplayTags.ini`. No code changes, no restarts -- just save the files and the tags appear in the editor.

### Option B: Add tags manually

Open **Project Settings > Gameplay Tags** and add tags one by one. This works fine for small additions, but for 135+ tags the `.ini` route is significantly faster.

---

## Before You Copy Everything

!!! warning "Don't blindly copy all 135 tags"
    Unused tags have **near-zero runtime performance cost** -- they're just entries in a hash table loaded once at startup. But having hundreds of irrelevant tags **clutters the editor tag picker** and makes it harder for designers to find what they need. Start with the namespaces relevant to your game and add others as needed.

    You can filter tag dropdowns per-property using the `Categories` meta specifier:

    ```cpp
    // Only shows tags under Weapon.* in this dropdown
    UPROPERTY(EditDefaultsOnly, meta=(Categories="Weapon"))
    FGameplayTag WeaponTag;
    ```

!!! danger "Renaming tags is painful"
    Anything referencing the old tag name **breaks silently** -- no compile error, no warning, just things stop working. Effects won't match, abilities won't block, cues won't trigger. Get your naming right early. It's safe to *remove* unused tags (nothing references them), but *renaming* active tags requires finding and updating every reference.

!!! tip "Safe to start broad, prune later"
    If you're unsure whether you need a namespace, include it. Removing unused tags later is trivial -- just delete the lines from the `.ini` file. The risk of having too many tags is clutter, not performance. The risk of missing a namespace is having to retrofit it later when your project is larger.

---

## Global Settings

Before adding tag files, make sure your `DefaultGameplayTags.ini` has `ImportTagsFromConfig=True`. This tells the engine to scan `Config/Tags/` for additional tag files.

??? example "DefaultGameplayTags.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True
    WarnOnInvalidTags=True
    ClearInvalidTags=False
    AllowEditorTagUnloading=True
    AllowGameTagUnloading=False
    FastReplication=False
    bDynamicReplication=False
    InvalidTagCharacters="\"',',"
    NumBitsForContainerSize=6
    NetIndexFirstBitSegment=16
    ```

| Setting | What It Does |
|---|---|
| `ImportTagsFromConfig` | Enables loading tags from `.ini` files in `Config/Tags/` |
| `WarnOnInvalidTags` | Logs a warning when code references a tag that doesn't exist |
| `FastReplication` | Enables integer-indexed tag replication (enable once your tag set is stable) |

---

## Ability Tags

Types of abilities, delivery methods, and activation failure reasons. Used for ability classification, tag requirements, and UI feedback when activation fails.

??? example "Tags_Ability.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Ability Root --
    +GameplayTagList=(Tag="Ability",DevComment="Root namespace for gameplay abilities (GAS).")

    ; -- Activation Type --
    +GameplayTagList=(Tag="Ability.Active",DevComment="Activatable ability (triggered by input/AI).")
    +GameplayTagList=(Tag="Ability.Passive",DevComment="Passive/always-on ability.")
    +GameplayTagList=(Tag="Ability.Ultimate",DevComment="High-impact ability with special gating/cooldown.")
    +GameplayTagList=(Tag="Ability.Toggle",DevComment="Toggle on/off ability (e.g., stance, aura).")

    ; -- Category --
    +GameplayTagList=(Tag="Ability.BasicAttack",DevComment="Basic attack ability (core combat loop).")
    +GameplayTagList=(Tag="Ability.Mobility",DevComment="Movement ability (dash/blink/leap).")
    +GameplayTagList=(Tag="Ability.Defensive",DevComment="Defense ability (block/parry/shield).")
    +GameplayTagList=(Tag="Ability.Support",DevComment="Support ability (buff/heal/utility).")
    +GameplayTagList=(Tag="Ability.Summon",DevComment="Summon ability (pet/minion/turret).")
    +GameplayTagList=(Tag="Ability.Combo",DevComment="Ability participates in the combo system.")

    ; -- Delivery Method --
    +GameplayTagList=(Tag="Ability.Targeted",DevComment="Requires a target actor.")
    +GameplayTagList=(Tag="Ability.Area",DevComment="Area-based (AoE/ground targeted).")
    +GameplayTagList=(Tag="Ability.Projectile",DevComment="Spawns projectile(s).")
    +GameplayTagList=(Tag="Ability.Melee",DevComment="Melee range delivery.")
    +GameplayTagList=(Tag="Ability.Ranged",DevComment="Ranged delivery.")
    +GameplayTagList=(Tag="Ability.Channeled",DevComment="Maintained while channeling.")
    +GameplayTagList=(Tag="Ability.Instant",DevComment="Resolves immediately.")

    ; -- Activation Failure --
    +GameplayTagList=(Tag="Ability.ActivateFail",DevComment="Ability activation failure reasons (used for UI/debug).")
    +GameplayTagList=(Tag="Ability.ActivateFail.Dead",DevComment="Failed because owner is dead.")
    +GameplayTagList=(Tag="Ability.ActivateFail.Stunned",DevComment="Failed because owner is stunned.")
    +GameplayTagList=(Tag="Ability.ActivateFail.Silenced",DevComment="Failed because owner is silenced.")
    +GameplayTagList=(Tag="Ability.ActivateFail.Rooted",DevComment="Failed because owner is rooted (movement ability).")
    +GameplayTagList=(Tag="Ability.ActivateFail.Disarmed",DevComment="Failed because owner is disarmed (weapon ability).")
    +GameplayTagList=(Tag="Ability.ActivateFail.InCooldown",DevComment="Failed because ability is on cooldown.")
    +GameplayTagList=(Tag="Ability.ActivateFail.InCost",DevComment="Failed due to insufficient resource.")
    +GameplayTagList=(Tag="Ability.ActivateFail.NoTarget",DevComment="Failed because target is missing.")
    +GameplayTagList=(Tag="Ability.ActivateFail.OutOfRange",DevComment="Failed because target is out of range.")
    +GameplayTagList=(Tag="Ability.ActivateFail.LineOfSight",DevComment="Failed due to line-of-sight.")
    +GameplayTagList=(Tag="Ability.ActivateFail.Blocked",DevComment="Failed because a blocking tag is present.")
    +GameplayTagList=(Tag="Ability.ActivateFail.AlreadyActive",DevComment="Failed because ability is already active.")
    +GameplayTagList=(Tag="Ability.ActivateFail.Requirement",DevComment="Failed due to custom requirement.")
    ```

| Tag | Purpose |
|---|---|
| `Ability.Active` / `.Passive` / `.Ultimate` / `.Toggle` | Classify how the ability activates |
| `Ability.BasicAttack` through `.Combo` | Functional category for UI grouping and tag requirements |
| `Ability.Melee` / `.Ranged` / `.Projectile` / `.Area` | Delivery method -- used by the damage pipeline to select hit detection |
| `Ability.ActivateFail.*` | Sent as gameplay events on failed activation -- drive UI feedback ("Not enough mana", "On cooldown") |

---

## AI Tags

NPC behavioral states, combat personalities, group roles, and perception awareness levels. These tags drive behavior tree decisions and are completely independent of the AI toolkit you choose (built-in BT, StateTree, custom FSM).

??? example "Tags_AI.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- AI State (what the AI brain is currently doing) --
    +GameplayTagList=(Tag="AI",DevComment="AI behavior and decision-making namespace.")
    +GameplayTagList=(Tag="AI.State.Idle",DevComment="Idle: no stimulus, standing/wandering.")
    +GameplayTagList=(Tag="AI.State.Patrol",DevComment="Patrol: following a patrol path.")
    +GameplayTagList=(Tag="AI.State.Alert",DevComment="Alert: stimulus detected, not yet engaged.")
    +GameplayTagList=(Tag="AI.State.Investigate",DevComment="Investigate: moving to investigate a stimulus.")
    +GameplayTagList=(Tag="AI.State.Combat",DevComment="Combat: actively fighting a target.")
    +GameplayTagList=(Tag="AI.State.Chase",DevComment="Chase: pursuing a target out of attack range.")
    +GameplayTagList=(Tag="AI.State.Flee",DevComment="Flee: retreating from combat.")
    +GameplayTagList=(Tag="AI.State.Return",DevComment="Return: leashing back to home/spawn.")
    +GameplayTagList=(Tag="AI.State.Dead",DevComment="Dead: AI is dead, cleanup pending.")
    +GameplayTagList=(Tag="AI.State.Staggered",DevComment="Staggered: AI is in hit-react, cannot decide.")
    +GameplayTagList=(Tag="AI.State.Spawning",DevComment="Spawning: playing intro/spawn animation.")

    ; -- AI Behavior (combat personality -- how it fights) --
    +GameplayTagList=(Tag="AI.Behavior.Passive",DevComment="Passive: won't attack unless provoked.")
    +GameplayTagList=(Tag="AI.Behavior.Defensive",DevComment="Defensive: prefers blocking, counter-attacks.")
    +GameplayTagList=(Tag="AI.Behavior.Aggressive",DevComment="Aggressive: prefers closing distance and attacking.")
    +GameplayTagList=(Tag="AI.Behavior.Berserk",DevComment="Berserk: all-out offense, ignores defense.")
    +GameplayTagList=(Tag="AI.Behavior.Support",DevComment="Support: prioritizes healing/buffing allies.")
    +GameplayTagList=(Tag="AI.Behavior.Ranged",DevComment="Ranged: prefers keeping distance, uses projectiles.")
    +GameplayTagList=(Tag="AI.Behavior.Hit-And-Run",DevComment="Hit-And-Run: attacks then retreats.")
    +GameplayTagList=(Tag="AI.Behavior.Ambush",DevComment="Ambush: waits hidden, strikes first.")

    ; -- AI Role (functional role in a group) --
    +GameplayTagList=(Tag="AI.Role.Melee",DevComment="Melee combatant.")
    +GameplayTagList=(Tag="AI.Role.Ranged",DevComment="Ranged combatant.")
    +GameplayTagList=(Tag="AI.Role.Tank",DevComment="Frontline / damage sponge.")
    +GameplayTagList=(Tag="AI.Role.Healer",DevComment="Healer / support caster.")
    +GameplayTagList=(Tag="AI.Role.Caster",DevComment="Magic damage dealer.")
    +GameplayTagList=(Tag="AI.Role.Assassin",DevComment="High burst, flanker.")
    +GameplayTagList=(Tag="AI.Role.Summoner",DevComment="Spawns adds/minions.")

    ; -- AI Awareness (perception state) --
    +GameplayTagList=(Tag="AI.Awareness.Unaware",DevComment="Has not detected any threat.")
    +GameplayTagList=(Tag="AI.Awareness.Suspicious",DevComment="Partial stimulus -- not confirmed.")
    +GameplayTagList=(Tag="AI.Awareness.Detected",DevComment="Target confirmed and tracked.")
    +GameplayTagList=(Tag="AI.Awareness.Lost",DevComment="Had target, now lost -- searching.")
    ```

| Tag | Purpose |
|---|---|
| `AI.State.*` | Current AI brain state -- behavior tree switches on these |
| `AI.Behavior.*` | Combat personality -- set on spawn or dynamically (e.g., `Berserk` at low health) |
| `AI.Role.*` | Group composition role -- used for encounter balancing and AI coordination |
| `AI.Awareness.*` | Perception pipeline output -- drives alert/investigate/combat transitions |

---

## Animation Tags

Slot claims that prevent abilities from fighting over the animation system, plus playback state tags for logic gating.

??? example "Tags_Animation.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Animation Slot Claims --
    +GameplayTagList=(Tag="Animation",DevComment="Animation namespace for slot claims and states.")

    +GameplayTagList=(Tag="Animation.Slot.FullBody",DevComment="Claims full-body animation (blocks all other slots).")
    +GameplayTagList=(Tag="Animation.Slot.UpperBody",DevComment="Claims upper-body animation only.")
    +GameplayTagList=(Tag="Animation.Slot.LowerBody",DevComment="Claims lower-body animation only.")
    +GameplayTagList=(Tag="Animation.Slot.Additive",DevComment="Additive layer (can blend on top of others).")

    ; -- Animation State --
    +GameplayTagList=(Tag="Animation.State.Attacking",DevComment="Currently in attack montage.")
    +GameplayTagList=(Tag="Animation.State.Casting",DevComment="Currently in cast montage.")
    +GameplayTagList=(Tag="Animation.State.HitReact",DevComment="Currently playing hit reaction.")
    +GameplayTagList=(Tag="Animation.State.Staggered",DevComment="Currently playing stagger animation.")
    +GameplayTagList=(Tag="Animation.State.Death",DevComment="Currently playing death animation.")
    +GameplayTagList=(Tag="Animation.State.Dodging",DevComment="Currently in dodge/evade animation.")
    +GameplayTagList=(Tag="Animation.State.Blocking",DevComment="Currently in block/guard pose.")
    +GameplayTagList=(Tag="Animation.State.Interacting",DevComment="Currently in interact animation.")
    ```

| Tag | Purpose |
|---|---|
| `Animation.Slot.*` | Added as ability tags to claim animation layers -- prevents a dodge from overwriting a cast montage |
| `Animation.State.*` | Granted while a montage plays -- other systems (AI, input) can query these to know what's happening |

---

## Combat and Damage Tags

Combat results, damage types, delivery methods, elemental resistances, and immunity rules. This is the tag backbone of the [damage pipeline](damage-pipeline.md).

??? example "Tags_CombatDamage.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Combat Results --
    +GameplayTagList=(Tag="Combat",DevComment="Combat classification and results.")
    +GameplayTagList=(Tag="Combat.Result.Hit",DevComment="A hit connected.")
    +GameplayTagList=(Tag="Combat.Result.Crit",DevComment="A critical hit connected.")
    +GameplayTagList=(Tag="Combat.Result.Block",DevComment="Attack was blocked.")
    +GameplayTagList=(Tag="Combat.Result.Parry",DevComment="Attack was parried.")
    +GameplayTagList=(Tag="Combat.Result.Dodge",DevComment="Attack was dodged.")
    +GameplayTagList=(Tag="Combat.Result.Miss",DevComment="Attack missed.")
    +GameplayTagList=(Tag="Combat.Result.Immune",DevComment="Target was immune.")
    +GameplayTagList=(Tag="Combat.Result.Absorbed",DevComment="Damage was absorbed by shield.")
    +GameplayTagList=(Tag="Combat.Result.Resisted",DevComment="Damage was resisted/mitigated heavily.")
    +GameplayTagList=(Tag="Combat.Result.Kill",DevComment="Target was killed by this hit.")
    +GameplayTagList=(Tag="Combat.Result.GuardBreak",DevComment="Block/guard was broken (stamina depleted).")

    ; -- Damage Type --
    +GameplayTagList=(Tag="Damage",DevComment="Damage classification: type and delivery.")
    +GameplayTagList=(Tag="Damage.Type.Physical",DevComment="Physical baseline damage.")
    +GameplayTagList=(Tag="Damage.Type.Slash",DevComment="Blade slash damage.")
    +GameplayTagList=(Tag="Damage.Type.Pierce",DevComment="Piercing damage (arrows/spears).")
    +GameplayTagList=(Tag="Damage.Type.Blunt",DevComment="Blunt impact damage.")
    +GameplayTagList=(Tag="Damage.Type.Fire",DevComment="Fire damage.")
    +GameplayTagList=(Tag="Damage.Type.Ice",DevComment="Ice damage.")
    +GameplayTagList=(Tag="Damage.Type.Lightning",DevComment="Lightning damage.")
    +GameplayTagList=(Tag="Damage.Type.Nature",DevComment="Nature/poison damage.")
    +GameplayTagList=(Tag="Damage.Type.Arcane",DevComment="Arcane/magic force damage.")
    +GameplayTagList=(Tag="Damage.Type.Holy",DevComment="Holy/radiant damage.")
    +GameplayTagList=(Tag="Damage.Type.Shadow",DevComment="Shadow/necrotic damage.")
    +GameplayTagList=(Tag="Damage.Type.Void",DevComment="Void/entropy damage.")
    +GameplayTagList=(Tag="Damage.Type.True",DevComment="Ignores mitigation (use sparingly).")
    +GameplayTagList=(Tag="Damage.Type.Chaos",DevComment="Exotic/untyped hybrid damage.")

    ; -- Damage Delivery --
    +GameplayTagList=(Tag="Damage.Delivery.Direct",DevComment="Instant application.")
    +GameplayTagList=(Tag="Damage.Delivery.Periodic",DevComment="Periodic tick damage.")
    +GameplayTagList=(Tag="Damage.Delivery.Aoe",DevComment="Area damage.")
    +GameplayTagList=(Tag="Damage.Delivery.Projectile",DevComment="Projectile hit delivery.")
    +GameplayTagList=(Tag="Damage.Delivery.Melee",DevComment="Melee hit delivery.")
    +GameplayTagList=(Tag="Damage.Delivery.Chain",DevComment="Chains between targets.")
    +GameplayTagList=(Tag="Damage.Delivery.Bounce",DevComment="Bounces between targets.")
    +GameplayTagList=(Tag="Damage.Delivery.Ground",DevComment="Ground zone/hazard damage.")
    +GameplayTagList=(Tag="Damage.Delivery.Reflect",DevComment="Reflected damage back to attacker.")

    ; -- Resistance --
    +GameplayTagList=(Tag="Resistance",DevComment="Resistances reduce damage taken by type.")
    +GameplayTagList=(Tag="Resistance.Physical",DevComment="Reduces physical damage taken.")
    +GameplayTagList=(Tag="Resistance.Fire",DevComment="Reduces fire damage taken.")
    +GameplayTagList=(Tag="Resistance.Ice",DevComment="Reduces ice damage taken.")
    +GameplayTagList=(Tag="Resistance.Lightning",DevComment="Reduces lightning damage taken.")
    +GameplayTagList=(Tag="Resistance.Nature",DevComment="Reduces nature damage taken.")
    +GameplayTagList=(Tag="Resistance.Arcane",DevComment="Reduces arcane damage taken.")
    +GameplayTagList=(Tag="Resistance.Holy",DevComment="Reduces holy damage taken.")
    +GameplayTagList=(Tag="Resistance.Shadow",DevComment="Reduces shadow damage taken.")
    +GameplayTagList=(Tag="Resistance.Void",DevComment="Reduces void damage taken.")
    +GameplayTagList=(Tag="Resistance.Chaos",DevComment="Reduces chaos damage taken.")

    ; -- Immunity --
    +GameplayTagList=(Tag="Immunity",DevComment="Immunity rules; prefer specific immunities.")
    +GameplayTagList=(Tag="Immunity.Damage",DevComment="Immune to all damage.")
    +GameplayTagList=(Tag="Immunity.Damage.Physical",DevComment="Immune to physical damage.")
    +GameplayTagList=(Tag="Immunity.Damage.Fire",DevComment="Immune to fire damage.")
    +GameplayTagList=(Tag="Immunity.CrowdControl",DevComment="Immune to all crowd control.")
    +GameplayTagList=(Tag="Immunity.CrowdControl.Stun",DevComment="Immune to stun.")
    +GameplayTagList=(Tag="Immunity.CrowdControl.Root",DevComment="Immune to root.")
    +GameplayTagList=(Tag="Immunity.CrowdControl.Silence",DevComment="Immune to silence.")
    +GameplayTagList=(Tag="Immunity.CrowdControl.Knockback",DevComment="Immune to knockback/knockup.")
    ```

| Tag | Purpose |
|---|---|
| `Combat.Result.*` | Output of hit resolution -- drives cue selection, event broadcasts, and UI |
| `Damage.Type.*` | Elemental/physical classification -- matched against `Resistance.*` in ExecCalc |
| `Damage.Delivery.*` | How damage reaches the target -- affects AoE falloff, projectile behavior |
| `Resistance.*` | Per-element damage reduction -- maps 1:1 to `Damage.Type.*` subtags |
| `Immunity.*` | Hard blocks -- checked via GE tag requirements before damage applies |

---

## Combo Tags

Combo chain system: input windows, step counters, branch selection, and combo state.

??? example "Tags_Combo.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Combo Window --
    +GameplayTagList=(Tag="Combo",DevComment="Combo system namespace.")
    +GameplayTagList=(Tag="Combo.Window.Open",DevComment="Combo input window is open -- next input will chain.")
    +GameplayTagList=(Tag="Combo.Window.Closed",DevComment="Combo input window is closed -- input resets.")

    ; -- Combo Count --
    +GameplayTagList=(Tag="Combo.Count.1",DevComment="First hit in combo chain.")
    +GameplayTagList=(Tag="Combo.Count.2",DevComment="Second hit in combo chain.")
    +GameplayTagList=(Tag="Combo.Count.3",DevComment="Third hit in combo chain.")
    +GameplayTagList=(Tag="Combo.Count.4",DevComment="Fourth hit in combo chain.")
    +GameplayTagList=(Tag="Combo.Count.Finisher",DevComment="Final/finisher hit in combo chain.")

    ; -- Combo Branch --
    +GameplayTagList=(Tag="Combo.Branch.Light",DevComment="Light attack branch in combo.")
    +GameplayTagList=(Tag="Combo.Branch.Heavy",DevComment="Heavy attack branch in combo.")
    +GameplayTagList=(Tag="Combo.Branch.Special",DevComment="Special/launcher branch in combo.")
    +GameplayTagList=(Tag="Combo.Branch.Aerial",DevComment="Aerial continuation branch.")

    ; -- Combo State --
    +GameplayTagList=(Tag="Combo.Reset",DevComment="Combo has been reset (timeout or explicit cancel).")
    +GameplayTagList=(Tag="Combo.Active",DevComment="A combo chain is currently in progress.")
    ```

| Tag | Purpose |
|---|---|
| `Combo.Window.*` | Granted/removed by anim notifies to open/close the input window |
| `Combo.Count.*` | Tracks current step -- used as activation requirements on combo abilities |
| `Combo.Branch.*` | Selects the next attack type -- light/heavy branching within a chain |
| `Combo.Active` / `.Reset` | Overall combo state -- used to gate combo-only abilities and reset counters |

---

## Cooldown Tags

Cooldown identifiers for abilities, actions, and items. GAS cooldown effects use these tags to track which cooldown is active.

??? example "Tags_Cooldown.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Cooldown Root --
    +GameplayTagList=(Tag="Cooldown",DevComment="Root namespace for ability/action cooldowns.")

    ; -- Ability Cooldowns --
    +GameplayTagList=(Tag="Cooldown.Ability.BasicAttack",DevComment="Cooldown for basic attack.")
    +GameplayTagList=(Tag="Cooldown.Ability.Secondary",DevComment="Cooldown for secondary attack.")
    +GameplayTagList=(Tag="Cooldown.Ability.Mobility",DevComment="Cooldown for mobility ability (dash/blink).")
    +GameplayTagList=(Tag="Cooldown.Ability.Defensive",DevComment="Cooldown for defensive ability.")
    +GameplayTagList=(Tag="Cooldown.Ability.Support",DevComment="Cooldown for support ability.")
    +GameplayTagList=(Tag="Cooldown.Ability.Ultimate",DevComment="Cooldown for ultimate ability.")
    +GameplayTagList=(Tag="Cooldown.Ability.Slot1",DevComment="Cooldown for ability slot 1.")
    +GameplayTagList=(Tag="Cooldown.Ability.Slot2",DevComment="Cooldown for ability slot 2.")
    +GameplayTagList=(Tag="Cooldown.Ability.Slot3",DevComment="Cooldown for ability slot 3.")
    +GameplayTagList=(Tag="Cooldown.Ability.Slot4",DevComment="Cooldown for ability slot 4.")
    +GameplayTagList=(Tag="Cooldown.Ability.Summon",DevComment="Cooldown for summon ability.")

    ; -- Action Cooldowns --
    +GameplayTagList=(Tag="Cooldown.Evade",DevComment="Cooldown between evades/dodges.")
    +GameplayTagList=(Tag="Cooldown.Block",DevComment="Cooldown after guard break before blocking again.")
    +GameplayTagList=(Tag="Cooldown.Parry",DevComment="Cooldown between parry attempts.")
    +GameplayTagList=(Tag="Cooldown.Interact",DevComment="Cooldown on interaction (prevent spam).")

    ; -- Item Cooldowns --
    +GameplayTagList=(Tag="Cooldown.Item",DevComment="Cooldown for item usage.")
    +GameplayTagList=(Tag="Cooldown.Item.QuickSlot.1",DevComment="Cooldown for quick slot 1 item.")
    +GameplayTagList=(Tag="Cooldown.Item.QuickSlot.2",DevComment="Cooldown for quick slot 2 item.")
    +GameplayTagList=(Tag="Cooldown.Item.QuickSlot.3",DevComment="Cooldown for quick slot 3 item.")
    +GameplayTagList=(Tag="Cooldown.Item.QuickSlot.4",DevComment="Cooldown for quick slot 4 item.")
    +GameplayTagList=(Tag="Cooldown.Item.Potion",DevComment="Shared potion cooldown.")
    ```

| Tag | Purpose |
|---|---|
| `Cooldown.Ability.*` | Applied by cooldown GEs -- the ability's `CooldownGameplayEffectClass` grants these |
| `Cooldown.Evade` / `.Block` / `.Parry` | Action cooldowns that aren't tied to a specific ability slot |
| `Cooldown.Item.*` | Shared or per-slot item cooldowns -- prevents potion spam |

---

## Cues and Events Tags

Gameplay Cues (cosmetic VFX/SFX), semantic gameplay events (for messaging and procs), and SetByCaller magnitude tags (runtime values passed into Gameplay Effects).

??? example "Tags_CuesEvents.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Gameplay Cues (cosmetic-only: VFX/SFX) --

    ; Combat Cues
    +GameplayTagList=(Tag="GameplayCue",DevComment="Cosmetic-only cues (VFX/SFX). Keep logic out of cues.")
    +GameplayTagList=(Tag="GameplayCue.Combat.Hit",DevComment="Cue for generic hit feedback.")
    +GameplayTagList=(Tag="GameplayCue.Combat.Hit.Light",DevComment="Cue for light hit feedback.")
    +GameplayTagList=(Tag="GameplayCue.Combat.Hit.Heavy",DevComment="Cue for heavy hit feedback.")
    +GameplayTagList=(Tag="GameplayCue.Combat.Crit",DevComment="Cue for critical hit feedback.")
    +GameplayTagList=(Tag="GameplayCue.Combat.Block",DevComment="Cue for block feedback.")
    +GameplayTagList=(Tag="GameplayCue.Combat.Parry",DevComment="Cue for parry feedback.")
    +GameplayTagList=(Tag="GameplayCue.Combat.GuardBreak",DevComment="Cue for guard break feedback.")
    +GameplayTagList=(Tag="GameplayCue.Combat.Kill",DevComment="Cue for killing blow feedback.")

    ; Status Cues
    +GameplayTagList=(Tag="GameplayCue.Status.Stun",DevComment="Cue for stunned feedback.")
    +GameplayTagList=(Tag="GameplayCue.Status.Slow",DevComment="Cue for slowed feedback.")
    +GameplayTagList=(Tag="GameplayCue.Status.Burn",DevComment="Cue for burning feedback.")
    +GameplayTagList=(Tag="GameplayCue.Status.Poison",DevComment="Cue for poison feedback.")
    +GameplayTagList=(Tag="GameplayCue.Status.Bleed",DevComment="Cue for bleeding feedback.")
    +GameplayTagList=(Tag="GameplayCue.Status.Freeze",DevComment="Cue for frozen feedback.")
    +GameplayTagList=(Tag="GameplayCue.Status.Stagger",DevComment="Cue for stagger/poise break.")

    ; Ability Cues
    +GameplayTagList=(Tag="GameplayCue.Ability.CastStart",DevComment="Cue for cast windup.")
    +GameplayTagList=(Tag="GameplayCue.Ability.CastEnd",DevComment="Cue for cast release/finish.")
    +GameplayTagList=(Tag="GameplayCue.Ability.Impact",DevComment="Cue for ability impact.")
    +GameplayTagList=(Tag="GameplayCue.Ability.Channel",DevComment="Cue for sustained channeling loop.")

    ; Movement Cues
    +GameplayTagList=(Tag="GameplayCue.Movement.Dash",DevComment="Cue for dash/dodge VFX.")
    +GameplayTagList=(Tag="GameplayCue.Movement.Land",DevComment="Cue for heavy landing impact.")
    +GameplayTagList=(Tag="GameplayCue.Movement.Sprint",DevComment="Cue for sprint VFX (dust/trails).")

    ; UI/Feedback Cues
    +GameplayTagList=(Tag="GameplayCue.UI.DamageNumber",DevComment="Cue for floating damage number.")
    +GameplayTagList=(Tag="GameplayCue.UI.HealNumber",DevComment="Cue for floating heal number.")
    +GameplayTagList=(Tag="GameplayCue.UI.LevelUp",DevComment="Cue for level-up fanfare.")

    ; -- Gameplay Events (semantic events for messaging/procs/UI) --
    +GameplayTagList=(Tag="Event",DevComment="Semantic gameplay events for messaging/procs/UI.")

    ; Combat Events
    +GameplayTagList=(Tag="Event.Combat.Hit",DevComment="Event emitted when a hit is confirmed.")
    +GameplayTagList=(Tag="Event.Combat.Crit",DevComment="Event emitted when a crit is confirmed.")
    +GameplayTagList=(Tag="Event.Combat.Kill",DevComment="Event emitted when a kill occurs.")
    +GameplayTagList=(Tag="Event.Combat.Block",DevComment="Event emitted when an attack is blocked.")
    +GameplayTagList=(Tag="Event.Combat.Parry",DevComment="Event emitted when an attack is parried.")
    +GameplayTagList=(Tag="Event.Combat.Dodge",DevComment="Event emitted when an attack is dodged.")
    +GameplayTagList=(Tag="Event.Combat.GuardBreak",DevComment="Event emitted when guard/block is broken.")
    +GameplayTagList=(Tag="Event.Combat.Death",DevComment="Event emitted when the owner dies.")

    ; Ability Events
    +GameplayTagList=(Tag="Event.Ability.Activated",DevComment="Event emitted when an ability activates.")
    +GameplayTagList=(Tag="Event.Ability.Ended",DevComment="Event emitted when an ability ends.")
    +GameplayTagList=(Tag="Event.Ability.Failed",DevComment="Event emitted when an ability fails activation.")
    +GameplayTagList=(Tag="Event.Ability.Interrupted",DevComment="Event emitted when an ability is interrupted.")

    ; Status Events
    +GameplayTagList=(Tag="Event.Status.Applied",DevComment="Event emitted when a status is applied.")
    +GameplayTagList=(Tag="Event.Status.Removed",DevComment="Event emitted when a status is removed.")
    +GameplayTagList=(Tag="Event.Status.Refreshed",DevComment="Event emitted when a status is refreshed/restacked.")

    ; Combo Events
    +GameplayTagList=(Tag="Event.Combo.Advance",DevComment="Event emitted when combo advances to next step.")
    +GameplayTagList=(Tag="Event.Combo.Finish",DevComment="Event emitted when combo finisher executes.")
    +GameplayTagList=(Tag="Event.Combo.Drop",DevComment="Event emitted when combo window expires.")

    ; Poise Events
    +GameplayTagList=(Tag="Event.Poise.Break",DevComment="Event emitted when poise hits zero (stagger).")
    +GameplayTagList=(Tag="Event.Poise.Recover",DevComment="Event emitted when poise fully recovers.")

    ; -- SetByCaller (runtime numeric magnitudes passed into GameplayEffects) --
    +GameplayTagList=(Tag="SetByCaller",DevComment="Runtime numeric magnitudes passed into GameplayEffects.")
    +GameplayTagList=(Tag="SetByCaller.Damage",DevComment="Damage amount passed into a GE.")
    +GameplayTagList=(Tag="SetByCaller.Heal",DevComment="Heal amount passed into a GE.")
    +GameplayTagList=(Tag="SetByCaller.Shield",DevComment="Shield amount passed into a GE.")
    +GameplayTagList=(Tag="SetByCaller.Duration",DevComment="Dynamic duration passed into a GE.")
    +GameplayTagList=(Tag="SetByCaller.Stacks",DevComment="Initial stacks passed into a GE.")
    +GameplayTagList=(Tag="SetByCaller.Radius",DevComment="AoE radius passed into ability/GE logic.")
    +GameplayTagList=(Tag="SetByCaller.CooldownDuration",DevComment="Cooldown duration passed into a cooldown GE.")
    +GameplayTagList=(Tag="SetByCaller.PoiseDamage",DevComment="Poise damage amount passed into a GE.")
    +GameplayTagList=(Tag="SetByCaller.StaminaCost",DevComment="Stamina cost passed into a cost GE.")
    +GameplayTagList=(Tag="SetByCaller.ManaCost",DevComment="Mana/magic cost passed into a cost GE.")
    +GameplayTagList=(Tag="SetByCaller.Magnitude",DevComment="Generic magnitude for flexible GE usage.")
    ```

| Tag | Purpose |
|---|---|
| `GameplayCue.Combat.*` | Hit/block/parry VFX and SFX -- triggered from the damage pipeline |
| `GameplayCue.Status.*` | Looping status VFX (burning particles, frozen overlay) |
| `Event.Combat.*` | Broadcast after hit resolution -- listened to by proc systems, kill feeds, achievements |
| `Event.Combo.*` | Combo system events -- drive UI updates and combo-dependent abilities |
| `SetByCaller.*` | Runtime magnitudes set on GE specs before application (see [SetByCaller](../gameplay-effects/set-by-caller.md)) |

---

## Enemy Tags

Hostile entity classification: difficulty ranks, creature types, and Diablo-style affix modifiers for elite/champion scaling.

??? example "Tags_Enemy.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Enemy Rank (difficulty tier) --
    +GameplayTagList=(Tag="Enemy",DevComment="Hostile entity classification and scaling.")
    +GameplayTagList=(Tag="Enemy.Rank.Minion",DevComment="Weak fodder enemy (dies fast, appears in groups).")
    +GameplayTagList=(Tag="Enemy.Rank.Normal",DevComment="Standard enemy (baseline difficulty).")
    +GameplayTagList=(Tag="Enemy.Rank.Elite",DevComment="Elite enemy (stronger, may have affixes).")
    +GameplayTagList=(Tag="Enemy.Rank.Champion",DevComment="Champion enemy (mini-boss tier, unique mechanics).")
    +GameplayTagList=(Tag="Enemy.Rank.Boss",DevComment="Boss enemy (scripted encounter, phases).")
    +GameplayTagList=(Tag="Enemy.Rank.WorldBoss",DevComment="World boss (open-world, multi-player scale).")

    ; -- Enemy Type (creature classification) --
    +GameplayTagList=(Tag="Enemy.Type.Humanoid",DevComment="Humanoid enemy (bandits, soldiers, mages).")
    +GameplayTagList=(Tag="Enemy.Type.Beast",DevComment="Beast/animal enemy (wolves, boars).")
    +GameplayTagList=(Tag="Enemy.Type.Undead",DevComment="Undead enemy (skeletons, zombies, wraiths).")
    +GameplayTagList=(Tag="Enemy.Type.Construct",DevComment="Construct enemy (golems, automatons).")
    +GameplayTagList=(Tag="Enemy.Type.Elemental",DevComment="Elemental enemy (fire/ice/nature spirits).")
    +GameplayTagList=(Tag="Enemy.Type.Demon",DevComment="Demon/fiend enemy.")
    +GameplayTagList=(Tag="Enemy.Type.Dragon",DevComment="Dragon/draconic enemy.")
    +GameplayTagList=(Tag="Enemy.Type.Aberration",DevComment="Aberration/eldritch enemy.")
    +GameplayTagList=(Tag="Enemy.Type.Spirit",DevComment="Ghost/spectral enemy.")
    +GameplayTagList=(Tag="Enemy.Type.Insectoid",DevComment="Insect/arachnid enemy.")

    ; -- Enemy Affix (modifiers for elite/champion scaling) --
    +GameplayTagList=(Tag="Enemy.Affix",DevComment="Runtime modifiers applied to elites/champions.")
    +GameplayTagList=(Tag="Enemy.Affix.Enraged",DevComment="Increased damage and attack speed.")
    +GameplayTagList=(Tag="Enemy.Affix.Shielded",DevComment="Periodic damage shield.")
    +GameplayTagList=(Tag="Enemy.Affix.Regenerating",DevComment="Passive health regeneration.")
    +GameplayTagList=(Tag="Enemy.Affix.Teleporting",DevComment="Periodically blinks to player.")
    +GameplayTagList=(Tag="Enemy.Affix.Splitting",DevComment="Spawns smaller copies on death.")
    +GameplayTagList=(Tag="Enemy.Affix.Vampiric",DevComment="Heals on hit.")
    +GameplayTagList=(Tag="Enemy.Affix.Molten",DevComment="Fire aura / leaves fire trails.")
    +GameplayTagList=(Tag="Enemy.Affix.Frozen",DevComment="Ice aura / freezing attacks.")
    +GameplayTagList=(Tag="Enemy.Affix.Thorns",DevComment="Reflects damage to attackers.")
    +GameplayTagList=(Tag="Enemy.Affix.Warlord",DevComment="Buffs nearby allies.")
    ```

| Tag | Purpose |
|---|---|
| `Enemy.Rank.*` | Difficulty tier -- drives stat scaling, loot tables, XP rewards |
| `Enemy.Type.*` | Creature classification -- used for damage bonuses ("deals +20% to Undead") and bestiary |
| `Enemy.Affix.*` | Runtime modifiers -- randomly applied to Elites/Champions for encounter variety |

---

## Equipment Tags

Equipment slots and weapon type classification. Used by the inventory system to enforce slot rules and by abilities to gate valid moves based on equipped weapon.

??? example "Tags_Equipment.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Equipment Slots --
    +GameplayTagList=(Tag="Equipment",DevComment="Equipment namespace for slots and weapon types.")
    +GameplayTagList=(Tag="Equipment.Slot.MainHand",DevComment="Main-hand weapon slot.")
    +GameplayTagList=(Tag="Equipment.Slot.OffHand",DevComment="Off-hand weapon/shield slot.")
    +GameplayTagList=(Tag="Equipment.Slot.TwoHand",DevComment="Two-handed weapon (blocks both hand slots).")
    +GameplayTagList=(Tag="Equipment.Slot.Head",DevComment="Head armor slot.")
    +GameplayTagList=(Tag="Equipment.Slot.Chest",DevComment="Chest armor slot.")
    +GameplayTagList=(Tag="Equipment.Slot.Legs",DevComment="Leg armor slot.")
    +GameplayTagList=(Tag="Equipment.Slot.Feet",DevComment="Feet armor slot.")
    +GameplayTagList=(Tag="Equipment.Slot.Hands",DevComment="Hand armor slot.")
    +GameplayTagList=(Tag="Equipment.Slot.Back",DevComment="Back slot (cloak/cape).")
    +GameplayTagList=(Tag="Equipment.Slot.Ring1",DevComment="Ring slot 1.")
    +GameplayTagList=(Tag="Equipment.Slot.Ring2",DevComment="Ring slot 2.")
    +GameplayTagList=(Tag="Equipment.Slot.Amulet",DevComment="Amulet/necklace slot.")

    ; -- Weapon Types --
    +GameplayTagList=(Tag="Weapon",DevComment="Weapon type classification.")
    +GameplayTagList=(Tag="Weapon.Type.Sword",DevComment="One-handed sword.")
    +GameplayTagList=(Tag="Weapon.Type.Axe",DevComment="One-handed axe.")
    +GameplayTagList=(Tag="Weapon.Type.Mace",DevComment="One-handed mace/club.")
    +GameplayTagList=(Tag="Weapon.Type.Dagger",DevComment="Dagger/short blade.")
    +GameplayTagList=(Tag="Weapon.Type.Spear",DevComment="Spear/polearm.")
    +GameplayTagList=(Tag="Weapon.Type.GreatSword",DevComment="Two-handed great sword.")
    +GameplayTagList=(Tag="Weapon.Type.GreatAxe",DevComment="Two-handed great axe.")
    +GameplayTagList=(Tag="Weapon.Type.Hammer",DevComment="Two-handed hammer/maul.")
    +GameplayTagList=(Tag="Weapon.Type.Bow",DevComment="Bow (ranged).")
    +GameplayTagList=(Tag="Weapon.Type.Crossbow",DevComment="Crossbow (ranged).")
    +GameplayTagList=(Tag="Weapon.Type.Staff",DevComment="Staff (magic catalyst).")
    +GameplayTagList=(Tag="Weapon.Type.Wand",DevComment="Wand (magic catalyst, one-hand).")
    +GameplayTagList=(Tag="Weapon.Type.Shield",DevComment="Shield (off-hand defensive).")
    +GameplayTagList=(Tag="Weapon.Type.Fist",DevComment="Unarmed/fist weapon.")
    ```

| Tag | Purpose |
|---|---|
| `Equipment.Slot.*` | Enforces one-item-per-slot -- `TwoHand` should block both `MainHand` and `OffHand` |
| `Weapon.Type.*` | Ability activation requirements -- a sword ability checks for `Weapon.Type.Sword` on the owner |

---

## Input Tags

Tag-driven input mapping for Enhanced Input. Each tag maps to an `UInputAction` and is matched against the `InputTag` property on your [base ability class](../getting-started/project-setup.md#6-the-base-ability-class).

??? example "Tags_Input.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Input Root --
    +GameplayTagList=(Tag="InputTag",DevComment="Root namespace for tag-driven input mapping.")

    ; -- Combat Inputs --
    +GameplayTagList=(Tag="InputTag.Combat",DevComment="Combat input group.")
    +GameplayTagList=(Tag="InputTag.Combat.Primary",DevComment="Primary attack input (basic attack).")
    +GameplayTagList=(Tag="InputTag.Combat.Secondary",DevComment="Secondary attack input (alt attack).")
    +GameplayTagList=(Tag="InputTag.Combat.Light",DevComment="Light attack branch (if distinct from primary).")
    +GameplayTagList=(Tag="InputTag.Combat.Heavy",DevComment="Heavy attack branch (if distinct).")
    +GameplayTagList=(Tag="InputTag.Combat.Block",DevComment="Block/guard input.")
    +GameplayTagList=(Tag="InputTag.Combat.Parry",DevComment="Parry/perfect guard input.")
    +GameplayTagList=(Tag="InputTag.Combat.Aim",DevComment="Aim/lock-on input (ranged or target lock).")

    ; -- Ability Slot Inputs --
    +GameplayTagList=(Tag="InputTag.Ability",DevComment="Ability slot input group.")
    +GameplayTagList=(Tag="InputTag.Ability.Slot1",DevComment="Ability slot 1.")
    +GameplayTagList=(Tag="InputTag.Ability.Slot2",DevComment="Ability slot 2.")
    +GameplayTagList=(Tag="InputTag.Ability.Slot3",DevComment="Ability slot 3.")
    +GameplayTagList=(Tag="InputTag.Ability.Slot4",DevComment="Ability slot 4.")
    +GameplayTagList=(Tag="InputTag.Ability.Ultimate",DevComment="Ultimate slot input.")

    ; -- Movement Inputs --
    +GameplayTagList=(Tag="InputTag.Movement",DevComment="Movement input group.")
    +GameplayTagList=(Tag="InputTag.Movement.Dodge",DevComment="Dodge/evade input.")
    +GameplayTagList=(Tag="InputTag.Movement.Sprint",DevComment="Sprint input.")
    +GameplayTagList=(Tag="InputTag.Movement.Jump",DevComment="Jump input.")
    +GameplayTagList=(Tag="InputTag.Movement.Crouch",DevComment="Crouch/slide input.")

    ; -- Interaction Inputs --
    +GameplayTagList=(Tag="InputTag.Interact",DevComment="Interact input (loot/talk/open/revive).")

    ; -- Quick Slot Inputs --
    +GameplayTagList=(Tag="InputTag.QuickSlot",DevComment="Quick item slot group (consumables).")
    +GameplayTagList=(Tag="InputTag.QuickSlot.1",DevComment="Quick slot 1.")
    +GameplayTagList=(Tag="InputTag.QuickSlot.2",DevComment="Quick slot 2.")
    +GameplayTagList=(Tag="InputTag.QuickSlot.3",DevComment="Quick slot 3.")
    +GameplayTagList=(Tag="InputTag.QuickSlot.4",DevComment="Quick slot 4.")

    ; -- Menu/UI Inputs --
    +GameplayTagList=(Tag="InputTag.Menu",DevComment="Menu/UI input group.")
    +GameplayTagList=(Tag="InputTag.Menu.Pause",DevComment="Pause/escape input.")
    +GameplayTagList=(Tag="InputTag.Menu.Inventory",DevComment="Inventory toggle input.")
    +GameplayTagList=(Tag="InputTag.Menu.Map",DevComment="Map toggle input.")
    ```

| Tag | Purpose |
|---|---|
| `InputTag.Combat.*` | Combat inputs -- matched to abilities via the `InputTag` property on your base ability |
| `InputTag.Ability.Slot*` | Hotbar-style ability slots -- bind to number keys or gamepad buttons |
| `InputTag.Movement.*` | Movement abilities (dodge, sprint) -- separate from raw movement input |
| `InputTag.QuickSlot.*` | Consumable item slots -- separated from ability slots |
| `InputTag.Menu.*` | UI toggle inputs -- useful for blocking gameplay input while menus are open |

---

## Movement Tags

Current locomotion state and camera/targeting modes. These are **actual state**, not input -- they're granted by the movement system and read by abilities and animation.

??? example "Tags_Movement.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Movement State --
    +GameplayTagList=(Tag="Movement",DevComment="Movement state namespace (what the character is doing).")

    +GameplayTagList=(Tag="Movement.State.Grounded",DevComment="On the ground.")
    +GameplayTagList=(Tag="Movement.State.Airborne",DevComment="In the air (jumping or falling).")
    +GameplayTagList=(Tag="Movement.State.Jumping",DevComment="Actively ascending from a jump.")
    +GameplayTagList=(Tag="Movement.State.Falling",DevComment="Falling (not from a jump, or jump apex passed).")
    +GameplayTagList=(Tag="Movement.State.Swimming",DevComment="In water.")
    +GameplayTagList=(Tag="Movement.State.Climbing",DevComment="On a climbable surface.")
    +GameplayTagList=(Tag="Movement.State.Sliding",DevComment="Sliding (slope or crouch-slide).")
    +GameplayTagList=(Tag="Movement.State.Crouching",DevComment="Crouched.")
    +GameplayTagList=(Tag="Movement.State.Sprinting",DevComment="Sprinting (increased speed, may drain stamina).")
    +GameplayTagList=(Tag="Movement.State.Walking",DevComment="Walking (reduced speed).")
    +GameplayTagList=(Tag="Movement.State.Mounted",DevComment="On a mount.")

    ; -- Movement Mode (camera/targeting mode) --
    +GameplayTagList=(Tag="Movement.Mode.Free",DevComment="Free movement (camera-relative).")
    +GameplayTagList=(Tag="Movement.Mode.Locked",DevComment="Target-locked movement (strafe around target).")
    +GameplayTagList=(Tag="Movement.Mode.Strafe",DevComment="Strafe mode (always face forward).")
    ```

| Tag | Purpose |
|---|---|
| `Movement.State.*` | Current locomotion -- abilities use these as activation requirements (e.g., "only while grounded") |
| `Movement.Mode.*` | Camera/targeting mode -- affects dodge direction, attack facing, animation blend |

---

## Phase Tags

Encounter phases for boss fights and multi-stage events. Applied to the encounter controller or the boss actor to drive phase-specific behavior.

??? example "Tags_Phase.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Encounter Phases --
    +GameplayTagList=(Tag="Phase",DevComment="Encounter phase namespace (boss fights, staged events).")
    +GameplayTagList=(Tag="Phase.1",DevComment="Phase 1 (initial).")
    +GameplayTagList=(Tag="Phase.2",DevComment="Phase 2.")
    +GameplayTagList=(Tag="Phase.3",DevComment="Phase 3.")
    +GameplayTagList=(Tag="Phase.Enraged",DevComment="Enraged phase (low health, increased aggression).")
    +GameplayTagList=(Tag="Phase.Transition",DevComment="Phase transition (invuln, cinematic, repositioning).")
    +GameplayTagList=(Tag="Phase.Intermission",DevComment="Intermission (adds phase, puzzle, etc).")
    +GameplayTagList=(Tag="Phase.Final",DevComment="Final/desperation phase.")
    ```

| Tag | Purpose |
|---|---|
| `Phase.1` through `.3` | Numbered phases -- abilities can require a specific phase to activate |
| `Phase.Enraged` / `.Final` | Named phases -- applied at health thresholds, grant new abilities or stat buffs |
| `Phase.Transition` / `.Intermission` | Non-combat phases -- typically grant invulnerability and trigger cinematic or add spawns |

---

## State, Status, and Crowd Control Tags

Broad action states, crowd control taxonomy (hard CC, soft CC, forced movement), status effects (buffs, debuffs, DoTs, HoTs). This is the largest and most important namespace for gameplay logic.

??? example "Tags_StateStatusCC.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Action States --
    +GameplayTagList=(Tag="State",DevComment="Broad action states used to gate/cancel abilities.")
    +GameplayTagList=(Tag="State.Dead",DevComment="Dead: blocks most actions.")
    +GameplayTagList=(Tag="State.InCombat",DevComment="In combat: combat rules/regen gating.")
    +GameplayTagList=(Tag="State.Casting",DevComment="Casting: can be interrupted/canceled.")
    +GameplayTagList=(Tag="State.Channeling",DevComment="Channeling: sustained ability in progress.")
    +GameplayTagList=(Tag="State.Blocking",DevComment="Actively blocking/guarding.")
    +GameplayTagList=(Tag="State.Evading",DevComment="Currently in evade/dodge animation.")
    +GameplayTagList=(Tag="State.Interruptible",DevComment="Current action can be interrupted.")
    +GameplayTagList=(Tag="State.Uninterruptible",DevComment="Current action cannot be interrupted (super armor).")
    +GameplayTagList=(Tag="State.Invulnerable",DevComment="Cannot take damage (short i-frame windows).")
    +GameplayTagList=(Tag="State.Untargetable",DevComment="Cannot be targeted (short windows).")
    +GameplayTagList=(Tag="State.Stealthed",DevComment="Stealth active.")
    +GameplayTagList=(Tag="State.Revealed",DevComment="Revealed: prevents stealth.")
    +GameplayTagList=(Tag="State.Staggered",DevComment="Currently in stagger/hit-react (poise broken).")
    +GameplayTagList=(Tag="State.Riposte",DevComment="In riposte window after successful parry.")
    +GameplayTagList=(Tag="State.ComboWindow",DevComment="Combo input window is currently open.")

    ; -- Crowd Control --
    +GameplayTagList=(Tag="CrowdControl",DevComment="Crowd control taxonomy (usually granted by effects).")

    ; Hard CC (loss of control)
    +GameplayTagList=(Tag="CrowdControl.Hard.Stun",DevComment="Stun: cannot act.")
    +GameplayTagList=(Tag="CrowdControl.Hard.Freeze",DevComment="Freeze: cannot act; often breaks on hit.")
    +GameplayTagList=(Tag="CrowdControl.Hard.Sleep",DevComment="Sleep: cannot act; often breaks on damage.")
    +GameplayTagList=(Tag="CrowdControl.Hard.Fear",DevComment="Fear: forced loss of control.")
    +GameplayTagList=(Tag="CrowdControl.Hard.Charm",DevComment="Charm: forced loss of control.")
    +GameplayTagList=(Tag="CrowdControl.Hard.Taunt",DevComment="Taunt: forced target selection.")
    +GameplayTagList=(Tag="CrowdControl.Hard.Petrify",DevComment="Petrify: turned to stone, cannot act.")

    ; Soft CC (partial restriction)
    +GameplayTagList=(Tag="CrowdControl.Soft.Slow",DevComment="Slow: reduced movement speed.")
    +GameplayTagList=(Tag="CrowdControl.Soft.Root",DevComment="Root: cannot move.")
    +GameplayTagList=(Tag="CrowdControl.Soft.Silence",DevComment="Silence: cannot cast.")
    +GameplayTagList=(Tag="CrowdControl.Soft.Disarm",DevComment="Disarm: cannot weapon attack.")
    +GameplayTagList=(Tag="CrowdControl.Soft.Blind",DevComment="Blind: reduced accuracy/targeting.")
    +GameplayTagList=(Tag="CrowdControl.Soft.Cripple",DevComment="Cripple: reduced attack speed.")

    ; Forced Movement
    +GameplayTagList=(Tag="CrowdControl.ForcedMovement.Knockback",DevComment="Push away with impulse.")
    +GameplayTagList=(Tag="CrowdControl.ForcedMovement.Knockup",DevComment="Launch upward (airborne).")
    +GameplayTagList=(Tag="CrowdControl.ForcedMovement.Pull",DevComment="Pull toward a point/source.")

    ; -- Status Effects --
    +GameplayTagList=(Tag="Status",DevComment="General buffs/debuffs/DoTs not strictly CC.")

    ; Buffs
    +GameplayTagList=(Tag="Status.Buff",DevComment="Positive status effects.")
    +GameplayTagList=(Tag="Status.Buff.Haste",DevComment="Increased movement speed.")
    +GameplayTagList=(Tag="Status.Buff.Might",DevComment="Increased damage output.")
    +GameplayTagList=(Tag="Status.Buff.Fortify",DevComment="Increased defense/damage reduction.")
    +GameplayTagList=(Tag="Status.Buff.Regeneration",DevComment="Health regen over time.")
    +GameplayTagList=(Tag="Status.Buff.Shield",DevComment="Absorb shield active.")
    +GameplayTagList=(Tag="Status.Buff.Berserk",DevComment="Increased damage, may reduce defense.")

    ; Debuffs
    +GameplayTagList=(Tag="Status.Debuff",DevComment="Negative status effects.")
    +GameplayTagList=(Tag="Status.Debuff.Vulnerable",DevComment="Takes increased damage.")
    +GameplayTagList=(Tag="Status.Debuff.Weaken",DevComment="Deals reduced damage.")
    +GameplayTagList=(Tag="Status.Debuff.Marked",DevComment="Marked for tracking/bonus interactions.")
    +GameplayTagList=(Tag="Status.Debuff.Exhausted",DevComment="Reduced stamina regen or max stamina.")
    +GameplayTagList=(Tag="Status.Debuff.Cursed",DevComment="Reduced healing received.")

    ; Damage Over Time
    +GameplayTagList=(Tag="Status.DamageOverTime",DevComment="Periodic damage effects (DoTs).")
    +GameplayTagList=(Tag="Status.DamageOverTime.Burn",DevComment="Fire-based DoT.")
    +GameplayTagList=(Tag="Status.DamageOverTime.Poison",DevComment="Toxin-based DoT.")
    +GameplayTagList=(Tag="Status.DamageOverTime.Bleed",DevComment="Physical laceration DoT.")
    +GameplayTagList=(Tag="Status.DamageOverTime.Corruption",DevComment="Shadow/void-based DoT.")

    ; Heal Over Time
    +GameplayTagList=(Tag="Status.HealOverTime",DevComment="Periodic healing effects (HoTs).")
    +GameplayTagList=(Tag="Status.HealOverTime.Regeneration",DevComment="Generic HoT.")
    ```

| Tag | Purpose |
|---|---|
| `State.*` | Broad gates -- `State.Dead` blocks everything, `State.Invulnerable` blocks damage |
| `CrowdControl.Hard.*` | Full loss of control -- used as blocking tags on abilities |
| `CrowdControl.Soft.*` | Partial restrictions -- applied as GE-granted tags that modify specific systems |
| `CrowdControl.ForcedMovement.*` | Physics impulses -- trigger root motion or launch characters |
| `Status.Buff.*` / `.Debuff.*` | Named status effects -- drive UI icons, tooltip text, and stacking behavior |
| `Status.DamageOverTime.*` | Periodic damage -- applied by GEs with `HasDuration` + `Period` |
| `Status.HealOverTime.*` | Periodic healing -- same pattern as DoTs |

---

## Team Tags

Faction allegiance and targeting rules. Used by AI for friend/foe decisions and by abilities to filter valid targets.

??? example "Tags_Team.ini"
    ```ini
    [/Script/GameplayTags.GameplayTagsSettings]
    ImportTagsFromConfig=True

    ; -- Team / Faction --
    +GameplayTagList=(Tag="Team",DevComment="Team/faction namespace for allegiance and targeting.")
    +GameplayTagList=(Tag="Team.Player",DevComment="Player team.")
    +GameplayTagList=(Tag="Team.Ally",DevComment="Allied NPCs (friendly, not player-controlled).")
    +GameplayTagList=(Tag="Team.Enemy",DevComment="Hostile team.")
    +GameplayTagList=(Tag="Team.Neutral",DevComment="Neutral: won't attack unless provoked.")

    ; -- Targeting Rules --
    +GameplayTagList=(Tag="Team.Target.Friendly",DevComment="Ability targets friendlies (heals/buffs).")
    +GameplayTagList=(Tag="Team.Target.Hostile",DevComment="Ability targets hostiles (damage/debuffs).")
    +GameplayTagList=(Tag="Team.Target.Self",DevComment="Ability targets self only.")
    +GameplayTagList=(Tag="Team.Target.All",DevComment="Ability targets all (AoE that hits everyone).")
    ```

| Tag | Purpose |
|---|---|
| `Team.Player` / `.Ally` / `.Enemy` / `.Neutral` | Faction identity -- granted on spawn, used by AI perception and targeting |
| `Team.Target.*` | Targeting filter -- set on abilities to control who they can hit |
