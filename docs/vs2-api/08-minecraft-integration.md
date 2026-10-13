# Minecraft Integration API

## Two Minecraft-facing surfaces

The integration artifact (`org.valkyrienskies:valkyrienskies-120-forge`) exposes
two Minecraft-facing surfaces:

1. **`org.valkyrienskies.mod.api`** — the public, intended-for-addons surface:
   `VsApi`, the `ValkyrienSkies` Java facade, `EntityPhysicsListener`,
   `BlockEntityPhysicsListener`, config/datagen helpers, and render events.
2. **`org.valkyrienskies.mod.common`** — the common implementation. It contains
   `ValkyrienSkiesMod` and the large `VSGameUtilsKt` extension set. This surface
   is available and widely used, but it is not the stable public facade. Prefer
   `mod.api`; isolate `mod.common` calls behind a small adapter when a Kotlin
   bridge is needed.

## API acquisition

```java
import org.valkyrienskies.mod.api.ValkyrienSkies;
import org.valkyrienskies.mod.api.VsApi;

VsApi api = ValkyrienSkies.api();
```

`ValkyrienSkies.api()` is the public accessor. The implementation also has
`ValkyrienSkiesMod.getApi()` (`org.valkyrienskies.mod.common`), which the
`ValkyrienSkies` facade calls reflectively. Both return the same `VsApi`
instance for the installed integration.

## `VsApi` helpers

Verified members are listed in
[02-core-api-and-lifecycle.md](02-core-api-and-lifecycle.md). In summary,
`VsApi` provides ship-world lookup, shipyard classification, managing-ship
lookup, intersection queries, dimension IDs, and ship-mount queries, all taking
Minecraft objects.

```java
ServerShipWorld world = api.getServerShipWorld(server);
ClientShipWorld clientWorld = api.getClientShipWorld(minecraft);
ShipWorld levelWorld = api.getShipWorld(level);

boolean inShipyard = api.isChunkInShipyard(level, chunkX, chunkZ);
Ship ship = api.getShipManagingBlock(level, blockPos);
Iterable<Ship> hits = api.getShipsIntersecting(level, aabb);
```

## The `ValkyrienSkies` Java facade

`ValkyrienSkies.kt` is declared `@file:JvmName("ValkyrienSkies")`, so its Kotlin
extensions are static methods on the Java class `ValkyrienSkies` (verified via
bytecode). This is the most convenient Java-friendly surface. Categories:

### Level / ship-world lookup

```java
ShipWorld getShipWorld(Level level);
ServerShipWorld getShipWorld(MinecraftServer server);
ClientShipWorld getShipWorld(Minecraft client);
String getDimensionId(Level level);          // null-safe overload also exists
Ship getShipById(Level level, long id);
```

### Shipyard classification and managing-ship lookup

```java
boolean isBlockInShipyard(Level level, int x, int y, int z);
boolean isBlockInShipyard(Level level, BlockPos pos);
boolean isChunkInShipyard(Level level, int chunkX, int chunkZ);
Ship getShipManagingBlock(Level level, int x, int y, int z);
Ship getShipManagingBlock(Level level, BlockPos pos);
Ship getShipManagingBlock(Level level, double x, double y, double z);
Ship getShipManagingBlock(Level level, Vector3dc pos);
Ship getShipManagingBlock(Level level, Position pos);
Ship getShipManagingChunk(Level level, int chunkX, int chunkZ);
Ship getShipManagingChunk(Level level, ChunkPos pos);
Ship getShipManagingEntity(Entity entity);   // level().getShipManagingBlock(position())
```

### Coordinate and AABB conversion

```java
Vec3  positionToWorld(Ship ship, Vec3 pos);
Vector3d positionToWorld(Ship ship, Vector3d pos);
Vector3d positionToWorld(Ship ship, Vector3dc pos, Vector3d dest);
Vec3  positionToShip(Ship ship, Vec3 pos);            // world -> ship space
Vector3d positionToShip(Ship ship, Vector3d pos);
Vector3d positionToShip(Ship ship, Vector3dc pos, Vector3d dest);

AABBd toWorld(Level level, AABBdc aabb, AABBd dest);  // transforms only if both corners share a ship
AABB  toWorld(Level level, AABB aabb);
Vec3  positionToWorld(Level level, Vec3 pos);         // resolves the managing ship
```

