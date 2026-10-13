# Advanced Integration Facilities

This chapter covers facilities beyond the core ship/physics path. Their
availability and stability are more version-sensitive than the fundamental ship
interfaces.

## Wings

`Wing` and `WingManager` (`org.valkyrienskies.core.api.ships`) model aerodynamic
wing groups on a `LoadedServerShip`. `WingManager` is reached through
`LoadedServerShip.getWingManager()` (a `@VsBeta` nullable property).

```kotlin
data class Wing(
    val wingNormal: Vector3dc,
    val wingLiftPower: Double,
    val wingDragPower: Double,
    val wingBreakingForce: Double?,       // null = infinite
    val wingCamberAttackAngleBias: Double // radians; non-zero = cambered airfoil
)
data class PositionedWing(val posX: Int, val posY: Int, val posZ: Int, val wing: Wing?)
typealias WingGroupId = Int
```

`WingManager` (verified):

```java
WingGroupId createWingGroup(boolean isContraption);   // default false
boolean deleteWingGroup(WingGroupId wingGroupId);
boolean setWing(WingGroupId id, int posX, int posY, int posZ, Wing wing); // true if it replaced one
Wing getWing(WingGroupId id, int posX, int posY, int posZ);
void setWingGroupTransform(WingGroupId id, Matrix4dc transform);
WingManagerChanges getWingChanges();
void clearWingChanges();       // VS Core internal use
WingGroupId getFirstWingGroupId();
```

`WingManagerChanges` / `WingGroupChanges` expose pending wing/transform changes
for the game-to-physics pipeline. Wing state is server/game state: do not update
it from `@PhysTickOnly` callbacks.

`ContraptionWingProvider` supplies wing data for integrations that derive wings
from contraptions:

```java
WingGroupId getWingGroupId(); void setWingGroupId(WingGroupId id);
Matrix4dc computeContraptionWingTransform();
```

## Drag and wind

`DragController` (`org.valkyrienskies.core.api.ships`, `@VsBeta @GameTickOnly`)
manages aerodynamic drag state on a `LoadedServerShip`. Reached through
`LoadedServerShip.getDragController()`.

```java
void enableDrag();  void disableDrag();   // disabling also falls back to Weather2 wind
void enableLift();  void disableLift();
void enableRotDrag(); void disableRotDrag();
Vector3dc getDragForce();
Vector3dc getLiftForce();
void setWindDirection(double x, double y, double z, String identifier); // default "default"
void setWindDirection(Vector3dc direction, String identifier);
void setWindSpeed(double speed, String identifier);
void applyWindImpulse(Vector3dc direction, double speed);  // direction normalized, speed >= 0
Vector3dc getWindVector();
```

Treat drag configuration as game-thread ship state. The controller is not a
replacement for `PhysShip` force application; use it when the VS2 aerodynamic
model is the intended source of drag.

## `AerodynamicUtils`

`AerodynamicUtils` (`@VsBeta`) provides atmospheric calculations and defaults,
available from both `PhysLevel.getAerodynamicUtils()` (physics) and
`ServerShipWorld.getAerodynamicUtils()` (game). Verified members:

```java
void registerDimension(String dimensionId, double maxY, double seaLevel, double gravity);
void updateGravityForDimension(String dimensionId, double gravity);
void updateAerodynamicInfoForDimension(String dimensionId, Double maxY, Double seaLevel);
Triple<Double,Double,Double> getAtmosphereForDimension(String id);   // <maxY, seaLevel, gravity>
double getAirDensityForY(double y, String dimension);     // kg/m^3
double getAirPressureForY(double y, String dimension);    // Pa
double getAirTemperatureForY(double y, String dimension); // K
```

Constants: `DEFAULT_MAX = 962.0`, `DEFAULT_SEA_LEVEL = 62.0`,
`GRAVITATIONAL_ACCELERATION = 10.0`, `UNIVERSAL_GAS_CONSTANT = 8.314`,
`AIR_MOLAR_MASS = 0.0289644`. The density/pressure/temperature formulas follow
the barometric formula; `Y = 60` is sea level and `Y = 320` the top of the
troposphere.

Aerodynamic calculations require correct world/model coordinates, atmospheric
inputs, and ship orientation. Do not mix a model-space wing position with a
world-space wind vector without applying the documented transform.

## Game-to-physics force adapter

`org.valkyrienskies.mod.common.util.GameToPhysicsAdapter` lets the game thread
queue force/torque/joint/collision requests that are applied during
`PhysTickEvent`. Verified methods include:

```java
void applyWorldForce(long shipId, Vector3dc force, Vector3dc pos);
void applyWorldTorque(long shipId, Vector3dc torque);
void applyModelForce(long shipId, Vector3dc force, Vector3dc pos);
void applyModelTorque(long shipId, Vector3dc torque);
void applyWorldForceToModelPos(long shipId, Vector3dc force, Vector3dc pos);
void applyBodyForce(long shipId, Vector3dc force, Vector3dc pos);
void applyBodyTorque(long shipId, Vector3dc torque);
void applyWorldForceToBodyPos(long shipId, Vector3dc force, Vector3dc pos);
void setStatic(long shipId, boolean value);
void addJoint(VSJoint joint, int delay, Consumer<VSJointId> callback);
void updateJoint(VSJointAndId jointAndId);
void removeJoint(VSJointId jointId);
VSJoint getJointById(VSJointId jointId);
Set<VSJointId> getJointsFromShip(long shipId);
Map<VSJointId, VSJoint> getAllJoints();
Map<Long, Set<VSJointId>> getJointsByShipIds();
void enableCollisionBetween(long a, long b);
void disableCollisionBetween(long a, long b);
List<Long> getAllConnectedShips(long start);
Double getLiquidOverlap(long id);
```

