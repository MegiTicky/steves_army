# Enlisted-Style AI Squad System Research for Minecraft Mod

## Overview

Enlisted is a squad-based first-person shooter where players command AI-controlled soldiers in large-scale World War II battles. The core mechanic revolves around the player leading a squad of AI soldiers while also controlling individual squad members directly.

---

## Core AI Squad System Features

### 1. Squad Composition
- **Player as Squad Leader**: The player commands a squad of AI soldiers
- **Squad Size**: Typically 4-9 soldiers per squad (varies by faction and campaign)
- **Specialized Roles**: Soldiers have specific roles (Assault, Medic, Engineer, Radio Operator, Anti-Tank, Machine Gunner, Sniper, Rifleman)
- **Equipment Loadouts**: Each AI soldier has their own weapons, ammunition, and equipment

### 2. AI Behavior Modes

#### Follow Mode
- AI soldiers follow the player within a defined radius
- They maintain formation but adapt to terrain
- They engage enemies automatically while following

#### Hold Position Mode
- AI stays at designated location
- Takes cover automatically
- Provides suppressing fire

#### Assault/Attack Mode
- AI aggressively moves toward objectives
- Coordinates attacks on enemy positions
- Uses grenades and special abilities

#### Defensive/Suppression Mode
- AI prioritizes cover and defensive positions
- Focuses on suppressing enemies rather than pushing

### 3. Command System

#### Basic Commands
- **Follow Me**: Squad regroups on player
- **Hold Position**: Squad stays at current location
- **Attack/Advance**: Squad moves to designated point
- **Take Cover**: Squad seeks nearest cover
- **Suppressing Fire**: Squad focuses fire on designated area

#### Advanced Commands
- **Individual Unit Control**: Command specific soldiers
- **Formation Selection**: Line, column, wedge formations
- **Target Prioritization**: Focus on infantry, vehicles, or objectives
- **Ability Activation**: Use special abilities (medic heal, engineer build, artillery strike)

### 4. AI Combat Behavior

#### Target Acquisition
- Line of sight based enemy detection
- Sound-based detection (gunfire, footsteps)
- Squad members share target information
- Radio operators can call in reconnaissance

#### Cover System
- AI identifies and moves to cover automatically
- Distinguishes between hard cover (indestructible) and soft cover
- Flanks exposed enemies
- Retreats from overwhelming firepower

#### Tactical Movement
- Bounding overwatch (alternating movement between subgroups)
- Flanking maneuvers
- Suppression and advance tactics
- Retreat when outmatched

#### Fire Discipline
- Automatic fire at spotted enemies
- Suppression fire on suspected positions
- Conservation of ammunition
- Reload management

### 5. Specialized AI Roles

#### Medic
- Automatically revives downed squadmates
- Heals injured soldiers
- Prioritizes wounded over combat (adjustable)
- Carries medical supplies

#### Engineer
- Builds fortifications (sandbags, barbed wire)
- Constructs rally points
- Repairs vehicles
- Places mines and explosives

#### Radio Operator
- Calls in artillery strikes
- Requests supply drops
- Provides reconnaissance
- Communicates with other squads

#### Anti-Tank
- Prioritizes armored vehicles
- Maintains safe distance from targets
- Coordinates with squad for protection
- Carries rocket launchers/AT grenades

#### Machine Gunner
- Provides suppressing fire
- Sets up in defensive positions
- Slow movement, high firepower
- Requires setup time for accuracy

#### Sniper
- Engages targets at long range
- Stays behind main squad
- Prioritizes high-value targets (officers, MGs)
- Relocates after shots

### 6. Communication & Coordination

#### Visual Indicators
- UI markers showing squad positions
- Status icons (health, ammo, suppression)
- Objective markers
- Enemy spotted indicators

#### Audio Cues
- Voice lines for orders and status
- Callouts for enemy positions
- Suppression and morale sounds
- Faction-specific voice acting

#### Information Sharing
- Spotted enemies shared among squad
- Coordination between multiple player squads
- Commander-level strategic overview

### 7. Morale & Suppression System

#### Suppression Mechanics
- AI under heavy fire takes cover
- Reduced accuracy when suppressed
- Panic behaviors (retreat, freeze)
- Recovery when fire stops

#### Morale
- Affects AI aggression and effectiveness
- Influenced by casualties, nearby leadership
- Rally points restore morale
- Broken morale causes retreat

---

## Implementation Recommendations for Minecraft Mod

### Phase 1: Core Framework
1. NPC entity with extended AI capabilities
2. Basic squad data structure (list of NPCs, leader reference)
3. Simple follow/hold commands
4. Basic combat AI (target, shoot, take cover)

### Phase 2: Combat AI
1. Cover detection system (blocks providing protection)
2. Suppression mechanics
3. Flanking behavior
4. Formation system
5. Ammo and reload management

### Phase 3: Specialized Roles
1. Role definitions (Medic, Engineer, etc.)
2. Role-specific behaviors
3. Equipment system
4. Special abilities

### Phase 4: Command System
1. Command UI (radial menu or keybinds)
2. Individual unit selection
3. Waypoint/destination marking
4. Formation presets

### Phase 5: Advanced Features
1. Morale and suppression
2. Inter-squad communication
3. Objective-based AI
4. Vehicle support
5. Building/fortification system

---

## Technical Considerations

### Performance
- Limit active AI calculations per tick
- Use LOD (Level of Detail) for distant AI
- Efficient pathfinding caching
- Squad-level updates vs individual updates

### Minecraft-Specific Adaptations
- Block-based cover system
- Redstone-compatible commands
- Integration with Minecraft combat (bows, crossbows, swords)
- Loot and equipment management
- Dimension/portaling behavior

### Data Structures
```
Squad {
    UUID leader
    List<UUID> members
    FormationType currentFormation
    Vector3d waypoint
    CommandMode mode
    Map<Role, List<UUID>> roles
}

NPCData {
    UUID squadId
    Role role
    Inventory equipment
    int ammo
    float morale
    float suppression
    boolean isPlayerControlled
}
```

---

## References

- Enlisted (Gaijin Entertainment/Darkflow Software)
- Similar systems: Brothers in Arms, Squad, Arma
- Minecraft mod references: Ancient Warfare 2, Minecraft Comes Alive, Custom NPCs

---

## Notes

- Balance between autonomous AI and player control is crucial
- Clear visual feedback for AI states
- Avoid too many micro-management options
- Consider multiplayer synchronization
- Test with varying squad sizes for performance
