# Steve's Army - Agent Instructions

## Build & Run

```powershell
# JDK 17 required - set every session
$env:JAVA_HOME = "C:\Program Files\Java\jdk-17"
cd steves_army
.\gradlew build
.\gradlew runClient        # first time only: .\gradlew prepareRuns
```

- **Workdir matters:** gradlew is in `steves_army/`, NOT repo root
- **Build config:** `-Xmx3G` heap, `org.gradle.daemon=false` (gradle.properties)
- **TaCZ dependency:** `libs/tacz-1.20.1-1.1.8-release.jar` loaded via `fileTree` (compile-time only)
- **Mixin processor:** `annotationProcessor 'org.spongepowered:mixin:0.8.5:processor'`
- Build jar: `steves_army/build/libs/steves_army-0.1.0-alpha.jar`
- Copy to test instance: `Copy-Item steves_army\build\libs\steves_army-0.1.0-alpha.jar C:\Users\lauya\curseforge\minecraft\Instances\Test\mods\ -Force`
- Logs: `C:\Users\lauya\curseforge\minecraft\Instances\Test\logs` or `C:\Modding\Steve's_Army_parent\Debug`

## Coding guidelines
- Do not always choose the easiest or quickest fix. Prioritize correctness and long-term stability.
- Perform deeper investigation when debugging issues. Do not assume the first explanation is the root cause.
- Verify and confirm root causes before applying changes. Use evidence from logs, code behavior, or system structure.
- Consider broader system impact before implementing changes. Avoid local fixes that may introduce architectural inconsistency.
- Prefer systematic solutions over patchwork fixes.
- When multiple solutions exist, evaluate trade-offs (performance, maintainability, scalability) before deciding.
- If uncertainty exists, explore multiple hypotheses instead of committing to a single assumption.
- Do research online and compare/explore existing solutions if needed.
- Do not blindly follow user's instruction. When you think the user's instruction is problematic, or there are better alternatives, discuss with the user about it first.

## Architecture

**Entrypoint:** `steves_army/src/main/java/com/stevesarmy/StevesArmyMod.java`
**MODID:** `steves_army`, Forge 1.20.1-47.4.0, Parchment mappings 2023.08.20-1.20.1

```
steves_army/src/main/java/com/stevesarmy/
├── StevesArmyMod.java       # @Mod entry, registers entities/items/menus/commands/creative tab
├── StevesArmyConfig.java    # ForgeConfigSpec: 14 fields across aim_quality, friendly_fire, threat_system
├── combat/                  # Core combat (GunIntegration, AimAccuracyManager, DetectionSystem, ThreatTracker, etc.)
│   └── cover/               # Cover system: CoverTacticalGoal delegates to PeekController + CoverPositionController
├── command/                 # /stevesarmy, /stevesarmy_debug, /stevesarmy_cover commands
├── entity/                  # SoldierEntity, EnemySoldierEntity, TargetEntity (each with custom AI)
│   └── ai/                  # CoverTacticalGoal, SoldierCombatGoal, PeekController, CoverPositionController, etc.
├── client/                  # Key bindings, renderers, debug overlays, formation/ping wheels
├── inventory/               # 9-slot SoldierInventory, menu, screen
├── item/                    # RecruitItem, spawn eggs (soldier, enemy_soldier, target)
├── mixins/                  # GunDisplayInstanceMixin (client-side TaCZ integration)
├── network/                 # Packets: squad mode, formation, debug, inventory, ping, CQB toggle
├── ping/                    # Ping system (Ping, PingManager, PingType)
├── respawn/                 # Player death handling, respawn camera, soldier respawn management
├── squad/                   # SquadData, SquadManager, SquadFormation, SquadCoverContext, etc.
├── registry/                # ModEntities, ModItems, ModCreativeTab, ModMenuTypes
└── util/                    # MathUtils, RateLimiter, FormationPositionCalculator, etc.
```

**Key bindings:** G = formation wheel, H = debug, Middle Mouse = ping wheel

## Key Architecture Decisions

