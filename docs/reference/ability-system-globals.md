
# AbilitySystemGlobals

`UAbilitySystemGlobals` is the singleton that holds global GAS configuration and provides factory methods for project-specific customization. As of UE 5.5, most configuration has moved to `UGameplayAbilitiesDeveloperSettings` (accessible via **Project Settings > Gameplay Abilities Settings**), but `UAbilitySystemGlobals` remains the place to override runtime behavior.

For concepts, see [Project Setup](../getting-started/project-setup.md).

---

## InitGlobalData

`InitGlobalData()` must be called once before any GAS usage. It is called automatically by `IGameplayAbilitiesModule::Get().GetAbilitySystemGlobals()` on first access.

**What it does (in order):**

1. Calls `InitGlobalTags()` — resolves configured tag names into `FGameplayTag` instances
2. Calls `InitTargetDataScriptStructCache()` — builds the net serialization cache
3. Loads the Global Curve Table (if configured)
4. Loads the Global Attribute MetaData Table (if configured)
5. Calls `InitAttributeDefaults()` — loads attribute default curve tables and creates the `FAttributeSetInitter`
6. Creates the `UGameplayCueManager` singleton

!!! note "Legacy requirement removed"
    In UE 4.x you had to manually call `UAbilitySystemGlobals::Get().InitGlobalData()` from an `AssetManager` subclass. Since UE 5.0+, the module calls this automatically. You no longer need to do this yourself.

---

## UGameplayAbilitiesDeveloperSettings

As of UE 5.5, this `UDeveloperSettings` class is the preferred way to configure GAS. It writes to the same config section (`/Script/GameplayAbilities.AbilitySystemGlobals`) for backwards compatibility.

Access in **Editor**: Project Settings > Gameplay Abilities Settings.
Access in **Code**: `GetDefault<UGameplayAbilitiesDeveloperSettings>()`.

### Gameplay Settings

| Property | Type | Default | Description |
|---|---|---|---|
| `AbilitySystemGlobalsClassName` | `FSoftClassPath` | `UAbilitySystemGlobals` | The class to instantiate as the globals singleton. Override for project-specific subclassing. |
| `bUseDebugTargetFromHud` | `bool` | `false` | Use the HUD's debug target for `ShowDebug AbilitySystem` instead of the ASC's own debug target |
| `PredictTargetGameplayEffects` | `bool` | `true` | Allow clients to predict GE application on non-local targets |
| `ReplicateActivationOwnedTags` | `bool` | `true` | Replicate tags granted by ability activations. Disable only for legacy compatibility. |
| `GlobalCurveTableName` | `FSoftObjectPath` | *None* | Global curve table for `FScalableFloat` default lookups |

### Attribute Settings

| Property | Type | Default | Description |
|---|---|---|---|
| `GlobalAttributeSetDefaultsTableNames` | `TArray<FSoftObjectPath>` | *Empty* | Curve tables used to initialize attributes to default values, keyed by Name/Level |
| `GlobalAttributeMetaDataTableName` | `FSoftObjectPath` | *None* | DataTable defining attribute metadata (min/max values, stacking rules). Currently unused by engine. |

### Gameplay Cue Settings

| Property | Type | Default | Description |
|---|---|---|---|
| `GlobalGameplayCueManagerClass` | `FSoftClassPath` | `UGameplayCueManager` | Class to use for the cue manager singleton |
| `GlobalGameplayCueManagerName` | `FSoftObjectPath` | *None* | Optional: reference to a specific Blueprint instance of your cue manager class |
| `GameplayCueNotifyPaths` | `TArray<FString>` | *Empty* | Content paths to scan for GameplayCueNotify assets. These are the "always loaded" set. |

### Gameplay Effect Settings

| Property | Type | Default | Description |
|---|---|---|---|
| `bAllowGameplayModEvaluationChannels` | `bool` | `false` | Enable [evaluation channels](modifier-formula.md) for modifier aggregation |
| `DefaultGameplayModEvaluationChannel` | `EGameplayModEvaluationChannel` | `Channel0` | Default channel when channels are enabled |
| `GameplayModEvaluationChannelAliases[10]` | `FName[10]` | *Empty* | Named aliases for each channel. Only channels with aliases are valid (except Channel0). |

