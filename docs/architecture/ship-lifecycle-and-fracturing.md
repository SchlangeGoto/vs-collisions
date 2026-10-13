# Ship lifecycle and fracturing

Related: [Documentation index](../index.md) · [Block mutation and destruction](block-mutation-and-destruction.md) · [VS2/Core API reference](../reference/vs2-core-api.md)

---

## 1. Getting the Ship World

`VsiServerShipWorld` (internal package — `@GameTickOnly`) is the server-side registry of all ships. There are two established accessors, both used by this codebase:

```kotlin
import org.valkyrienskies.mod.common.VSGameUtilsKt          // Java-style statics
import org.valkyrienskies.mod.common.shipObjectWorld        // Kotlin extension
import net.minecraft.server.MinecraftServer
import net.minecraft.server.level.ServerLevel

// From a level (what DamageApplicator does):
val shipWorld = VSGameUtilsKt.getShipObjectWorld(level)     // VsiServerShipWorld

// Idiomatic Kotlin:
val shipWorld2 = level.shipObjectWorld                      // same object
val shipWorld3 = server.shipObjectWorld                     // server-wide
```

> [!warning] `@GameTickOnly`
> `VsiServerShipWorld` is annotated `@GameTickOnly`. Resolved ships (`LoadedServerShip`) are also `@GameTickOnly`. All lookups and mutations in this page must happen on the game thread — which is satisfied automatically when called from `DamageApplicator.onServerTick`, and **never** from a collision event handler.

## 2. Querying Ships

`loadedShips` / `allShips` are `QueryableShipData` collections (verified):

```kotlin
interface QueryableShipData<out ShipType : Ship> : Collection<ShipType> {
    fun getById(shipId: ShipId): ShipType?
    fun getByChunkPos(chunkX: Int, chunkZ: Int, dimensionId: DimensionId): ShipType?
    fun getIntersecting(aabb: AABBdc, dimensionId: DimensionId): Iterable<ShipType>
    fun contains(shipId: ShipId): Boolean = getById(shipId) != null
    // …Collection members…
}
```

The collision-processing lookup used by `DamageApplicator`:

```kotlin
val level = server.getLevelFromDimensionId(collision.dimensionId) ?: return
val shipWorld = level.shipObjectWorld

val shipA = shipWorld.loadedShips.getById(collision.shipIdA)  // null = unloaded / the world
val shipB = shipWorld.loadedShips.getById(collision.shipIdB)

// A collision can involve the ground body (id resolves to nothing):
//   shipA != null && shipB == null  →  ship <-> world impact
//   both null                       →  stale event, log & skip (current behavior)
```

Key semantics:

- **`loadedShips`** contains ships with loaded chunk claims (physics-active). Damage processing should use this one — as the codebase does.
- **`allShips`** additionally includes unloaded/offline ships — use only for bookkeeping that must see dormant ships.
- One side of a ship↔world collision is VS2's *ground body* (`dimensionToGroundBodyIdImmutable`), which is not a ship — hence the nullable pattern.

## 3. Creating Ships

Verified signature from `VsiServerShipWorld.kt`:

```kotlin
// "Creates a new ShipData centered at the block at blockPosInWorldCoordinates.
//  If createShipObjectImmediately is true then a ShipObject will be created immediately."
fun createNewShipAtBlock(
    blockPosInWorldCoordinates: Vector3ic,
    createShipObjectImmediately: Boolean,
    scaling: Double = 1.0,          // JVM callers must pass it — no @JvmOverloads
    dimensionId: DimensionId
): ServerShip

fun allocateShipId(dimensionId: DimensionId): ShipId
```

```kotlin
import org.joml.Vector3i

val dimensionId: DimensionId = level.dimensionId
val newShip = shipWorld.createNewShipAtBlock(
    Vector3i(fragmentCenterWorldX, fragmentCenterWorldY, fragmentCenterWorldZ),
    /* createShipObjectImmediately = */ true,
    /* scaling = */ 1.0,
    dimensionId
)
```

> [!important] What `createNewShipAtBlock` does NOT do
> Per its own documentation it creates the *ShipData* (the ship record/claim), it does **not** harvest existing world blocks into the ship. Populating a freshly created ship from a parent fragment is your own code's job — copy block states into the new ship's chunk claim, then clear them from the parent, then let the physics object (`createShipObjectImmediately = true`) cook.

## 4. Deleting Ships

```kotlin
fun deleteShip(ship: ServerShip)
fun teleportShip(ship: ServerShip, teleportData: ShipTeleportData)
```

Deletion is the bookend of catastrophic failure (e.g., "vessel disintegrates"). Safe recipe:

```kotlin
fun vaporizeShip(ship: LoadedServerShip, level: ServerLevel, shipWorld: VsiServerShipWorld) {
    // 1. Clear every block in the claim FIRST, so no ghost voxels remain in the shipyard.
    val aabb = ship.shipAABB
    if (aabb != null) {
        for (x in aabb.minX()..aabb.maxX())
            for (y in aabb.minY()..aabb.maxY())
                for (z in aabb.minZ()..aabb.maxZ()) {
                    val pos = BlockPos(x, y, z)
                    if (!level.getBlockState(pos).isAir) {
                        level.removeBlock(pos, false)   // no drops, no neighbor shaping
                    }
                }
    }
    // 2. Then delete the ship record.
    shipWorld.deleteShip(ship)
}
```

> [!caution] Deleting without clearing blocks first
> Leaves orphaned blocks sitting in the shipyard chunk claim with no ship to render or simulate them. Always scrub, then delete.

## 5. Fracturing via VS2's Own Splitter

VS2 continuously runs a connectivity graph (`SplitHandler`, `ValkyrienSkiesMod.splitHandler`) over ship voxels. When you `level.destroyBlock(...)` enough hull to cut it in two, VS2 **automatically splits the ship** — this is exactly what the codebase relies on (`BlockDamageResolver`: *"VSplit automatically handles structural separation"*).

Observability for this is the beta `splitEvent` on the core API (verified):

```kotlin
// org.valkyrienskies.core.api.events.SplitEvent — @VsBeta @GameTickOnly
interface SplitEvent {
    val wasPocket: Boolean          // true if the split component was an isolated pocket
    val oldRoot: Vector3ic
    val newRootA: Vector3ic
    val newRootB: Vector3ic
    val voxelType: Int              // 1 = solids, 0 = air
    var dimensionId: DimensionId?
}
```

Subscribe to react (e.g., re-parent your `ShipDamageData` so fatigue follows the right fragments):

```kotlin
@OptIn(GameTickOnly::class)
fun registerSplitReaction(api: VsApi) {
    api.splitEvent.on { ev ->
        ev.dimensionId ?: return@on
        // Re-derive: the fragments are new ships; copy proportional fatigue
        // from the old ShipDamageData based on which blocks moved where.
    }
}
```

### Suppressing splits during batch destruction

When you destroy a *large* connected region in one tick (a whole magazine detonation), the split handler can fire repeatedly mid-batch. VS2 ships `SplittingDisablerAttachment` (`org.valkyrienskies.mod.common.util`) to gate this:

```kotlin
import org.valkyrienskies.mod.common.util.SplittingDisablerAttachment

fun destroyRegionAtomically(ship: LoadedServerShip, blocks: List<BlockPos>, level: ServerLevel) {
    val disabler = SplittingDisablerAttachment(/* splitt = */ true)
    disabler.disableSplitting()
    ship.setAttachment(disabler)          // splits suppressed for this ship

    try {
        blocks.forEach { level.destroyBlock(it, true) }
    } finally {
        disabler.enableSplitting()        // one clean split pass afterwards
        ship.setAttachment(disabler)
    }
}
```

(Verified: `SplittingDisablerAttachment` is a `data class(private var splitt)` with `enable/disable/canSplit`; VS2's split logic consults it.)

> [!warning] Ship identity changes under you
> After an auto-split, the `LoadedServerShip` instance you resolved at the start of `processCollision` may no longer own the blocks you think it does (a fragment became a *new* ship id). Practical rules:
> - Re-resolve with `loadedShips.getById(id)` after any step that destroyed blocks before using the ref for a *second* zone.
> - Attachments live on the ship object — plan a migration strategy on `splitEvent` if fatigue must survive splitting.
> - Never store `LoadedServerShip` across ticks. Store `ShipId`s.

## 6. Lifecycle Cheat-Sheet

| Operation | API | Thread | Notes |
|---|---|---|---|
| Resolve ship from collision id | `shipWorld.loadedShips.getById(id)` | Game | null for ground body / unloaded |
| List damageable ships in volume | `loadedShips.getIntersecting(aabb, dimId)` | Game | don't use the deprecated dimension-less overload |
| Create fragment ship | `createNewShipAtBlock(pos, true, 1.0, dim)` | Game | creates record only; move blocks yourself |
| Reserve id ahead of time | `allocateShipId(dim)` | Game | rarely needed; creation allocates |
| Delete ship | `deleteShip(ship)` | Game | clear its blocks first |
| Teleport ship (debris fling) | `teleportShip(ship, ShipTeleportData)` | Game | intact hull, new pose |
| Disable auto-split during batch | `setAttachment(SplittingDisablerAttachment(...))` | Game | re-enable in `finally` |
| Observe splits | `vsApi.splitEvent.on { ... }` | Game | migrate attachments here |
