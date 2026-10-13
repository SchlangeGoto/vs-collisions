# API Inventory and Provenance

This chapter records the exact resolved surface the reference was written
against, and how it was inspected. It is grouped by API responsibility rather
than by this repository's implementation.

## Artifacts reviewed

| Coordinate | Contents |
| --- | --- |
| `org.valkyrienskies:valkyrienskies-120-forge:2.4.11` (+ `-sources.jar`) | Minecraft integration, `mod.api` and `mod.common` |
| `org.valkyrienskies.core:api:1.1.0+cf208d8b56` (+ `-sources.jar`) | Public Core API |
| `org.valkyrienskies.core:internal:1.1.0+cf208d8b56` (+ `-sources.jar`) | Internal Core interfaces |
| `org.valkyrienskies.core:util:1.1.0+cf208d8b56` (bytecode only) | Utilities, event emitter, data structures |
| `org.valkyrienskies.core:impl:1.1.0+cf208d8b56` (bytecode only) | Internal Core implementation |

These were resolved by this project's Gradle build (`gradle.properties`:
`vs2_version=2.4.11`, `vs_core_version=1.1.0+cf208d8b56`) and read from the
Gradle dependency cache.

### How the surface was verified

* **Source review** of every `.kt` file in the `api`, `internal`, and Forge
  `-sources.jar`s: the API facade, all events, event plumbing, attachments and
  serializers, bodies and properties, shapes, ships and ship properties,
  worlds/connectivity, physics block-state shapes, utilities, internal ship and
  world interfaces, joints, config, and the Minecraft facades.
* **Bytecode review** (`javap -public`) of the `api`, `internal`, `util`, and
  `impl` JARs for JVM signatures and for concrete classes where source was
  unavailable (`util`, `impl`). This confirmed, among others, that
  `PhysLevel.enableCollisionBetween` returns `boolean`, that `Ship.kinematics`
  is `BodyKinematics`, that `ValkyrienSkies.api()` is the static accessor, and
  that `EventEmitterImpl` implements `SingleEvent`.

Statements in this reference that go beyond source/bytecode (for example,
backend emission cadence) are explicitly framed as unspecified or
backend-dependent, not as facts.

### Generated Dokka reference

The exhaustive, member-level surface is published as a Dokka site generated from
the resolved VS2/VS Core sources. It is not committed to the repository
(generated output is gitignored). Build it with:

```bash
./gradlew dokkaGeneratePublicationHtml   # -> docs/vs2-api/api/index.html
```

The published copy lives at <https://schlangegoto.github.io/vs-collisions/>
(see [14-tooling-and-publishing.md](14-tooling-and-publishing.md)). Chapter
[13](13-additional-api-surface.md) describes public types that have no source
JAR in this build (`util`, `impl`, native Krunch), which Dokka therefore cannot
document.

## Public Core API packages

