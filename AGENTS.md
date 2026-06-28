# Steve's Army - Agent Instructions

## Project Overview

Minecraft 1.20.1 Forge mod implementing an Enlisted-style military squad system with AI soldiers, cover mechanics, and TaCZ gun integration.

## Build Commands

```powershell
# Must set JAVA_HOME to JDK 17 for Forge 1.20.1
$env:JAVA_HOME = "C:\Program Files\Java\jdk-17"

# Build the mod
.\gradlew build

# Setup dev environment (run once after clone)
.\gradlew prepareRuns

# Run client for testing
.\gradlew runClient
```

## Project Structure

```
steves_army/                    # Main mod project
├── src/main/java/com/stevesarmy/
│   ├── StevesArmyMod.java      # Main mod entry point
│   ├── combat/                 # Combat system
│   │   ├── GunIntegration.java # TaCZ reflection integration
│   │   ├── TargetAcquisition.java
│   │   └── ThreatTracker.java
│   ├── entity/                 # Entities
│   │   ├── SoldierEntity.java
│   │   └── ai/
│   │       ├── SoldierCombatGoal.java
│   │       ├── SoldierFollowOwnerGoal.java
│   │       └── SoldierHoldPositionGoal.java
│   ├── inventory/              # Inventory system
│   │   ├── SoldierInventory.java
│   │   ├── SoldierInventoryMenu.java
│   │   └── SoldierInventoryMenuProvider.java
│   ├── client/screen/          # Client GUI
│   │   └── SoldierInventoryScreen.java
│   ├── network/                # Network packets
│   │   ├── NetworkHandler.java
│   │   ├── ToggleSquadModeMessage.java
│   │   ├── DebugMessage.java
│   │   └── OpenSoldierInventoryMessage.java
│   └── squad/                  # Squad management
│       ├── SquadData.java
│       └── SquadManager.java
├── build.gradle
├── gradle.properties
└── src/main/resources/META-INF/mods.toml

ReferenceMod/                   # Reference source code for study
├── TACZ-1.20.1/                # TaCZ gun mod (GPLv3) - weapon integration
├── AncientWarfare2-1.12.x/     # NPC AI reference (1.12.2 only, not direct dependency)
├── recruits-main/              # Villager Recruits (medieval NPCs, closed source)
└── CustomNPC-Plus-master/      # CustomNPC (1.7.10, NOT for 1.20.1)
```

## Key Decisions

- **Simplified scope**: Follow/Hold commands only (no Assault mode), all riflemen (no specialized roles)
- **Cover system is critical**: Without cover, soldiers die too quickly
- **Fair AI**: No X-ray vision - line of sight, sound detection, squad communication only
- **Inventory-based ammo**: Soldiers use ammo from their 9-slot inventory, not dummy ammo

## Mod Metadata

- MODID: `steves_army`
- Version: 0.1.0-alpha
- Author: erika
- License: GPL-3.0
- Minecraft: 1.20.1
- Forge: 47.4.0

## Development Plan

See `steves_army_plan.md` for full roadmap. Current phase: Combat AI with TaCZ integration.

## Dependencies (Planned)

- TaCZ (Timeless and Classics Zero) - required for weapon system
- Will need to add to build.gradle as `fg.deobf()` dependency

## Java Requirements

- **JDK 17 required** - Forge 1.18+ ships Java 17 to end users
- Located at `C:\Program Files\Java\jdk-17`

## TaCZ Integration Notes

### Key API Methods (via reflection)

```java
IGunOperator operator = IGunOperator.fromLivingEntity(entity);

// Initialization (call once when gun equipped)
operator.initialData();
operator.draw(() -> gunStack);

// Combat
operator.aim(true);                              // Start aiming (improves accuracy 33x)
ShootResult result = operator.shoot(pitch, yaw); // Shoot with target angles
operator.bolt();                                 // Bolt-action cycle
operator.reload();                               // Reload from inventory

// State checks
operator.getSynShootCoolDown();                  // Shoot cooldown (ms)
operator.getSynAimingProgress();                 // Aim progress 0-1
operator.getSynIsBolting();                      // Is bolting
operator.getSynReloadState().getStateType().isReloading();
```

### ShootResult Values

- `SUCCESS` - Shot fired
- `NEED_BOLT` - Manual action rifle needs bolting
- `NO_AMMO` - Out of ammo
- `COOL_DOWN` - Fire rate cooldown active
- `IS_BOLTING`, `IS_RELOADING`, `IS_DRAWING` - Action in progress
- `NOT_DRAWN` - Gun not drawn yet

### Accuracy (InaccuracyType)

| State | Inaccuracy |
|-------|------------|
| STAND | 5.0 |
| MOVE | 5.75 |
| SNEAK | 3.5 |
| LIE | 2.5 |
| AIM | 0.15 |

Soldiers aim before shooting for best accuracy.

### Ammo System

- Soldiers use inventory ammo, not dummy ammo
- `IGun.getCurrentAmmoCount(gunStack)` - Current magazine ammo
- `IGun.hasAmmoInBarrel(gunStack)` - Barrel round (for closed bolt)
- `GunData.getMagazineSize()` - Magazine capacity
- `GunData.getUseInventoryAmmo()` - Whether gun uses inventory ammo
