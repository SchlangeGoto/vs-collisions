# Coordinate transforms and spatial math

Related: [Documentation index](../index.md) · [Collision data flow](collision-data-flow.md) · [VS2/Core API reference](../reference/vs2-core-api.md)

---

## 1. The Two Spaces You Must Keep Separate

| Space | What lives there | Frame |
|---|---|---|
| **World space** | Physics contact positions, normals, relative velocities, rendered ship positions | Absolute world coordinates |
| **Shipyard (ship / model) space** | The actual `BlockState`s of the ship — stored in far-away "shipyard" chunks of the same `ServerLevel` | Block coordinates inside the ship's `ChunkClaim` |

Physics collision events arrive **entirely in world space**. Every `level.getBlockState(...)` / `level.destroyBlock(...)` call on a ship must be made with **shipyard coordinates**. The bridge is the ship's transform.

## 2. The Transform API (verified against `BodyTransform.kt`)

```kotlin
// org.valkyrienskies.core.api.bodies.properties.BodyTransform
interface BodyTransform {
    val position: Vector3dc        // world-space position of the body
    val rotation: Quaterniondc     // model → world rotation
    val scaling: Vector3dc         // (1,1,1) for normal ships
    val positionInModel: Vector3dc // where the body sits in the shipyard (~center of mass)

    val toWorld: Matrix4dc         // shipyard → world   (homogeneous)
    val toModel: Matrix4dc         // world → shipyard   (= toWorld inverted)
}
```

Definition of the matrices (from the docs):

```kotlin
val toWorld = Matrix4d()
    .translate(position)                 // 4. world position
    .rotate(rotation)                    // 3. orientation
    .scale(scaling)                      // 2. scale
    .translate(positionInModel.negate()) // 1. leave shipyard origin
val toModel = Matrix4d(toWorld).invert()
```

On a `Ship` you also get convenience aliases (verified in `Ship.kt`):

```kotlin
val Ship.shipToWorld: Matrix4dc get() = transform.toWorld
val Ship.worldToShip: Matrix4dc get() = transform.toModel
```

> [!note] `ShipTransform` is soft-deprecated
> `ShipTransform` (the old interface with `positionInWorld`, `shipCoordinatesToWorldCoordinatesRotation`, …) exists only for compatibility; it is always safe to cast `BodyTransform` → `ShipTransform`, and `ship.transform` returns the legacy type. The codebase uses `ship.transform.worldToShip`. New code should prefer `ship.transform.toModel` / `ship.kinematics.transform`.

## 3. Contact Point → Ship Block (the pin-point-damage recipe)

This is the exact transform chain used by `BlockImpactInjector.inject`:

```kotlin
import org.joml.Vector3d
import org.valkyrienskies.core.api.ships.LoadedServerShip

fun ImpactZone.toShipLocal(ship: LoadedServerShip): Pair<Vector3d, Vector3d> {
    // 1. World-space impact centroid to a shipyard-space position
    val localPos = ship.transform.worldToShip.transformPosition(Vector3d(position))

    // 2. World-space impact direction to a shipyard-space DIRECTION
    //    (uses the same matrix here because scaling is (1,1,1); see §4)
    val localDir = ship.transform.worldToShip.transformDirection(Vector3d(impactDirection))
    if (localDir.lengthSquared() > 0) localDir.normalize()

    return localPos to localDir
}

// Then snap to a block grid position:
val seed = BlockPos(
    Math.floor(localPos.x).toInt(),
    Math.floor(localPos.y).toInt(),
    Math.floor(localPos.z).toInt()
)
```

and the inverse, for feedback effects in world space (particles, sounds, camera shake):

```kotlin
val worldPos = ship.transform.shipToWorld.transformPosition(Vector3d(localPos))
```

`Vector3d` constructors around every input (`new Vector3d(...)` in Java, `Vector3d(x)` in Kotlin) are **defensive copies** — preserve this habit, because both JOML and VS2 internals reuse buffer vectors.

## 4. Positions vs. Directions — the Pitfall

`Matrix4dc` has two different transform methods, and mixing them up is the #1 coordinate bug:

```kotlin
matrix.transformPosition(v)  // applies translation + rotation + scale   — for POINTS
matrix.transformDirection(v) // applies rotation + scale, NO translation — for DIRECTIONS/NORMALS
```