There are also `ShipTransform` overloads of `positionToWorld`/`positionToShip`
that use `shipToWorld`/`worldToShip` directly (falling back to the input when the
transform is `null`).

### Ship-aware distance and intersection

```java
double distance(Level level, double x1,y1,z1, double x2,y2,z2);
double distanceSquared(Level level, double x1,y1,z1, double x2,y2,z2);
double distanceSquared(Level level, Position a, Position b);
double distanceSquared(Level level, Vec3i a, Vec3i b);
double distanceToCenterSquared(Level level, Vec3i a, double x, double y, double z);
boolean closerThan(Level level, Position a, Position b, double distance);
boolean closerToCenterThan(Level level, Vec3i a, double x, double y, double z, double distance);

Iterable<Ship> getShipsIntersecting(Level level, double x, double y, double z);
Iterable<Ship> getShipsIntersecting(Level level, AABBdc aabb);
Iterable<Vector3d> positionToNearbyShips(Level level, double x, double y, double z);
Iterable<Vector3d> positionToNearbyShipsAndWorld(Level level, double x, double y, double z);
// ...plus AABB-radius and DoubleTernaryConsumer callback variants
```

The distance functions transform points into world space using the managing
ship's transform before measuring.

### JOML ↔ Minecraft conversions

`ValkyrienSkies` also exposes static conversions used throughout addons:
`Vector3ic.toBlockPos()`, `Vector3dc.toMinecraft()`, `AABBdc.toMinecraft()`,
`AABB.toJOML()`, `Vector2ic.toChunkPos()`, `ChunkPos.toJOML()`,
`Vec3.toJOML()`, `Vec3i.toJOML()/toJOMLd()/toJOMLf()`, `Position.toJOML()`,
`Quaterniondc.toFloat()`, `Matrix4dc.toFloat()`,
`Matrix4dc.transformPosition(Position)`, and
`PoseStack.multiply(Matrix4dc)`.

## `VSGameUtilsKt` (`org.valkyrienskies.mod.common`)

`VSGameUtils.kt` is a large Kotlin extension set exposed to Java as static
methods on `VSGameUtilsKt`. It is `mod.common`, not the stable public facade, but
it is the source of several operations addons commonly need:

```java
ServerLevel getLevelFromDimensionId(MinecraftServer server, String dimensionId);
VsiServerShipWorld getShipObjectWorld(ServerLevel level);   // Level?.shipObjectWorld
boolean isBlockInShipyard(Level level, int x, int y, int z);
LoadedShip getLoadedShipManagingPos(Level level, ...);      // many overloads
Ship getShipManagingPos(Level level, ...);                  // many overloads
Vec3 toWorldCoordinates(Level level, BlockPos pos);
LevelYRange getYRange(Level level);
void executeOrSchedule(Level level, Runnable runnable);     // run now if game thread, else schedule
ShipMountedToData getShipMountedToData(Entity passenger, Float partialTicks);
LoadedShip getShipMountedTo(Entity entity);
boolean isPositionSealed(Level level, BlockPos pos);
void applyShipVelocity(Entity entity, Ship ship);
```

Notes:

* `getShipObjectManagingPos` is deprecated in favour of
  `getLoadedShipManagingPos`.
* `executeOrSchedule(level, runnable)` is a useful utility: it runs the task
  immediately when already on the game thread and otherwise schedules it, so you
  can safely route work out of physics callbacks.
* `VSGameUtilsKt` also contains `getShipObjectWorld` (property `shipObjectWorld`)
  which returns a `VsiServerShipWorld`; the public equivalent is
  `ValkyrienSkies.getShipWorld(level)`.

`RaycastUtilsKt` (`org.valkyrienskies.mod.common.world`) provides
`clipIncludeShips(Level, ClipContext[, shouldTransformHitPos, skipShip, skipWorld])`
and `raytraceEntities(...)`, which extend Minecraft ray tracing to ships. Treat
these as integration utilities, not portable Core API.

## Entity and block-entity physics listeners

`org.valkyrienskies.mod.api.EntityPhysicsListener`:

```java
void physTick(PhysLevel physLevel);   // physics thread
String dimension;                     // must be set by the implementer
```

`org.valkyrienskies.mod.api.BlockEntityPhysicsListener` (`@PhysTickOnly`):

```java
void physTick(PhysShip physShip, PhysLevel physLevel);  // physShip is null if the block is not on a ship
String dimension;
```

