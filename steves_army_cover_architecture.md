# Cover System Architecture - Complete Flow

## Overview

The cover system has **two separate systems** that sometimes conflict:

1. **CoverBehaviorManager** (State Machine) - Runs every tick in `SoldierEntity.tick()`
2. **SeekCoverGoal** (AI Goal) - Runs via Minecraft's goal selector

---

## Component Breakdown

### 1. CoverBehaviorManager (State Machine)

**Location:** `CoverBehaviorManager.java`

**Purpose:** Tracks cover state and makes decisions about cover transitions.

**States:**
```
NO_COVER → SEEKING_COVER → IN_COVER → SUPPRESSED_IN_COVER
                ↑              ↓
                └──────────────┘
```

**Tick Flow (runs every game tick via `SoldierEntity.tick()`):**

```
tick(soldier)
├── suppressionTracker.tick(inCover)
└── switch (state)
    ├── NO_COVER → handleNoCover()
    │   └── if shouldSeekCover() → transitionTo(SEEKING_COVER)
    │
    ├── SEEKING_COVER → handleSeekingCover()
    │   ├── if seekingTime > 10s → transitionTo(NO_COVER)
    │   └── if currentCover != null && distance < 2.0 → transitionTo(IN_COVER)
    │
    ├── IN_COVER → handleInCover()
    │   ├── if FOLLOW mode && !suppressed → clearCover() → NO_COVER
    │   ├── if pinned → transitionTo(SUPPRESSED_IN_COVER)
    │   └── if !isCoverStillValid() → transitionTo(SEEKING_COVER) ⚠️
    │
    └── SUPPRESSED_IN_COVER → handleSuppressed()
        └── if !pinned && canPeek() → transitionTo(IN_COVER)
```

**Key Methods:**

| Method | Returns | Called By |
|--------|---------|-----------|
| `shouldSeekCover(soldier)` | boolean if soldier should seek cover | `handleNoCover()`, `shouldGoalSeekCover()` |
| `shouldGoalSeekCover(soldier)` | boolean if SeekCoverGoal should run | `SeekCoverGoal.canUse()` |
| `isCoverStillValid(soldier)` | boolean if current cover is still valid | `handleInCover()`, `SeekCoverGoal.canUse()` |
| `setCover(cover)` | void | `SeekCoverGoal.onCoverReached()` |
| `clearCover()` | void | `handleInCover()` (FOLLOW mode) |
| `transitionTo(newState)` | void | Internal state transitions |

---

### 2. SeekCoverGoal (AI Goal)

**Location:** `SeekCoverGoal.java`

**Purpose:** Actually moves the soldier to cover using Minecraft's pathfinding.

**Goal Priority:** 2 (higher than combat at 4, lower than ping movement at 1)

**Goal Flags:** `MOVE`, `LOOK`

**Flow:**

```
canUse() → Can the goal start?
├── cooldown > 0? → false
├── not alive? → false
├── FOLLOW mode && !suppressed && health >= 30%? → false
├── targetCover != null && navigation not done? → false
├── isInCover()?
│   ├── timeInCover < 3s? → false
│   └── isCoverStillValid()? → false
└── shouldGoalSeekCover()? → true

start() → Find and navigate to cover
├── stuckTicks = 0
└── findAndMoveToCover()
    ├── Get threats from getThreats() (just current target)
    ├── analyzeThreats() → threat direction
    ├── findBestCover() → search radius 12
    ├── Hysteresis check (new > current * 1.25?)
    ├── Reserve cover
    └── navigation.moveTo()

tick() → Monitor progress
├── distance < 1.5 blocks?
│   └── if !isInCover() → onCoverReached()
│       └── setCover(targetCover)
└── navigation stuck > 60 ticks?
    └── findAndMoveToCover() again

stop() → Cleanup
├── Release cover reservation if not in cover
└── cooldown = 40 ticks
```

---

### 3. Goal Priority & Conflicts

**Goal Registration Order (SoldierEntity:136-148):**
```
Priority 0: FloatGoal (swimming)
Priority 1: SoldierMoveToPingGoal
Priority 2: SeekCoverGoal ⚠️
Priority 3: SoldierFollowOwnerGoal / SoldierHoldPositionGoal
Priority 4: SoldierCombatGoal ⚠️
Priority 5: SoldierStrollGoal
Priority 6: LookAtPlayerGoal
Priority 7: RandomLookAroundGoal
```

**Conflict Analysis:**

| Goal | Flags | Conflict with SeekCover? |
|------|-------|-------------------------|
| SeekCoverGoal | MOVE, LOOK | - |
| SoldierFollowOwnerGoal | MOVE | **YES** - Both use MOVE |
| SoldierHoldPositionGoal | MOVE | **YES** - Both use MOVE |
| SoldierCombatGoal | LOOK | Partial - Combat may move soldier to aim |