If you transform an impact *direction* with `transformPosition`, the ship's world position gets added to the vector — the stress injection direction then points somewhere in outer space and your directional falloff silently corrupts.

> [!warning] Non-uniform scaling
> `transformDirection`, like the matrix, includes scaling. For directions you want pure rotation when `scaling != (1,1,1)`:
> ```kotlin
> val localDir = ship.transform.rotation.transformInverse(Vector3d(worldDir)) // world → model
> val worldDir = ship.transform.rotation.transform(Vector3d(localDir))        // model → world
> ```
> (Verified in `BodyTransform.rotation` docs and the deprecated-but-correct `ShipTransform.transformDirectionNoScalingFromWorldToShip`.)
>
> Normal ships use `scaling = (1,1,1)`, which is why the current matrix approach is safe today. If you ever enable VS2's experimental ship scaling, normals also need inverse-transpose treatment — rotation-only is the cheap correct approximation.

## 5. Geometry You Get for Free

From `Ship.kt` / `ShipTransform.kt`:

| Accessor | Space | Use in this codebase |
|---|---|---|
| `ship.worldAABB: AABBdc` | world | broad-phase sanity checks, effect placement |
| `ship.shipAABB: AABBic?` | shipyard | `ZoneLoadCalculator` computes ship radius for torque falloff (note: **nullable** — see warning) |
| `ship.chunkClaim` | shipyard chunks | the region of the level the ship owns |
| `ship.prevTickTransform` | — | interpolation, impact-relative velocity reconstruction |
| `ship.inertiaData.centerOfMass` | shipyard | lever arm `r = impactPos − CoM` for torque factor |
| `ship.velocity` / `ship.angularVelocity` | world | kinematics of the whole body |

> [!warning] `shipAABB` can be `null`
> A ship with **no blocks** has `shipAABB == null`. `ZoneLoadCalculator` currently calls `ship.getShipAABB()` unguarded — if a ship loses its last block while a collision is being processed, that's an NPE on the game thread. Guard it:
> ```kotlin
> val aabb = ship.shipAABB ?: return null   // treat as no geometry to damage
> ```

## 6. Lever-Arm / Torque Math (as implemented)

`ZoneLoadCalculator.compute` does, per ship:

```kotlin
val localZonePos = ship.transform.worldToShip.transformPosition(Vector3d(zone.position))
val r = Vector3d(localZonePos).sub(ship.inertiaData.centerOfMass)   // lever from CoM
val leverArm = Vector3d(r).cross(localForceDir)                     // r × F̂

val aabb = ship.shipAABB!!                                          // ⚠ see warning above
val shipRadius = maxOf(1.0, sqrt(
    (aabb.maxX() - aabb.minX()).toDouble().pow(2) +
    (aabb.maxY() - aabb.minY()).toDouble().pow(2) +
    (aabb.maxZ() - aabb.minZ()).toDouble().pow(2)
) / 2.0)

val leverRatio = leverArm.length() / shipRadius
val torqueFactor = leverRatio / (leverRatio + 1.0)                  // ∈ [0, 1)
val spinLoss = torqueFactor * SPIN_ENERGY_LOSS_FACTOR               // 0.3
val damageEnergy = kineticEnergy * (1.0 - spinLoss)
```

Interpretation: a hit far off-axis converts part of its energy into *rotation* of the ship rather than structural damage — the torque factor subtracts that portion and simultaneously *widens* the injection radius in `BlockImpactInjector` (`torqueBonus = torqueFactor * 3.0`), modeling a glancing blow raking along the hull.

## 7. Why the Impact Block Must Be Searched

A physics contact point (world space) transformed into the shipyard rarely lands exactly on a voxel — ship colliders slightly overlap their mesh, and the centroid of a contact cluster floats outside the hull. Hence `BlockImpactInjector.findNearestSolidBlock` does a growing shell search (`r = 0..3`) returning the first non-air, in-shipyard position. Only shipyard positions are accepted (`VSGameUtilsKt.isBlockInShipyard(level, pos)`), so a miss transformed into far-away shipyard wilderness is rejected rather than damaging a random location.

Shell-only iteration (`max(|dx|,|dy|,|dz|) == r`) avoids re-testing the interior of smaller radii — O(shell) instead of O(cube).
