# ⚡ Effect & Action System — Technical Breakdown

> **[← Back to README](./README)**
> **Category:** Data-Driven Combat Framework · Event Architecture · Ability Pipeline

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)  
2. [Runtime Architecture](#2-runtime-architecture)  
3. [Trigger Wiring — TriggerRuntime](#3-trigger-wiring--triggerruntime)  
4. [The Action Queue — Sequenced Execution](#4-the-action-queue--sequenced-execution)  
5. [ActionContext as a Struct — Allocation Strategy](#5-actioncontext-as-a-struct--allocation-strategy)  
6. [Key Engineering Decisions](#6-key-engineering-decisions)  
7. [Extensibility](#7-extensibility)  

---

## 1. Design Philosophy

The central goal was to make effects **entirely authorable by designers** without touching code. Every effect — a regenerating shield, a stacking poison, a reactive counter-attack — is expressed as a `ScriptableObject` (`EffectData`) containing an array of `TriggerBlock` entries.

Each `TriggerBlock` follows a strict **when / if / then** structure:

```
TriggerBlock
  ├── TriggerType        "when does this fire?"       (OnAttack, OnEffectApplied, OnRoundStart…)
  ├── ConditionNode      "under what circumstances?"  (composable condition tree)
  └── EffectBehaviour[]  "what happens?"              (serialised, subclass-selectable actions)
```

`EffectBehaviour` uses `[SerializeReference]` with a `[SubclassSelector]` Odin drawer — designers pick from a dropdown of all available behaviour types and configure them inline. A new behaviour type is a single class implementing the interface; it becomes available to every effect without any registration or factory code.

The full effect set — every trigger, condition, and consequence — lives in the asset. The runtime only reads it. This separation means **designers own the design space entirely**, and the runtime system never needs to change to accommodate new effect types.

Custom Odin Inspector tooling on `EffectData` enforces authoring constraints, surfaces relevant fields conditionally (e.g. duration fields hidden when `HasDuration` is false), and keeps the inspector clean regardless of trigger set complexity.

---

## 2. Runtime Architecture

```
EffectData (ScriptableObject) — immutable blueprint, never modified
  └── TriggerBlock[] EffectSet

EffectManager
└── Effect (runtime instance)
      └── TriggerRuntime (runtime instance)
```

`EffectData` is a shared, read-only asset. Any number of live `Effect` instances can reference the same `EffectData` simultaneously with no cross-entity contamination — all runtime state (stacks, duration, counter) lives exclusively in `Effect`.

`Effect` is a **plain C# class**, not a MonoBehaviour. It exists precisely as long as the owning `EffectManager` holds a reference — no Unity lifecycle overhead, no `Update()` cost when idle, no dependency on scene hierarchy.

---

## 3. Trigger Wiring — `TriggerRuntime`

When an effect becomes active, `Effect.Activate()` instantiates one `TriggerRuntime` per `TriggerBlock`. Each runtime classifies its trigger as **internal** (reacting to the effect's own lifecycle) or **external** (reacting to global combat events via the entity's action listener bus):

```csharp
public void Start()
{
    if (IsExternalTrigger())
    {
        // Routes through the entity's EffectAbilityManager action bus
        effectable.EffectManager.AddActionListener(_block.GetActionType(), _listener);
    }
    else
    {
        switch (_block.Trigger)
        {
			case TriggerType.OnThisEffectApplied:        
				_effect.OnEffectApplied    += _listener; 
				break;
            case TriggerType.OnThisEffectStackChange:    
	            _effect.OnStacksChanged    += _listener; 
	            break;
            case TriggerType.OnThisEffectDurationChange: 
	            _effect.OnDurationChanged  += _listener; 
	            break;
            // ...
        }
    }
}
```

`Stop()` mirrors every subscription exactly on deactivation. This is made reliable by a deliberate allocation decision: **the delegate is created once in the constructor and stored** as a field.

```csharp
public TriggerRuntime(TriggerBlock block, Effect effect)
{
    _block = block;
    _effect = effect;
    _listener = OnTriggerAction; // one stable reference — subscribe and unsubscribe use the same instance
}
```

Using an inline lambda (`_listener = ctx => _block.OnTrigger(...)`) would allocate a new delegate object on every `Start()` call _and_ silently break `Stop()` — C# event unsubscription requires reference equality, and a new lambda is a new object. Storing the delegate explicitly eliminates this class of bug and avoids the allocation.

---

## 4. The Action Queue — Sequenced Execution

Effects don't react immediately when triggered. Every reaction routes through the `AbilityQueue`, a frame-safe pipeline ensuring cascading events resolve in a defined, deterministic order.

### The Register → Fire → Notify Protocol

```csharp
/// Action Context Struct
public readonly void Queue()  
{  
	// 1. snapshot queue depth before any entries arrive
    abilityQueue.RegisterAction(ActionID);      
    
    // 2. fire — listeners enqueue their ability entries
    ActionContextEvent.TriggerAction(this);     
    
    // 3. done — queue checks delta and advances if needed
    abilityQueue.ActionContextQueued(ActionID); 
}
```

`RegisterAction` snapshots the current queue depth. After `TriggerAction` fires, `ActionContextQueued` compares the new depth to the snapshot:

- **No change** → no reactions were queued; the action completes immediately.
- **Depth increased** → reactions are pending; the queue waits for them to drain before resolving the parent action.

This gives every action composable sequencing guarantees without any action needing to know about its dependents. The depth-delta check is the only coordination mechanism needed.

### Internal Events Use the Same Protocol

Internal effect events (stack changes, duration ticks, expiry) use `QueueInternalEffect`, which applies the identical three-step pattern:

```csharp
public static void QueueInternalEffect(ActionType type, Action<ActionContext> internalTrigger, Effect effect)
{
    ActionContext context = new(type) { Effect = effect };

    abilityQueue.RegisterAction(context.ActionID);
    internalTrigger.Invoke(context);
    abilityQueue.ActionContextQueued(context.ActionID);
}
```

A stack reduction from a poison tick and an attack from a player input both participate in the same queue with identical ordering guarantees. There is no special execution path for effect-internal events.

### Frame Deferral

`Advance()` defers to the next `Update()` tick via `_advancePending` rather than executing inline. This is a deliberate tradeoff: one frame of latency per reaction step, in exchange for eliminating re-entrant execution within a single frame — a common source of ordering bugs and stack overflows when synchronous event chains call back into their own triggers.

---

## 5. `ActionContext` as a Struct — Allocation Strategy

`ActionContext` is a **value type**. In a system that creates one context per action, per trigger, and per internal effect event, using a class would generate continuous heap pressure and GC activity during combat — precisely when frame consistency matters most.

```csharp
public struct ActionContext
{
    public readonly int ActionID { get; }
    public readonly ActionType Type { get; }
    public IActor Source;
    public Effect Effect;
    public int StackCount;
    public int RoundNumber;
    // ... targeting fields (PartyMember Target0–4)
}
```

Because `ActionContext` is a struct, passing it through the trigger chain costs nothing beyond a stack copy. Where behaviours need to write results back — damage values, selected targets — the context is passed by `ref`, enabling downstream communication without allocating a return object:

```csharp
public void ExecuteAllBehaviours(Effect effect, ref ActionContext ctx)
{
    foreach (var behav in Behaviours)
        behav.Execute(effect, ctx);
}
```

The context is a lightweight, frame-scoped data carrier: created on the stack, threaded through the system, discarded with no GC cost.

> **Note on targeting fields:** `ActionContext` stores up to five `PartyMember` target slots as direct fields rather than a `PartyMember[]` array. This avoids a heap allocation per context for what is always a small, fixed-size set of targets in a turn-based game with a bounded party size.

---

## 6. Key Engineering Decisions

|Decision|Rationale|
|---|---|
|`Effect` as a plain C# class|No MonoBehaviour, no `Update()`, no per-frame cost when idle|
|`EffectData` never mutated at runtime|Read-only ScriptableObject safely shared across all live instances|
|Stable delegate stored in `TriggerRuntime`|Prevents silent unsubscription failures; eliminates one allocation per trigger activation|
|`ActionContext` as a struct|Eliminates per-action heap allocation during combat; `ref` passing enables downstream data flow|
|`EffectManager` implements `IDisposable`|Clears all effects and disposes the ability manager on entity teardown; prevents listener leaks|
|`_advancePending` deferral|Single-frame deferral prevents re-entrancy without locks or coroutines|
|`[SerializeReference]` + `[SubclassSelector]`|Polymorphic inspector authoring with zero registration overhead; new types auto-available|
|Fixed target fields vs. array|Avoids per-context array allocation for a known bounded set|

---

## 7. Extensibility

The architecture is structured so that new content touches exactly one layer:

- **New trigger type** — add an enum value, handle it in `TriggerRuntime.Start/Stop` and `GetActionType`. No other files change.
- **New behaviour** — implement `EffectBehaviour`. Immediately appears in the inspector dropdown for every existing and future effect.
- **New condition** — extend `ConditionNode`. All existing effects gain access automatically.
- **New stacking rule** — add an enum value and a case in `EffectManager.ApplyStacksToExisting`.

No subclassing per effect. No factory registration. No runtime type switching outside the designated dispatch points.