Both callbacks run in the **physics domain**. Their source explicitly warns:
do not access non-volatile/non-atomic Minecraft object fields from `physTick`.
Use atomic variables, `volatile`, or concurrent queues, and schedule Minecraft
mutations back to the game thread (`Level.executeOrSchedule`, `TickEndEvent`, or
a queue drained on the server tick).

Registration is done through `ValkyrienSkiesMod` (`mod.common`), which also
runs the listeners during `physTickEvent`:

```java
ValkyrienSkiesMod.addEntityPhysTicker(dimensionId, entity);
ValkyrienSkiesMod.addBlockEntityPhysTicker(dimensionId, blockPos, blockEntity);
ValkyrienSkiesMod.removeEntityPhysTicker(entity, dimensionId);
ValkyrienSkiesMod.removeBlockEntityPhysTicker(pos, dimensionId);
```

## Rendering

`PreRenderShipEvent` and `PostRenderShipEvent` are `@Experimental` marker
interfaces registered via `api.preRenderShipEvent` / `api.postRenderShipEvent`.
They belong to the client/render path and must not mutate server ships or
persistent state. `StartUpdateRenderTransformsEvent` marks the client transform
update phase.

Client ships use `renderTransform` / `renderAABB` (interpolated), which can
differ from the server transform. Never use a render transform as authoritative
physics or gameplay state.

## Config API

* `org.valkyrienskies.mod.api.config.VSConfigApi` builds Forge config specs from
  annotated config objects and applies updates.
* `org.valkyrienskies.mod.common.config.VSGameConfig` holds the client/server/
  common game settings; `org.valkyrienskies.core.impl.config.VSCoreConfig` holds
  Core settings (`SERVER.pt` physics-thread settings, `SERVER.sp` connectivity,
  `SERVER.ds` drag, `SERVER.physics` Krunch config).
* `org.valkyrienskies.core.internal.config` defines the annotations
  (`ConfigEntry`, `ConfigCategory`) and the model interfaces
  (`VsiConfigModel`, `VsiConfigModelCategory`, `VsiConfigModelEntry`).

`ConfigEntry` fields accept `description`, `min`, and `max` (doubles, default
`Double.NaN`). `VSGameConfig` includes client rendering/tooltip/sound/connectivity
options and server gameplay options (pocket buoyancy, block defaults, splitting,
teleport/movement checks, explosion force, AI on ships, mini-ship size and
scaling, command permission levels, and compat toggles for FTB Chunks,
ComputerCraft, Weather2, Dynmap, and Create Big Cannons).

Read the installed platform's config API rather than depending on internal
implementation models.

## Datagen

`org.valkyrienskies.mod.api.datagen` contains:

* `VSBlockInfoDataProvider` — generates a `vs_mass` datapack JSON describing
  block/tag mass, friction, elasticity, and priority.
* `VSShipyardEntityDataProvider` — generates `vs_entities` JSON marking entity
  types as handled in the shipyard.

Extend the abstract provider and register it with your platform's data
generator. Datagen providers run in the data-generation environment, not in the
game/physics tick; do not access live ship worlds or attachments from them.

## Assembly, schematics, and copying

`org.valkyrienskies.mod.common.assembly` provides experimental copy/paste hooks
for ships and blocks, plus events:

* `ICopyableAttachment` — ship attachments that participate in schematic
  copy/paste (`onCopy`, `onAfterCopy`, `onPaste`).
* `ICopyableBlock` — blocks (not block entities) that participate in copy/paste
  (`onCopy`, `onPaste`, optional `pasteSurvivalCost`).
* `VSAssemblyEvents` and `VSSchematicEvents` — `EventEmitterImpl` streams for
  `BeforeCopy`, `OnPasteBeforeBlocksAreLoaded`, `OnPasteAfterBlocksAreLoaded`,
  and paste survival cost.

These are `@Experimental` / `TODO`-marked and should be treated as
version-sensitive.

## Client/server separation

* Do not load client classes from common/server code.
* Do not treat a `ClientShip` as a server ship.
* Do not use render transforms as authoritative physics transforms.
* Resolve a server level and server ship world for gameplay mutations.
* `EntityShipCollisionUtils` and the Forge mixins are implementation helpers,
  not portable Core contracts; their signatures and behaviour are
  version-specific.
