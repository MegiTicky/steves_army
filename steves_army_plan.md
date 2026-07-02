# Steve's Army - Development Plan

## Vision

Create an Enlisted-style military squad system for Minecraft 1.20.1 where players command AI soldiers in combat. Focus on core tactical gameplay: squad command, cover system, and fair combat AI.

---

## Core Features (MVP)

### 1. Squad System

**Squad Composition**
- Player leads 1 squad of up to 9 AI soldiers
- Squad members can be respawned/replaced
- Simple recruitment system (item-based: RecruitItem)
- 9-slot soldier inventory accessible via Middle Mouse click

**Squad Data**
```
SquadData {
    SquadMode mode (FOLLOW / HOLD)
    BlockPos holdPosition
}
SquadManager {
    Map<UUID, SquadData> squadDataMap
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
- G key to toggle between Follow/Hold (client-side packet)
- Ping wheel on Middle Mouse button (look-at, move-to, enemy-spotted, etc.)
- Visual indicator in chat showing current mode

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

| Type | Blocks | Behavior |
|------|--------|----------|
| **HALF Cover** | 1-block tall obstacles | Soldier crouches/crawls behind, pops up to peek (stance-only) |
| **FULL Cover** | 2+ block tall obstacles | Soldier stands behind, side-steps out to peek (1-block velocity slide) |
| **NONE** | No valid cover | Soldier exposed |

**Cover Detection Algorithm**
```
1. Scan 12-block radius for potential cover positions
2. For each position:
   - Check if solid block exists (voxel shape check)
   - Classify as HALF (1-block) or FULL (2-block)
   - Score based on: protection, distance, firing quality, peek angle
3. Select best cover position (or find better while in cover)
4. Pathfind to cover via vanilla navigation + ExactCoverMoveControl
5. Crawl/peek behavior based on cover type
```

**Cover Behavior State Machine**

```
NO_COVER → SEEKING_COVER → IN_COVER → SUPPRESSED_IN_COVER → IN_COVER
                               ↓                              ↓
                          REPOSITIONING ←── non-peekable ──→ DUCKING_BACK
```

**Peek System (3-state)**
```
HIDING ──→ EXPOSED ──→ DUCKING_BACK ──→ HIDING
  ↑                                        │
  └────────────────────────────────────────┘