### Activation Failure Tags

These tags are sent as events when ability activation fails, allowing UI/feedback systems to react.

| Property | Tag Sent When... |
|---|---|
| `ActivateFailCanActivateAbilityTag` | `CanActivateAbility()` returns false |
| `ActivateFailCooldownTag` | Ability is on cooldown |
| `ActivateFailCostTag` | Ability cannot pay its cost |
| `ActivateFailTagsBlockedTag` | Ability is blocked by current tags |
| `ActivateFailTagsMissingTag` | Required tags are missing |
| `ActivateFailNetworkingTag` | Net execution policy prevents activation |

### Advanced

| Property | Type | Default | Description |
|---|---|---|---|
| `MinimalReplicationTagCountBits` | `int32` | `5` | Bits used for tag count in `FMinimalReplicationTagCountMap::NetSerialize` |
| `GameplayTagResponseTableName` | `FSoftObjectPath` | *None* | Asset reference to a `UGameplayTagReponseTable` for automatic tag-based responses |

---

## UGameplayAbilitiesEditorDeveloperSettings

Editor-only settings backed by CVars (per-user, not committed to source control).

| Property | CVar | Default | Description |
|---|---|---|---|
| `bIgnoreCooldowns` | `AbilitySystem.IgnoreCooldowns` | `false` | Skip cooldown checks during development |
| `bIgnoreCosts` | `AbilitySystem.IgnoreCosts` | `false` | Skip cost checks during development |
| `AbilitySystemGlobalScaler` | `AbilitySystem.GlobalAbilityScale` | `1.0` | Global rate/duration scaler for ability tasks |
| `DebugDrawMaxDistance` | `AbilitySystem.DebugDrawMaxDistance` | `2048.0` | Max distance for debug draw visualizations |

---

## Subclassing UAbilitySystemGlobals

Override the globals class for project-specific behavior. Set your subclass in Project Settings > Gameplay Abilities Settings > **Ability System Globals Class**.

### Common Overrides

| Virtual Method | Purpose |
|---|---|
| `AllocAbilityActorInfo()` | Return a project-specific `FGameplayAbilityActorInfo` subclass with additional cached data |
| `AllocGameplayEffectContext()` | Return a project-specific `FGameplayEffectContext` subclass with additional fields (e.g., critical hit flag) |
| `GlobalPreGameplayEffectSpecApply()` | Global hook before any GE spec is applied — use for logging, analytics, or global modifiers |
| `GetGameplayCueManager()` | Return a custom `UGameplayCueManager` subclass |
| `GetGameplayCueNotifyPaths()` | Customize which content paths are scanned for gameplay cue notifies |
| `InitGlobalTags()` | Extend global tag initialization |

### Example: Custom Effect Context

```cpp
// MyAbilitySystemGlobals.h
UCLASS()
class UMyAbilitySystemGlobals : public UAbilitySystemGlobals
{
    GENERATED_BODY()
public:
    virtual FGameplayEffectContext* AllocGameplayEffectContext() const override;
};

// MyAbilitySystemGlobals.cpp
FGameplayEffectContext* UMyAbilitySystemGlobals::AllocGameplayEffectContext() const
{
    return new FMyGameplayEffectContext();
}
```

---

## Key Static Methods

| Method | Description |
|---|---|
| `Get()` | Returns the singleton instance, creating it if necessary |
| `GetAbilitySystemComponentFromActor()` | Finds an ASC on an actor — checks `IAbilitySystemInterface` first, then falls back to component search |
| `DeriveGameplayCueTagFromAssetName()` | Generates a `GameplayCue.*` tag from an asset's name |
| `NonShipping_ApplyGlobalAbilityScaler_Rate()` | Applies the global ability scale CVar to a rate value (dev builds only) |
| `NonShipping_ApplyGlobalAbilityScaler_Duration()` | Applies the global ability scale CVar to a duration value (dev builds only) |

## Related Pages

- [Project Setup](../getting-started/project-setup.md) -- where AbilitySystemGlobals configuration is typically performed
- [Gameplay Cue Manager](../gameplay-cues/cue-manager.md) -- the cue manager singleton created during InitGlobalData
- [Blueprint Library](blueprint-library.md) -- static helper functions that interact with the globals singleton