| Package | Principal types |
| --- | --- |
| `core.api` | `VsCoreApi`, `VsBeta`, `ValkyrienSkiesException` |
| `core.api.event` | `ListenableEvent`, `EmittableEvent`, `SingleEvent`, `EventConsumer`, `RegisteredListener` |
| `core.api.events` | `CollisionEvent`, `PhysTickEvent`, `ShipLoadEvent`, `ShipLoadEventClient`, `ShipUnloadEventClient`, `TickEndEvent`, `StartUpdateRenderTransformsEvent`, `MergeEvent`, `SplitEvent`, `JointSerializedEvent`, `JointDeserializedEvent` |
| `core.api.attachment` | `AttachmentHolder`, `AttachmentRegistration`, `AttachmentSerializer`, `JacksonAttachmentSerializer`, `LegacyAttachmentSerializer`, `TransientAttachmentSerializer`, `BinaryAttachmentSerializer` |
| `core.api.bodies` | `VsBody`, `BaseVsBody`, `ServerBaseVsBody`, `ServerVsBody`, `PhysicsVsBody` |
| `core.api.bodies.properties` | `BodyId`, `BodyTransform`, `BodyKinematics`, `BodyInertia`, `BodyPose` |
| `core.api.bodies.shape` | `BodyShape`, `Box`, `Capsule`, `Sphere`, `Voxel`, `VoxelType`, `VoxelUpdate`, `Wheel` |
| `core.api.physics` | `ContactPoint`, `RayCastResult` |
| `core.api.physics.blockstates` | `BlockShape`, `BoxBlockShape`, `BoxesBlockShape`, `SolidBlockShape`, `LiquidBlockShape`, `SolidState`, `LiquidState`, `CollisionPoint` |
| `core.api.ships` | `Ship`, `LoadedShip`, `ServerShip`, `LoadedServerShip`, `ClientShip`, `PhysShip`, `QueryableShipData`, `ServerTickListener`, `ShipPhysicsListener`, `ShipForcesInducer`, `ClientShipTransformProvider`, `ServerShipTransformProvider`, `ContraptionWingProvider`, `DragController`, `Wing`, `PositionedWing`, `WingGroupId`, `WingGroupChanges`, `WingManagerChanges`, `WingManager` |
| `core.api.ships.properties` | `ShipId`, `ShipTransform`, `ShipInertiaData`, `PhysPose`, `ChunkClaim`, `IShipActiveChunksSet` |
| `core.api.world` | `ShipWorld`, `ServerShipWorld`, `ClientShipWorld`, `PhysLevel`, `LevelYRange` |
| `core.api.world.connectivity` | `Component`, `ConnectionStatus`, `SparseVoxelPosition`, `DoubleAugmentation`, `DoubleComponentAugmentation` |
| `core.api.world.properties` | `DimensionId` |
| `core.api.util` | `GameTickOnly`, `PhysTickOnly`, `Identified`, `AerodynamicUtils`, `DoublePair` |
| `core.api.util.functions` | `IntBinaryConsumer`, `IntTernaryConsumer`, `DoubleTernaryConsumer` |
| `core.game.ships` | deprecated typealiases `ShipData`, `ShipObject`, `ShipObjectClient`, `ShipObjectServer` |
| `core.impl.hooks` | deprecated `VSEvents` |
| `com.github.imifou.jsonschema...` | Jackson JSON-schema annotations bundled into the API JAR (not VS API) |

## Public Minecraft API (`mod.api`)

| Type | Purpose |
| --- | --- |
| `VsApi` | `VsCoreApi` + Minecraft helpers |
| `ValkyrienSkies` | Java facade for Kotlin extensions and `api()` |
| `EntityPhysicsListener` | Physics-tick entity listener |
| `BlockEntityPhysicsListener` | Physics-tick block-entity listener |
| `SeatedControllingPlayer` | Deprecated ship-seat input holder |
| `events.PreRenderShipEvent` / `events.PostRenderShipEvent` | Render events |
| `events.RegisterBlockStateEvent` | Physics block-state registration |
| `config.VSConfigApi` | Forge config model/spec builder |
| `datagen.VSBlockInfoDataProvider` | Block mass datagen |
| `datagen.VSShipyardEntityDataProvider` | Shipyard entity datagen |

## Internal and implementation inventory (`internal`, `impl`)

Selected public-visible internal types (all under
`org.valkyrienskies.core.internal` unless noted):

* Root/program: `VsiCore`, `VsiCoreClient`, `VsiCoreServer`, `VsiCoreFactory`,
  `VsiCoreCommands`, `ShipTeleportData`, `VsiGameServer`.
* Ships: `VsiShip`, `VsiLoadedShip`, `VsiServerShip`, `VsiLoadedServerShip`,
  `VsiClientShip`, `VsiPhysShip`, `VsiQueryableShipData`,
  `VsiMutableQueryableShipData`.
* Worlds: `VsiShipWorld`, `VsiServerShipWorld`, `VsiClientShipWorld`,
  `VsiPhysLevel`, `VsiPipeline`, `VsiPlayer`, `VsiPlayerState`.
