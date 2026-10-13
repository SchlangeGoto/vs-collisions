# Ships, Bodies, Transforms, and Forces

## The ship type hierarchy

```text
Identified (id: Long)
└── Ship                     (common view: id, slug, transform, chunk claim, AABBs, velocity)
    ├── ServerShip           (game-thread server data; no attachments)
    │   └── LoadedServerShip (also AttachmentHolder + LoadedShip; has wingManager/dragController)
    ├── ClientShip           (client representation; render transform/AABB)
    └── PhysShip             (physics-thread body; force/torque API)
```

Supporting interfaces: `LoadedShip : Ship` (marker for a loaded representation);
`LoadedServerShip : AttachmentHolder, LoadedShip, ServerShip`.

Internal sub-interfaces add behaviour: `VsiShip` (`setFromTransform`,
`unsafeSetKinematics`), `VsiServerShip` (`kinematicTarget`), `VsiLoadedServerShip`
(portal cooldown handling), `VsiPhysShip` (`kinematicTarget`, `dragController`),
`VsiClientShip`, `VsiLoadedShip`.

## `Ship` — the common view

Source: `org/valkyrienskies/core/api/ships/Ship.kt`. Verified members:

| Member | Type | Notes |
| --- | --- | --- |
| `id` | `ShipId` (`Long`) | Unique; never `-1` |
| `slug` | `String?` | UI name (e.g. `/vs` commands); mutable on `ServerShip` |
| `kinematics` | `BodyKinematics` | `@VsBeta`; transform + velocity + angular velocity |
| `transform` | `ShipTransform` | Alias for `kinematics.transform` |
| `prevTickTransform` | `ShipTransform` | Previous game tick, or current if none |
| `chunkClaim` | `ChunkClaim` | Shipyard region owning the ship's blocks |
| `chunkClaimDimension` | `DimensionId` | Dimension of the claim |
| `worldAABB` | `AABBdc` | World-space bounds |
| `shipAABB` | `AABBic?` | Model-space (integer) bounds; `null` if no blocks |
| `activeChunksSet` | `IShipActiveChunksSet` | Loaded/ticking ship chunks |
| `velocity` | `Vector3dc` | Alias for `kinematics.velocity` |
| `angularVelocity` | `Vector3dc` | Alias for `kinematics.angularVelocity` |
| `shipToWorld` | `Matrix4dc` | Alias for `transform.toWorld` |
| `worldToShip` | `Matrix4dc` | Alias for `transform.toModel` |

`worldAABB` is the ship's `shipAABB` transformed to world space, or a zero-volume
AABB at the transform position when the ship has no blocks. It does **not**
account for non-full-block shapes: a single-slab ship still has a full-block
bounding box.

Deprecated aliases on `Ship`: `prevTickShipTransform`, `omega`, `shipTransform`,
`shipVoxelAABB`, `shipActiveChunksSet`. Prefer the names above.

### `ServerShip`

Game-thread server data. Adds:

* `slug` is overridable (`var slug: String?`);
* `isStatic: Boolean`;
* `inertiaData: ShipInertiaData` (deprecated in favour of `BodyInertia`);
* `transformProvider: ServerShipTransformProvider?`;
* `unsafeSetTransform(BodyTransform)` — `@Experimental`.

`ServerShip` cannot store attachments. Its `saveAttachment`/`getAttachment`
helpers are deprecated at `DeprecationLevel.ERROR` since VS 2.5; attachments
belong to `LoadedServerShip`.

### `LoadedServerShip`

`@GameTickOnly`, and an `AttachmentHolder`. Adds:

* `wingManager: WingManager?` (`@VsBeta`);
* `dragController: DragController?`;
* the full `AttachmentHolder` API (`getAttachment`, `getOrPutAttachment`,
  `setAttachment`, `removeAttachment`, `getAllAttachments`).

There is a legacy `setAttachment(clazz, value)` overload kept only for
compatibility.

### `ClientShip`

`@GameTickOnly`. Adds:

* `renderTransform: ShipTransform` — the transform used for rendering
  (interpolated; distinct from the authoritative server transform);
* `renderAABB: AABBdc`;
* `transformProvider: ClientShipTransformProvider?`.

Never treat a `ClientShip` as server state. Server and client transforms can
differ; when sending a position between sides, convert to ship-local space with
one side's transform and back with the other's.

### `PhysShip`

`@PhysTickOnly`. Adds the live physics properties and the force/torque API:

| Property | Type | Notes |
| --- | --- | --- |
| `isStatic` | `Boolean` | Kinematic vs dynamic body |
| `buoyantFactor` | `Double` | `@Experimental` |
| `doFluidDrag` | `Boolean` | `@Experimental` |
| `mass` | `Double` | kg, scaling applied |
| `centerOfMass` | `Vector3dc` | model space, `@Experimental` getter |
| `momentOfInertia` | `Matrix3dc` | model space, scaling applied |
| `liquidOverlap` | `Double` | 0–1 fluid overlap, updated after each physics tick, `@Experimental` |
| `isSleeping` | `Boolean` | `@Experimental`; sleeping bodies are not simulated |

