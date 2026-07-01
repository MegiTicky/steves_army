# Steve's Army - Agent Instructions

## Build & Run

```powershell
# JDK 17 required
$env:JAVA_HOME = "C:\Program Files\Java\jdk-17"
.\gradlew build
.\gradlew runClient        # needs .\gradlew prepareRuns first
```

- Mod is in `steves_army/` subdirectory, NOT repo root
- Must set `JAVA_HOME` to JDK 17 every session
- Build jar at `steves_army/build/libs/steves_army-0.1.0-alpha.jar`
- Copy to test instance: `Copy-Item ...\build\libs\steves_army-0.1.0-alpha.jar C:\Users\lauya\curseforge\minecraft\Instances\Test\mods\ -Force`

## Architecture

**Entrypoint:** `steves_army/src/main/java/com/stevesarmy/StevesArmyMod.java`
**MODID:** `steves_army`, Forge 1.20.1, Parchment mappings

```
steves_army/src/main/java/com/stevesarmy/
├── StevesArmyMod.java       # @Mod entry, registers entities/items/menus/commands
├── StevesArmyConfig.java    # ForgeConfigSpec: accuracy, shot threshold, target switch
├── combat/                  # Core combat (gun integration, detection, threat, aim)
│   └── cover/               # Cover system: Manager, Finder, Evaluator, Suppression, etc.
├── command/                 # /stevesarmy_debug and /stevesarmy_cover commands
├── entity/                  # SoldierEntity (PathfinderMob + 9-slot Container), TargetEntity
│   └── ai/                  # CoverTacticalGoal, SoldierCombatGoal, Follow/Hold/Stroll/Ping
├── client/                  # Key bindings (G=toggle, H=debug, MB3=ping wheel), renderers, screens
├── inventory/               # 9-slot SoldierInventory, menu, screen
├── item/                    # RecruitItem, spawn eggs
├── network/                 # Packets: squad mode, debug, inventory, ping, targets
├── ping/                    # Ping system (look-at, move-to, enemy-spotted, etc.)
├── squad/                   # SquadData, SquadManager (FOLLOW/HOLD modes)
├── registry/                # ModEntities, ModItems, ModMenuTypes
└── util/                    # MathUtils, RateLimiter, ScreenPos
```

**Key bindings:** G = toggle Follow/Hold, H = debug, Middle Mouse = ping wheel

## Key Decisions

- **Simplified scope:** Follow/Hold commands only (no Assault mode), all riflemen
- **Cover system is critical** and has its own state machine: NO_COVER → SEEKING → IN_COVER → SUPPRESSED_IN_COVER → REPOSITIONING
- **Peek system is 3-state:** HIDING → EXPOSED → DUCKING_BACK. Half-cover = stance-only pop-up. Full-cover = side-step 1 block with velocity manipulation. Both return to original cover position
- **LOS validation before peek:** Half-cover checks LOS at standing eye height. Full-cover checks LOS from each candidate peek position. Cover is marked non-peekable if no valid angle exists, triggering reposition after ~8s
- **Cover scoring prefers fightable positions:** `FIGHTABILITY_BONUS` (0.2) added to cover score when `canShootFrom()` is true
- **SoldierCombatGoal is simplified:** Peek state machine moved to CoverTacticalGoal. SoldierCombatGoal only checks `peekState == EXPOSED` before shooting
- **TaCZ via reflection:** `GunIntegration.java` wraps reflection calls; graceful fallback to melee
- **Inventory-based ammo:** Soldiers use ammo from their 9-slot inventory, not dummy ammo
- **TaCZ dependency is optional** (`mods.toml: mandatory=false`)
- Reference mods in `ReferenceMod/` for study (not compiled, not our code)

## Commands

| Command | Purpose |
|---------|---------|
| `/stevesarmy_debug` | Combat debug info |
| `/stevesarmy_cover` | Cover system debug (state, threats, reservations, force-peek) |

## ShootResult (TaCZ)

`SUCCESS`, `NEED_BOLT`, `NO_AMMO`, `COOL_DOWN`, `IS_BOLTING`, `IS_RELOADING`, `IS_DRAWING`, `NOT_DRAWN`

## Accuracy (InaccuracyType)

STAND=5.0, MOVE=5.75, SNEAK=3.5, LIE=2.5, AIM=0.15 — soldiers should aim before shooting.

## Plan Docs

| File | Content |
|------|---------|
| `steves_army_plan.md` | Full MVP roadmap, feature specs, AI behavior |
| `steves_army_cover_architecture.md` | Refactored single-responsibility cover system |
| `cover_system_plan.md` | Cover implementation details (scoring, phases) |
| `combat/cover/COVER_SYSTEM.md` | Cover detection algorithm, height classification |

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

## Peek System Flow

```
IN_COVER
  ├── HIDING:
  │   ├── Check cooldown (time since last peek)
  │   ├── Check target exists
  │   ├── HALF cover: LOS check from standing eye height → EXPOSED if clear
  │   ├── FULL cover: velocity slide to peek position → EXPOSED when reached
  │   └── No valid peek → mark non-peekable, reposition after ~8s
  │
  ├── EXPOSED:
  │   ├── Soldier standing (crawl=false)
  │   ├── Can shoot (SoldierCombatGoal checks peekState == EXPOSED)
  │   ├── Duck back after EXPOSURE_TIME_MS (1500ms)
  │   └── Duck back sooner if target dies
  │
  └── DUCKING_BACK:
      ├── HALF: instant transition after 200ms
      └── FULL: velocity slide back to cover position
      └── Sets lastPeekEndTime, resets to HIDING
```
