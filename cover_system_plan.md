# Cover Behavior System Implementation Plan

## Overview

Implementation of AI cover behavior system for Steve's Army soldiers, based on research from Arma 3 and Squad game mechanics.

## Design Principles

### Threat Direction Handling
- **Weighted by danger, not just proximity** - closer enemies + active shooters = higher weight
- **Flanking detection via angular spread** - enemies spanning >90° triggers flanking response
- **Cover must protect against multiple threats** - not just primary target

### Suppression Mechanics
- **Near misses add suppression** - bullets within 3 blocks
- **Decay over time** - 15%/sec (2x faster in cover)
- **Affects all combat decisions** - accuracy, peek willingness, movement

### Cover Behavior
- **Stay committed to cover** - minimum 2.5 seconds before re-evaluating
- **Peek to shoot only when safe** - suppression < 70%, has clear shot
- **Use TaCZ crawl** - best accuracy (2.5 inaccuracy vs 5.0 standing)
- **No stacking** - max 2 soldiers per cover point

## Recommended Defaults

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Flanking threshold | 90° | Standard military doctrine |
| Suppression decay | 15%/sec (2x in cover) | Forgiving but punishing |
| Min cover time before peek | 2.5 sec | Balanced aggression |
| Crawl usage | Always in HALF/FULL cover | Best accuracy |
| Cover scoring weights | 40/30/15/15 | Protection prioritized |
| Max soldiers per cover | 2 | Prevents stacking |

## Cover Scoring Formula

```
Score = (PrimaryProtection * 0.40) +
        (FlankingProtection * 0.30) +
        (DistanceToSoldier * 0.15) +
        (FiringPositionQuality * 0.15)
```

- **PrimaryProtection**: How well cover blocks primary threat direction (0-1)
- **FlankingProtection**: % of all known threats blocked by cover (0-1)
- **DistanceToSoldier**: Closer to soldier = higher score (inverse distance)
- **FiringPositionQuality**: Can shoot from cover = 1.0, else 0.5

## Implementation Status

**All phases are implemented.** See `steves_army_cover_architecture.md` for current architecture.

### Key Implementation Notes

**Actual scoring weights (CoverFinder.java):**
- PRIMARY_PROTECTION_WEIGHT = 0.30 (not 0.40)
- FLANKING_PROTECTION_WEIGHT = 0.20 (not 0.30)
- DISTANCE_WEIGHT = 0.10 (not 0.15)
- FIRING_QUALITY_WEIGHT = 0.25 (not 0.15)
- PEEK_ANGLE_WEIGHT = 0.15 (added)
- FIGHTABILITY_BONUS = 0.20 (added, not in original design)

**Key Bug Fixes Discovered During Implementation:**

1. **Walking while crawling** — `clearCover()` must call `GunIntegration.crawl(false)` because all cover exits (SEEKING, REPOSITIONING, FOLLOW exit, NO_COVER timeout) go through `clearCover()`. Without this, the CRAWLING flag persists and the soldier walks with the crawling pose.

2. **Frozen at peek position when suppressed** — The state transition `IN_COVER → SUPPRESSED_IN_COVER` skips `tickDuckingBack()`. `tickSuppressedInCover()` must call `tickDuckingBack()` when peek state is `DUCKING_BACK` to process the velocity slide back to cover.

### Updated State Machine Flow

```
NO_COVER ──────────────────────────────────────► SEEKING_COVER
    │                                              (find cover, pathfind)
    │  [suppressed, low health, hold mode,         │
    │   being flanked]                             │
    │                              [reached cover] │
    │                                              ▼
    │                                         IN_COVER
    │                                    [tickPeekState]
    │                              ┌──────────────────┐
    │                              │ HIDING → EXPOSED │
    │                              │   → DUCKING_BACK │
    │                              └──────────────────┘
    │                              [suppressed]    │
    │  [suppression clears]           ↓            ▼
    │  ┌──────────────────────── IN_COVER ← SUPPRESSED_IN_COVER
    │  │                                          │
    │  │                              [tickDuckingBack() while DUCKING_BACK]
    │  │
    │◄─┘
    [cover invalid, threat moved, follow mode]
    [clearCover() → GunIntegration.crawl(false)]
```

