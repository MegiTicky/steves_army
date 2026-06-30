# Cover System Architecture - Refactored (Single-Responsibility)

## Overview

The cover system now uses a **single-responsibility architecture** where:
- **CoverBehaviorManager** = passive data holder (state, cover points, suppression)
- **CoverTacticalGoal** = behavior owner (state machine, decisions, movement)
- **CoverFinder/CoverScorer** = cover math (searching, scoring)
- **CoverReservationManager** = multiplayer/squad coordination

This eliminates the race condition between dual state management that caused oscillation.

---

## Component Breakdown

### 1. CoverBehaviorManager (Passive Data Holder)

**Location:** `CoverBehaviorManager.java`

**Purpose:** Stores state and provides getters/setters. NO autonomous logic.

**Data:**
```
CoverState state            - Current state
CoverPoint currentCover     - Cover currently occupied
CoverPoint targetCover      - Cover being navigated to
CoverPoint lastCover        - Recently abandoned cover (hysteresis)
long coverEntryTime         - Time entered cover
long seekingStartTime       - Time started seeking
SuppressionTracker          - Suppression data
ThreatDirectionCalculator   - Threat analysis helper
```

**States:**
```
NO_COVER           - Default, exposed
SEEKING_COVER      - Moving to cover (no currentCover)
IN_COVER           - Stationary, can peek/shoot
SUPPRESSED_IN_COVER - Pinned, cannot peek
REPOSITIONING      - Moving to better cover (has currentCover)
```

**Key Methods:**
| Method | Returns | Purpose |
|--------|---------|---------|
| `getState()` | CoverState | Query current state |
| `setState()` | void | Set state (called by Goal) |
| `getCurrentCover()` | CoverPoint | Query occupied cover |
| `setCurrentCover()` | void | Set cover (called by Goal) |
| `getTargetCover()` | CoverPoint | Query navigation target |
| `setTargetCover()` | void | Set target (called by Goal) |
| `getLastCover()` | CoverPoint | Query abandoned cover (hysteresis) |
| `clearCover()` | void | Clear cover and set NO_COVER |
| `tickSuppression()` | void | Tick suppression tracker only |

**NO:**
- `tick()` method (removed)
- `handleNoCover()`, `handleInCover()` (removed)
- `transitionTo()` (removed)
- Autonomous state decisions

---

### 2. CoverTacticalGoal (Behavior Owner)

**Location:** `CoverTacticalGoal.java` (renamed from SeekCoverGoal)

**Purpose:** Owns the full cover lifecycle state machine and all behavior decisions.

**Goal Priority:** 2

**Goal Flags:** Dynamic (MOVE+LOOK when navigating, LOOK only when in cover)

**State Machine (owned by Goal):**
```
canUse() checks:
├── NO_COVER → shouldSeekCover() → start SEEKING_COVER
├── SEEKING_COVER → always true (goal running)
├── IN_COVER → shouldContinueInCover() → may exit or stay
├── SUPPRESSED_IN_COVER → always true
└── REPOSITIONING → always true

tick() transitions:
├── SEEKING_COVER:
│   ├── reached cover? → setCurrentCover → IN_COVER
│   ├── stuck? → findAndMoveToCover
│   └── timeout? → NO_COVER
│
├── IN_COVER:
│   ├── suppressed? → SUPPRESSED_IN_COVER
│   ├── cover invalid? → startRepositioning
│   ├── better cover? (hysteresis) → startRepositioning
│   ├── FOLLOW + not suppressed + player far? → clearCover → NO_COVER
│   └── can peek? → set MOVE+LOOK flags temporarily
│
├── SUPPRESSED_IN_COVER:
│   ├── not pinned + canPeek? → IN_COVER
│   └── FOLLOW + not suppressed? → clearCover
│
└── REPOSITIONING:
    ├── reached new cover? → setCurrentCover → IN_COVER
    └── stuck? → back to IN_COVER
```

**Key Methods:**
| Method | Purpose |
|--------|---------|
| `canUse()` | Cheap check: needs cover or already in cover |
| `canContinueToUse()` | True while SEEKING/IN_COVER/SUPPRESSED/REPOSITIONING |
| `start()` | Find and navigate to cover |
| `tick()` | Run state machine, manage flags |
| `stop()` | Cleanup reservations |
| `shouldSeekCover()` | Decision: should seek cover? |
| `shouldExitCoverForFollow()` | Decision: should leave cover to follow player? |
| `canPeekAndShoot()` | Decision: can expose to shoot? |
| `isCoverStillValid()` | Decision: is current cover still good? |
| `evaluateCoverState()` | Periodic cover quality check |
| `startRepositioning()` | Move to better cover |
| `findAndMoveToCover()` | Cover search and navigation |

**Dynamic Flag Management:**
```java
tick() {
    if (state == IN_COVER && !canPeekAndShoot()) {
        setFlags(EnumSet.of(Flag.LOOK)); // No MOVE, allow follow goal
    } else if (state == SEEKING_COVER || wantsToPeek) {
        setFlags(EnumSet.of(Flag.MOVE, Flag.LOOK));
    }
}
```

---

### 3. SoldierCombatGoal (Cover-Aware)

**Changes:**
```java
tickGunCombat() {
    CoverBehaviorManager coverManager = soldier.getCoverBehaviorManager();
    
    if (coverManager.isInCover()) {
        if (coverManager.isPinned()) {
            // Stay down, don't shoot
            GunIntegration.aim(soldier, false);
            GunIntegration.crawl(soldier, true);
            return;
        }
        
        if (!coverManager.getSuppressionTracker().canPeek()) {
            // Can't peek yet
            return;
        }
        
        // Pop up to shoot
        GunIntegration.crawl(soldier, false);
    }
    
    // Normal shooting logic...
    
    if (shotSuccess && coverManager.isInCover()) {
        coverManager.onPeekShot(); // Reset peek timer
    }
}
```

