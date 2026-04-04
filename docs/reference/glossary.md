
# Glossary

Alphabetical definitions of GAS-specific terms. For acronym expansions, see [Acronyms](acronyms.md).

---

**Ability System Component (ASC)**
:   `UAbilitySystemComponent` — the `UActorComponent` that serves as the central hub for GAS. Owns granted abilities, active effects, attribute sets, and tag state. See [ASC](../core-concepts/ability-system-component.md).

**Ability Task**
:   `UAbilityTask` — a latent action that runs within the scope of an active `UGameplayAbility`. Ends when the ability ends. Blueprint-exposable via output delegates. See [Ability Tasks](../gameplay-abilities/ability-tasks.md).

**Active Gameplay Effect**
:   `FActiveGameplayEffect` — the runtime wrapper for a `UGameplayEffect` that has been applied to an ASC. Tracks remaining duration, stacks, and inhibition state.

**Active Gameplay Effect Handle**
:   `FActiveGameplayEffectHandle` — a lightweight handle (int32) used to reference an active effect without holding a pointer. Safe to store; becomes invalid when the effect is removed.

**Aggregator**
:   `FAggregator` — internal engine struct that collects all modifiers targeting a single attribute and evaluates them via the [modifier formula](modifier-formula.md). One aggregator exists per attribute per ASC.

**Async Ability Task**
:   `UAbilityAsync` — a latent task similar to `UAbilityTask` but usable outside of abilities (e.g., in UMG widgets, AI). Does not require an active ability. See [Async Ability Tasks](../gameplay-abilities/async-ability-tasks.md).

**Attribute**
:   A single numeric property (float) within a `UAttributeSet`, wrapped in `FGameplayAttributeData`. Attributes are the only values that Gameplay Effects can modify.

**Attribute Set**
:   `UAttributeSet` — a `UObject` that defines a group of related attributes. Registered with an ASC. See [Attributes and Attribute Sets](../core-concepts/attributes-and-attribute-sets.md).

**CDO (Class Default Object)**
:   The singleton template instance of a UClass. GAS uses CDOs for abilities with `InstancedPerExecution` or `NonInstanced` instancing policies — the CDO is shared across all activations.

**Cooldown Effect**
:   A `UGameplayEffect` with `HasDuration` used as an ability's cooldown. The ability checks for the cooldown tag before activation. See [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md).

**Cost Effect**
:   A `UGameplayEffect` (typically `Instant`) applied when an ability commits. Deducts resource attributes (mana, stamina, etc.).

**Dynamic Gameplay Effect**
:   A `UGameplayEffect` created at runtime via `NewObject<UGameplayEffect>()` rather than a data asset. Useful for procedural modifiers. See [Dynamic Effects](../gameplay-effects/dynamic-effects.md).

**Evaluation Channel**
:   `EGameplayModEvaluationChannel` — an optional channel that allows modifiers to be evaluated in separate passes. The output of one channel feeds as the base value into the next. Disabled by default; enabled via Developer Settings.

**Execution Calculation (ExecCalc)**
:   `UGameplayEffectExecutionCalculation` — a C++ class that can read multiple attributes and output multiple modifier results in a single pass. Used for complex operations like damage formulas. See [Execution Calculations](../gameplay-effects/execution-calculations.md).

**FGameplayAbilitySpec**
:   The runtime record of a granted ability. Contains the ability CDO/instance, level, input ID, activation state, and the granting GE handle (if any).

**FGameplayAbilitySpecHandle**
:   A lightweight handle used to reference a granted ability without holding a pointer.

**FGameplayAttributeData**
:   The struct wrapping a single attribute value. Contains `BaseValue` (before modifiers) and `CurrentValue` (after modifiers are evaluated).

**FGameplayEffectContext**
:   Runtime context passed with a GE application — instigator, causer, hit result, etc. Can be subclassed via `UAbilitySystemGlobals::AllocGameplayEffectContext()`.

**FGameplayEffectSpec**
:   The runtime instance created when a `UGameplayEffect` is about to be applied. Contains the GE definition plus captured context: level, source/target tags, SetByCaller magnitudes, and modifier snapshots.

**FGameplayTag**
:   A single hierarchical tag registered in the engine's tag system (e.g., `Status.Debuff.Stun`). Lightweight (stored as `FName` internally). See [Gameplay Tags](../core-concepts/gameplay-tags.md).

**FGameplayTagContainer**
:   An unordered set of `FGameplayTag` values. Used throughout GAS for tag queries, requirements, and grants.