### Additional States Added
- **REPOSITIONING**: Moving to better cover while maintaining currentCover for hysteresis (prevents oscillation)
- **NON_PEEKABLE**: Cover position with no valid LOS angles; soldier repositions after ~8s (160 ticks)

### Peek System (Not in Original Design)

A 3-state sub-machine within IN_COVER:
1. **HIDING** → check LOS, cooldown, expose if valid
2. **EXPOSED** → soldier shoots, auto-ducks after 1500ms, early duck on target death/LOS loss
3. **DUCKING_BACK** → half-cover: instant pose switch; full-cover: velocity slide back to cover position

### Phase 1: Core Infrastructure

**Files to create:**

1. `SuppressionTracker.java` (~80 lines)
   - Track suppression level (0.0 - 1.0)
   - `onNearMiss(Vec3 bulletPath)` - Called when bullet passes nearby
   - `onIncomingFire(LivingEntity shooter)` - Called when being shot at
   - `tick(boolean inCover)` - Decay over time
   - Getters: `isSuppressed()`, `isPinned()`, `getAccuracyModifier()`, `canPeek()`

2. `ThreatDirectionCalculator.java` (~120 lines)
   - `calculatePrimaryThreatDirection(soldier, threats)` - Weighted threat vector
   - `calculateThreatWeight(soldier, threat)` - Distance + active shooter bonus
   - `isBeingFlanked(soldier, threats)` - Angular spread > 90°
   - `getThreatCoverage(coverPos, threats)` - % of threats blocked by cover

3. `CoverReservationManager.java` (~60 lines)
   - Static map: `Map<BlockPos, Set<UUID>> coverReservations`
   - `reserve(coverPos, soldierUUID)` - Claim a cover point
   - `release(coverPos, soldierUUID)` - Release reservation
   - `isAvailable(coverPos)` - Check if < 2 soldiers using it
   - `getReservationCount(coverPos)` - How many soldiers using it

### Phase 2: Cover Selection Enhancement

**Files to modify:**

4. `CoverFinder.java`
   - New method: `findBestCover(soldier, threatDirection, allThreats)`
   - New scoring: Primary protection (40%) + Flanking protection (30%) + Distance (15%) + Firing quality (15%)
   - Add `evaluateFlankingProtection(cover, allThreats)`
   - Filter out reserved cover points (via `CoverReservationManager`)

5. `CoverQualityEvaluator.java`
   - Add `isDirectionProtected(cover, direction)` helper
   - Used by flanking protection calculation

### Phase 3: Cover Behavior State Machine

**Files to create:**

6. `CoverBehaviorManager.java` (~200 lines)
   - State enum: `NO_COVER`, `SEEKING_COVER`, `IN_COVER`, `SUPPRESSED_IN_COVER`
   - State transitions based on suppression, health, squad mode
   - `tick(SoldierEntity)` - Main state machine loop
   - `shouldSeekCover(soldier)` - Decision logic
   - `isCoverStillValid(soldier)` - Check if current cover still protects
   - `peekAndShoot(soldier)` - Exit crawl, shoot, return to crawl

### Phase 4: AI Goal Integration

**Files to create:**

7. `SeekCoverGoal.java` (~150 lines)
   - Priority: Between flee and combat
   - `canUse()`: Suppressed, low health, hold mode, being flanked
   - `start()`: Find best cover, pathfind to it, reserve it
   - `tick()`: Monitor path progress, check if cover still valid
   - `stop()`: Release cover reservation

**Files to modify:**

8. `SoldierCombatGoal.java`
   - Add `CoverBehaviorManager` instance
   - Add `SuppressionTracker` instance
   - In `tick()`: Check suppression before engaging
   - When suppressed: Let `SeekCoverGoal` take over
   - Apply suppression accuracy modifier to shots

9. `SoldierEntity.java`
   - `private CoverPoint currentCover`
   - `private CoverBehaviorManager.CoverState coverState`
   - `private float suppressionLevel`
   - Sync to client for rendering (optional)

### Phase 5: TaCZ Crawl Integration

**Files to modify:**

