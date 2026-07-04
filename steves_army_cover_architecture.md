# Cover System Architecture - Refactored (Single-Responsibility)

## Overview

The cover system uses a **single-responsibility architecture** where:
- **CoverBehaviorManager** = passive data holder (state, cover points, suppression, peek state)
- **CoverTacticalGoal** = behavior owner (state machine, decisions, movement orchestration)
- **CoverPositionController** = custom MoveControl (direct velocity for positioning/peeking/returning)
- **CoverFinder/CoverScorer** = cover math (searching, scoring)
- **CoverReservationManager** = multiplayer/squad coordination

---

## Component Breakdown

### 1. CoverBehaviorManager (Passive Data Holder)

**Location:** `combat/cover/CoverBehaviorManager.java`

**Purpose:** Stores state and provides getters/setters. NO autonomous logic.

**Data:**
- `CoverState state` — Current cover state (NO_COVER, SEEKING_COVER, IN_COVER, SUPPRESSED_IN_COVER, REPOSITIONING)
- `CoverPoint currentCover` — Cover currently occupied
- `CoverPoint targetCover` — Cover being navigated to
- `CoverPoint lastCover` — Recently abandoned cover (hysteresis)
- `long coverEntryTime` — Time entered cover
- `long seekingStartTime` — Time started seeking
- `SuppressionTracker suppressionTracker` — Suppression data
- `PeekState peekState` — HIDING, EXPOSED, DUCKING_BACK
- `BlockPos peekPosition` — Target block for full-cover side-step peek
- `long peekStartTime` — Time entered current peek state
- `long lastPeekEndTime` — Time last peek cycle ended (for cooldown)
- `boolean nonPeekableCover` — Flag: no valid peek angle exists
- `Vec3 entryThreatDirection` — Threat direction when cover was entered
- `int peekCountSameCover` — Peek cycles at same cover (penalty for repositioning)

**States:**
```
NO_COVER              - Default, exposed
SEEKING_COVER         - Moving to cover (no currentCover)
IN_COVER              - Stationary, can peek/shoot
SUPPRESSED_IN_COVER   - Pinned, cannot peek
REPOSITIONING         - Moving to better cover (has currentCover)
```

**Peek States:**
```
HIDING       - Behind cover, waiting to peek
EXPOSED      - Peeking out, can shoot
DUCKING_BACK - Returning to cover after exposure
```

---

### 2. CoverTacticalGoal (Behavior Owner)

**Location:** `entity/ai/CoverTacticalGoal.java`

**Purpose:** Owns the full cover lifecycle state machine and all behavior decisions.

**Goal Priority:** 2

**Goal Flags:** Dynamic (MOVE+LOOK when navigating, none when in cover to avoid vanilla movement interference)

**State Machine:**
```
canUse() checks:
├── NO_COVER → shouldSeekCover() → start SEEKING_COVER
├── SEEKING_COVER → always true
├── IN_COVER → check abandon distance, then true
├── SUPPRESSED_IN_COVER → always true
└── REPOSITIONING → always true

tick() transitions:
├── SEEKING_COVER:
│   ├── reached cover (dist < 1.5)? → setCurrentCover → IN_COVER
│   ├── close (dist < 2.0)? → use POSITIONING (slow walk to exact position)
│   ├── stuck? → blacklist → find another cover
│   └── timeout (10s)? → NO_COVER
│
├── IN_COVER:
│   ├── drifted too far (dist > 8.0)? → clearCover → NO_COVER
│   ├── pushed out (dist > 2.0, not peeking)? → SEEKING_COVER
│   ├── not peeking + controller idle? → set POSITIONING to cover center
│   ├── suppressed? → set DUCKING_BACK → SUPPRESSED_IN_COVER
│   ├── shouldExitCoverForFollow()? → clearCover
│   ├── enemy too close (dist < 4.0)? → startRepositioning()
│   └── tickPeekState() → HIDING/EXPOSED/DUCKING_BACK
│
├── SUPPRESSED_IN_COVER:
│   ├── if DUCKING_BACK → tickDuckingBack()
│   ├── not pinned + canPeek? → IN_COVER
│   └── FOLLOW + not suppressed? → clearCover
│
└── REPOSITIONING:
    ├── reached new cover? → setCurrentCover → IN_COVER
    └── stuck? → back to IN_COVER or SEEKING_COVER
```

