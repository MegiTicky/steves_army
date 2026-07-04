# Steve's Army — Movement & Peek System Bug Log

## Problem: Soldier stuck in DUCKING_BACK, never completes return to cover

### Observed Behavior
- Soldier 9 goes through full peek cycle: HIDING → (slide) → EXPOSED (~1.6s) → DUCKING_BACK
- DUCKING_BACK persists for 3–4 seconds until `tickInCover()` gives up (`dist > COVER_VALID_DISTANCE = 2.0`), transitions to SEEKING_COVER, finds new cover, and restarts
- Key log line: `peek=DUCKING_BACK(600ms,sinceLast=9223372036854775807ms) intent=NONE ctrlDist=0.44 distToCover=0.67 speed=0.0000 peekCount=0`
- `sinceLast=Long.MAX_VALUE` → `setLastPeekEndTime()` never called → peek cycle never completed
- `peekCount=0` → `recordPeekCycle()` never called
- `distToCover=0.67` → soldier 0.67 blocks from cover center (stable, doesn't change)
- `speed=0.0000` → no movement happening
- `intent=NONE` → controller not actively moving

### Diagnosis

#### Root Cause: `timeToReturnToCover()` used wrong target position

```java
// OLD CODE (broken):
Vec3 targetPos = coverPos.getCenter();
Set<Direction> protectedDirs = cover.getProtectedDirections();
if (protectedDirs != null && !protectedDirs.isEmpty()) {
    Direction protectDir = protectedDirs.iterator().next();
    targetPos = targetPos.add(-protectDir.getStepX() * 0.5, 0, -protectDir.getStepZ() * 0.5);
}
double dist = distance(soldier, targetPos);
if (dist < PEEK_POSITION_REACHED_DISTANCE) // 0.5
    return true;
```

This offset the return target by 0.5 blocks in the **protected direction** (away from threat). But the soldier peeked in the **unprotected direction**, so the offset moved the target **farther away** from the soldier. The distance to the adjusted target was ~1.17 instead of ~0.67 — always > 0.5, so `timeToReturnToCover()` never returned true.

#### Secondary Issue: Dual-target conflict

`tickInCover()` line 407 set POSITIONING toward **raw cover center**, but `timeToReturnToCover()` set it toward the **adjusted target**. These two callers fought each other every tick:

1. `tickInCover()` line 407 → sets target to cover center
2. `timeToReturnToCover()` line 757–758 → checks adjusted target distance → too far → sets target to adjusted position
3. Controller runs → sees POSITIONING with adjusted target → moves toward it
4. Next tick: line 407 resets to cover center again

The controller's target oscillated between two positions, never converging on either.

#### Tertiary: `setDeltaMovement()` weakness

`tickPositioning()` uses `setDeltaMovement()` with speed 0.08. On ground, `LivingEntity.travel()` applies 0.91 friction per tick, reducing effective movement to ~0.073/tick. Combined with the dual-target oscillation, net movement was effectively zero.

### Fix Applied (commit pending)

**`timeToReturnToCover()` now uses raw `coverPos.getCenter()`** — same target as `tickInCover()` line 407 — so both consistently target the same position.

```java
// NEW CODE:
Vec3 targetPos = cover.getPosition().getCenter();
double dist = distance(soldier, targetPos);
if (dist < PEEK_POSITION_REACHED_DISTANCE) // 0.5
    return true;
// else set POSITIONING toward coverPos.getCenter()
```

### Still Open Questions
1. Will 0.08 speed (0.073 effective after friction) be enough to close 0.67 → 0.5 in a reasonable time? (~3 ticks = 0.15s should be enough)
2. The soldier drifts during EXPOSED (distToCover goes from 0.67 to 0.98) — is the peek slide pushing them too far? This is separate from the stuck-DUCKING_BACK issue.

---

## Problem: `setDeltaMovement()` vs vanilla `MoveControl` friction

### Observed Behavior
- Controller sets `setDeltaMovement(0.08, ...)` in `tickPositioning()`
- Next tick, soldier hasn't moved (speed=0.0000)
- Vanilla `LivingEntity.aiStep()` → `travel()` applies friction (`deltaMovement = deltaMovement.scale(0.91)`) and calls `move()`
- After friction: 0.08 * 0.91 = 0.0728 — should still produce movement
- Yet logs show zero movement over many ticks

### Diagnosis
The `setDeltaMovement()` approach works for direct velocity, but `travel()` also reads `zza` (forward input). When `zza = 0` (no input), `travel()` applies only friction, not acceleration. But `setDeltaMovement()` should bypass this — the velocity is set directly, and `move()` uses `deltaMovement` regardless of `zza`.

Possible cause: something resets `deltaMovement` between the controller tick and `travel()`. The `super.tick()` call for NONE/NAVIGATING cases calls `MoveControl.tick()` which may call `setSpeed()` or `setWantedMovement()`. If intent transitions through NONE briefly, `super.tick()` could interfere.

### Status: Not fully diagnosed, but the dual-target fix should resolve the immediate stuck-DUCKING_BACK issue.

---

## Known Minor Issues

### `getTimeSinceLastPeek()` returns `Long.MAX_VALUE` when `lastPeekEndTime == 0`
- By design: means "never peeked"
- But labels/logs show absurdly large numbers
- Fix: cap at 0 or show "never" in display

### `getDebugTargetPos()` returns `Vec3.ZERO` when controller idle
- Shows soldier distance to world origin when intent is NONE
- Fix: return null when intent is NONE