# Post-bf45406 Revert Attempts

## Baseline
**bf45406 "Broke stuff"** — the last commit before the poke/flail period. This commit was built and tested as the baseline.

## Attempt 1: Selective Revert + CoverPositionController Rewiring
**Goal:** Keep `bf45406` CoverTacticalGoal and CoverBehaviorManager, wire them to the new `CoverPositionController` instead of the removed `ExactCoverMoveControl`.

**Changes:**
- Reverted `CoverTacticalGoal.java` and `CoverBehaviorManager.java` to `bf45406`
- Replaced all `ExactCoverMoveControl` references with `CoverPositionController`:
  - `lockToBlock(pos, dir)` → `setTarget(pos.center(), POSITIONING, 0.5, 0.08)`
  - `clearLock()` → `clearIntent()`
  - `isLocked()` → `getIntent() != NONE`
- Fixed `CoverState.POSITIONING` → `REPOSITIONING` in `SoldierFollowOwnerGoal` and `SoldierHoldPositionGoal`
- Removed unused `getProtectedDirection()` and `LOCK_DISTANCE`

**Result:** BUILD SUCCESSFUL. Deployed and tested.
**Status:** Working better — soldier peeks and shoots again. But stuck-in-ducking_back bug discovered.

## Attempt 2: Reset Peek State + Widen Threshold
**Goal:** Fix soldier stuck in `DUCKING_BACK` during `REPOSITIONING`/`SEEKING_COVER` state transitions.

**Root Cause:** Peek state (`DUCKING_BACK`) persisted when cover state transitioned to `REPOSITIONING`. `tickPeekState()` only runs in `tickInCover()`, so ducking back never completes.

**Changes:**
- `PEEK_POSITION_REACHED_DISTANCE`: 0.3 → 0.5 (match `CoverPositionController` tolerance, prevent oscillation)
- `startRepositioning()` both variants: added `resetPeekState()` + `clearIntent()`
- `canUse()` cover abandon: added `resetPeekState()` + `clearIntent()`
- `tickSeekingCover()` timeout: added `resetPeekState()` + `clearIntent()`
- `shouldExitCoverForFollow()`: added `resetPeekState()` + `clearIntent()`

**Result:** BUILD SUCCESSFUL. Deployed. Committed as `31bfd9e`.
**Status:** DUCKING_BACK stuck issue partly resolved, but movement still problematic.

## Attempt 3: Route Movement Through CoverPositionController
**Goal:** Replace all direct `setDeltaMovement()` calls in peek/return with `CoverPositionController.startPeek()`/`startReturn()`.

**Root Cause:** `timeToReturnToCover()` and the full-cover peek slide in `tickHiding()` used direct `setDeltaMovement()` calls, bypassing the controller entirely. The controller's `startPeek()` and `startReturn()` were dead code.

**Changes:**
- `tickHiding()` full-cover peek: replaced direct slide with `controller.startPeek(coverCenter, direction, MAX_PEEK_OFFSET)`, state machine checks `getIntent()` for completion
- `tickDuckingBack()`: replaced `timeToReturnToCover()` with `controller.startReturn(center)`, checks `getIntent() == NONE` for completion
- Removed `timeToReturnToCover()` method and `FULL_COVER_PEEK_SPEED` constant
- Added `MAX_PEEK_OFFSET = 1.0` constant

**Result:** BUILD SUCCESSFUL. Deployed.
**Status:** Movement routed through controller, but behavior not fully verified in-game.

## Attempt 4: Full Revert to bf45406
**Goal:** Undo everything and start fresh from `bf45406`.

**Action:** `git stash` + `git reset --hard bf45406`

**Result:** BUILD FAILED — `bf45406` references code that was in the earlier commit (`ExactCoverMoveControl`, `SoldierModel`, `SoldierGroundNavigation`, `PoseConfig`, `CoverHudRenderer`) which are all missing from the tree.

**Status:** At `bf45406` but can't build without the supporting files that were in the prior commit tree.

## Key Missing Files After Revert
- `CoverPositionController.java`
- `ExactCoverMoveControl.java` (removed, never in repo — was transitioned to CoverPositionController)
- `SoldierModel.java`
- `SoldierGroundNavigation.java`
- `PoseConfig.java`
- `CoverHudRenderer.java`
- `IncomingFireHandlerTaCZ.java`

## Lessons Learned
1. `bf45406` was a mid-transition commit — it deleted old code but hadn't fully wired the new patterns
2. The `CoverPositionController` approach (intent-based movement) is architecturally cleaner than direct `setDeltaMovement()`
3. Peek state must be reset when cover state transitions happen
4. `tickPeekState()` only runs in `tickInCover()` — peek state stalls during `REPOSITIONING`/`SEEKING_COVER`
