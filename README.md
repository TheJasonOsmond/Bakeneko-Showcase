<h1 align="center">🐾 Bakeneko 🐾</h1>

<p align="center">
  <b>Rogue-like Auto-Battler | Systems Showcase | In Development</b><br>
  A technical and design-focused public showcase documenting architecture, gameplay systems, and development progress.
</p>

<p align="center">
  <a href="YOUR_ITCH_IO_LINK_HERE">
    <img src="https://img.shields.io/badge/🎮%20Play%20on%20itch.io-FA5C5C?style=for-the-badge&logo=itch.io&logoColor=white" alt="Play on itch.io">
  </a>
  <br><br>
  <a href="YOUR_YOUTUBE_LINK_HERE">
    <img src="https://img.shields.io/badge/▶%20Watch%20Development%20Demo-FF0000?style=for-the-badge&logo=youtube&logoColor=white" alt="Watch Development Demo">
  </a>
</p>

---

## 📸 Gameplay Preview

<p align="center">
  <img src="MarketingMaterial/GameplayGif1.gif" width="400" alt="Gameplay GIF 1">
  <img src="MarketingMaterial/GameplayGif2.gif" width="400" alt="Gameplay GIF 2">
</p>

---

## 🩸 Project Overview

**Bakeneko** is an in-development rogue-like auto-battler centered around modular systems design, replayability, and strategic emergent gameplay.

### Core Pillars:
- 🧩 **Modular Action & Event Architecture** — Flexible, decoupled systems enabling complex interactions  
- 🎮 **Custom UI Focus Framework** — Seamless state-driven transitions between mouse and controller  
- 🃏 **Extensible Drag-and-Drop Framework** — Rapidly expandable card and interaction authoring  
- 🔥 **Rogue-like Replayability** — Build experimentation through layered systemic design  

---

## 🧠 Technical Highlights

### ⚙️ Modular Action & Event System
Designed a highly decoupled architecture allowing actors, abilities, and systems to communicate through broadcasted gameplay events.

```csharp
public void Broadcast(ActionType type, ActionContext context)
{
    foreach (var listener in listeners[type])
        listener.Execute(context);
}