---

### 3. CoverPositionController (Custom Movement Control)

**Location:** `entity/ai/CoverPositionController.java`

**Purpose:** Custom `MoveControl` that uses direct `setDeltaMovement()` for precise positioning instead of vanilla pathfinding. Prevents `super.tick()` from fighting custom velocity.

**MovementIntents:**
```
NONE        - Idle. Calls super.tick() only if navigation has active path
NAVIGATING  - Pathfinding. Calls super.tick() only if navigation has active path
POSITIONING - Direct velocity toward target at speed=min(0.08, dist*0.8). Clears to NONE when within tolerance.
PEEKING     - Slide to peek position at 0.15 speed. Clears to NONE when within 0.25 blocks.
RETURNING   - Slide back to cover center at speed=min(0.2, dist*0.6). Clears to NONE when within tolerance.
```

**Key Design Decision:** `super.tick()` is only called when `!navigation.isDone()` (active path). When idle or in custom movement modes, the controller handles everything itself. This prevents vanilla MoveControl from setting `zza=0` and fighting the custom `setDeltaMovement()`.

---

### 4. Peek System (3-state, ticked from tickInCover)

```
tickInCover()
├── Positioning: if not peeking + controller idle → set POSITIONING to cover center
├── Suppressed? → set DUCKING_BACK → SUPPRESSED_IN_COVER → return
├── shouldExitCoverForFollow? → clearCover → return
├── Enemy proximity (< 4.0 blocks)? → startRepositioning() → return
└── tickPeekState()
    └── switch peekState:
        ├── HIDING → tickHiding()
        │   ├── Non-peekable timer (3s) → reposition
        │   ├── doCrawlIfHalfCover() (crawl behind half-cover)
        │   ├── Cooldown check (1s base, +2s if suppressed)
        │   ├── HALF cover: LOS check from standing eye height
        │   │   ├── LOS clear? → clearIntent() → EXPOSED + stand up
        │   │   └── No LOS? → mark nonPeekable
        │   └── FULL cover: side-step peek
        │       ├── Re-evaluate targets + peek position (every 1.5s)
        │       ├── Has valid peekPos + target? → startPeekAt(peekPos) → controller slides
        │       ├── Already at peekPos (dist < 0.5) + LOS? → setPos → EXPOSED
        │       └── No valid peek? → mark nonPeekable
        │
        ├── EXPOSED → tickExposed(timeInState)
        │   ├── Time exceeded (1500ms)? → DUCKING_BACK + doCrawlIfHalfCover()
        │   ├── Target dead + time > min(800ms)? → trySwitchTarget or DUCKING_BACK
        │   ├── LOS lost + time > 200ms? → trySwitchTarget or DUCKING_BACK
        │   └── faceTarget()
        │
        └── DUCKING_BACK → tickDuckingBack()
            └── timeToReturnToCover(cover)
                ├── HALF: timeInState > 200ms → resetPeekState → HIDING
                └── FULL: check distance to cover center
                    ├── dist < 0.5? → resetPeekState + recordPeekCycle → HIDING
                    └── dist >= 0.5? → set POSITIONING to cover center
```

**timeToReturnToCover():** Uses raw `coverPos.getCenter()` (same as `tickInCover()` line 407 positioning target). This is critical — previously used an adjusted target offset by -0.5 in the protected direction, which placed the target farther from the soldier and caused infinite DUCKING_BACK.

---