**FPredictionKey**
:   A key generated on the client to associate predicted actions (ability activations, GE applications) with their server-confirmed counterparts. See [Prediction](../networking/prediction.md).

**Gameplay Cue**
:   A cosmetic-only notification (VFX, SFX, camera shake) triggered by tags matching `GameplayCue.*`. Never affects gameplay state. See [Gameplay Cues](../core-concepts/gameplay-cues-overview.md).

**Gameplay Effect**
:   `UGameplayEffect` — a data-only blueprint that defines attribute modifiers, tag grants, duration, stacking, and more. The primary data pipeline of GAS. See [Gameplay Effects](../core-concepts/gameplay-effects-overview.md).

**Gameplay Effect Component (GEC)**
:   `UGameplayEffectComponent` — a modular behavior object that lives within a `UGameplayEffect` (UE 5.3+). Replaces the monolithic tag/ability/immunity fields with composable components. See [GE Components](../gameplay-effects/ge-components.md).

**Granted Tag**
:   A Gameplay Tag added to the target's owned tag set while a duration or infinite GE is active. Removed when the GE expires or is removed.

**I-frames (Invulnerability Frames)**
:   A brief window during an action (like a dodge roll) where the character cannot take damage. Typically implemented by granting `State.Invulnerable` via Activation Owned Tags, then checking for that tag in the damage pipeline. See [Dodge Roll Example](../examples/dodge-roll.md).

**Inhibited Effect**
:   An active GE whose ongoing tag requirements are not met. It remains applied but dormant -- its modifiers and granted tags are not active. Becomes active again when requirements are satisfied.

**Meta Attribute**
:   An attribute used as a transient calculation channel (e.g., `IncomingDamage`) that is never replicated or persisted. Consumed immediately in `PostGameplayEffectExecute` and reset to zero. See [Damage Pipeline](../patterns/damage-pipeline.md).

**Modifier**
:   A single attribute modification within a GE — defined by an attribute, operation (`EGameplayModOp`), and magnitude. See [Modifiers](../gameplay-effects/modifiers.md).

**Modifier Magnitude Calculation (MMC)**
:   `UGameplayModifierMagnitudeCalculation` — a C++ class that computes a single modifier's magnitude by reading attribute values and tags at runtime. See [Magnitude Calculations](../gameplay-effects/magnitude-calculations.md).

**Net Execution Policy**
:   `EGameplayAbilityNetExecutionPolicy` — controls where an ability's logic runs: `LocalPredicted`, `LocalOnly`, `ServerInitiated`, or `ServerOnly`. See [Net Execution Policies](../networking/net-execution-policies.md).

**Prediction Key**
:   See *FPredictionKey*.

**Prediction Window**
:   A scope during which the client predicts the outcome of an action before the server confirms or rejects it. Created with `FScopedPredictionWindow`. See [Prediction](../networking/prediction.md).

**Net Relevancy**
:   Whether an actor is close enough to a player to be replicated. Actors outside a player's net cull distance are not replicated, which affects ASC state visibility for distant actors.

**Replication Mode**
:   `EGameplayEffectReplicationMode` — controls what GE data is replicated to which clients: `Full`, `Mixed`, or `Minimal`. See [Replication Modes](../networking/replication-modes.md).

**Scalable Float**
:   `FScalableFloat` — a float value optionally multiplied by a curve table row evaluated at a given level. Used for magnitudes, durations, periods, and cooldowns. See [Scalable Floats](scalable-float.md).

**SetByCaller**
:   A magnitude type where the value is set on the `FGameplayEffectSpec` at runtime using a Gameplay Tag as a key. See [SetByCaller](../gameplay-effects/set-by-caller.md).

**Spec Handle**
:   See *FGameplayAbilitySpecHandle* or *FActiveGameplayEffectHandle* depending on context.

**Stacking**
:   The rules governing what happens when multiple instances of the same GE definition are applied: `AggregateBySource`, `AggregateByTarget`, or no stacking. See [Stacking](../gameplay-effects/stacking.md).

**Target Data**
:   `FGameplayAbilityTargetData` — polymorphic struct carrying targeting results (hit results, actor lists, locations). Replicated from client to server for confirmed targeting. See [Target Data](../gameplay-abilities/targeting/target-data.md).

## Related Pages

- [Acronyms](acronyms.md) -- quick-reference table of GAS abbreviations (ASC, GE, GA, etc.)
- [Getting Started](../getting-started/index.md) -- recommended starting point for newcomers to GAS