`PhysShip` is obtained via `PhysLevel.getShipById(long)` on the physics thread, or
as the argument to a `ShipPhysicsListener`. It must not be stored on the game
thread.

## Transforms and coordinate spaces

VS Core distinguishes three coordinate spaces:

* **World space** — ordinary dimension coordinates.
* **Model space** (a.k.a. shipyard space) — the block grid the ship's blocks
  occupy; where Minecraft reports block positions.
* **Body space** — an offset basis centred on the centre of mass, with no
  rotation or scaling applied.

`BodyTransform` (source: `org/valkyrienskies/core/api/bodies/properties/BodyTransform.kt`)
is the canonical transform:

| Property | Meaning |
| --- | --- |
| `position` | Body position in world space |
| `rotation` | Normalised world rotation quaternion |
| `scaling` | Model-to-world scale (physics currently only supports uniform scale) |
| `positionInModel` | Body position in model space (typically the centre of mass) |
| `toWorld` | Affine model → world matrix |
| `toModel` | Inverse world → model matrix |

The documented construction is:

```text
toWorld = translate(position) * rotate(rotation) * scale(scaling)
          * translate(-positionInModel)
toModel = inverse(toWorld)
```

Use `toWorld.transformPosition(p)` for points and `toWorld.transformDirection(d)`
for directions. A direction transform ignores translation but **includes
scaling**; use `rotation.transform(d)` to rotate without scaling, and normalise
when a unit vector is required.

`BodyTransform` also provides `withVelocity(velocity, angularVelocity)`,
`toBuilder()`, and `writeTransform(DataOutput)` (13 doubles: position xyz,
positionInModel xyz, scaling xyz, rotation xyzw). The `@VsBeta` `Factory`
(`VsCoreApi.getTransformFactory()`) offers `newBuilder`, `newBuilder(from)`,
`readFrom(DataInput)`, `build { ... }`, `create(...)`, and
`createFromSlerp(previous, current, alpha)`.

### `ShipTransform` (compatibility)

`ShipTransform extends BodyTransform` and is `@NonExtendable` and
"soft-deprecated"; it exists for compatibility and it is always safe to cast a
`BodyTransform` to `ShipTransform`. It adds aliases:

| Compatibility name | Preferred name |
| --- | --- |
| `positionInWorld` | `position` |
| `positionInShip` | `positionInModel` |
| `shipToWorldRotation` | `rotation` |
| `shipToWorldScaling` | `scaling` |
| `shipToWorld` | `toWorld` |
| `worldToShip` | `toModel` |

Deprecated members: `createEmptyAABB()`, `copy(...)`,
`transformDirectionNoScalingFromShipToWorld(...)`,
`transformDirectionNoScalingFromWorldToShip(...)`, and the older
`shipPositionInWorldCoordinates` / `shipPositionInShipCoordinates` /
`shipCoordinatesToWorldCoordinates*` / `shipToWorldMatrix` / `worldToShipMatrix`
aliases. Prefer the `BodyTransform` names and the plain rotation methods.

### `BodyKinematics`, `BodyInertia`, `BodyPose`

* `BodyKinematics` (`@VsBeta`) = `BodyTransform` + `velocity` (blocks/s) +
  `angularVelocity` (rad/s), with convenience getters that delegate to the
  transform and `withTransform`, `toBuilder`, and a `Builder`
  (`position`, `positionInModel`, `rotation`, `scaling`, `velocity`,
  `angularVelocity`, `fromTransform`, `from`, `build`).
* `BodyInertia` = `inertiaTensor: Matrix3dc` (model space), `centerOfMass:
  Vector3dc` (model space), `mass: Double`. The interface documents angular
  momentum `L = I·ω` and moment about an axis `nᵀIn`.
* `BodyPose` (`@VsBeta`) = `position` (world) + `rotation` (normalised).
* `ShipInertiaData : BodyInertia` and `PhysPose : BodyPose` are deprecated
  compatibility types; use `BodyInertia` / `BodyPose`.

## Force and torque application

Apply force and torque to a `PhysShip` from the physics thread. The vector space
(how the force is oriented) and the application-position space (where it acts)
are independent, and each method names both.

Verified `PhysShip` methods (`org/valkyrienskies/core/api/ships/PhysShip.kt`):