* Chunks/terrain: `VsiTerrainUpdate`, `VsiBlockType`, `VsiBlockTypes`,
  `VsiChunkWatchTask`, `VsiChunkUnwatchTask`, `VsiChunkWatchTasks`.
* Physics: `VsiBlockState`, `PhysicsEntityData`, `PhysicsEntityServer`,
  `VSCollisionShapeData` and variants, `RigidBodyDefaults`.
* Joints: `VSJoint`, `VSJointId`, `VSJointType`, `VSJointPose`,
  `VSJointMaxForceTorque`, `VSJointAndId`, `VSFixedJoint`, `VSDistanceJoint`,
  `VSRevoluteJoint`, `VSPrismaticJoint`, `VSSphericalJoint`, `VSGearJoint`,
  `VSRackAndPinionJoint`, `VSD6Joint`, `UnavailableInKrunch`.
* Collision: `VsiConvexPolygonc`, `VsiEntityPolygonCollider`,
  `VsiSolidShapeUtils`.
* Hooks/config: `VsiCoreHooks`, `VsiCoreHooksIn`, `VsiCoreHooksOut`,
  `VsiPlayState`, `VsiConfigModel`, `VsiConfigModelCategory`,
  `VsiConfigModelEntry`, `ConfigEntry`, `ConfigCategory`.
* Networking: `SimplePacket`, `SimplePacketNetworking`
  (`impl.networking.simple`).
* Config models (`impl`): `VSCoreConfig`, `PhysicsConfig`,
  `ConfigPhysicsBackendType`, `VsiConfigModelImpl`.

The `impl` artifact additionally contains Dagger components
(`VSCoreImpl`, `VSCoreClientImpl`, `VSCoreServerImpl`, `DaggerVSCore*Factory`),
serialization DTOs/converters, networking packets, and a relocated `shadow`
package. None of these are addon API.

## VS2 common implementation inventory (`mod.common`)

Notable public/common types used by addons and by VS2 itself:

* `ValkyrienSkiesMod` — mod entry point, API/core accessors, attachment
  registration, physics-listener registries, split handler.
* `VSGameUtilsKt` — large extension set (lookups, conversions, distance).
* `RaycastUtilsKt` — ship-aware ray tracing.
* `EntityShipCollisionUtils` — entity/ship collision helpers.
* `VSCoreProvider`, `IShipObjectWorldProvider` (+ server/client variants) —
  service interfaces.
* `util.ShipSettings`, `util.BuoyancyHandlerAttachment`,
  `util.SplittingDisablerAttachment`, `util.SplitHandler`,
  `util.DimensionIdProvider`, `util.GameToPhysicsAdapter`, `util.VSServerLevel`.
* `entity.ShipMountedToData`, `entity.ShipMountedToDataProvider`,
  `entity.ShipyardPosSavable`, `entity.ShipMountingEntity`, `entity.VSPhysicsEntity`.
* `assembly.ICopyableAttachment`, `assembly.ICopyableBlock`,
  `assembly.VSAssemblyEvents`, `assembly.VSSchematicEvents`,
  `assembly.ShipAssembler`.
* `config.VSGameConfig`, `config.ShipRenderer`, `config.ConfigType`,
  `config.DimensionParametersResolver`, `config.MassDatapackResolver`.
* `hooks.VSGameEvents` — additional (implementation) event emitters.
* `command.*` — VS2's own `/vs` commands and argument types.
* `networking.*` — VS2 packet types.
* `mixin.*`, `mixinducks.*` — mixins and duck interfaces.
* `compat.*` — compatibility modules (Create, Flywheel, Sodium, Hex Casting,
  Dynmap, Weather2, ...).

## Scope boundary

This directory documents VS2 and VS Core. It does not document:

* this repository's collision-energy formulas;
* its stress propagation;
* its material thresholds;
* its damage or destruction policy;
* its queue deduplication;
* its project-specific attachments.

Those are application behaviour and belong in the
[`../architecture`](../architecture) pages.
