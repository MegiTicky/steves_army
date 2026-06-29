# Reference Mod Research - Implementation Patterns for Steve's Army

## Overview

Research findings from three reference mods for implementing AI soldiers in Minecraft 1.20.1 Forge.

---

## 1. Ancient Warfare 2 (1.12.2) - NPC AI Patterns

**Most Applicable Patterns:**

### Entity Base Class Structure
```
NpcBase extends EntityCreature
├── NpcPlayerOwned (player-owned NPCs)
│   └── NpcCombat (combat specialization)
└── NpcFaction (faction-based NPCs)
```

**Key Implementation:**
```java
// NpcBase.java - Task bitfield for AI state tracking (debug rendering)
public static final int TASK_ATTACK = 1;
public static final int TASK_FOLLOW = 128;
public static final int TASK_MOVE = 512;

private static final DataParameter<Integer> AI_TASKS = 
    EntityDataManager.createKey(NpcBase.class, DataSerializers.VARINT);

public final void addAITask(int task);
public final void removeAITask(int task);

// Owner system with team integration
private Owner owner = Owner.EMPTY;
public abstract boolean isHostileTowards(Entity e);
public abstract boolean canTarget(Entity e);
```

### AI Goal Pattern
```java
// NpcAI.java - Generic base class
public abstract class NpcAI<T extends NpcBase> extends EntityAIBase {
    protected T npc;
    protected double moveSpeed = 1.d;
    
    @Override
    public boolean shouldExecute() {
        return npc.getIsAIEnabled() && !npc.isAIFlagStopped(getMutexBits());
    }
    
    protected final void moveToEntity(Entity target, double sqDist);
    protected final void moveToPosition(BlockPos pos, double sqDist);
}

// NpcAIFollowPlayer.java - Simple follow behavior
@Override
public void updateTask() {
    npc.getLookHelper().setLookPositionWithEntity(target, 10.0F, ...);
    double distance = npc.getDistanceSq(target);
    if (distance > FOLLOW_STOP_DISTANCE) {
        npc.addAITask(TASK_MOVE);
        moveToEntity(target, distance);
    }
}
```

### Dynamic Weapon Switching
```java
// NpcCombat.java - Switch AI based on equipped weapon
@Override
public void onWeaponInventoryChanged() {
    tasks.removeTask(arrowAI);
    tasks.removeTask(meleeAI);
    ItemStack stack = getHeldItemMainhand();
    if (isBow(stack.getItem())) {
        tasks.addTask(4, arrowAI);
    } else {
        tasks.addTask(4, meleeAI);
    }
}
```

### Command System
```java
// CommandType enum
public enum CommandType {
    MOVE, ATTACK, ATTACK_AREA, GUARD,
    SET_HOME, SET_UPKEEP, CLEAR_HOME, CLEAR_UPKEEP, CLEAR_COMMAND, NONE;
    
    public boolean isPersistent() {
        return (this == ATTACK || this == GUARD || this == ATTACK_AREA);
    }
}
```

**Files to Reference:**
- `ReferenceMod/AncientWarfare2-1.12.x/src/main/java/net/shadowmage/ancientwarfare/npc/entity/NpcBase.java`
- `ReferenceMod/AncientWarfare2-1.12.x/src/main/java/net/shadowmage/ancientwarfare/npc/ai/NpcAI.java`
- `ReferenceMod/AncientWarfare2-1.12.x/src/main/java/net/shadowmage/ancientwarfare/npc/ai/NpcAIFollowPlayer.java`

---

## 2. TaCZ (1.20.1) - Weapon Integration

### Making NPCs Fire Guns

**Main Interface:**
```java
// IGunOperator is mixed into LivingEntity
IGunOperator gunOperator = IGunOperator.fromLivingEntity(livingEntity);

// Fire at direction
ShootResult result = gunOperator.shoot(
    () -> pitch,  // Supplier<Float> vertical angle
    () -> yaw     // Supplier<Float> horizontal angle
);

// Other operations
gunOperator.reload();
gunOperator.aim(true);
```

**ShootResult Enum:**
- `SUCCESS` - Shot fired
- `NO_AMMO` - Needs reload
- `COOL_DOWN` - Fire rate delay
- `NOT_GUN`, `NOT_DRAW`, `IS_RELOADING`, etc.

### Creating Gun ItemStacks
```java
GunItemBuilder.create()
    .setId(gunId)                    // ResourceLocation
    .setFireMode(FireMode.AUTO)
    .setAmmoCount(30)
    .build();
```

### Weapon Stats Access
```java
// Via TimelessAPI
TimelessAPI.getCommonGunIndex(gunId).ifPresent(index -> {
    GunData data = index.getGunData();
    int rpm = data.getRoundsPerMinute();
    float damage = data.getBulletData().getDamageAmount();
    float speed = data.getBulletData().getSpeed();
});
```

### Forge Events
```java
@SubscribeEvent
public void onGunHurt(EntityHurtByGunEvent.Pre event) {
    LivingEntity attacker = event.getAttacker();
    float damage = event.getBaseAmount();
    boolean headshot = event.isHeadShot();
}

@SubscribeEvent
public void onGunShoot(GunShootEvent event) {
    LivingEntity shooter = event.getShooter();
}
```

**Files to Reference:**
- `ReferenceMod/TACZ-1.20.1/src/main/java/com/tacz/guns/api/entity/IGunOperator.java`
- `ReferenceMod/TACZ-1.20.1/src/main/java/com/tacz/guns/api/item/builder/GunItemBuilder.java`
- `ReferenceMod/TACZ-1.20.1/src/main/java/com/tacz/guns/api/TimelessAPI.java`