```

- **HIDING**: Soldier waits behind cover, checks LOS cooldown, validates peek opportunity
- **EXPOSED**: Soldier shoots (SoldierCombatGoal checks peekState == EXPOSED), auto-ducks after 1500ms
- **DUCKING_BACK**: Half cover = instant pose switch. Full cover = velocity slide back to cover position
- **NON_PEEKABLE detection**: If no valid LOS from any peek position, marks cover non-peekable, repositions after ~8s

---

### 6. Combat AI

**Basic Combat Behavior**

| Action | Trigger |
|--------|---------|
| **Shoot** | Enemy in line of sight, peekState == EXPOSED |
| **Take Cover** | Taking fire, low health, or in Hold mode |
| **Reload** | Ammo depleted (uses inventory ammo, not dummy ammo) |
| **Heal** | Low health + has healing item |

**Accuracy System (TaCZ InaccuracyType)**
- STAND=5.0, MOVE=5.75, SNEAK=3.5, LIE=2.5, AIM=0.15
- Soldiers should aim before shooting
- Configurable via ForgeConfigSpec (StevesArmyConfig)

**Gun Integration**
- TaCZ via reflection (GunIntegration.java wraps reflection calls)
- Fallback to melee if TaCZ not loaded
- TaCZ is optional dependency (mods.toml: mandatory=false)

**Damage Balance**
- Soldiers die in 3-5 hits (not instant death)
- Gives player time to react and command
- Armor extends survival

---

### 7. Equipment System

**Simple Loadout**
- 9-slot inventory per soldier (SoldierInventory extends Container)
- Weapons from TaCZ integration
- Inventory-based ammo system (no dummy ammo)

**No Specialized Roles (Keep It Simple)**
- All soldiers are riflemen
- Same behavior, same capabilities
- Reduces complexity, easier to balance

---

## Implemented Features

### Cover System
- [x] Cover detection (HALF/FULL classification via voxel shape checks)
- [x] Cover scoring (protection, distance, firing quality, peek angle, fightability bonus)
- [x] Cover reservation system (prevents two soldiers sharing same cover)
- [x] Peek system (HIDING → EXPOSED → DUCKING_BACK for both half and full cover)
- [x] Full cover side-step peek (velocity slide 1 block to side)
- [x] Half cover crawl (SWIMMING pose, persistent per-tick)
- [x] Suppression system (decay rates, pinned/suppressed thresholds)
- [x] Non-peekable cover detection → reposition after ~8s
- [x] Suppression while peeked correctly triggers duck-back (velocity slide/physical return)
- [x] ClearCover() disables crawling (prevents walking while crawling)

### Prone/Crawl Pose
- [x] Custom SoldierModel extends HumanoidModel with prone pose
- [x] PoseConfig with adjustable angles (live via /stevesarmy_cover pose set)
- [x] swimAmount-driven body rotation (smooth lerp 0° to -90°)
- [x] Instant limb pose switching (no lerp, only body rotation provides visual transition)
- [x] Exit with threshold at swimAmount < 0.15 (body at -13.5°, minimal snap)
- [x] Prone hitbox (0.6F × 0.8F)
- [x] Armor layers use vanilla HumanoidModel (not SoldierModel)
- [x] Leg position offsets for body rotation compensation

### Commands
- [x] /stevesarmy_debug - Combat debug info
- [x] /stevesarmy_cover - Cover system debug (state, threats, reservations, force-peek)
- [x] /stevesarmy_cover pose [crawl|stand|status|noai] - Force state
- [x] /stevesarmy_cover pose set <part> <axis> <value> - Live angle adjustment
- [x] /stevesarmy_cover pose [angles|deg] - Angle reports
- [x] /stevesarmy_cover pose reset - Reset to defaults

### Network
- [x] ToggleSquadModeMessage (G key toggle)
- [x] Debug messages
- [x] Ping messages (broadcast to squad)
- [x] Inventory open messages
- [x] PotentialTargetsDebugMessage

### Client
- [x] Custom SoldierRenderer with swimAmount-driven setupRotations
- [x] Ping wheel (Middle Mouse)
- [x] Combat debug renderer (LOS lines, threat indicators)
- [x] Cover debug renderer (cover points, state labels)

---

## Technical Architecture

### Entity Structure

```
SoldierEntity extends PathfinderMob implements Container {
    SquadData squadData
    SoldierInventory (9 slots)
    CoverBehaviorManager coverBehaviorManager
    ThreatAwareness threatAwareness
    
    Data Parameters:
    - CRAWLING (boolean)
    - SQUAD_MODE (int ordinal)
    
    AI Goals (priority order):
    - CoverTacticalGoal (HIGHEST) - cover seeking/hiding/peeking
    - SoldierCombatGoal - shooting when exposed
    - SoldierFollowOwnerGoal - follow mode pathfinding
    - SoldierHoldPositionGoal - hold mode staying
    - SoldierMoveToPingGoal - ping destination
    - SoldierStrollGoal - idle wandering
}
```

### AI Goal Priority

1. CoverTacticalGoal - Find cover, hide, peek, suppress
2. SoldierCombatGoal - Shoot when peekState == EXPOSED
3. SoldierFollowOwnerGoal - Follow player
4. SoldierHoldPositionGoal - Hold position
5. SoldierMoveToPingGoal - Move to ping
6. SoldierStrollGoal - Idle

### Data Structures

```
SquadData {
    SquadMode mode (FOLLOW / HOLD)
    BlockPos holdPosition
}

CoverPoint {
    BlockPos position
    CoverType type (HALF / FULL / NONE)
    Set<Direction> protectedDirections
    float quality
    boolean canShootFrom
}

