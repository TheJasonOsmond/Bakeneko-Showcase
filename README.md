<h1 align="center">🐾 Bakeneko 🐾</h1>

<p align="center">
  <b>Rogue-like Auto-Battler | Systems Showcase | In Development</b><br>
  A technical and design-focused public showcase documenting architecture, gameplay systems, and development progress.
</p>

<!-- <p align="center">
  <a href="YOUR_ITCH_IO_LINK_HERE">
    <img src="https://img.shields.io/badge/🎮%20Play%20on%20itch.io-FA5C5C?style=for-the-badge&logo=itch.io&logoColor=white" alt="Play on itch.io">
  </a>
  <br><br>
  <a href="YOUR_YOUTUBE_LINK_HERE">
    <img src="https://img.shields.io/badge/▶%20Watch%20Development%20Demo-FF0000?style=for-the-badge&logo=youtube&logoColor=white" alt="Watch Development Demo">
  </a>
</p> -->

---

## Gameplay Preview

<p align="center">
  <img src="MarketingMaterial/BN_Gameplay_Item.gif" width="32%" alt="Gameplay GIF 1">
  <img src="MarketingMaterial/BN_Gameplay_Party.gif" width="32%" alt="Gameplay GIF 2">
  <img src="MarketingMaterial/BN_Gameplay_Boss.gif" width="32%" alt="Gameplay GIF 3">
</p>

---

## Bakeneko — Project Overview

**Bakeneko** is a rogue-like auto-battler built around modular system design and emergent strategic gameplay, where players construct and maintain a deterministic “party engine” under continuously shifting constraints.

The player does not directly control combat units; instead, they assemble and evolve a systemic build that functions as a scoring machine, tested against increasingly complex rule-based encounters.

### Core Pillars

#### 1. Party Engine Construction (Build Crafting)
Players build a deterministic party system rather than a collection of units.
Synergies, positioning, evolution, artifacts, and item usage combine to form a composable scoring engine. The design focus is on engine diversity, not stat optimization.

#### 2. Rule-Based Encounter Puzzles
Encounters act as constraint layers on the engine rather than traditional enemy fights.
Each encounter modifies scoring rules via positioning limits, multipliers, caps, or structural distortions, forcing players to evaluate whether their build remains valid under shifting conditions.

#### 3. Adaptive Run Evolution (Instability System)
No build remains stable throughout a run.
Units evolve, stage rules shift, encounters escalate, and systemic modifiers introduce mid-run disruption, enforcing continuous adaptation rather than static optimization.

_Inspiration: Balatro + Super Auto Pets_

---

## Technical Highlight - Modular Action & Event Architecture

A modular, data-driven combat framework built around **Trigger → Condition → Behaviour** pipelines authored entirely through Unity’s inspector.  
  
### System Goal:  
Allow designers to create:  
- Status effects    
- Reactive abilities    
- Synergies    
- Buff / Debuff systems    
- Conditional combat logic    
  
**Without requiring effect-specific runtime code.**  

---
### Key Features  
  
- **Designer-Owned Content Pipeline** — Effects are authored through composable `ScriptableObject` blocks    
- **Decoupled Event Architecture** — Combat systems communicate through broadcasted actions rather than direct references    
- **Extensible Behaviour System** — New behaviours are implemented once and immediately become globally authorable    
- **Frame-Safe Action Queue** — Sequenced execution prevents re-entrant combat bugs    
- **Allocation-Conscious Runtime Design** — Struct-based `ActionContext` minimizes heap pressure    
- **Inspector Tooling** — Odin-powered validation and clean conditional authoring workflows    
  
---  
### Effect Data (what designers see)

Effects are designed as a set of:
  1. Trigger   - "when?"       
  2. Condition - "under what circumstances?"  
  3. Behaviour - "what happens?"              

The designer also has control over the animation played, stack and duration, and what to display on the UI. 

<p align="center">
  <img src="MarketingMaterial/BN_Design_EffectInspector.png" height="600" alt="Bakeneko Effect System Architecture Diagram">
</p>

---
### Why This Architecture Matters  
  
### Production Value:  
- Reduces programmer bottlenecks    
- Enables rapid design iteration    
- Supports scalable content expansion    
- Minimizes coupling between systems    
- Maintains deterministic combat sequencing    
  
### Engineering Value:  
- ScriptableObject data separation    
- Runtime state isolation    
- Event-driven system design    
- Extensibility-first architecture    
- Performance-conscious execution    
  
---  
  
### Architecture Overview

<p align="center">
  <img src="MarketingMaterial/BN_Design_EffectSystem.png" height="600" alt="Bakeneko Effect System Architecture Diagram">
</p>

---

### Selected Snippet — Action Queue Protocol

Every combat action follows a **register → fire → resolve** pipeline:

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

### Why This Matters

This protocol ensures:

- Deterministic chain resolution
- Safe cascading reactions
- Shared execution model for both external and internal events
- Predictable sequencing across combat systems

---

### Engineering Outcomes
This system shifts combat implementation from **Hardcoded one-off ability logic** to  **Composable systemic gameplay architecture**

- Reduced programmer dependency for effect creation
- Enabled designer-authored combat content
- Minimized runtime allocations on Hot Path
- Created scalable foundations for:
    - Abilities
    - Status systems
    - Item synergies
    - Encounter modifiers
    - Future game modes

**[Read Full Technical Breakdown](/effect-system.md)**