### 5. SoldierCombatGoal (Cover-Aware)

**Changes from vanilla combat:**
- Checks `coverManager.isInCover()` to modify shooting behavior
- Checks `coverManager.isPinned()` to stay down
- Only shoots when `peekState == EXPOSED`
- Sets target via `setTarget()` (called by CoverTacticalGoal during peek)

---

### 6. CoverType System

**Classification:**
```
HALF Cover: 1-block tall obstacle. Soldier crawls behind, pops up to peek.
FULL Cover: 2+ block tall obstacle. Soldier stands behind, side-steps 1 block to peek.
CONCEALMENT: No ballistic protection but breaks LOS (e.g. foliage).
```

**Detection (CoverFinder.java):**
- Voxel shape check for solid blocks
- Classify based on height (1-block = HALF, 2-block = FULL)
- Score based on weights:
  - PRIMARY_PROTECTION_WEIGHT = 0.30
  - FLANKING_PROTECTION_WEIGHT = 0.20
  - DISTANCE_WEIGHT = 0.10
  - FIRING_QUALITY_WEIGHT = 0.25
  - PEEK_ANGLE_WEIGHT = 0.15
  - FIGHTABILITY_BONUS = 0.20 (added if canShootFrom())

---

### 7. Crawl System Integration

- `doCrawlIfHalfCover()` called in `tickHiding()` and `tickDuckingBack()` for half-cover
- `clearCover()` always calls `GunIntegration.crawl(soldier, false)` — single exit point, prevents crawling pose from persisting

---

## Key Constants

### CoverTacticalGoal.java
| Constant | Value | Purpose |
|----------|-------|---------|
| `COVER_REACHED_DISTANCE` | 1.5 | Transition to IN_COVER |
| `COVER_VALID_DISTANCE` | 2.0 | Still in cover (normal) |
| `COMBAT_COVER_VALID_DISTANCE` | 6.0 | Still in cover (shooting) |
| `COVER_ABANDON_DISTANCE` | 8.0 | Definitely left cover |
| `MIN_COVER_DWELL_TIME_MS` | 4000 | Minimum time before reposition allowed |
| `MIN_SUPPRESSED_DWELL_TIME_MS` | 6000 | Minimum time when suppressed |
| `MIN_PEEK_INTERVAL_MS` | 2000 | Time between peeks |
| `REEVALUATE_INTERVAL_TICKS` | 60 | Periodic cover quality check (~3s) |
| `HYSTERESIS_THRESHOLD` | 0.35 | 35% better to switch cover |
| `MAX_SEEKING_TIME_MS` | 10000 | Seeking timeout |
| `LOW_HEALTH_THRESHOLD` | 0.3 | Health % to seek cover |
| `FOLLOW_COVER_SEARCH_RADIUS` | 15.0 | Prefer cover near player |
| `FOLLOW_REGROUP_DISTANCE` | 10.0 | Exit cover if player this far |
| `NON_PEEKABLE_REPOSITION_TICKS` | 60 | Wait ~3s before abandoning non-peekable cover |
| `PEEK_NO_LOS_REPOSITION_TICKS` | 100 | Wait ~5s of no LOS before reposition |
| `FULL_COVER_PEEK_SPEED` | 0.15 | Velocity slide (blocks/tick) |
| `PEEK_POSITION_REACHED_DISTANCE` | 0.5 | Tolerance for peek position reached |
| `PEEK_SEARCH_RADIUS` | 2 | Block radius for full-cover peek candidates |
| `ENEMY_PROXIMITY_REPOSITION_DISTANCE` | 4.0 | Reposition if enemy this close |
| `THREAT_ANGLE_REPOSITION_THRESHOLD` | 2.09 rad (120°) | Reposition if threat angle changes this much |

