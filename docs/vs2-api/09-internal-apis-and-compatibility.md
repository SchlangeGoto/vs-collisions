# Internal APIs and Compatibility

## Public versus internal

Preferred addon dependencies:

* `org.valkyrienskies.core.api.*` — public Core API (compatibility-oriented).
* `org.valkyrienskies.mod.api.*` — public Minecraft API.

Version-pinned implementation dependencies:

* `org.valkyrienskies.core.internal.*` — internal Core interfaces.
* `org.valkyrienskies.core.impl.*` — internal Core implementation.
* `org.valkyrienskies.mod.common.*` — VS2 common implementation.
* `org.valkyrienskies.mod.mixin` / `mixinducks` — mixins and duck interfaces.
* Generated Kotlin `*Kt` utilities where no public facade exists.

Internal APIs are useful for advanced server, physics, joint, chunk, and
lifecycle work, but they are **not** compatibility promises. Anything under
`internal` or `impl` may change or disappear between builds.

## Important internal types

### Core root

* `VsiCore : VsCoreApi, VsiCoreCommands` — root of the Core implementation.
  Adds `simplePacketNetworking`, `hooks`, `blockTypes`, `entityPolygonCollider`,
  `solidShapeUtils`, dummy ship worlds, terrain-update builders, pipeline
  (de)serialization, `newChunkClaim`, block-state builders, `newShipTransform`,
  `newShipTeleportData`, `newShipInertiaData`, `getServerConfig`,
  `defaultMapper`, and `registerBlockStates`. Obtained from
  `ValkyrienSkiesMod.getVsCore()`.
* `VsiCoreClient` / `VsiCoreServer` — client/server specialisations;
  `VsiCoreClient` adds `newShipWorldClient()`.
* `VsiCoreFactory` — `newVsCoreClient(hooks)` / `newVsCoreServer(hooks)`;
  `VsiCoreFactory.instance` is discovered reflectively.
* `VsiCoreCommands` — `deleteShips`, `renameShip`, `scaleShip`, `teleportShip`;
  also defines `ShipTeleportData` and its `createNewShipTransform(old)`.

### Server world and lifecycle

`VsiServerShipWorld : VsiShipWorld, ServerShipWorld` (game thread) adds:

```java
Set<VsiPlayer> getPlayers(); void setPlayers(...);
void setGameServer(VsiGameServer);
Iterator<VsiPlayer> getIPlayersWatchingShipChunk(int, int, String);
VsiChunkWatchTasks getChunkWatchTasks();
void setExecutedChunkWatchTasks(Iterable<VsiChunkWatchTask>, Iterable<VsiChunkUnwatchTask>);
ServerShip createNewShipAtBlock(Vector3ic blockPos, boolean createShipObjectImmediately,
                                double scaling, String dimensionId);
PhysicsEntityServer createPhysicsEntity(PhysicsEntityData, String dimensionId);
void deletePhysicsEntity(long id);
long allocateShipId(String dimensionId);
Map<Long, PhysicsEntityServer> retrieveLoadedPhysicsEntities();
void onDisconnect(VsiPlayer player);
void deleteShip(ServerShip ship);
void teleportShip(ServerShip ship, ShipTeleportData data);
void teleportPhysicsEntity(PhysicsEntityServer, ShipTeleportData);
Map<String, Long> getDimensionToGroundBodyIdImmutable();
```

`VsiShipWorld : ShipWorld` adds terrain and dimension management:
`onSetBlock(...)`, `addTerrainUpdates(...)`,
`forceUpdateConnectivityChunk(...)`, `addDimension(...)`,
`updateDimension(...)`, `removeDimension(...)`.

`VsiGameServer` provides `moveTerrainAcrossDimensions(...)`.

### Physics world

`VsiPhysLevel : PhysLevel` (`@PhysTickOnly`) adds:

```java
Vector3dc getGravity();
Set<PhysShip> getAllPhysShips();
VSJoint getJointById(int jointId);
Set<Integer> getJointsFromShip(long shipId);
Map<Integer, VSJoint> getAllJoints();
Map<Long, Set<Integer>> getJointsByShipIds();
int addJoint(VSJoint joint);
boolean updateJoint(int jointId, VSJoint joint);
boolean removeJoint(int jointId);
```

`getAllPhysShips` currently returns loaded physics ships (the source notes it may
eventually be per-physics-island).

### Ship implementation interfaces

* `VsiShip : Ship` — `setFromTransform(BodyTransform)`,
  `unsafeSetKinematics(BodyKinematics)`.
* `VsiLoadedShip : LoadedShip, VsiShip`.
* `VsiServerShip : ServerShip, VsiShip` — `var kinematicTarget: PhysPose?`.
* `VsiLoadedServerShip : LoadedServerShip, VsiServerShip, VsiLoadedShip` —
  `isOnPortalCoolDown()`, `handleInsidePortal()`, `decayPortalCoolDown()`.
* `VsiPhysShip : PhysShip, VsiShip` (`@PhysTickOnly`) — `kinematicTarget`,
  `dragController`.
* `VsiClientShip : VsiLoadedShip, ClientShip`.
* `VsiQueryableShipData<out S : Ship> : QueryableShipData<S>` — adds a
  dimensionless `getByChunkPos(chunkX, chunkZ)` (not for general use).
* `VsiMutableQueryableShipData<ShipType : Ship> : VsiQueryableShipData<ShipType>,
  MutableIterable<ShipType>` — `add`, `remove(ship)`, `remove(id)` (plus
  deprecated `addShipData`/`removeShipData` aliases).