10. `GunIntegration.java`
    - New method: `crawl(LivingEntity entity, boolean isCrawl)`
    - Reflection call to `IGunOperator.crawl(boolean)`
    - New method: `isCrawling(LivingEntity entity)`
    - Check `entity.getPose() == Pose.SWIMMING && !entity.isSwimming()`

### Phase 6: Incoming Fire Detection

**Files to create:**

11. `IncomingFireHandler.java` (~100 lines)
    - Listen for `LivingHurtEvent` - soldier taking damage
    - Listen for projectile tick events - bullets passing nearby
    - Call `SuppressionTracker.onNearMiss()` when bullet within 3 blocks
    - Call `SuppressionTracker.onIncomingFire()` when shot at

### Phase 7: Debug/Testing Tools

**Files to modify:**

12. `CoverDebugCommand.java`
    - `/stevesarmy_cover suppression` - Show current suppression level
    - `/stevesarmy_cover state` - Show cover state machine state
    - `/stevesarmy_cover threats` - Show threat direction calculation

13. `CoverDebugRenderer.java`
    - Render threat direction arrow
    - Render suppression level indicator
    - Render cover reservations

## State Machine Flow

```
NO_COVER ──────────────────────────────────────► SEEKING_COVER
    │                                              (find cover, pathfind)
    │  [suppressed, low health, hold mode,         │
    │   being flanked]                             │
    │                                              │
    │                              [reached cover] │
    │                                              ▼
    │                                         IN_COVER
    │                                              │
    │                              [suppressed]    │
    │                                              ▼
    │                                   SUPPRESSED_IN_COVER
    │                                              │
    │                              [suppression clears]
    │                                              │
    │◄─────────────────────────────────────────────┘
    [cover invalid, threat moved, follow mode]
```

## Testing Checklist

### Phase 1-2 Testing
- [x] `/stevesarmy_cover threats` shows weighted threat direction
- [x] Place multiple targets, verify flanking detection (>90°)
- [x] Verify cover scoring includes flanking protection
- [x] Verify cover reservations prevent stacking

### Phase 3-4 Testing
- [x] Spawn soldier + enemies, verify soldier seeks cover
- [x] Verify soldiers use different cover points (no stacking)
- [x] Verify suppression triggers cover-seeking
- [x] Verify cover state transitions work

### Phase 5-6 Testing
- [x] Verify soldier crawls when in HALF/FULL cover
- [x] Shoot near soldier, verify suppression increases
- [x] Verify suppression decays over time
- [x] Verify accuracy reduced when suppressed

### Phase 7 Testing
- [ ] Full combat scenario with multiple enemies
- [ ] Verify soldier uses cover intelligently
- [ ] Verify flanking response (repositioning)
- [ ] Verify debug visualizations work

## Estimated Effort

| Phase | Files | Lines | Complexity |
|-------|-------|-------|------------|
| 1 | 3 new | ~260 | Low |
| 2 | 2 modify | ~150 | Medium |
| 3 | 1 new | ~200 | Medium |
| 4 | 3 files | ~450 | High |
| 5 | 1 modify | ~40 | Low |
| 6 | 1 new | ~100 | Medium |
| 7 | 2 modify | ~150 | Low |
| **Total** | **11 files** | **~1010** | - |

## Dependencies

### Existing Components
- `CoverFinder`, `CoverQualityEvaluator`, `CoverPoint`, `CoverType`
- `ThreatTracker`, `DetectionSystem`, `TargetAcquisition`
- `GunIntegration` (TaCZ reflection)
- `SoldierEntity`, `SoldierCombatGoal`

### TaCZ API Used
- `IGunOperator.crawl(boolean)` - Enter/exit crawl state
- `Pose.SWIMMING` - Crawl pose (check via `!entity.isSwimming() && entity.getPose() == Pose.SWIMMING`)
- Crawling benefits: 2.5 inaccuracy (vs 5.0 standing), 50% recoil reduction

## Future Enhancements (Post-MVP)

- Cover indicators for player (highlight nearby cover blocks)
- Smoke grenade usage for breaking contact
- Suppression communication between squad members
- Cover quality decay (cover degrades under sustained fire)
- Dynamic cover destruction (explosions destroy cover)