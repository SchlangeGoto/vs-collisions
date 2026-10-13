# Additional API Surface

The earlier chapters cover the addon-facing Core and Mod API in depth. This
chapter catalogues the remaining public surface that the completeness audit
found, so nothing is invisible. For exact member signatures, build the generated
Dokka site (`./gradlew dokkaGeneratePublicationHtml`); the types here are the
ones Dokka cannot see because they ship without a source JAR.

> Stability reminder: only `org.valkyrienskies.core.api.*` and
> `org.valkyrienskies.mod.api.*` are compatibility-oriented. Everything else in
> this chapter is **internal**, **implementation**, **compat**, or **mixin**
> code and can change between builds.

## `org.valkyrienskies.core.util` (utilities)

A bytecode-only module (no source JAR was published for this build). It contains
genuinely useful, stable-in-practice helpers. Treat it as internal but usable.

### Block-position data structures

| Type | Summary |
| --- | --- |
| `IBlockPosSet` | `Set<Vector3ic>` with primitive `add/remove/contains/canStore(int,int,int)` and `forEach(IntTernaryConsumer)` |
| `DenseBlockPosSet` | Dense implementation keyed by chunk; `forEachChunk(...)`, `getChunk(x,y,z)` |
| `SingleChunkDenseBlockPosSet` | Per-chunk bitset (`getData()`, `forEach(...)`) |
| `SparseVoxelOpenHashMap<T>` | `HashMap<SparseVoxelPosition, T>` with extent-aware `get/put/remove/containsKey`, `findContainingVoxel`, `getBorderingSides`, `getBorderingDirection` |
| `BlockPos2ObjectOpenHashMap<T>` | Primitive-key open hash map; `getOrPut(x,y,z, supplier)`, `forEach(...)` |
| `BlockPos2IntOpenHashMap`, `BlockPos2ByteOpenHashMap` | Primitive-value variants |
| `MurmurHash3` | Hashing helper |

### AABB, vector, and misc helpers

`AABBdUtilKt` (`set`, `expand`, `getSize`, `extend`, `signedDistanceTo`,
`toAABBd`, `intersectsAABB`), `VectorConversionsKt` (`component1..3`, `getX/Y/Z`,
`multiplyTerms`, `addTerms`, `horizontalLength(Sq)`, `differenceHorLengthSq`,
`toAABBi`, `expand`), `VSCoreUtilKt` (`squared`, `variance`, `toImmutableSet`,
`mapToArray`, `filterNotNullToArray`, `filterToArray`, `asLong`, `asInts`,
`tryAndPrint`, and **Netty codecs** `readVec3fAsDouble`, `writeVec3AsFloat`,
`readQuatfAsDouble`, `writeQuatfAsFloat`, `writeNormQuatdAs3F`,
`read3FAsNormQuatd`, `writeQuatd`, `readQuatd`, `writeVec3d`, `readVec3d`,
`writeAABBi`, `readAABBi`), `UtilKt` (`iterateBits`, `pollUntilEmpty`,
`wrapIndex`, `unwrapIndex`), and `RateLimiter` (`maybeRun(Runnable)`).

## `org.valkyrienskies.core.impl` (implementation)

### `impl.api` internal ship interfaces

These extend the `internal` ship interfaces with mutation hooks used by the
Minecraft integration. Useful when you must drive ships directly; internal.

| Type | Adds |
| --- | --- |
| `ShipInternal : VsiShip` | `chunkClaimDimension` setter; `onSetBlock(...)` block-change hook (exact parameter meaning is not documented in the resolved artifact) |
| `LoadedShipInternal` | `ShipInternal + VsiLoadedShip` |
| `ServerShipInternal : ShipInternal, VsiServerShip` | `asShipDataCommon()`, `updateShipAABBGenerator(...)`, `onLoadChunk/onUnloadChunk`, `areVoxelsFullyLoaded()` |
| `LoadedServerShipInternal` | `LoadedShipInternal + ServerShipInternal + VsiLoadedServerShip`; `shipTeleportId`, `teleportShip(ShipTeleportData)` |
| `ClientShipInternal` | `LoadedShipInternal + VsiClientShip` |
| `ServerShipProvider` / `ServerShipUser` | `getShip()` / `setShip(...)` |

### Event and physics implementations