| Method | Force vector space | Application position space |
| --- | --- | --- |
| `applyWorldForce(force, pos)` | World | World (default: `kinematics.position`) |
| `applyWorldTorque(torque)` | World | centre of mass |
| `applyModelForce(force, pos)` | Model | Model (default: `kinematics.positionInModel`) |
| `applyModelTorque(torque)` | Model | centre of mass |
| `applyWorldForceToModelPos(force, pos)` | World | Model (default: `kinematics.positionInModel`) |
| `applyBodyForce(force, pos)` | Body | Body offset (default `(0,0,0)`) |
| `applyBodyTorque(torque)` | Body | centre of mass |
| `applyWorldForceToBodyPos(force, pos)` | World | Body offset (default `(0,0,0)`) |

Space semantics, quoting the source documentation:

* **World** force/torque is independent of the ship's transform; "up" is always
  `(0,1,0)`. A world position is translated into the shipyard before being
  offset from the centre of mass.
* **Model** force/torque rotates and scales with the ship; `(0,1,0)` is
  perpendicular to world up when the ship is on its side. A model position is a
  shipyard coordinate.
* **Body** force/torque is relative to the centre of mass with no transform
  applied; a body position is an offset from the centre of mass. This is the
  rawest path and applies no extra transforms.

Passing a non-shipyard coordinate to a *model* method offsets the force by a
huge distance and will fling the ship; use a *world* method for external
sources.

Deprecated (replace as shown):

| Deprecated | Replacement |
| --- | --- |
| `applyRotDependentForce` | `applyModelForce` or `applyBodyForce` |
| `applyInvariantForce` | `applyWorldForce` |
| `applyRotDependentForceToPos` | `applyModelForce` or `applyBodyForce` |
| `applyInvariantForceToPos` | `applyWorldForce` |
| `applyRotDependentTorque` | `applyModelTorque` or `applyBodyTorque` |
| `applyInvariantTorque` | `applyWorldTorque` |

### Generic bodies

Core also exposes a lower-level body abstraction used for physics entities
(see `org.valkyrienskies.core.api.bodies`):

```text
BaseVsBody (id, dimension, transform: BodyKinematics, aabb, velocity)
└── ServerBaseVsBody (mass, centerOfMass, momentOfInertia, isStatic, buoyantFactor,
    │                 doFluidDrag, setTransform/setPosition/setRotation/setScaling)
    ├── ServerVsBody
    └── PhysicsVsBody (force/torque API)
VsBody : BaseVsBody (@NonExtendable)
```

`PhysicsVsBody` uses a **reason string** and separate rotating/invariant
variants:

```java
applyForceInWorld(force);            applyForceInLocal(force);
applyForceInWorld(reason, force);    applyForceInLocal(reason, force);
applyRotatingForceInWorld(...);      applyInvariantForceInLocal(...);
applyTorqueInWorld(torque);          applyTorqueInLocal(torque);
applyRotatingTorqueInWorld(...);     applyInvariantTorqueInLocal(...);
// ...and position-taking overloads
```

By default, a local-space force rotates with the body and a world-space force
does not. Override with the invariant/rotating variants. The default reason is
`PhysicsVsBody.REASON_UNKNOWN` (`"unspecified"`). This API is separate from the
ship-oriented `PhysShip` force methods and is mostly relevant to physics
entities and custom bodies.

## Per-ship listeners

### `ShipPhysicsListener`

The attachment callback for per-ship physics work (source:
`org/valkyrienskies/core/api/ships/ShipPhysicsListener.kt`):

```java
void physTick(PhysShip physShip, PhysLevel physLevel);                 // required
default void physTick(PhysShip physShip, PhysLevel physLevel, double delta) {
    physTick(physShip, physLevel);
}
```

The three-argument overload defaults to the two-argument one. `delta` is
milliseconds since the previous physics tick. Attach it to a
`LoadedServerShip` as an attachment and it is invoked on the physics thread.
This is the recommended way to apply forces safely.

### `ServerTickListener`

```java
void onServerTick();   // game thread, per ship
```

Use for game-thread per-ship work.

### `ShipForcesInducer`

Deprecated at `DeprecationLevel.ERROR` with message "use `ShipPhysicsListener`
instead". Do not use it in new code.

## Transform providers

`ServerShipTransformProvider` lets an addon override the pose the physics engine
computes:

```java
NextTransformAndVelocityData provideNextTransformAndVelocity(
    ShipTransform prevShipTransform, ShipTransform shipTransform);

record NextTransformAndVelocityData(
    BodyTransform nextTransform, Vector3dc nextVel, Vector3dc nextOmega) {}
```

Return `null` to fall back to VS2's default behaviour. Setting
`ServerShip.transformProvider` is game-thread work.

`ClientShipTransformProvider` controls client interpolation:

```java
BodyTransform provideNextTransform(
    ShipTransform prevShipTransform, ShipTransform shipTransform,
    ShipTransform latestNetworkTransform);

BodyTransform provideNextRenderTransform(
    ShipTransform prevShipTransform, ShipTransform shipTransform,
    double partialTick);
```

Return `null` for VS2's default interpolation. Set through
`ClientShip.transformProvider` on the client thread.