CoverType enum { HALF, FULL, NONE }
SquadMode enum { FOLLOW, HOLD }
```

---

## TaCZ Integration

**Weapon System**
- Soldiers use TaCZ guns via reflection
- GunIntegration.java wraps all reflection calls
- ShootResult enum: SUCCESS, NEED_BOLT, NO_AMMO, COOL_DOWN, IS_BOLTING, IS_RELOADING, IS_DRAWING, NOT_DRAWN

**Implementation Approach**
- Depend on TaCZ as optional mod
- TaCZ PlayerAnimator prone animations only work for AbstractClientPlayer — SoldierEntity uses custom SoldierModel
- Graceful fallback to melee if TaCZ not loaded

---

## Key Decisions

- **Simplified scope:** Follow/Hold commands only (no Assault mode), all riflemen
- **Cover state machine:** NO_COVER → SEEKING → IN_COVER → SUPPRESSED_IN_COVER → REPOSITIONING
- **Peek system 3-state:** HIDING → EXPOSED → DUCKING_BACK. Half-cover = stance-only pop-up. Full-cover = side-step 1 block with velocity manipulation
- **LOS validation before peek:** Half-cover checks LOS at standing eye height. Full-cover checks LOS from each candidate peek position
- **Cover scoring:** FIGHTABILITY_BONUS (0.2) added when canShootFrom() is true
- **SoldierCombatGoal simplified:** Peek state machine moved to CoverTacticalGoal. Only checks peekState == EXPOSED before shooting
- **Inventory-based ammo:** Soldiers use ammo from 9-slot inventory, not dummy ammo
- **TaCZ via reflection:** Graceful fallback to melee
- **TaCZ optional dependency** (mods.toml: mandatory=false)
- **Crawl uses vanilla SWIMMING pose** with persistent flag per-tick re-application
- **Exit crawl crawl false in clearCover()** prevents walking while crawling
- **Suppressed duck-back processed in SUPPRESSED_IN_COVER** via tickDuckingBack() call

---

## Implementation Phases

### Phase 1: Foundation ✓
- [x] Set up Forge 1.20.1 development environment
- [x] Create SoldierEntity basic class
- [x] Basic follow AI (follow player around)
- [x] Simple spawn system (RecruitItem)
- [x] Squad system (FOLLOW/HOLD modes)
- [x] Key bindings (G=toggle, H=debug, MB3=ping wheel)

### Phase 2: Combat ✓
- [x] Target acquisition system (line of sight, threat awareness)
- [x] Basic shooting AI (SoldierCombatGoal)
- [x] TaCZ weapon integration (reflection-based)
- [x] Damage and health system
- [x] Aim accuracy (TaCZ InaccuracyType)
- [x] Suppression system
- [ ] Fix accruacy issue - They seems to be not affected by distance now
- [ ] Suppressive fire at enemy behind cover?

### Phase 3: Cover System ✓
- [x] Cover detection algorithm (HALF/FULL classification)
- [x] Cover scoring system (protection, distance, firing quality, peek angle)
- [x] Pathfinding to cover (ExactCoverMoveControl)
- [x] Cover behavior AI (state machine, peek system)
- [x] Prone/crawl animations (SoldierModel, PoseConfig)
- [x] Full cover side-step peek
- [x] Suppression integration (duck-back on fire, decay, pinned threshold)
- [x] Non-peekable cover → reposition
- [x] Crawl state correctly cleared on cover exit
- [ ] Do pathfind from cover to current position too, to confirm that the soldier wont fall into a hole and cant get out
- [x] Accruacy issue. The soldier is aiming for the edge of the block instead of at the centre. This make using 1 block cover very hard
- [ ] When peeking, the soldier only consider one target. Need to fix this
- [ ] Make peeking less robotic
- [x] Now the when the soldier fires from cover, the soldier consider that a bulelt flying close to himself. And get suppressed by himself. Need to fix this

### Phase 4: Squad Commands ✓
- [x] Follow/Hold toggle
- [x] Hold position with location
- [x] Ping system (look-at, move-to, enemy-spotted)
- [x] Ping wheel UI
- [x] Squad HUD (debug overlays)
- [ ] Attack mode
- [ ] Detailed Planning UI


### Phase 5: Respawn System (Next)
- [ ] Death event handler
- [ ] Squadmate selection UI
- [ ] Transfer player to squadmate
- [ ] Squad persistence on player swap

### Phase 5b: Cover Bugfixes & AI Polish (Current Priority)
- [X] **Fix suppression near-miss detection** — Bullets landing within 3 blocks should trigger `SuppressionTracker.onNearMiss()`. Investigate `IncomingFireHandler` event wiring — projectile impact events may not fire reliably. Add debug logging to verify trigger rate
- [ ] **Fix weird head aim in cover** — Soldiers stare at odd angles (into walls/sky) while hiding. Head tracking should lock to cover-relative threat direction (behind-cover safe zone), not vanilla look-vector. In HIDING state, head should face threat direction at crouch height, not randomly
- [X] **Fix non-peekable slide into wall** — When all adjacent blocks from current full cover offer no LOS to any target, the soldier still velocity-slides toward one side. The peek position computation should return null when no valid position exists, and the soldier should go straight to REPOSITIONING instead of sliding into a wall each peek attempt
- [X] **Reconsider cover when non-peekable** — Currently soldier waits ~8s (160 ticks) before abandoning non-peekable cover. Reduce this threshold or make it dynamic: if no LOS from any peek position for >3s AND no active target in a direction that could be valid, reposition sooner. Also check during `evaluateCoverState()` periodic ticks
- [ ] **Cover path exposure** — When scoring cover positions, check if the navigation path to the cover is exposed to known threats. Use raycasts along the path segments. If the only approach is through open fire zones, apply a penalty to the cover score. Soldiers should prefer cover they can reach safely
- [ ] **Tactical reload in cover** — Soldiers should only tactical-reload while in HIDING or SUPPRESSED_IN_COVER. No reloading while EXPOSED. If magazine is empty and soldier needs to peek, they should reload first while hidden, then peek. Emergency reload (completely out of ammo in EXPOSED state) should return to cover first
- [ ] **Prevent slide when all peek positions invalid** — In `computePeekPosition()` / `tickHiding()` for full cover: before setting peek state to EXPOSED or starting the velocity slide, validate that at least one candidate peek position has LOS. If none do, set non-peekable immediately and don't start the side-step movement
- [ ] **Cover path find issue** - the AI now go for straight line when close to cover, and they sometime hit obstacle.d

### Phase 6: Polish
- [ ] Balance testing
- [ ] Performance optimization
- [ ] UI improvements
- [ ] Sound effects
- [ ] Multiplayer testing

---

## What We're NOT Doing (Out of Scope for MVP)

| Feature | Reason |
|---------|--------|
| Specialized roles (Medic, Engineer, etc.) | Adds complexity, all riflemen is simpler |
| Assault/Attack mode | Follow mode covers attack behavior |
| Morale system | Too complex for MVP |
| Vehicle support | Post-MVP feature |
| Artillery/support calls | Requires separate systems |
| Formations (line, wedge, etc.) | Post-MVP feature |
| Individual soldier commands | Keep it squad-level only |
| Modded uniform system | Rely on external mods |

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
1. [x] Player can recruit 4-8 soldiers
2. [x] Soldiers follow player and engage enemies
3. [x] Soldiers take cover intelligently
4. [x] Player can toggle Follow/Hold
5. [ ] Player can respawn as squadmate
6. [x] Fair target acquisition (no X-ray)
7. [x] Compatible with TaCZ weapons

**Nice to Have (Post-MVP):**
- Cover indicators
- Squad loadout customization
- Sound detection for enemies
- Multiple squads per player
- Squad statistics (kills, deaths)

---

## File Structure (Current)

```
steves_army/src/main/java/com/stevesarmy/
├── StevesArmyMod.java       # @Mod entry, registers entities/items/menus/commands
├── StevesArmyConfig.java    # ForgeConfigSpec: accuracy, shot threshold, target switch
├── combat/                  # Core combat (gun integration, detection, threat, aim)
│   └── cover/               # Cover system: Manager, Finder, Evaluator, Suppression, etc.
├── command/                 # /stevesarmy_debug, /stevesarmy_cover, /stevesarmy
├── entity/                  # SoldierEntity (PathfinderMob + 9-slot Container), TargetEntity
│   └── ai/                  # CoverTacticalGoal, SoldierCombatGoal, Follow/Hold/Stroll/Ping
├── client/                  # Key bindings, debug renderers, model, renderer, screens
├── inventory/               # 9-slot SoldierInventory, menu, screen
├── item/                    # RecruitItem, spawn eggs
├── network/                 # Packets: squad mode, debug, inventory, ping, targets
├── ping/                    # Ping system (look-at, move-to, enemy-spotted, etc.)
├── squad/                   # SquadData, SquadManager (FOLLOW/HOLD modes)
├── registry/                # ModEntities, ModItems, ModMenuTypes
└── util/                    # MathUtils, RateLimiter, ScreenPos
```

---

## Cover System Constants

### CoverTacticalGoal.java
```
COVER_REACHED_DISTANCE       = 1.5   blocks
COVER_VALID_DISTANCE         = 2.0   blocks (normal)
COMBAT_COVER_VALID_DISTANCE  = 6.0   blocks (while shooting)
COVER_ABANDON_DISTANCE       = 8.0   blocks (definitely left)
SEARCH_RADIUS                = 12    blocks
MIN_COVER_DWELL_TIME_MS      = 4000  ms
MAX_SEEKING_TIME_MS          = 10000 ms
HYSTERESIS_THRESHOLD         = 0.35  (35% better to switch cover)
NON_PEEKABLE_REPOSITION_TICKS= 160   (~8s before abandoning non-peekable cover)
FULL_COVER_PEEK_SPEED        = 0.15  blocks/tick (velocity slide)
```

### CoverBehaviorManager.java (Peek)
```
EXPOSURE_TIME_MS             = 1500  ms max exposure
MIN_EXPOSURE_TIME_MS         = 800   ms minimum exposure
DUCK_COOLDOWN_MS             = 1000  ms between peeks
SUPPRESSED_HIDE_EXTRA_MS     = 2000  ms extra hide when suppressed
```

### SuppressionTracker.java
```
DECAY_RATE                   = 0.15  base/tick (0.015/s in cover, 0.0075/s out)
SUPPRESSED_THRESHOLD         = 0.5   (suppressed)
PINNED_THRESHOLD             = 0.7   (pinned)
MIN_PEEK_TIME_MS             = 2500  ms before can peek after suppression
DIRECT_FIRE_SUPPRESSION      = 0.3   per incoming fire event
DAMAGE_SUPPRESSION           = 0.4   per damage taken
NEAR_MISS_SUPPRESSION        = 0.25  per near miss within 3 blocks
```

### CoverFinder.java (Scoring Weights)
```
PRIMARY_PROTECTION_WEIGHT    = 0.30
FLANKING_PROTECTION_WEIGHT   = 0.20
DISTANCE_WEIGHT              = 0.10
FIRING_QUALITY_WEIGHT        = 0.25
PEEK_ANGLE_WEIGHT            = 0.15
FIGHTABILITY_BONUS           = 0.20  added if canShootFrom()
```

---

## Prone/Crawl System

**PoseConfig.java** — static fields for all prone pose angles/positions:
- RA_X=-3.14, RA_POS_Z=-2.0 (right arm fully inverted, forward offset)
- LA_X=-3.14, LA_POS_Z=-2.0 (left arm)
- H_X=-3.0 (head nearly fully inverted)
- RL_POS_Z, LL_POS_Z (leg forward offsets)
- H_CLAMP_MIN=-4.0, H_CLAMP_MAX=0.5 (head clamp range)
- getAngleReport() / getDegreesReport() for debug

**SoldierModel setupAnim() logic:**
- isCrawling() == true: full prone pose via Mth.lerp for smooth entry
- isCrawling() == false && swimAmount > 0.15F: maintain prone pose during exit
- isCrawling() == false && swimAmount <= 0.15F: let super.setupAnim() standing pose

**SoldierRenderer.setupRotations():**
- Uses soldier.getSwimAmount(partialTick) to lerp body rotation 0° to -90°
- Translation from (0,0) to (-1, 0.25)

**Critical Context:**
- Negative xRot = forward/downward in HumanoidModel (verified via walking swing, riding code, elytra)
- Arm xRot = -3.14 (-π): fully inverted; after -90° body X rotation, points forward in world
- No limb lerp on exit — body rotation handles visual transition; snap at swimAmount < 0.15 (~2 ticks, body at -13.5°)
- swimAmount flows naturally (vanilla LivingEntity.updateSwimAmount(), 0.09/tick, ~11 ticks full transition)
- TaCZ PlayerAnimator prone animations only work for AbstractClientPlayer — not applicable

---

## Next Steps

1. **Respawn system** - Death event handler, squadmate selection UI, player transfer
2. **Polish** - Balance testing, performance optimization, UI improvements
3. **Sound effects** - Gunshots, commands, footsteps
4. **Multiplayer testing** - Sync edge cases, performance under load

---

## Future Features (Post-MVP)

### Detailed Planning UI
Soldier behavior customization beyond Follow/Hold:
- **Engagement rules**: Scout mode (observe only, engage if engaged), Battle mode (engage on sight), Stealth mode (avoid combat)
- **Trigger discipline**: Hold fire until fired upon, fire only on ping, free engagement
- **Formation**: Thin line (spread out), Spearhead (follow in wedge), Column (single file), Skirmish (loose grouping)
- **Movement speed**: Walk/crawl only, run only, adaptive

### Human-like Behavior
Break the "perfect AI" feel:
- **Fix x-ray prefire**: Soldiers should not prefire when peeking from cover — they need time to acquire targets after LOS is established, mirroring human reaction time
- **Hesitation**: Random decision delays (50-200ms) before committing to actions like returning fire, changing position, or exposing
- **Not always optimal**: Occasionally choose sub-optimal cover positions, take longer routes, or delay shots to feel more natural
- **Target switching delay**: Brief delay (200-400ms) before switching to a new higher-priority target

### Friendly Fire Prevention
- Raycast check from soldier to target before firing — ensure no friendly entity (squad member, player) is in the bullet path
- Non-squad entities (villagers, passive mobs) should not cause hold-fire, only soldiers/players
- Friendly check should account for target movement — lead the shot check along bullet trajectory

### Squad-level Decision Making
- **Cover stacking avoidance**: Soldiers should communicate cover choices at the squad level, not just via `CoverReservationManager` (which only prevents 2-per-cover)
- **Crossfire positioning**: Soldiers should spread out to cover different angles, not cluster behind the same obstacle
- **Mutual support**: If a squadmate is suppressed, other members prioritize suppressing the enemy that's pinning them

### Recoil Simulation
- Aim tracking progress should degrade proportional to weapon recoil
- After each shot, the soldier's aim point drifts based on TaCZ gun recoil stats
- Tracking recovery time scales with recoil severity (light rifle = fast recovery, heavy = slow)
- Prevents laser-accurate full-auto at long range

### VS (Valkyrien Skies) Compatibility
- Enable pathfinding on/around VS ships — treat VS ship blocks as valid navigation surfaces
- Ship movement should not break soldier pathfinding (continuous collision updates?)
- Soldiers should be able to take cover on ship structures

### VS Vehicle Compatibility (Tanks & Transports)
- **Mount/Dismount**: Soldiers can get on/off VS vehicles (tanks, transport trucks)
- **Vehicle as cover**: Soldiers can use vehicles as cover points (detect vehicle hull as solid protection)
- **Tank threat response**: VS vehicles with CBC cannons are treated as high-priority threats — soldiers avoid their line of fire, seek hard cover (not vehicles), and suppress/damage them if possible
- **Transport**: Soldiers can ride in/on transport vehicles, dismount when ordered or under fire

### CBC (Cannons-Bombs-Cannons?) Compatibility
- AI-controlled cannon crew (separate from soldier AI — dedicated gun crew)
- Detect and operate CBC cannons on ships/vehicles
- Target tracking and firing at enemy vehicles/structures
- Could share detection systems with soldier AI

### Model Improvements
- **Base model**: Steve with tank top + underwear (modern military look without full uniform)
- **Rely on modded uniform system**: Rather than building a complex skin system, let other uniform mods handle the appearance
- Custom `SoldierModel` already extends `HumanoidModel` — add skin layer support for modded uniform textures