`impl.api_impl.physics` contains concrete `CollisionEventImpl`,
`ContactPointImpl`, `PhysTickEventImpl`, `JointSerializedEventImpl`,
`JointDeserializedEventImpl`; `impl.api_impl.connectivity` contains
`MergeEventImpl`, `SplitEventImpl`; `impl.api_impl.physics.blockstates` contains
`BoxBlockShapeImpl`, `BoxesBlockShapeImpl`, `SolidStateImpl`, `LiquidStateImpl`.
These are data holders; depend on the API interfaces, not these classes.

### Bodies, config, program, networking

* `impl.bodies.properties`: `BodyTransformImpl`, `BodyTransformBuilderImpl`,
  `BodyTransformFactory`, `BodyKinematicsImpl`, `BodyKinematicsBuilderImpl`,
  `BodyKinematicsFactory`.
* `impl.api_impl.config`: `ConfigPhysicsBackendType`
  (`KRUNCH_CLASSIC`, `KRUNCH_PHYSX`, `KRUNCH_JOLT`), `VsiConfigModelImpl`,
  `VsiConfigModelCategoryImpl`, `VsiConfigModelEntryImpl`.
* `impl.config`: `VSCoreConfig`, `PhysicsConfig`.
* `impl.program`: Dagger components (`VSCoreImpl`, `VSCoreClientImpl`,
  `VSCoreServerImpl`, `VsiCoreFactoryImpl`, `DaggerVSCore*Factory`).
* `impl.game.ships` / `impl.game.phys_entities` / `impl.chunk_tracking`:
  `ShipData`, `ShipDataCommon`, `ShipObjectClientWorld`, `ShipTransformImpl`,
  `ShipInertiaDataImpl`, `PhysShipImpl`, `PhysPoseImpl`, `ChunkClaimImpl`,
  `ChunkAllocator`, `ShipActiveChunksSet`, `PhysicsEntityClient`.
* `impl.game.ships.serialization.*`: the persisted-format DTOs and updaters
  (`ServerShipDataV0..V4`, `VSPipelineDataV1..V5`, `ShipTransformDataV0`,
  `ShipInertiaDataV0`, converters and modules). These define the on-disk save
  format; do not depend on them.
* `impl.networking.impl`: `PacketShipDataCreate`, `PacketShipRemove`,
  `PacketPhysEntityCreate/Remove`, `PacketCommonConfigUpdate`,
  `PacketServerConfigUpdate`, `PacketRequestUdp`, `PacketUdpState`, and
  `PhysEntityCreateData`.
* `impl.util` / `impl.util.serialization`: serialization helpers.

## Native Krunch backend (`org.valkyrienskies.physics_api_krunch`)

The `impl` artifact ships the low-level **native Krunch** bindings:
`KrunchNativePhysicsWorldReference`, `KrunchNativeRigidBodyReference`,
`KrunchNativeCompoundShapeReference`, `KrunchNativePhysicsWorldIsland`,
`KrunchNative{Box,Capsule,Sphere,Voxel,Wheel}ShapeReference`,
`KrunchCollisionShapeFactory`, `KrunchNativePhysicsThreadPoolReference`,
`KrunchByteBuffer`, `KrunchDebug`, and the voxel types
(`KrunchVoxelChunk16`, `KrunchUnsafeVoxelChunk16Reference`,
`KrunchLod1BlockRegistry`).

These classes implement relocated (obfuscated `core.impl.shadow.*`) interfaces
and are JNI bindings. They are **not addon API** and their members are
unstable/obfuscated. Do not reference them from an addon; use `PhysShip` /
`PhysLevel` instead.

## VS2 Minecraft integration — additional areas

Beyond the `mod.api` facade documented in
[08-minecraft-integration.md](08-minecraft-integration.md), the integration
contains:

### Entity handling

* `VSEntityHandler` — the entity behaviour contract:
  `freshEntityInShipyard`, `entityRemovedFromShipyard`, `applyRenderTransform`,
  `positionSetFromVehicle`, `getTeleportPos`, `applyRenderOnMountedEntity`.
* `VSEntityManager` — registers handlers by `ResourceLocation`/`EntityType`,
  `pair(...)`, `getHandler(entity)`, `syncHandlers(player)`,
  `isShipyardEntity(entity)`.
