# Core API and Lifecycle

## `VsCoreApi` — the platform-neutral facade

`org.valkyrienskies.core.api.VsCoreApi` is the single entry point to VS Core. Its
documentation states that Core works hard to keep classes and functions in
`org.valkyrienskies.core.api.*` backward compatible. There is generally one
instance per running game, obtained through the Minecraft-dependent Mod API.

Verified members (source: `org/valkyrienskies/core/api/VsCoreApi.kt`):

### Event streams

All are `ListenableEvent<...>` properties. Every event is `@VsBeta` unless noted.

| Property | Payload | Emitting domain |
| --- | --- | --- |
| `shipLoadEvent` | `ShipLoadEvent` (`LoadedServerShip`) | Game |
| `shipLoadEventClient` | `ShipLoadEventClient` (`ClientShip`) | Client |
| `shipUnloadEventClient` | `ShipUnloadEventClient` (`ClientShip`) | Client |
| `startUpdateRenderTransformsEvent` | `StartUpdateRenderTransformsEvent` (`ClientShipWorld`) | Client |
| `tickEndEvent` | `TickEndEvent` (`ServerShipWorld`) | Game |
| `collisionStartEvent` | `CollisionEvent` | Physics |
| `collisionEndEvent` | `CollisionEvent` | Physics |
| `collisionPersistEvent` | `CollisionEvent` | Physics |
| `mergeEvent` | `MergeEvent` | Game |
| `splitEvent` | `SplitEvent` | Game |
| `physTickEvent` | `PhysTickEvent` | Physics |
| `jointSerializedEvent` | `JointSerializedEvent` | Physics |
| `jointDeserializedEvent` | `JointDeserializedEvent` | Physics |

See [04-events-and-callbacks.md](04-events-and-callbacks.md) for payload fields.

### Factories

```java
BodyTransform.Factory getTransformFactory();          // @VsBeta

BodyKinematics newBodyKinematics(
    Vector3dc velocity, Vector3dc angularVelocity, BodyTransform transform);

BodyKinematics newBodyKinematics(
    Vector3dc velocity, Vector3dc angularVelocity, Vector3dc position,
    Quaterniondc rotation, Vector3dc scaling, Vector3dc positionInModel);

BodyKinematics.Builder newBodyKinematicsBuilder();

BodyTransform newBodyTransform(
    Vector3dc position, Quaterniondc rotation,
    Vector3dc scaling, Vector3dc positionInModel);

BodyTransform.Builder newBodyTransformBuilder();
```

Use these instead of constructing implementation types directly; they hide the
backend representation and are the intended public construction path.

### Attachment registration

```java
<T> AttachmentRegistration.Builder<T> newAttachmentRegistrationBuilder(Class<T> attachmentClass);

<T> void registerAttachment(AttachmentRegistration<T> registration);
<T> void registerAttachment(Class<T> attachmentClass);                       // default settings
<T> void registerAttachment(Class<T> attachmentClass,
        Function1<AttachmentRegistration.Builder<T>, Unit> configure);        // Kotlin DSL

void registerAttachmentForRemoval(String attachmentKey);
```

Registration must happen during mod initialisation, before first use. See
[05-attachments-and-serialization.md](05-attachments-and-serialization.md).

## `VsApi` — the Minecraft facade

`org.valkyrienskies.mod.api.VsApi` extends `VsCoreApi` and adds the
Minecraft-facing surface. Obtain it with `ValkyrienSkies.api()`:

```java
VsApi api = org.valkyrienskies.mod.api.ValkyrienSkies.api();
```

`ValkyrienSkies.api()` is a lazy accessor that resolves
`ValkyrienSkiesMod.getApi()` reflectively and throws `IllegalStateException` if
VS2 is absent. `ValkyrienSkiesMod.getApi()` returns the same instance and lives
in `org.valkyrienskies.mod.common` (the implementation package).

`VsApi` adds (verified via bytecode):

| Member | Purpose |
| --- | --- |
| `registerBlockStateEvent` | `ListenableEvent<RegisterBlockStateEvent>` for physics block-state registration |
| `preRenderShipEvent`, `postRenderShipEvent` | Client render events |
| `isShipMountingEntity(Entity)` | Whether an entity is a VS2 seat/mount |
| `getDimensionId(Level)` | The `DimensionId` string for a level |
| `getShipMountedTo(Entity)`, `getMountPosInShip(Entity[, float])` | Ship-mount queries |
| `getServerShipWorld(MinecraftServer?)`, `getServerShipWorld()` | Server ship world lookup |
| `getClientShipWorld(Minecraft?)`, `getClientShipWorld()` | Client ship world lookup |
| `getShipWorld(Level?)` | Server or client ship world for a level |
| `isChunkInShipyard(Level?, int, int)` | Shipyard chunk test |
| `getShipManagingBlock(Level?, BlockPos?)` | Ship owning a shipyard block |
| `getShipManagingChunk(Level?, ChunkPos?)`, `getShipManagingChunk(Level?, int, int)` | Ship owning a shipyard chunk |
| `getShipsIntersecting(Level?, AABBdc?)`, `getShipsIntersecting(Level?, double, double, double)` | Ship intersection queries |

The no-argument `getServerShipWorld()`/`getClientShipWorld()` overloads are
`@Experimental`; they are only valid when exactly one server/client is loaded.
Prefer the argument-taking overloads.

## Initialization order