Obtain one per dimension with `ValkyrienSkiesMod.getOrCreateGTPA(dimensionId)`.
This is the bridge the Minecraft integration uses so that game-thread code can
request physics work without touching `PhysShip` directly. It is `mod.common`
implementation, not public Core API.

## Simple packet networking

`org.valkyrienskies.core.impl.networking.simple.SimplePacketNetworking` can
register handlers and send `SimplePacket`s:

```java
<T : SimplePacket> RegisteredListener registerServerHandler(KClass<T>, Function2<T, VsiPlayer, Unit>);
<T : SimplePacket> RegisteredListener registerClientHandler(KClass<T>, Function1<T, Unit>);
void sendToServer(SimplePacket packet);
void sendToClient(SimplePacket packet, VsiPlayer player);
void sendToClients(SimplePacket packet, VsiPlayer... players);
void sendToAllClients(SimplePacket packet);
void register(KClass<? extends SimplePacket> klass);
```

**Do not use this.** Its own documentation says addons should not use it, that
it is soft-deprecated without alternative, and that it may be removed. Use your
platform's networking (Forge `SimpleChannel`, etc.).

## Commands and server control

VS Core internal sources contain command and server-control classes
(`VsiCoreCommands`, and VS2's own `mod.common.command` package). They are useful
for VS2's own `/vs` commands and diagnostics but are internal. Do not expose
internal command objects from an addon API. Register commands through the
platform command lifecycle and call game-thread ship operations from command
execution.

## Assembly and schematics

`org.valkyrienskies.mod.common.assembly` contains experimental copy/paste
support (see [08-minecraft-integration.md](08-minecraft-integration.md)):

* `ShipAssembler` — assembles a set of blocks into a new ship.
* `ICopyableAttachment` / `ICopyableBlock` — opt-in copy/paste hooks.
* `VSAssemblyEvents` / `VSSchematicEvents` — `EventEmitterImpl` streams for
  `BeforeCopy`, `OnPasteBeforeBlocksAreLoaded`, `OnPasteAfterBlocksAreLoaded`,
  and `PasteSurvivalCost`.

`SplitHandler` (`mod.common.util`) is a worked example of using the connectivity
APIs: it queues a split after a grace timer, finds the largest connected solid
component, indexes disconnected components with
`indexSolidComponentVoxels(...)`, and assembles each into a new ship.

## Ship mounts and player control

* `ShipMountedToData(shipMountedTo: LoadedShip, mountPosInShip: Vector3dc)` and
  `ShipMountedToDataProvider` describe an entity mounted to a ship.
* `VsApi.isShipMountingEntity(Entity)`, `getShipMountedTo(Entity)`, and
  `getMountPosInShip(Entity[, float])` expose mount queries.
* `SeatedControllingPlayer` is a deprecated attachment holding player input for
  a `ShipMountingEntity`; it will be removed. Addons should make their own seat
  entities and handle their own input.
* `ShipyardPosSavable` lets an entity persist an unloaded shipyard position.

## Ship attachments shipped with VS2

VS2 registers several attachments on load (source: `ValkyrienSkiesMod.init()`):

| Attachment | Serializer | Purpose |
| --- | --- | --- |
| `ShipSettings` | default | Per-ship settings (`shouldGenerateChunks`, `changeDimensionOnTouchPortals`) |
| `SeatedControllingPlayer` | legacy | Deprecated player input |
| `SplittingDisablerAttachment` | legacy | Guards against splitting a ship while it is being processed |
| `BuoyancyHandlerAttachment` | default | `ShipPhysicsListener` that sets `buoyantFactor` from air-pocket volume |

`LoadedServerShip.settings` is a Kotlin extension returning `ShipSettings`.
`BuoyancyHandlerAttachment` reads `pocketVolumeTotal`/`pocketCenterAverage` and
sets `physShip.buoyantFactor`; the buoyancy factor is configured by
`VSGameConfig.SERVER.enablePocketBuoyancy` and
`buoyancyFactorPerPocketVolume`.

## Data ownership summary

| Facility | Primary thread | Stability |
| --- | --- | --- |
| Wing manager | Game | Public/Beta, version-scoped |
| Drag controller | Game | Public/Beta, version-scoped |
| Aerodynamic utilities | Game/Physics world | Public/Beta |
| `GameToPhysicsAdapter` | Game producer / physics consumer | Internal (`mod.common`) |
| Configuration models | Initialisation/game | Mixed public/internal |
| Datagen providers | Datagen process | Public integration API |
| Entity/block-entity physics listeners | Physics callback | Public integration API |
| Simple packet networking | Platform/network | Internal, soft-deprecated |
| Core commands | Game/server | Internal |
| Assembly/schematics | Game/server | Experimental |