---

## 3. Villager Recruits (1.20.1) - Squad Management

### Entity Data Sync
```java
// AbstractRecruitEntity.java - EntityDataAccessor pattern
private static final EntityDataAccessor<Optional<UUID>> OWNER_ID = 
    SynchedEntityData.defineId(AbstractRecruitEntity.class, EntityDataSerializers.OPTIONAL_UUID);
private static final EntityDataAccessor<Integer> FOLLOW_STATE = 
    SynchedEntityData.defineId(AbstractRecruitEntity.class, EntityDataSerializers.INT);
private static final EntityDataAccessor<Integer> STATE = // 0=NEUTRAL, 1=AGGRESSIVE, 2=RAID, 3=PASSIVE
    SynchedEntityData.defineId(AbstractRecruitEntity.class, EntityDataSerializers.INT);
```

### Follow State System
```java
// States 0-6
public void setFollowState(int state) {
    switch (state) {
        case 0 -> { setShouldFollow(false); setShouldHoldPos(false); } // Wander
        case 1 -> { setShouldFollow(true); }                           // Follow
        case 2 -> { setShouldHoldPos(true); setHoldPos(position()); }  // Hold your position
        case 3 -> { setShouldHoldPos(true); }                          // Back to position
        case 4 -> { setShouldHoldPos(true); setHoldPos(owner.position()); } // Hold my position
        case 5 -> { setShouldProtect(true); }                          // Protect
    }
}
```

### Squad/Group Data
```java
public class RecruitsGroup {
    private UUID uuid;
    public List<UUID> members = new ArrayList<>();
    private UUID playerUUID;
    private String name;
    public int aggroState;
    public int followState;
    
    public CompoundTag toNBT();
    public static RecruitsGroup fromNBT(CompoundTag tag);
}
```

### Fair Target Acquisition
```java
// Line of sight gating
if (!this.recruit.getSensing().hasLineOfSight(target)) {
    this.recruit.setTarget(null);
    return false;
}

// Target search with randomization
public void searchForTargets() {
    AABB searchBox = this.getBoundingBox().inflate(40);
    List<LivingEntity> nearby = level.getEntitiesOfClass(LivingEntity.class, searchBox, predicate);
    nearby.sort(Comparator.comparingDouble(e -> e.distanceToSqr(this)));
    
    int pool = Math.min(10, nearby.size());
    LivingEntity target = nearby.get(this.getRandom().nextInt(pool)); // Random from closest 10
}
```

### Inventory Slots
```java
// 15 slots total
// 0-3: Armor (HEAD, CHEST, LEGS, FEET)
// 4: OFFHAND (shield)
// 5: MAINHAND
// 6-14: General inventory (3x3)
```

**Files to Reference:**
- `ReferenceMod/recruits-main/src/main/java/com/tomlouiscourt/recruits/entities/AbstractRecruitEntity.java`
- `ReferenceMod/recruits-main/src/main/java/com/tomlouiscourt/recruits/entities/ai/RecruitFollowOwnerGoal.java`
- `ReferenceMod/recruits-main/src/main/java/com/tomlouiscourt/recruits/world/RecruitsGroup.java`

---

## Recommended Implementation Order for Steve's Army

### Phase 1: Foundation
1. **SoldierEntity** - Base on Villager Recruits `AbstractRecruitEntity` pattern
   - EntityDataAccessor for owner UUID, follow state
   - 15-slot inventory (armor + mainhand + 9 general)
   
2. **SquadData** - Based on Villager Recruits `RecruitsGroup`
   - UUID-based membership
   - NBT serialization for world save

3. **Follow AI** - Based on Ancient Warfare 2 `NpcAIFollowPlayer`
   - Generic `SoldierAI<T>` base class
   - Task bitfield for debug rendering

### Phase 2: Combat
4. **Target Acquisition** - Based on Villager Recruits fair AI
   - Line of sight check
   - Random selection from closest N targets
   
5. **TaCZ Integration** - Use `IGunOperator` interface
   - `shoot(pitch, yaw)` for firing
   - Handle `ShootResult` for reloads

### Phase 3: Cover System
6. **Cover Detection** - New implementation (no direct reference)
   - Directional scan toward enemy
   - Score by distance, material, firing angle

---

## Key Differences from Reference Mods

| Feature | Ancient Warfare 2 | Villager Recruits | Steve's Army |
|---------|-------------------|-------------------|--------------|
| Minecraft | 1.12.2 | 1.20.1 | 1.20.1 |
| Roles | Multiple | Multiple | Single (rifleman) |
| Commands | Complex | Follow/Hold/Protect | Follow/Hold only |
| Weapons | Vanilla bows | Vanilla bows | TaCZ guns |
| Cover | No | No | Yes (critical) |
| Formations | Yes | Yes | No (MVP) |

---

## 1.20.1 API Changes from 1.12.2

When adapting Ancient Warfare 2 code:

| 1.12.2 | 1.20.1 |
|--------|--------|
| `EntityAIBase` | `Goal` |
| `EntityDataManager` | `SynchedEntityData` |
| `DataParameter` | `EntityDataAccessor` |
| `RegistryEvent.Register<EntityEntry>` | `DeferredRegister<EntityType<?>>` |
| `EntityCreature` | `PathfinderMob` |

---

## Next Steps

1. Create `SoldierEntity` using Villager Recruits patterns
2. Implement basic follow AI using Ancient Warfare 2 patterns
3. Add TaCZ dependency and test `IGunOperator` interface
4. Design cover system (no direct reference - new implementation)