**Minecraft Goal System Behavior:**
- Only ONE goal with `MOVE` flag can run at a time
- Higher priority goal preempts lower priority
- But `SeekCoverGoal` (priority 2) can run alongside `SoldierCombatGoal` (priority 4) because combat only has `LOOK` flag

---

## The Core Problem

### Issue 1: Dual State Management

**CoverBehaviorManager maintains state independently of SeekCoverGoal:**

1. **CoverBehaviorManager.tick()** runs every tick and decides state transitions
2. **SeekCoverGoal** only runs when the goal selector picks it

**This creates a race condition:**

```
Tick 1:
  CoverBehaviorManager: IN_COVER → isCoverStillValid() returns false → SEEKING_COVER
  SeekCoverGoal: Not running yet

Tick 2:
  CoverBehaviorManager: SEEKING_COVER → currentCover is null
  SeekCoverGoal: canUse() returns true → start() → findAndMoveToCover()

Tick 3:
  CoverBehaviorManager: SEEKING_COVER → currentCover still null
  SeekCoverGoal: tick() → navigating to cover

Tick 4:
  SeekCoverGoal: tick() → reached cover → onCoverReached() → setCover()
  CoverBehaviorManager: Still SEEKING_COVER

Tick 5:
  CoverBehaviorManager: handleSeekingCover() → currentCover != null → distance < 2.0 → IN_COVER

Tick 6:
  CoverBehaviorManager: handleInCover() → isCoverStillValid() → false → SEEKING_COVER
  (Loop repeats)
```

### Issue 2: isCoverStillValid() is Too Strict

```java
public boolean isCoverStillValid(SoldierEntity soldier) {
    if (currentCover == null) return false;
    
    double distance = soldier.position().distanceTo(currentCover.getPosition().getCenter());
    if (distance > MAX_DISTANCE_TO_COVER * 1.5) {  // 2.0 * 1.5 = 3.0 blocks
        return false;  // ⚠️ Very strict!
    }
    
    // FOLLOW mode check...
    return true;
}
```

**The problem:** 3 blocks is very strict. If the soldier:
- Turns to aim at a target
- Takes a step while shooting
- Moves slightly for pathfinding

...they can exceed 3 blocks from cover center, triggering a re-seek.

### Issue 3: transitionTo(SEEKING_COVER) Clears currentCover

```java
private void transitionTo(CoverState newState) {
    if (newState == CoverState.SEEKING_COVER) {
        this.seekingStartTime = System.currentTimeMillis();
        this.coverEntryTime = 0;
        if (currentCover != null) {
            CoverReservationManager.release(currentCover.getPosition(), null);
        }
        this.currentCover = null;  // ⚠️ Cleared!
    }
}
```

**This breaks hysteresis:** When `SeekCoverGoal.findAndMoveToCover()` runs, it checks `getCoverManager().getCurrentCover()` for hysteresis. But `currentCover` is now `null`, so the soldier seeks the same cover again!

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SoldierEntity.tick()                               │
│                                                                              │
│  1. super.tick() ──────────────────────────────────────────────────────────►│
│  2. coverBehaviorManager.tick(this) ───────────────────────────────────────►│
│                                                                              │
│     ┌─────────────────────────────────────────────────────────────────────┐  │
│     │              CoverBehaviorManager.tick()                            │  │
│     │                                                                     │  │
│     │  handleInCover()                                                    │  │
│     │  ├── isCoverStillValid()?                                          │  │
│     │  │   └── distance > 3.0? → return false                            │  │
│     │  └── false? → transitionTo(SEEKING_COVER)                          │  │
│     │      └── currentCover = null ⚠️                                    │  │
│     └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  3. Goal Selector (Minecraft) picks goals                                   │
│                                                                              │
│     ┌─────────────────────────────────────────────────────────────────────┐  │
│     │              SeekCoverGoal (Priority 2)                             │  │
│     │                                                                     │  │
│     │  canUse()?                                                          │  │
│     │  ├── FOLLOW mode && !suppressed? → false                           │  │
│     │  └── shouldGoalSeekCover()?                                         │  │
│     │      └── state == SEEKING_COVER? → true                            │  │
│     │                                                                     │  │
│     │  start()                                                            │  │
│     │  └── findAndMoveToCover()                                           │  │
│     │      └── getCurrentCover() → null (was cleared!) ⚠️                 │  │
│     │      └── No hysteresis, finds same cover again                      │  │
│     └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  4. SeekCoverGoal.tick() runs                                               │
│     └── navigation.moveTo() moves soldier                                   │
│     └── Soldier moves AWAY from cover to navigate around obstacles!         │
│                                                                              │
│  5. Next tick: CoverBehaviorManager.tick()                                  │
│     └── handleInCover() (if soldier is near cover)                          │
│         └── isCoverStillValid() → distance changed → false again!           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Current Behavior Summary