- **Scope:** All riflemen, Follow/Hold + CQB modes (no separate Assault mode, but CQB toggle exists)
- **Cover system:** 5-state machine: NO_COVER → SEEKING_COVER → IN_COVER → SUPPRESSED_IN_COVER → REPOSITIONING
- **Peek system:** 4-state in `PeekController`: HIDING → MOVING_TO_PEEK → EXPOSED → RETURNING_TO_COVER
- **Peek architecture:** `PeekController` handles state machine; `CoverPositionController` (extends MoveControl) handles velocity-based sliding
- **CoverType:** NONE, CONCEALMENT, HALF, FULL (no SHOOTABLE)
- **Half-cover peek:** stance-only pop-up via `GunIntegration.lowCrouch()`
- **Full-cover peek:** side-step slide to peek position with velocity manipulation
- **LOS validation before peek:** Both types check LOS; no valid peek → mark non-peekable, reposition after ~40 ticks
- **Cover scoring:** FIGHTABILITY_BONUS (0.2) added when `canShootFrom()` is true
- **SoldierCombatGoal:** Checks `peekState == EXPOSED` before shooting; peek state machine is in PeekController
- **TaCZ integration:** Reflection-based via `GunIntegration`; graceful fallback to melee if TaCZ missing
- **TaCZ dependency:** Optional (`mods.toml: mandatory=false, ordering=AFTER, version [1.1.0,)`)
- **Mixin:** `GunDisplayInstanceMixin` for client-side TaCZ gun display
- **Inventory-based ammo:** Soldiers use ammo from 9-slot inventory
- **Formation system:** Formation wheel, SquadFormation, FormationPositionCalculator, FormationMessage packets
- **EnemySoldierEntity:** Hostile soldier with EnemyDefendGoal, separate spawn egg
- **Aim system:** `AimAccuracyManager` with continuous aim quality model (replaces old InaccuracyType constants)

## Commands

| Command | Purpose |
|---------|---------|
| `/stevesarmy` | Main command (likely debug/utility) |
| `/stevesarmy_debug` | Combat debug info |
| `/stevesarmy_cover` | Cover system debug (state, threats, reservations, force-peek) |

## ShootResult (TaCZ)

`SUCCESS`, `NEED_BOLT`, `NO_AMMO`, `COOL_DOWN`, `IS_BOLTING`, `IS_RELOADING`, `IS_DRAWING`, `NOT_DRAWN`

## Config (StevesArmyConfig.java)

SERVER-side config with 14 fields in 3 groups:

| Group | Field | Default | Purpose |
|-------|-------|---------|---------|
| aim_quality | baseAccuracy | 0.50 | Max aim quality under ideal conditions |
| aim_quality | thresholdScale | 0.35 | Fraction of aim quality before firing |
| aim_quality | slowGunThresholdScale | 0.60 | Higher threshold for bolt-action rifles |
| aim_quality | buildRate | 0.08 | Aim quality increase per tick |
| aim_quality | recoilScale | 0.07 | Per-shot aim penalty from recoil |
| aim_quality | losDecayRate | 0.15 | Decay when target breaks LOS |
| aim_quality | moveDecayRate | 0.02 | Decay while soldier moving |
| aim_quality | targetMovePenalty | 0.05 | Extra decay when target moving |
| aim_quality | switchReset | 0.30 | Aim quality retained on target switch |
| aim_quality | targetSwitchImprovement | 0.20 | Minimum improvement to switch targets |
| aim_quality | targetReevaluateInterval | 20 | Ticks between target re-evaluation |
| friendly_fire | squadFriendlyFire | true | Squad protection for players/soldiers |
| threat_system | smoothBlendFactor | 0.5 | Threat direction adaptation rate |
| threat_system | smoothDecayTimeMs | 60000 | Threat memory decay time |

## Plan Docs

| File | Content |
|------|---------|
| `steves_army_plan.md` | Full MVP roadmap, feature specs, AI behavior |
| `steves_army_cover_architecture.md` | Refactored single-responsibility cover system |
| `cover_system_plan.md` | Cover implementation details (scoring, phases) |
| `steves_army/src/main/java/com/stevesarmy/combat/cover/COVER_SYSTEM.md` | Cover detection algorithm (note: CoverType.SHOOTABLE listed but not in actual code) |

## Cover System Constants