### Terrain, blocks, and physics entities

* `VsiBlockType` — opaque block type; `toInt()` is deprecated ("API consumers
  should not have to worry about how block types are internally represented").
* `VsiBlockTypes` — `air`, `solid`, `lava`, `water`, `empty` types and their
  states, plus `getType(state)` / `getState(type)`.
* `VsiBlockState(solidState: SolidState?, liquidState: LiquidState?)`.
* `VsiTerrainUpdate` — `chunkX/Y/Z` and a `Builder.addBlock(x, y, z, block:
  VsiBlockType)`.
* `PhysicsEntityData` and `PhysicsEntityServer` — `@Deprecated("To be replaced by
  VsBodies eventually.")`.
* `VSCollisionShapeData` — `VOXEL`, `SPHERE`, `WHEEL`, `BOX`, `CAPSULE`
  descriptors (`VSVoxelCollisionShapeData`, `VSSphereCollisionShapeData`, ...).
* `VsiBlockType` and terrain updates are consumed by
  `VsiShipWorld.addTerrainUpdates(...)`.

### Networking, hooks, players, pipeline

* `SimplePacketNetworking` (`org.valkyrienskies.core.impl.networking.simple`) —
  register handlers and send `SimplePacket`s. Its own docs say: **"Addons
  SHOULD NOT use this. This is soft-deprecated without alternative and may be
  removed in the future."** Use your platform's networking instead.
* `VsiCoreHooks : VsiCoreHooksOut, VsiCoreHooksIn` — platform callbacks for
  networking, play state, connectivity/drag toggles, and config directory.
* `VsiPlayer` — player position, dimension, UUID, admin flags, forced-watching
  ship set.
* `VsiPipeline` — `arePhysicsRunning`, `deleteResources`, `isUsingDummyPhysics`,
  `shipWorld`, `preTickGame()`, `postTickGame()`, `computePhysTps()`,
  `getLoadedVoxelChunks()`, `getPhysicsBackendType()`.
* `VsiPlayState` — `SERVERSIDE`, `CLIENT_SINGLEPLAYER`, `CLIENT_TITLESCREEN`,
  `CLIENT_MULTIPLAYER`.

### Implementation (`impl`) types

The `impl` artifact contains concrete classes such as
`org.valkyrienskies.core.impl.api.ShipInternal`,
`LoadedServerShipInternal`, `LoadedShipInternal`, `ServerShipInternal`,
`ServerShipProvider`, `ServerShipUser`; the Dagger-based program
(`VSCoreImpl`, `VSCoreClientImpl`, `VSCoreServerImpl`,
`DaggerVSCore*Factory`); and the `shadow` package (relocated third-party code).
These are implementation details; depend on the interfaces above, not on these.

## Internal joints

Joint types live in `org.valkyrienskies.core.internal.joints` and are
`@PhysTickOnly`. See
[06-physics-and-collision-api.md](06-physics-and-collision-api.md) for the full
list and semantics. `VSGearJoint`, `VSRackAndPinionJoint`, and `VSD6Joint` are
annotated `@UnavailableInKrunch` (require the PhysX backend). Do not assume an
internal joint type is supported by every backend.

## Adapter recommendation

Keep internal access behind one module or adapter:

```text
addon feature
  -> project VS adapter
     -> pinned VsiServerShipWorld / VsiPhysLevel / VsiCore calls
```

The adapter should:

* expose application-level concepts rather than internal classes;
* validate nullability and lifecycle state;
* centralise version migrations;
* keep internal types out of public addon APIs;
* make backend limitations explicit.

## Version migration checklist

When updating VS2 or VS Core:

1. Compare API and internal source JARs, not only compiled signatures.
2. Search for renamed or deprecated properties and extension functions.
3. Recheck `@GameTickOnly` and `@PhysTickOnly` annotations.
4. Recheck attachment serializer keys and migration behaviour.
5. Recheck event payloads and backend emission cadence.
6. Recheck transform aliases and matrix scaling behaviour.
7. Recheck internal lifecycle, terrain, and joint methods.
8. Recompile against the new artifacts.
9. Run a dedicated server/client smoke test.
10. Verify both supported physics backends if applicable.

## Deprecations to watch

The resolved API contains deprecated alternatives for:

* old attachment setter overloads (`ServerShip.saveAttachment`,
  `LoadedServerShip.setAttachment(Class, T)`);
* old ship-transform aliases (`shipToWorld`, `positionInShip`,
  `transformDirectionNoScalingFrom*`, ...);
* dimensionless AABB queries (`getIntersecting(AABBdc)`,
  `getShipDataIntersecting`);
* old ship-data query names (`getShipDataFromChunkPos`, `idToShipData`);
* older force/torque methods (`applyInvariant*`, `applyRotDependent*`);
* `ShipForcesInducer` (Kotlin `DeprecationLevel.ERROR`);
* `PhysPose`, `ShipInertiaData`, `ShipTransform.copy/createEmptyAABB`;
* `VsiCore.isChunkInShipyard(chunkX, chunkZ)` (no dimension);
* `SimplePacketNetworking` (soft-deprecated without replacement);
* `SeatedControllingPlayer` (will be removed);
* `org.valkyrienskies.core.impl.hooks.VSEvents`;
* `PhysicsEntityData` / `PhysicsEntityServer` (to be replaced by VS bodies).

Use the replacement named by the source annotation. Do not suppress a
deprecation globally when a local migration is possible.