**What happens:**
1. Soldier reaches cover
2. `setCover()` is called, state becomes IN_COVER (after next tick's handleSeekingCover)
3. `handleInCover()` checks `isCoverStillValid()` 
4. If soldier moved even slightly (turned, stepped), distance > 3.0 → returns false
5. `transitionTo(SEEKING_COVER)` clears `currentCover`
6. `SeekCoverGoal.canUse()` returns true
7. `findAndMoveToCover()` has no `currentCover` for hysteresis
8. Soldier finds same cover again
9. Repeat

**Why soldier seems to "not find cover":**
- The soldier IS finding cover
- But they keep re-seeking the same cover because:
  1. `isCoverStillValid()` fails due to distance
  2. Hysteresis is broken because `currentCover` is cleared

**Why soldier "moves between 2 covers":**
- If there are two equally good covers nearby
- And the soldier oscillates between them because:
  1. Reaches cover A
  2. Moves slightly, `isCoverStillValid()` fails
  3. Seeks again, finds cover B (slightly better now)
  4. Reaches cover B
  5. Moves slightly, `isCoverStillValid()` fails
  6. Seeks again, finds cover A (slightly better now)
  7. Repeat

---

## Design Questions

1. **Should the state machine (CoverBehaviorManager) and the goal (SeekCoverGoal) be merged?**
   - Currently they're separate, which causes synchronization issues
   - Option: Make CoverBehaviorManager only track state, let SeekCoverGoal handle all decisions

2. **What's the minimum time a soldier should stay in cover?**
   - Currently: `MIN_COVER_DWELL_TIME_MS = 3000` (3 seconds)
   - But `isCoverStillValid()` can override this by returning false

3. **What's the correct distance threshold?**
   - Currently: 3.0 blocks (2.0 * 1.5)
   - Is this too strict? Should it be 4.0? 5.0?

4. **Should soldiers in combat have different cover behavior?**
   - Currently: No distinction
   - Option: Allow more distance tolerance when actively shooting

5. **Should `currentCover` be preserved during SEEKING_COVER?**
   - Currently: Cleared on transition
   - This breaks hysteresis
   - Option: Keep `currentCover` but add a `seekingNewCover` flag

---

## Proposed Simplification

### Option A: Single-Responsibility Architecture

**CoverBehaviorManager:** Only tracks state and provides queries
- No state transitions except via explicit calls
- No `handleInCover()` logic that triggers transitions
- Just: `getState()`, `getCurrentCover()`, `setCover()`, `clearCover()`

**SeekCoverGoal:** Handles all cover decision logic
- `canUse()` decides when to seek cover
- `tick()` decides when to leave cover
- Owns all the logic currently in `handleInCover()`

**Benefit:** No race conditions, single source of truth

### Option B: Keep Dual System, Fix Synchronization

**Fixes:**
1. Don't clear `currentCover` in `transitionTo(SEEKING_COVER)`
2. Add `MIN_COVER_DWELL_TIME` check in `isCoverStillValid()` - don't invalidate until 3s passed
3. Increase distance threshold from 3.0 to 4.0 or 5.0
4. Add combat-aware distance tolerance (more lenient when shooting)

**Benefit:** Less invasive changes, keeps existing structure

---

## Constants Reference

| Constant | Value | Location | Purpose |
|----------|-------|----------|---------|
| `MAX_DISTANCE_TO_COVER` | 2.0 | CoverBehaviorManager | Distance to consider "at cover" |
| `MIN_COVER_TIME_MS` | 2500 | CoverBehaviorManager | Minimum time in cover |
| `MIN_PEEK_INTERVAL_MS` | 1500 | CoverBehaviorManager | Time between peek shots |
| `MAX_SEEKING_TIME_MS` | 10000 | CoverBehaviorManager | Timeout for seeking cover |
| `LOW_HEALTH_THRESHOLD` | 0.3f | CoverBehaviorManager | Health % to seek cover |
| `MIN_COVER_DWELL_TIME_MS` | 3000 | SeekCoverGoal | Time before re-evaluating |
| `COVER_REACHED_THRESHOLD` | 1.5 | SeekCoverGoal | Distance to consider reached |
| `SEARCH_RADIUS` | 12 | SeekCoverGoal | Radius to search for cover |
| `HYSTERESIS_THRESHOLD` | 0.25 | SeekCoverGoal | Quality improvement needed to switch |
| `MAX_STUCK_TICKS` | 60 | SeekCoverGoal | Ticks before re-seeking (3s) |
| `COOLDOWN_TICKS` | 40 | SeekCoverGoal | Ticks between seeks (2s) |

---

## Debug Commands

| Command | Purpose |
|---------|---------|
| `/stevesarmy_cover state` | Show cover state for nearby soldiers |
| `/stevesarmy_cover log on` | Enable cover behavior logging |
| `/stevesarmy_cover soldiers` | Toggle soldier cover visualization |
| `/stevesarmy_cover threats` | Show threat direction analysis |
| `/stevesarmy_cover reservations` | Show cover reservations |