### CoverTacticalGoal.java
```
COVER_REACHED_DISTANCE       = 1.5   blocks
COVER_VALID_DISTANCE         = 2.0   blocks (normal)
COMBAT_COVER_VALID_DISTANCE  = 6.0   blocks (while shooting)
COVER_ABANDON_DISTANCE       = 8.0   blocks (definitely left)
SEARCH_RADIUS                = 12    blocks
MIN_COVER_DWELL_TIME_MS      = 4000  ms
MIN_COVER_DWELL_TIME_DAMAGE_MS = 2000 ms
MIN_SUPPRESSED_DWELL_TIME_MS = 6000  ms
MAX_SEEKING_TIME_MS          = 10000 ms
HYSTERESIS_THRESHOLD         = 0.20  (20% better to switch cover)
MIN_PEEK_INTERVAL_MS         = 2000  ms
NON_PEEKABLE_REPOSITION_TICKS = 40   (~2s before abandoning non-peekable cover)
FLANKING_PROTECTION_THRESHOLD = 0.7  (70% flanking risk)
THREAT_ANGLE_REPOSITION_THRESHOLD = 2.09 (~120° threat shift)
```

### PeekController.java
```
EXPOSURE_TIME_MIN_MS         = 3000  ms minimum exposure
EXPOSURE_TIME_MAX_MS         = 8000  ms maximum exposure (random range)
MIN_EXPOSURE_TIME_MS         = 800   ms absolute minimum
DUCK_COOLDOWN_MS             = 1000  ms between peeks
SUPPRESSED_HIDE_EXTRA_MS     = 2000  ms extra hide when suppressed
PEEK_REACHED_DISTANCE        = 0.05  blocks
RETURN_REACHED_DISTANCE      = 0.5   blocks
PEEK_SPEED                   = 0.75  blocks/tick (slide to peek)
RETURN_SPEED                 = 1.0   blocks/tick (slide back)
NON_PEEKABLE_REPOSITION_TICKS = 40   (~2s, same as CoverTacticalGoal)
```

### SuppressionTracker.java (from COVER_SYSTEM.md - verify if still accurate)
```
DECAY_RATE                   = 0.15  base/tick (0.015/s in cover, 0.0075/s out)
SUPPRESSED_THRESHOLD         = 0.5   (suppressed)
PINNED_THRESHOLD             = 0.7   (pinned)
MIN_PEEK_TIME_MS             = 2500  ms before can peek after suppression
DIRECT_FIRE_SUPPRESSION      = 0.3   per incoming fire event
DAMAGE_SUPPRESSION           = 0.4   per damage taken
NEAR_MISS_SUPPRESSION        = 0.25  per near miss within 3 blocks
```

### CoverFinder.java (Scoring Weights - from COVER_SYSTEM.md)
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
  │   ├── Check cooldown (time since last peek, +2000ms if suppressed)
  │   ├── Check threat direction exists
  │   ├── HALF cover: LOS check from standing eye height → EXPOSED if clear
  │   ├── FULL cover: compute peek position, slide to it (MOVING_TO_PEEK)
  │   └── No valid peek → mark non-peekable, reposition after 40 ticks
  │
  ├── MOVING_TO_PEEK:
  │   ├── Full-cover only: velocity slide toward peek position
  │   ├── REACHED_TARGET → EXPOSED
  │   └── FAILED → back to HIDING
  │
  ├── EXPOSED:
  │   ├── Soldier standing (half-cover: lowCrouch=false)
  │   ├── Can shoot (SoldierCombatGoal checks peekState == EXPOSED)
  │   ├── Duck back after random exposure time (3000-8000ms)
  │   ├── Duck back sooner if target dies or suppressed
  │   └── Full-cover: stop movement, clear navigation
  │
  └── RETURNING_TO_COVER:
      ├── HALF: instant transition after 200ms → HIDING
      ├── FULL: velocity slide back to cover position
      └── REACHED_TARGET → HIDING, sets lastPeekEndTime
```

## Reference Mods

`ReferenceMod/` contains 7 reference projects for study (not compiled, not our code):
- AncientWarfare2, CustomNPC-Plus, Forge MDK, Gunners, Minecraft-Ping-Wheel, recruits, TACZ

## cover_simulator/

Standalone offline test project for cover algorithm simulation. **Note:** Source `.java` files are missing; only compiled `.class` files remain in `build/`. Package: `com.stevesarmy.simulator` (core, cover, entity, suppression, threat, test, world).