* `WorldEntityHandler` (default for world entities), `AbstractShipyardEntityHandler`
  and `DefaultShipyardEntityHandler` (shipyard entities/contraptions).
* `mod.api.datagen.VSShipyardEntityDataProvider` generates the data that pairs
  entity types with the shipyard handler.

This is a public-ish extension point for mods that need custom entity behaviour
on ships. It is not on the `VsApi` facade.

### Assembly

* `ShipAssembler` (`@JvmStatic`): `findMinAndMax(blocks)`,
  `assembleToShipFull(level, blocks, scale)`, `moveBlocksFromTo(...)`,
  `assembleToShip(level, blocks, scale)`,
  `assembleToShip(level, blocks, removeOriginal, scale, shouldDisableSplitting)`.
* `VSAssemblyEvents` / `VSSchematicEvents` (see
  [11-advanced-integration-facilities.md](11-advanced-integration-facilities.md)).

### Block state / mass data

* `BlockStateInfoProvider` and `DefaultBlockStateInfoProvider` — map Minecraft
  `BlockState`s to physics block-state info; `BlockStateInfo` caches them and is
  consumed by `LevelChunkSection.toDenseVoxelUpdate(...)`.
* `mod.api.datagen.VSBlockInfoDataProvider` generates the `vs_mass` datapack.

### Config

* `VSGameConfig` (`CLIENT`, `SERVER`, `COMMON`) — the game config model.
* `ConfigType` (client/common/server), `ShipRenderer` (default renderer
  selection), `DimensionParametersResolver` (per-dimension gravity/atmosphere),
  `MassDatapackResolver` (block mass from datapacks), `VSConfigUpdater`,
  `VSEntityHandlerDataLoader`.
* `mod.api.config.VSConfigApi` — builds Forge config specs from annotated models.

### Networking

`VSGamePackets` plus `PacketChangeKnownShips`, `PacketEntityShipMotion`,
`PacketMobShipRotation`, `PacketPlayerDriving`, `PacketPlayerShipMotion`,
`PacketStopChunkUpdates`, `PacketSyncVSEntityTypes`, `PacketRestartChunkUpdates`.
These are VS2's own protocol, not an addon API. Use your platform's networking.

### Commands

`VSCommands` and the `mod.common.command` package implement the `/vs` command
tree and argument types (`ShipArgument`, `ShipSelector`, `RelativeVector3Argument`,
...). Internal; register your own commands through the platform.

### Hooks and events

`VSGameEvents` is an implementation `EventEmitterImpl` stream for render and
lifecycle hooks (`registriesCompleted`, `tagsAreLoaded`, `configUpdated`,
`renderShip`, `postRenderShip`, `shipSplit`, Sodium render variants). Prefer the
public `VsApi` events where they exist.

### Client rendering

`mod.client` contains `IVSCamera`, `VSPhysicsEntityRenderer`,
`VSPhysicsEntityModel`, `EmptyRenderer`, and audio helpers
(`SimpleSoundInstanceOnShip`, `VelocityTickableSoundInstance`). Client-only.

### Compat modules

`mod.compat.*` and `mod.forge.compat.*` provide integrations for Create,
Create Big Cannons, Flywheel, Sodium, Hex Casting (Hexal, HexTweaks), Dynmap,
BlueMap, TIS-3D, Immersive Portals, Weather2, Mekanism, EMF/ETF, and others.
These are version- and mod-specific implementation code, not portable API.

### Mixins and duck interfaces

`mod.mixin.*` (and the Forge-specific `mod.forge.mixin.*`) plus
`mod.mixinducks.*` are the implementation's mixins and duck interfaces. They are
not API; depending on them is unsupported and fragile.

## Bundled third-party

The `api` artifact bundles the JSON-schema annotations
`com.github.imifou.jsonschema.module.addon` (`JSData`, `JsonSchema`,
`TypeFormat`). They are used internally for schema generation and are not VS
API.

## Where to look next

* Exact public members for every type that has a source JAR: the generated
  Dokka site (`./gradlew dokkaGeneratePublicationHtml` →
  `docs/vs2-api/api/index.html`).
* Types that have no source JAR (`util`, `impl`, native Krunch) are described
  here in prose because Dokka cannot see their sources.
