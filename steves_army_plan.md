# Steve's Army - Development Plan

## Vision

Create an Enlisted-style military squad system for Minecraft 1.20.1 where players command AI soldiers in combat. Focus on core tactical gameplay: squad command, cover system, and fair combat AI.

---

## Core Features (MVP)

### 1. Squad System

**Squad Composition**
- Player leads 1 squad of 4-8 AI soldiers
- Squad members can be respawned/replaced
- Simple recruitment system (item-based or block-based)

**Squad Data**
```
Squad {
    UUID leader (player)
    List<UUID> members
    Map<UUID, Integer> memberHealth
}
```

---

### 2. Command System

**Simplified Commands (2 modes)**

| Command | Behavior |
|---------|----------|
| **Follow Me** | Squad follows player, engages enemies on sight, stays in formation radius |
| **Hold Position** | Squad stays at location, takes cover, defends area |

**Implementation**
- Keybind to toggle between Follow/Hold
- No complex menus - keep it simple
- Visual indicator (chat message or HUD) showing current mode

**Attack Behavior**
- No separate "Attack Mode" - soldiers in Follow mode automatically engage enemies
- In Hold mode, they engage enemies within range but don't pursue

---

### 3. Respawn as Squadmate

**Feature**
- When player dies, they can respawn as one of their AI squad members
- Prevents "game over" when player dies
- Tactical choice: risk yourself or let AI take hits

**Implementation**
- On player death, show UI to select squadmate to respawn as
- Selected squadmate becomes player-controlled
- Remaining squad continues following new player
- If no squadmates alive, normal respawn

---

### 4. Target Acquisition (Fair System)

**Design Goals**
- No X-ray vision - soldiers only see what's realistically visible
- Creates fair and believable combat

**Detection Methods**

| Type | Range | Conditions |
|------|-------|------------|
| **Direct Line of Sight** | 64 blocks | Clear path to target, target in front arc |
| **Sound Detection** | 32 blocks | Gunfire, explosions trigger investigation |
| **Squad Communication** | 48 blocks | If one member sees enemy, all know approximate position |

**Line of Sight Rules**
- Raycast from soldier's eye position to target
- Check for solid blocks in between
- Must be within front 180° arc (no seeing behind)
- Affected by:
  - Day/night (reduced range at night)
  - Weather (rain reduces range)
  - Foliage (leaves partially block vision)

**Sound Detection**
- Gunshots heard within 32 blocks
- Soldiers investigate sound origin
- Doesn't reveal exact enemy position, just general direction
- Cooldown to prevent spam detection

---

### 5. Cover System ⭐ CRITICAL

**Why It's Critical**
- Without cover, soldiers die too quickly
- Cover extends survival and creates tactical gameplay
- Encourages player to use terrain strategically

**Cover Definition**

A block provides cover if:
1. It's between soldier and enemy
2. It's at least 1 block tall
3. Soldier can crouch behind it

**Cover Types**

| Type | Blocks | Effectiveness |
|------|--------|---------------|
| **Full Cover** | Stone, bricks, concrete, metal | 100% protection when behind |
| **Partial Cover** | Wood, hay bales, fences | 50% protection, some bullets penetrate |
| **Soft Cover** | Leaves, glass, tall grass | 25% protection, mostly visual concealment |

**Cover Detection Algorithm**
```
1. Scan 8-block radius for potential cover positions
2. For each position:
   - Check if solid block exists at foot or head level
   - Calculate angle to nearest enemy
   - Score based on: distance to player, cover quality, path access
3. Select best cover position
4. Pathfind to cover
5. Crouch animation when in cover
```

**Cover Behavior**
- In **Follow mode**: Take cover while moving, prioritize staying close to player
- In **Hold mode**: Find and stay in cover, peek to shoot
- When suppressed: Automatically seek nearest cover

**Cover Indicators** (Optional UI)
- Subtle highlight on nearby cover blocks when holding command key
- Helps player understand where squad will take cover

---

### 6. Combat AI

**Basic Combat Behavior**

| Action | Trigger |
|--------|---------|
| **Shoot** | Enemy in line of sight, has clear shot |
| **Take Cover** | Taking fire, low health, or in Hold mode |
| **Reload** | Ammo depleted |
| **Heal** | Low health + has healing item |

**Accuracy System**
- Base accuracy: 60% at optimal range
- Affected by:
  - Distance (accuracy drops over distance)
  - Movement (-20% when moving)
  - Suppression (-30% when being shot at)
  - Cover (+15% when in cover)

**Damage Balance**
- Soldiers die in 3-5 hits (not instant death)
- Gives player time to react and command
- Armor extends survival

---

### 7. Equipment System

**Simple Loadout**
- Each soldier has: 1 primary weapon, ammo, optional healing item
- Weapons from TaCZ integration
- Uniform appearance (configurable?)

**No Specialized Roles (Keep It Simple)**
- All soldiers are riflemen
- Same behavior, same capabilities
- Reduces complexity, easier to balance

---

## Technical Architecture

### Entity Structure

```
SoldierEntity extends PathfinderMob {
    SquadData squadData
    Inventory equipment
    CoverPosition currentCover
    
    AI Goals:
    - FollowLeaderGoal
    - HoldPositionGoal
    - FindCoverGoal
    - CombatGoal
    - ReloadGoal
}
```

### AI Goal Priority

1. Take cover (when suppressed/low health)
2. Reload (when empty)
3. Attack enemy (when has target)
4. Follow player / Hold position
5. Idle

