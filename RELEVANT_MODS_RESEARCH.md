# Relevant Mods Research for 1.20.1 Military Simulation Mod

## Overview

This document catalogs existing Minecraft mods relevant to developing an Enlisted-style military squad system. All mods listed are for **Minecraft 1.20.1 Forge**.

---

## Gun Mods (1.20.1)

### 1. Timeless and Classics Zero (TaCZ) ⭐ RECOMMENDED
**CurseForge**: https://www.curseforge.com/minecraft/mc-mods/timeless-and-classics-zero

| Metric | Value |
|--------|-------|
| Downloads | 30.2M |
| Last Updated | May 2026 |
| Loader | Forge |
| License | GPLv3 (code), CC BY-NC-ND 4.0 (assets) |

**Features**:
- Most popular gun mod for modern Minecraft
- AAA-quality gunplay and animations
- Attachment system (scopes, grips, suppressors, etc.)
- Custom gun packs support (JSON-based, no coding required)
- Workbench crafting system (survival-friendly)
- Compatible with shaders
- LOD optimization
- Third-person animations (with Player Animator mod)

**Why It's Ideal for Your Mod**:
- Open-source (GPLv3) - can study implementation
- Active development and community
- Gun pack system allows easy content creation
- Well-documented API
- Performance-optimized

**Integration Potential**:
- Your NPCs could use TaCZ weapons natively
- Gun pack system for military equipment
- Attachment system for soldier customization

**Official Gun Packs**:
- Cyber Armorer
- Create Armorer
- Applied Armorer
- Immersive Armorer

---

### 2. Vic's Modern Warfare (Legacy)
**Status**: Not updated for 1.20.1

**Notes**: Popular in older versions (1.12.2), but no 1.20.1 release. Community has moved to TaCZ.

---

### 3. MrCrayfish's Gun Mod
**Status**: Not updated for 1.20.1

**Notes**: Simpler gun mod, popular for learning. Last updated for 1.16.5.

---

## NPC/Squad Command Mods

### 1. Villager Recruits ⭐ MOST SIMILAR
**CurseForge**: https://www.curseforge.com/minecraft/mc-mods/recruits

| Metric | Value |
|--------|-------|
| Downloads | 2.5M |
| Last Updated | June 2026 |
| Loader | Forge |
| License | All Rights Reserved |

**Features**:
- Hire villagers with emeralds to create an army
- Command and manage through custom GUIs
- PvP and faction battles
- Team/faction system integration
- Smart eating/healing AI
- Shield blocking AI
- Group commands for large armies
- Spawn from villages, disband when needed
- Compatible with Epic Knights mod, Small Ships mod

**Relevance to Your Project**:
- **Directly implements squad command system**
- Villagers follow, fight, and take orders
- Group management UI
- Team-based PvP
- Active development (1.20.1 supported)

**Key Learnings**:
- Study their command GUI implementation
- AI behavior for combat followers
- Team/faction system integration

**Limitations**:
- Medieval-focused (swords, shields, bows)
- No modern gun support
- No cover system
- Basic AI (no advanced tactics)

---

### 2. Ancient Warfare 2
**CurseForge**: https://www.curseforge.com/minecraft/mc-mods/ancient-warfare-2

| Metric | Value |
|--------|-------|
| Downloads | 6.8M |
| Last Updated | April 2021 |
| Loader | Forge |
| License | GPLv3 |
| Version | 1.12.2 only (no 1.20.1) |

**Features**:
- NPC-driven automation and combat
- Soldier types: grunts, archers, horsemen, healers, siege engineers
- Equipment system (armor, weapons, shields)
- Commander tools for battlefield control
- Team-based warfare (scoreboard teams, FTB Utils)
- Siege weapons (catapults, ballistae, trebuchets, cannons)
- Worker NPCs for automation
- World generation with 2000+ structures
- 45+ NPC tribes and nations

**Why Study This**:
- **Open-source (GPLv3)** - can read code
- Comprehensive NPC command system
- Multiple soldier classes
- Siege weapon implementation
- Team warfare mechanics
- Large-scale battles

**Limitations**:
- **Not available for 1.20.1** (last version: 1.12.2)
- Medieval theme
- No modern gun support

**Code Value**:
- Excellent reference for NPC AI architecture
- Squad command implementation
- Team/faction system
- World-gen structure spawning

---

### 3. Custom NPCs
**CurseForge**: https://www.curseforge.com/minecraft/mc-mods/custom-npcs

| Metric | Value |
|--------|-------|
| Downloads | 27.1M |
| Last Updated | May 2022 |
| Loader | Forge |
| License | CC BY-NC 3.0 |
| Version | 1.16.5 (no 1.20.1) |