### CoverBehaviorManager.java (Peek)
| Constant | Value | Purpose |
|----------|-------|---------|
| `EXPOSURE_TIME_MS` | 1500 | Max exposure during peek |
| `MIN_EXPOSURE_TIME_MS` | 800 | Minimum exposure before early duck |
| `DUCK_COOLDOWN_MS` | 1000 | Cooldown between peeks |
| `SUPPRESSED_HIDE_EXTRA_MS` | 2000 | Extra hide time when suppressed |
| `PEEK_COUNT_PENALTY_THRESHOLD` | 4 | Peek count before cover quality penalty |
| `MAX_COVER_PENALTY` | 0.60 | Max penalty for overused cover |

### CoverPositionController.java
| Constant | Value | Purpose |
|----------|-------|---------|
| POSITIONING speed | min(0.08, dist*0.8) | Slow walk to exact position |
| PEEKING speed | 0.15 | Slide to peek position |
| RETURNING speed | min(0.2, dist*0.6) | Slide back to cover |
| PEEKING tolerance | 0.25 | Distance to snap to peek position |

---

## Hysteresis Implementation

```
IN_COVER → isCoverStillValid() fails
         → findBetterCover() (not same as current)
         → setTargetCover(newCover), setState(REPOSITIONING)
         → currentCover preserved for hysteresis

CoverBehaviorManager:
  currentCover    // Occupied cover
  targetCover     // Navigation target  
  lastCover       // Recently abandoned

findAndMoveToCover() skips failedCoverPositions (blacklist)
startRepositioning() preserves currentCover until new cover reached
```

---

## FOLLOW Mode Cover Behavior

```
FOLLOW + NO_COMBAT:     Don't seek cover, stay close to player
FOLLOW + HAS_TARGET:    Take nearby cover (within 15 blocks of player)
FOLLOW + SUPPRESSED:    Take ANY cover, ignore player distance

FOLLOW + IN_COVER + NOT_SUPPRESSED:
├── Player > 10 blocks away → Exit cover, follow player
├── Player < 10 blocks away → Stay in cover, engage target
└── No target → Exit cover, follow player

FOLLOW + IN_COVER + SUPPRESSED:
└── Stay in cover, ignore player distance
```

---

## Data Flow

```
SoldierEntity.tick()
  └── Goal Selector runs CoverTacticalGoal

CoverTacticalGoal.tick()
  ├── coverManager.tickSuppression(inCover)
  ├── switch (coverManager.getState()):
  │   ├── SEEKING_COVER → navigate, check reached/stuck/timeout
  │   ├── IN_COVER → validate, position, tickPeekState()
  │   ├── SUPPRESSED_IN_COVER → tickDuckingBack(), check canPeek
  │   └── REPOSITIONING → navigate to new cover
  └── coverManager.setState/setCurrentCover/setTargetCover (Goal decides)

CoverPositionController.tick()  (runs after goals, via LivingEntity.aiStep)
  ├── NONE/NAVIGATING → super.tick() only if navigation has active path
  ├── POSITIONING → direct velocity toward target
  ├── PEEKING → slide velocity toward peek position
  └── RETURNING → slide velocity toward cover center

SoldierCombatGoal checks:
  - peekState == EXPOSED → can shoot
  - coverManager.isPinned() → stay down
```

---

## Debug Commands

All debug is now under `/stevesarmy_debug`:

| Command | Purpose |
|---------|---------|
| `/stevesarmy_debug all` | Enable ALL debug (logging + render + combat overlay) |
| `/stevesarmy_debug log cover [on\|off]` | Toggle cover behavior logging |
| `/stevesarmy_debug render soldiers` | Toggle soldier cover viz lines/labels |
| `/stevesarmy_debug render peekcandidates` | Toggle peek candidate viz |
| `/stevesarmy_debug render rays` | Toggle raycast viz |
| `/stevesarmy_debug render solid` | Toggle solid block viz |
| `/stevesarmy_debug render coverpoints` | Toggle cover point wireframes |
| `/stevesarmy_debug render mode [off\|minimal\|verbose]` | Combat overlay mode |
| `/stevesarmy_debug info state` | Cover state + peek timing + movement intent + ctrlDist + speed |
| `/stevesarmy_debug info threats` | Threat direction analysis |
| `/stevesarmy_debug info reservations` | Cover reservations |
| `/stevesarmy_debug info suppression` | Suppression levels |
| `/stevesarmy_debug info scan [radius]` | Scan for cover points |
| `/stevesarmy_debug info target [entity]` | Scan with threat |
| `/stevesarmy_debug info best [radius]` | Find best cover |
| `/stevesarmy_debug info debug <x> <y> <z>` | Debug specific position |
| `/stevesarmy_debug control peek` | Force peek |
| `/stevesarmy_debug control reposition` | Force reposition |
| `/stevesarmy_debug control pose [...]` | Pose commands (crawl, stand, status, noai, angles, set, reset) |
| `/stevesarmy_debug status` | Show all toggle states |

`/stevesarmy debug` still works as shortcut for `/stevesarmy_debug all`.

---

## Files

| File | Location | Purpose |
|------|----------|---------|
| `CoverBehaviorManager.java` | combat/cover/ | Passive data holder (state, cover, suppression, peek) |
| `CoverTacticalGoal.java` | entity/ai/ | Full state machine, peek system, decisions |
| `CoverPositionController.java` | entity/ai/ | Custom MoveControl (direct velocity positioning) |
| `CoverFinder.java` | combat/cover/ | Cover search and scoring |
| `CoverQualityEvaluator.java` | combat/cover/ | Cover raycast quality evaluation |
| `CoverPoint.java` | combat/cover/ | Cover position data class |
| `CoverType.java` | combat/cover/ | HALF/FULL/CONCEALMENT/NONE enum |
| `CoverReservationManager.java` | combat/cover/ | Multiplayer cover coordination |
| `CoverDebugManager.java` | combat/cover/ | Debug visualization helpers |
| `SuppressionTracker.java` | combat/cover/ | Suppression decay and thresholds |
| `PoseConfig.java` | client/model/ | Prone pose adjustable angles |
| `SoldierModel.java` | client/model/ | Custom HumanoidModel with prone pose |
| `SoldierRenderer.java` | client/renderer/ | swimAmount-driven setupRotations |
| `GunIntegration.java` | combat/ | TaCZ reflection wrapper (includes crawl) |
| `CoverDebugRenderer.java` | client/ | Cover debug overlay rendering |
| `CombatDebugRenderer.java` | client/ | Combat detection debug overlay |
| `CombatDebugCommand.java` | command/ | `/stevesarmy_debug` command |
| `StevesArmyCommand.java` | command/ | `/stevesarmy` command (redirect) |

---

## Key Bug Fixes

| Bug | Root Cause | Fix | File |
|-----|-----------|-----|------|
| Soldier stuck in DUCKING_BACK forever | `timeToReturnToCover()` used target offset by -0.5 in protected direction, making it farther from soldier. Combined with `tickInCover()` targeting raw center, created dual-target oscillation | Use raw `coverPos.getCenter()` in `timeToReturnToCover()`, same as `tickInCover()` line 407 | CoverTacticalGoal.java |
| Soldier frozen, no movement during peek | `CoverPositionController.tick()` had empty NONE/NAVIGATING cases, vanilla `MoveControl.tick()` never ran | Call `super.tick()` for NONE/NAVIGATING when navigation has active path | CoverPositionController.java |
| Controller velocity fighting super.tick() | `super.tick()` running unconditionally set `zza=0`, friction killed custom velocity | Only call `super.tick()` when `!navigation.isDone()` | CoverPositionController.java |
| Soldier walks while crawling after cover exit | `clearCover()` didn't disable crawl | `clearCover()` calls `GunIntegration.crawl(false)` | CoverBehaviorManager.java |