---

## Key Constants (Updated)

| Constant | Value | Purpose |
|----------|-------|---------|
| `COVER_REACHED_DISTANCE` | 1.5D | Distance to consider reached |
| `COVER_VALID_DISTANCE` | 5.0D | Still considered in cover |
| `COMBAT_COVER_VALID_DISTANCE` | 6.0D | In cover while shooting |
| `COVER_ABANDON_DISTANCE` | 8.0D | Definitely left cover |
| `MIN_COVER_DWELL_TIME_MS` | 4000 | Minimum time in cover |
| `MIN_SUPPRESSED_DWELL_TIME_MS` | 6000 | Minimum time when suppressed |
| `MIN_PEEK_INTERVAL_MS` | 2000 | Time between peeks |
| `REEVALUATE_INTERVAL_TICKS` | 60 | Periodic cover check (3s) |
| `HYSTERESIS_THRESHOLD` | 0.35f | 35% better to switch cover |
| `MAX_SEEKING_TIME_MS` | 10000 | Timeout for seeking |
| `LOW_HEALTH_THRESHOLD` | 0.3f | Health % to seek cover |
| `FOLLOW_MAX_COMBAT_DISTANCE` | 20.0D | Max distance during combat |
| `FOLLOW_COVER_SEARCH_RADIUS` | 15.0D | Prefer cover near player |
| `FOLLOW_REGROUP_DISTANCE` | 10.0D | Exit cover if player this far |

---

## Hysteresis Implementation

**Problem (Before):**
```
IN_COVER → isCoverStillValid() fails
         → transitionTo(SEEKING_COVER) clears currentCover
         → findAndMoveToCover() has no currentCover for hysteresis
         → finds same cover again
         → oscillates
```

**Solution (Now):**
```java
CoverPoint currentCover    // Occupied cover
CoverPoint targetCover     // Navigation target
CoverPoint lastCover       // Recently abandoned

// In findAndMoveToCover():
if (lastCover != null && cover.getPosition().equals(lastCover.getPosition())) {
    // Skip recently abandoned cover
    return;
}

// In startRepositioning():
// currentCover is NOT cleared - preserved for hysteresis
coverManager.setTargetCover(newCover);
coverManager.setState(REPOSITIONING);
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

## Layered Distance Thresholds

```
≤ 1.5 blocks: COVER_REACHED → transition to IN_COVER
≤ 5.0 blocks: COVER_VALID → still in cover (normal)
≤ 6.0 blocks: COMBAT_COVER_VALID → still in cover (shooting)
> 8.0 blocks: COVER_ABANDON → definitely left cover

If 5-8 blocks AND timeInCover > MIN_DWELL_TIME → reposition
If > 8 blocks → abandon cover immediately
```

This prevents tiny combat/pathfinding movement from invalidating state.

---

## Data Flow (Single Source of Truth)

```
┌──────────────────────────────────────────────────────────────┐
│                    SoldierEntity                              │
│                                                              │
│  tick() {                                                    │
│      super.tick();                                           │
│      // NO coverBehaviorManager.tick() call                  │
│  }                                                           │
│                                                              │
│  Goal Selector picks CoverTacticalGoal                       │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              CoverTacticalGoal.tick()                  │ │
│  │                                                        │ │
│  │  coverManager.tickSuppression(inCover)                │ │
│  │                                                        │ │
│  │  switch (coverManager.getState()) {                   │ │
│  │      SEEKING_COVER → navigate, check reached          │ │
│  │      IN_COVER → evaluate, manage flags                │ │
│  │      SUPPRESSED → manage suppression decay            │ │
│  │      REPOSITIONING → navigate to new cover            │ │
│  │  }                                                    │ │
│  │                                                        │ │
│  │  coverManager.setState(newState) // Goal decides       │ │
│  │  coverManager.setCurrentCover(cover) // Goal decides   │ │
│  │  coverManager.setTargetCover(cover) // Goal decides    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  SoldierCombatGoal checks:                                   │
│      coverManager.isInCover() → modify shooting             │
│      coverManager.isPinned() → don't shoot                  │
│      coverManager.onPeekShot() → after successful shot      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Key Principle:**
- CoverBehaviorManager = MEMORY (passive)
- CoverTacticalGoal = BEHAVIOR (active)
- Single decision point = no race conditions

---

## Debug Commands

| Command | Purpose |
|---------|---------|
| `/stevesarmy_cover state` | Show cover state for nearby soldiers |
| `/stevesarmy_cover log on` | Enable cover behavior logging |
| `/stevesarmy_cover soldiers` | Toggle soldier cover visualization |
| `/stevesarmy_cover threats` | Show threat direction analysis |
| `/stevesarmy_cover reservations` | Show cover reservations |

---

## Files Modified

| File | Change |
|------|--------|
| `CoverBehaviorManager.java` | Refactored to passive data holder |
| `CoverTacticalGoal.java` | New file, owns full state machine |
| `SeekCoverGoal.java` | Deleted (replaced by CoverTacticalGoal) |
| `SoldierEntity.java` | Updated goal registration, removed tick() call |
| `SoldierCombatGoal.java` | Made cover-aware (check state before shooting) |
| `CoverDebugCommand.java` | Updated references to CoverTacticalGoal |

---

## Future Improvements

1. **Bounding Overwatch**: One soldier moves while others cover
2. **Cover-to-cover movement**: Planned tactical movement
3. **Peek direction optimization**: Face threat direction while in cover
4. **Suppression per-threat**: Track which enemy is suppressing most
5. **Cover destruction detection**: Detect when cover block is destroyed