**Features**:
- Create custom NPCs with configurable AI
- Dialog system
- Quest system
- Faction system
- Jobs and roles
- Custom skins and models

**Relevance**:
- Great for adventure map creation
- Flexible NPC scripting

**Limitations**:
- No 1.20.1 version
- Not focused on combat squads
- More for RPG/story content

---

### 4. CustomNPC+ (1.20.1 Fork)
**CurseForge**: https://www.curseforge.com/minecraft/mc-mods/customnpc-plus

| Metric | Value |
|--------|-------|
| Downloads | 1.7M |
| Last Updated | March 2026 |
| Loader | Forge |
| Version | 1.20.1 |

**Features**:
- Backport of CustomNPCs to 1.20.1
- Adds new features
- Active development

**Potential Use**:
- Could integrate with your mod
- Custom NPC creation for variety

---

## Other Relevant Mods

### Minecolonies
**Focus**: Colony management, not military
**Relevance**: NPC job system, building automation

### Millénaire
**Focus**: Village culture simulation
**Relevance**: Complex NPC behaviors

### Minecraft Comes Alive (MCA)
**Focus**: Villager relationships and family
**Relevance**: Villager interaction system
**Status**: No 1.20.1 Forge version

---

## Compatibility Considerations

### TaCZ + Villager Recruits Combination
**Potential**: Combine modern guns with squad command
**Challenge**: Villager Recruits uses vanilla weapons
**Solution**: Your mod could bridge the gap

---

## Gap Analysis: What's Missing

### Existing Mods Provide:
| Feature | TaCZ | Villager Recruits | Ancient Warfare 2 |
|---------|------|-------------------|-------------------|
| Modern guns | ✅ | ❌ | ❌ |
| NPC squads | ❌ | ✅ | ✅ |
| Command UI | ❌ | ✅ | ✅ |
| Team battles | ❌ | ✅ | ✅ |
| Cover system | ❌ | ❌ | ❌ |
| Suppression AI | ❌ | ❌ | ❌ |
| Specialized roles | ❌ | ❌ | ✅ |
| Vehicle support | ❌ | ❌ | ✅ (boats) |
| Open source | ✅ | ❌ | ✅ |

### Your Mod's Unique Value:
1. **Modern military AI** - No existing mod has Enlisted-style squad AI with modern weapons
2. **Cover system** - Untapped in Minecraft
3. **Suppression mechanics** - Unique to your mod
4. **TaCZ integration** - Make NPCs use modern firearms
5. **Tactical AI** - Flanking, bounding overwatch, etc.

---

## Recommended Approach

### Option A: Standalone Mod (Recommended)
- Build your own NPC system from scratch
- Integrate with TaCZ for weapons
- Study Villager Recruits and Ancient Warfare 2 for reference
- Implement unique features (cover, suppression, tactics)

### Option B: TaCZ Addon
- Create an addon for TaCZ
- Focus purely on NPC AI
- Leverage TaCZ's gun system directly
- Smaller scope

### Option C: Villager Recruits Fork/Addon
- Extend Villager Recruits with modern weapons
- Add advanced AI features
- Dependent on Villager Recruits updates

---

## Technical Resources

### Open-Source Code to Study
1. **TaCZ** - Gun system, animations, attachments
   - GitHub: https://github.com/MCModderAnchor/TACZ
   - License: GPLv3

2. **Ancient Warfare 2** - NPC AI, squad commands, team warfare
   - GitHub: https://github.com/P3pp3rF1y/AncientWarfare2
   - License: GPLv3

### Documentation
- TaCZ Wiki: https://tacwiki.mcma.club/
- Ancient Warfare 2: Has in-game manual

---

## Modpack Examples

### Military/Combat Modpacks (1.20.1)
These can provide inspiration for balancing and integration:

1. **BODYBLOCK** - Bodycam-style combat
   - Uses TaCZ
   - Focus on tactical gameplay

---

## Summary

### Best Gun Mod for 1.20.1
**TaCZ** - Most popular, actively maintained, open-source, supports gun packs

### Most Similar Mod
**Villager Recruits** - Directly implements villager army command system, but lacks modern weapons and advanced AI

### Best Code Reference
**Ancient Warfare 2** - Comprehensive open-source NPC command system (but 1.12.2 only)

### Your Mod's Niche
**Modern military squad simulation** - No existing mod combines:
- Modern firearms (TaCZ-style)
- Enlisted-style AI squad commands
- Cover and suppression mechanics
- Tactical behaviors (flanking, overwatch)
- Specialized military roles

---

## Next Steps

1. **Set up development environment** with TaCZ installed
2. **Study TaCZ API** for gun integration
3. **Read Ancient Warfare 2 source** for NPC AI patterns
4. **Test Villager Recruits** to understand user expectations
5. **Design your NPC entity** with TaCZ gun support from day 1