### Data Structures

```
class SquadData {
    UUID leaderId
    List<UUID> memberIds
    SquadMode mode (FOLLOW / HOLD)
    BlockPos holdPosition
}

class CoverPosition {
    BlockPos position
    CoverType type
    float protectionValue
}
```

---

## TaCZ Integration

**Weapon System**
- Soldiers use TaCZ guns
- Read weapon stats from TaCZ data (damage, fire rate, accuracy, range)
- Use TaCZ rendering for held weapons

**Implementation Approach**
- Depend on TaCZ as required mod
- Create custom AI goal for firing TaCZ weapons
- Sync animation with TaCZ third-person animations

---

## Implementation Phases

### Phase 1: Foundation (Week 1-2)
- [ ] Set up Forge 1.20.1 development environment
- [ ] Create SoldierEntity basic class
- [ ] Basic follow AI (follow player around)
- [ ] Simple spawn system (item to recruit)

### Phase 2: Combat (Week 3-4)
- [ ] Target acquisition system (line of sight)
- [ ] Basic shooting AI
- [ ] TaCZ weapon integration
- [ ] Damage and health system

### Phase 3: Cover System (Week 5-7)
- [ ] Cover detection algorithm
- [ ] Cover scoring system
- [ ] Pathfinding to cover
- [ ] Cover behavior AI
- [ ] Crouch/peek animations

### Phase 4: Squad Commands (Week 8)
- [ ] Follow/Hold toggle
- [ ] Hold position with location
- [ ] Squad HUD (member count, health)
- [ ] Sound detection

### Phase 5: Respawn System (Week 9)
- [ ] Death event handler
- [ ] Squadmate selection UI
- [ ] Transfer player to squadmate
- [ ] Squad persistence on player swap

### Phase 6: Polish (Week 10-12)
- [ ] Balance testing
- [ ] Performance optimization
- [ ] UI improvements
- [ ] Sound effects
- [ ] Multiplayer testing

---

## What We're NOT Doing (Out of Scope)

| Feature | Reason |
|---------|--------|
| Specialized roles (Medic, Engineer, etc.) | Adds complexity, all riflemen is simpler |
| Assault/Attack mode | Follow mode covers attack behavior |
| Morale system | Too complex for MVP |
| Suppression mechanics | May add later, not critical |
| Vehicle support | Major scope increase |
| Artillery/support calls | Requires separate systems |
| Formations (line, wedge, etc.) | Nice to have, not essential |
| Individual soldier commands | Keep it squad-level only |

---

## Performance Targets

- 1 squad (8 soldiers) per player
- Up to 10 players on server = 80 soldiers max
- Target: 60 TPS with full load
- Optimizations:
  - Tick AI every 2-3 ticks (not every tick)
  - Reduce detection range for distant soldiers
  - Cache cover positions

---

## Success Criteria

**Minimum Viable Product:**
1. Player can recruit 4-8 soldiers
2. Soldiers follow player and engage enemies
3. Soldiers take cover intelligently
4. Player can toggle Follow/Hold
5. Player can respawn as squadmate
6. Fair target acquisition (no X-ray)
7. Compatible with TaCZ weapons

**Nice to Have (Post-MVP):**
- Cover indicators
- Squad loadout customization
- Sound detection for enemies
- Multiple squads per player
- Squad statistics (kills, deaths)

---

## Reference Code

| Project | What to Study | Link |
|---------|---------------|------|
| Ancient Warfare 2 | Squad AI, command system | https://github.com/P3pp3rF1y/AncientWarfare2 |
| TaCZ | Weapon system, animations | https://github.com/MCModderAnchor/TACZ |
| Villager Recruits | GUI design, team system | Closed source (play test) |

---

## Technical Questions to Resolve

1. **Cover caching** - Recalculate every tick or cache for N ticks?
2. **Pathfinding in combat** - Use vanilla pathfinder or custom?
3. **TaCZ API** - How to make NPCs fire TaCZ weapons?
4. **Multiplayer sync** - How much AI state needs syncing?
5. **Performance** - What's the AI tick budget?

---

## File Structure (Planned)

```
steves_army/
├── src/main/java/com/stevesarmy/
│   ├── entity/
│   │   ├── SoldierEntity.java
│   │   └── ai/
│   │       ├── FollowLeaderGoal.java
│   │       ├── HoldPositionGoal.java
│   │       ├── FindCoverGoal.java
│   │       ├── CombatGoal.java
│   │       └── TargetAcquisitionGoal.java
│   ├── squad/
│   │   ├── SquadData.java
│   │   ├── SquadManager.java
│   │   └── SquadMode.java
│   ├── cover/
│   │   ├── CoverSystem.java
│   │   ├── CoverPosition.java
│   │   └── CoverType.java
│   ├── item/
│   │   └── RecruitItem.java
│   └── network/
│       └── SquadSyncPacket.java
└── resources/
    └── assets/steves_army/
        ├── textures/
        ├── models/
        └── sounds/
```

---

## Next Steps

1. **Set up dev environment**
   - Install JDK 17
   - Set up Forge 1.20.1 MDK
   - Add TaCZ as dependency

2. **Create first soldier**
   - Basic entity that follows player
   - Test with vanilla items

3. **Study TaCZ API**
   - How to programmatically fire weapons
   - How to equip NPCs with TaCZ guns

4. **Prototype cover system**
   - Simple block detection
   - Test pathfinding to cover
