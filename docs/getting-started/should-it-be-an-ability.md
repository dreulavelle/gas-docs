---
icon: material/help-rhombus
---

# Should It Be an Ability?

This is the question you'll ask yourself every time you add a new action to your game. Not everything needs to be a GAS ability — but more things should be abilities than you'd initially think. This page gives you a concrete decision framework so you're not guessing.

## The Decision Framework

Ask yourself these six questions about the action. The more "yes" answers, the stronger the case for making it a GAS ability.

| Question | If Yes → Ability | If No → Maybe Not |
|---|---|---|
| **Should crowd control block it?** (stun, freeze, silence) | Tag-based blocking is free | You'd skip the CC check anyway |
| **Does it have a resource cost?** (mana, stamina, energy) | Built-in cost checking | No cost to manage |
| **Does it have a cooldown?** | Built-in cooldown system | No cooldown needed |
| **Do other abilities interact with it?** (cancel, combo, block) | Tag queries handle this | It's independent of other actions |
| **Does it need client-side prediction?** | GAS prediction built-in | Server-only or cosmetic |
| **Should AI be able to use it?** | ASC activation works for AI and players identically | Player-only action |

**Score it:**

- **4-6 yes**: Definitely an ability
- **2-3 yes**: Probably an ability — the GAS infrastructure will save you work
- **0-1 yes**: Probably not an ability — direct implementation is simpler

## The Jump Example

Jump is the perfect case study because it sits right on the line. Let's compare both approaches.

### Jump as Regular Input

```
Player presses Jump
    → Character::Jump()
        → if (bIsStunned) return;          // manual check
        → if (bIsRooted) return;           // manual check
        → if (Stamina < JumpCost) return;  // manual check
        → if (bIsOnCooldown) return;       // manual timer
        → Stamina -= JumpCost;             // manual subtract
        → StartCooldownTimer(0.5);         // manual cooldown
        → LaunchCharacter(JumpForce);
```

Every check is hand-written. Every new status effect that should block jumping requires finding this code and adding another `if` statement. When you add a new CC type six months from now, you have to remember every action that should be blocked by it.

### Jump as a GAS Ability

```
Player presses Jump
    → ASC::TryActivateAbility(GA_Jump)
        → [GAS checks Activation Blocked Tags: CrowdControl.Hard, CrowdControl.Root]
        → [GAS checks Cost GE: has enough Stamina?]
        → [GAS checks Cooldown: Cooldown.Ability.Jump tag present?]
        → ActivateAbility()
            → LaunchCharacter(JumpForce)
            → EndAbility()
```

The stun check, root check, stamina cost, and cooldown are all configuration — no code. When you add a new CC type, every ability that blocks on `CrowdControl.Hard` automatically respects it. Zero changes to existing abilities.

!!! tip "The maintenance multiplier"
    The GAS version looks like more setup for a single action. But that setup pays for itself the moment you have 5+ actions that all need the same CC/cost/cooldown checks. The maintenance cost of the manual approach grows linearly with every action and every status effect. The GAS approach stays flat.

## Common Actions: Ability or Not?

Here's how typical game actions shake out:

| Action | Ability? | Reasoning |
|---|---|---|
| **Basic Attack** | Yes | Costs, cooldown, CC-blocked, montage, damage pipeline, AI uses it |
| **Dodge / Roll** | Yes | Costs stamina, CC-blocked, grants i-frames (tag), has cooldown |
| **Block / Parry** | Yes | CC-blocked, interacts with incoming damage, other abilities check for it |
| **Jump** | Usually yes | CC-blocked, often costs stamina, AI needs it for navigation |
| **Sprint** | Depends | See [Gray Areas](#gray-areas) below |
| **Crouch** | Depends | See [Gray Areas](#gray-areas) below |
| **Interact** (loot, talk) | Depends | See [Gray Areas](#gray-areas) below |
| **Open Inventory** | No | Pure UI action — not blocked by CC, no cost, no cooldown |
| **Camera Look** | No | Raw input — always available, never blocked, no gameplay interaction |
| **Pause** | No | Meta-action — outside the gameplay simulation entirely |

## Gray Areas

Some actions are genuinely ambiguous. The right answer depends on your specific game.

### Sprint

**As an ability:** Makes sense if sprinting costs stamina per second (Infinite duration GE that drains stamina), can be interrupted by CC, and AI needs to sprint in combat. The ability grants a `State.Sprinting` tag that other systems can query.

**As regular input:** Makes sense if sprint is just "hold shift to go faster" with no cost, no interaction with other systems, and no CC check. A simple movement component toggle is simpler.

**The deciding question:** Would getting stunned mid-sprint need to cancel it? If yes, make it an ability.

### Crouch

**As an ability:** Makes sense in tactical shooters where crouching affects accuracy (modifies a Spread attribute), interacts with abilities (can't crouch while knocked down), or grants a tag (`State.Crouching`) that other effects check for.

**As regular input:** Makes sense if it's purely a movement state with no gameplay interactions beyond collision changes.

**The deciding question:** Does any other system in your game need to know you're crouching? If yes, a tag from an ability is cleaner than a boolean.

### Interact (Loot / Talk / Activate)

**As an ability:** Makes sense if you want CC to block interaction (can't loot while stunned), if interaction has a cast time or channel, or if interaction should be cancelled by taking damage.

**As regular input:** Makes sense if interaction is instant and should always work regardless of character state.

**The deciding question:** Should a stun prevent the player from picking up items? If yes, ability.

!!! info "There's no wrong answer"
    For gray area actions, both approaches work. The key is being consistent within your project. If most of your actions are abilities, make the gray area ones abilities too — consistency reduces cognitive load. If you only have 2-3 abilities total, keeping gray area actions as regular inputs avoids over-engineering.

## The Rule of Thumb

Here's the simplest heuristic:

!!! quote "The `if (isStunned) return;` Rule"
    **If you'd ever write an `if (isStunned) return;` check for this action, it should probably be a GAS ability.**

    That single check is a sign that the action participates in your game's status/interaction system. And if it interacts with stun, it probably also interacts with (or will eventually interact with) silence, root, fear, sleep, and every other CC type you add. GAS handles all of that with tag configuration. Manual checks don't scale.

## What About Passive Effects?

Not everything that uses GAS needs to be an Ability. This distinction trips people up.

**Abilities are for actions** — things a character *does*. Press a button, something happens.

**Effects can exist independently** — things that *happen to* a character without an ability triggering them. Examples:

- **Environmental damage zones** — A fire floor applies a burning GE directly to any ASC that enters its overlap. No ability involved. The zone just calls `ApplyGameplayEffectToTarget` on the victim's ASC.

- **Equipment stat bonuses** — Equipping a sword grants an Infinite GE that adds +10 to an Attack attribute. Removing it removes the GE. No ability.

- **Aura effects** — A healing totem applies a periodic healing GE to all allies within range. The totem manages the effect application, not an ability.

- **Buff pickups** — Walk over a speed boost, it applies a Duration GE that modifies MoveSpeed for 10 seconds.

- **Status effects from other abilities** — A frost mage's ice bolt ability applies a slow GE to the target. The *ability* belongs to the caster, but the *effect* lives on the target. The target doesn't need an ability to receive the effect — just an ASC.

The rule: if there's no player-initiated action, there's no ability. Effects, tags, and attribute modifications can all happen without an ability as the source. The ASC is the entry point for applying effects — abilities are just one of many things that can apply them.

??? question "When does a passive NEED to be an ability?"
    When the passive has activation conditions that GAS should manage. For example, a "Second Wind" passive that triggers when Health drops below 20% — this is best implemented as a passive ability that listens for an attribute change event. The ability framework gives you activation conditions, tag requirements, and cooldowns for free. A raw effect can't conditionally activate itself.

    See [Gameplay Abilities Overview](../core-concepts/gameplay-abilities-overview.md) for more on passive vs. active abilities.

## Putting It All Together

Here's the mental checklist to run for every new action:

1. **Does it need CC blocking, costs, cooldowns, or prediction?** If 2+ apply, it's an ability.
2. **Does it interact with other abilities?** (cancels, combos, blocks) If yes, ability.
3. **Would you write `if (isStunned) return;`?** If yes, ability.
4. **Is it a passive thing that happens TO the character?** If yes, it's probably just an effect — no ability needed.
5. **Is it pure UI or meta?** (inventory, pause, camera) Not an ability.
6. **Still unsure?** Make it an ability. The overhead is small, and you can always simplify later. It's much harder to migrate a complex manual system *into* GAS than to start with GAS and remove it if unneeded.

## What's Next

You now have the foundation (from [Project Setup](project-setup.md)), a working ability (from [Your First Ability](your-first-ability.md)), and a framework for deciding what should and shouldn't use GAS. Time to go deeper.

Head to [Core Concepts](../core-concepts/index.md) to learn each piece of GAS in detail — the Ability System Component, Gameplay Tags, Attributes, Effects, Abilities, and Cues. Each page builds on the mental model and practical foundation you've established here.