1. Load the VS2/Core dependency.
2. Obtain the API facade (`ValkyrienSkies.api()`).
3. Register attachments, serializers, event listeners, and block states during
   mod initialisation.
4. Only after registration, access attachment- or event-dependent state.
5. Start game/physics work from the appropriate lifecycle callback.

Registration is **not** lazy. Reading an unregistered attachment is an
integration error.

## Ship representations and lifecycle

A ship can exist simultaneously as:

```text
persisted server data
  -> ServerShip          (game-thread server data, may be unloaded from disk)
  -> LoadedServerShip    (loaded server representation; an AttachmentHolder)
  -> PhysShip            (physics-thread body)
  -> ClientShip          (client representation used for rendering)
```

All representations share the same `ShipId` (a `Long`). Unloading one
representation does not imply the others are immediately gone, and a lookup can
fail at any stage. Always resolve the current representation on its owning
thread.

Typical transitions:

* **Assembly / creation** creates a `ServerShip` (and often a
  `LoadedServerShip`) on the game thread.
* **Loading** turns persisted data into a `LoadedServerShip` and fires
  `shipLoadEvent`.
* **Physics** mirrors a loaded server ship as a `PhysShip`.
* **Client sync** creates a `ClientShip` and fires `shipLoadEventClient`.

## Server ship world and queries

`ServerShipWorld` is `@GameTickOnly` and **global**: one instance covers all
dimensions. Its public collections are:

```java
QueryableShipData<ServerShip>       allShips;      // may fetch unloaded ships from disk
QueryableShipData<LoadedServerShip> loadedShips;   // currently loaded only
```

`allShips` can include unloaded persisted ships and may be slow because it can
read from disk. `loadedShips` contains loaded representations only. A collision
participant may be the ground/world body, so looking up its ID in `loadedShips`
can return `null`.

`QueryableShipData<ShipType>` is a `Collection<ShipType>` (verified signatures):

```java
ShipType getById(long shipId);                                  // null if absent
ShipType getByChunkPos(int chunkX, int chunkZ, String dimensionId); // null if absent
Iterable<ShipType> getIntersecting(AABBdc aabb, String dimensionId);
boolean contains(long shipId);
```

Deprecated on this interface:

| Deprecated | Replacement |
| --- | --- |
| `getIntersecting(AABBdc)` | `getIntersecting(AABBdc, dimensionId)` |
| `getShipDataIntersecting(AABBdc)` | `getIntersecting(AABBdc, dimensionId)` |
| `getShipDataFromChunkPos(x, z, dim)` | `getByChunkPos(x, z, dim)` |
| `idToShipData` | `getById` / iteration |

`getById` and `getByChunkPos` return `null` when there is no result. Always pass
a dimension to spatial queries: the server ship world is not dimension-filtered.

## Lifecycle operations

In this version, **creating, deleting, teleporting, renaming, and scaling ships
is not on the public `ServerShipWorld` interface**. It lives on internal types:

* `VsiServerShipWorld` adds `createNewShipAtBlock`, `deleteShip`,
  `allocateShipId`, `teleportShip`, `createPhysicsEntity`,
  `deletePhysicsEntity`, `teleportPhysicsEntity`, and chunk-watch/player
  management.
* `VsiCore` (and its `VsiCoreCommands` super-interface) adds
  `deleteShips`, `renameShip`, `scaleShip`, and `teleportShip`, plus
  `newShipTeleportData(...)`.

```java
// Internal, game thread only.
VsiServerShipWorld world =
        (VsiServerShipWorld) org.valkyrienskies.mod.api.ValkyrienSkies.getShipWorld(level);
ServerShip ship = world.createNewShipAtBlock(
    blockPos, /* createShipObjectImmediately = */ true, /* scaling = */ 1.0,
    dimensionId);
world.deleteShip(ship);
```

Because these are internal, keep the casts in a narrow, version-pinned adapter
and re-verify them on every VS2/Core update. See
[09-internal-apis-and-compatibility.md](09-internal-apis-and-compatibility.md).

## Events involved in lifecycle

* `ShipLoadEvent` — game thread; payload `LoadedServerShip`.
* `ShipLoadEventClient` / `ShipUnloadEventClient` — client; payload `ClientShip`.
* `TickEndEvent` — game thread; payload `ServerShipWorld`.
* `MergeEvent` / `SplitEvent` — game thread; connectivity graph changes.

Lifecycle events identify state transitions. They do not guarantee that a
physics-thread reference remains valid after the callback returns.

## `VsiCore` — the Core implementation root

`org.valkyrienskies.core.internal.VsiCore` is the internal root of the Core
implementation. Beyond `VsCoreApi` and `VsiCoreCommands` it exposes
(verified): `simplePacketNetworking`, `hooks`, `blockTypes`,
`entityPolygonCollider`, `solidShapeUtils`, dummy ship worlds,
`registerBlockStates(...)`, terrain-update builders, pipeline (de)serialization,
`newChunkClaim(...)`, block-state builders, `newShipTransform(...)`,
`newShipTeleportData(...)`, `newShipInertiaData(...)`, `getServerConfig()`, and
`defaultMapper`. It is obtained through the `VSCoreProvider` service:

```java
// org.valkyrienskies.mod.common.ValkyrienSkiesMod
VsiCore core = ValkyrienSkiesMod.getVsCore();
```

This is internal API; do not expose it from an addon's public surface.
