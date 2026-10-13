# Valkyrien Skies 2 / VS Core API reference

This repository reference covers the VS2 and VS Core API available to this
project, rather than this mod's damage implementation. Names, signatures,
annotations, and deprecations were verified against the resolved source JARs
and bytecode identified in [Version and provenance](#version-and-provenance).

## Version scope and API boundary

| Component | Resolved version |
| --- | --- |
| Minecraft / Forge | 1.20.1 / 47.4.10 |
| Valkyrien Skies Forge | 2.4.11 |
| VS Core API, internal, util, impl | 1.1.0+cf208d8b56 |
| Java target | 17 |

These versions come from gradle.properties. This reference applies to those
artifacts. Core describes org.valkyrienskies.core.api as its
compatibility-oriented public API. VsBeta, experimental, and deprecated
symbols require extra migration care.

| Package family | Status | Guidance |
| --- | --- | --- |
| org.valkyrienskies.core.api | Public Core API | Preferred for portable addons. |
| org.valkyrienskies.mod.api | Public Minecraft API | Preferred for Minecraft integration. |
| org.valkyrienskies.mod.common | VS2 common implementation | Available here; isolate uses. |
| core.internal and impl | Internal | Pin versions and do not expose to addon callers. |

## Architecture, threads, and ticks

~~~text
Game/server tick                               Physics tick
Minecraft Level, entities, blocks              PhysLevel and PhysShip
ServerShipWorld, LoadedServerShip              contacts, ray casts, forces
attachments and lifecycle                       collision/physics events
                    \                          /
                     \ concurrent handoff      /
~~~

GameTickOnly and PhysTickOnly state ownership; they do not lock or schedule
work. CollisionEvent, ContactPoint, PhysLevel, and PhysShip are physics-tick
data. LoadedServerShip, ServerShipWorld, Minecraft blocks/entities, and ship
lifecycle operations are game-tick data. Client ships and render events belong
to the client/render path.

PhysTickEvent occurs at the start of a physics tick after game-frame queue
updates. Its documented delta is milliseconds since the preceding physics tick,
normally 1/60 unless synchronized physics is enabled; do not assume a fixed
tick rate.

The Core documentation recommends ConcurrentLinkedQueue for physics-to-game
handoff. Copy values that must outlive the physics callback:

~~~java
record ContactSnapshot(String dimension, long shipA, long shipB,
        Vector3d position, Vector3d normal, float separation, Vector3d velocity) {}

private final ConcurrentLinkedQueue<ContactSnapshot> pending =
        new ConcurrentLinkedQueue<>();

private void capture(CollisionEvent event) { // physics thread
    for (ContactPoint p : event.getContactPoints()) {
        pending.add(new ContactSnapshot(event.getDimensionId(), event.getShipIdA(),
                event.getShipIdB(), new Vector3d(p.getPosition()),
                new Vector3d(p.getNormal()), p.getSeparation(),
                new Vector3d(p.getVelocity())));
    }
}
// Drain pending on a server tick, then resolve Level and LoadedServerShip there.
~~~

The queue safely publishes snapshots. It does not make PhysShip, PhysLevel,
Minecraft objects, or mutable backend-owned vectors safe to use on another
thread.

## Package map

| Package | Contents |
| --- | --- |
| core.api.event, events | listeners; collision, physics, lifecycle, connectivity, joint events |
| core.api.bodies, bodies.properties | generic bodies, IDs, inertia, transforms, kinematics |
| core.api.ships, ships.properties | ships, queries, claims, wings, drag |
| core.api.world, world.connectivity | ship/physics worlds and component queries |
| core.api.physics | contacts, ray casts, block-state collision shapes |
| core.api.attachment | addon-owned state and serializers |
| core.api.util | ownership annotations and aerodynamics |
| mod.api | VsApi, ValkyrienSkies, entity/block-entity physics listeners |

Core also defines public shape descriptors including BodyShape, Box, Capsule,
Sphere, Voxel, Wheel, SolidState, and LiquidState. This project does not use
them directly.

### Public Core inventory

The resolved API source contains the following groups. This is an index, not a
claim that each type is suitable for an addon use case.

| Group | Types |
| --- | --- |
| Annotations and base | VsBeta, GameTickOnly, PhysTickOnly, ValkyrienSkiesException, Identified |
| Events | CollisionEvent, PhysTickEvent, ShipLoadEvent, ShipLoadEventClient, ShipUnloadEventClient, TickEndEvent, StartUpdateRenderTransformsEvent, MergeEvent, SplitEvent, JointSerializedEvent, JointDeserializedEvent |
| Event plumbing | ListenableEvent, EmittableEvent, SingleEvent, EventConsumer, RegisteredListener |
| Bodies | VsBody, BaseVsBody, ServerBaseVsBody, ServerVsBody, PhysicsVsBody, BodyId, BodyInertia, BodyKinematics, BodyPose, BodyTransform |
| Ships | Ship, LoadedShip, ServerShip, LoadedServerShip, ClientShip, PhysShip, QueryableShipData, ServerTickListener, ShipPhysicsListener, ShipForcesInducer, DragController, Wing, WingManager |
| Ship properties | ShipId, ShipTransform, ShipInertiaData, PhysPose, ChunkClaim, IShipActiveChunksSet |
| Worlds | ShipWorld, ServerShipWorld, ClientShipWorld, PhysLevel, LevelYRange, DimensionId |
| Physics | ContactPoint, RayCastResult, BlockShape, BoxBlockShape, BoxesBlockShape, SolidBlockShape, LiquidBlockShape, SolidState, LiquidState, CollisionPoint |
| Attachments | AttachmentHolder, AttachmentRegistration, AttachmentSerializer, BinaryAttachmentSerializer |
| Connectivity/utilities | Component, ConnectionStatus, SparseVoxelPosition, DoubleAugmentation, DoubleComponentAugmentation, AerodynamicUtils, DoublePair and primitive consumer interfaces |

Kotlin extension functions in AttachmentHolderKt add reified attachment get,
get-or-put, and remove helpers. VSGameUtilsKt is the large Minecraft extension
set exposed to Java as static methods through ValkyrienSkies; its relevant
lookups and conversions are described below.

## Entry points and events

Use the public Java facade in new code:

~~~java
VsApi api = ValkyrienSkies.api();
~~~

VsApi extends VsCoreApi. The repository currently calls
ValkyrienSkiesMod.getApi() from mod.common; it returns the same facade in these
artifacts, but ValkyrienSkies.api() is the public surface.

ListenableEvent<T> has on, once, and predicate once; each returns a
RegisteredListener with unregister(). Registration does not change the
emitter's thread.

| API property | Payload | Ownership |
| --- | --- | --- |
| shipLoadEvent | LoadedServerShip | Game tick |
| shipLoadEventClient, shipUnloadEventClient | ClientShip | Client |
| startUpdateRenderTransformsEvent | ClientShipWorld | Client/render |
| tickEndEvent | ServerShipWorld | Game tick |
| collisionStartEvent, collisionEndEvent, collisionPersistEvent | CollisionEvent | Physics tick |
| physTickEvent | PhysTickEvent | Physics tick |
| mergeEvent, splitEvent | connectivity roots | Game tick |
| jointSerializedEvent, jointDeserializedEvent | joint bookkeeping | Physics tick |

All three collision streams are declared by the API in this version. Their
emission cadence and contact-manifold policy are not specified as identical
across physics backends. Tolerate repeated persistent events and empty contact
collections; do not depend on implementation classes.

MergeEvent and SplitEvent report graph roots, voxelType (currently documented
as 1 solid and 0 air), and a nullable dimension ID. They are connectivity
notifications, not ship collision notifications. Joint events report optional
endpoint IDs, a type string, and serialization operations or deserialization
outcomes.

## Collisions and physics worlds

CollisionEvent is VsBeta and PhysTickOnly. It contains a string dimensionId,
its physLevel, pair IDs shipIdA and shipIdB, and contactPoints. A ContactPoint
supplies world-space position, normal, signed separation (negative means
penetration), and relative velocity (A's point velocity minus B's). The public
type has no solver impulse, material, block identity, or documented normal
orientation convention.

PhysLevel is dimension-specific and provides:

* getShipById(long) for a live PhysShip;
* rayCast(start, normalizedDirection, length, ignoredIds...);
* enableCollisionBetween and disableCollisionBetween; and
* aerodynamic utilities and physics ticking.

RayCastResult contains the hit PhysShip, distance, normal, velocity, and
angular velocity. It remains physics-thread data because it retains a PhysShip.

## Ships, worlds, and queries

Ship supplies an ID, slug, current and previous kinematics/transform, chunk
claim and claim dimension, world AABB, model-space ship AABB, active chunks,
linear velocity, and angular velocity.

| Type | Meaning |
| --- | --- |
| ServerShip | Game-thread server data: slug, static state, inertia, transform provider. |
| LoadedServerShip | Loaded server ship plus AttachmentHolder, WingManager, DragController. |
| ClientShip | Client representation with render transform/AABB/provider. |
| PhysShip | Physics-thread body for live state and force application. |
| ServerShipWorld | Game-thread global server ship world, across dimensions. |
| ClientShipWorld | Client ship world. |
| PhysLevel | Physics world for one dimension. |

allShips can include unloaded persisted ships and may fetch from disk.
loadedShips contains loaded representations. A collision participant can be a
ground/world body, so looking up its ID in loadedShips may return null.

QueryableShipData<T> is a collection with getById,
getByChunkPos(x, z, dimension), and getIntersecting(aabb, dimension). The first
two may return null. The dimensionless AABB query, idToShipData, and
getShipData aliases are deprecated; use dimension-aware methods.

ShipWorld adds shipyard tests and experimental air/solid connectivity queries.
They can return CONNECTED, DISCONNECTED, or UNKNOWN when chunks are unavailable.
Server sum/component augmentations are experimental and explicitly not
serialized.

## Transforms and coordinate spaces

BodyTransform has position, rotation, scaling, positionInModel, toWorld, and
toModel. BodyKinematics adds linear and angular velocity. ShipTransform extends
BodyTransform and retains compatibility aliases:

| Current name | Ship alias |
| --- | --- |
| position | positionInWorld |
| positionInModel | positionInShip |
| toWorld | shipToWorld |
| toModel | worldToShip |

Model space is shipyard/block-grid space. World space is ordinary dimension
coordinates. Body space is an offset basis centered on the body center of mass.
worldAABB is world-space; shipAABB is model-space.

~~~java
Vector3d modelPoint = ship.getTransform().getToModel()
        .transformPosition(worldPoint, new Vector3d());
Vector3d worldDirection = ship.getTransform()
        .transformDirectionNoScalingFromShipToWorld(modelDirection, new Vector3d())
        .normalize();
~~~

Use transformPosition for points and transformDirection for vectors. Directions
avoid translation but may include scaling; normalize a direction when length
matters. Do not submit shipyard BlockPos values as world positions without
conversion.

## Forces, torque, and listeners

PhysShip is physics-thread only. It exposes static state, mass, model-space
center of mass/inertia, fluid settings, liquid overlap, sleeping state, and the
following current force API:

| Method | Vector space | Position space |
| --- | --- | --- |
| applyWorldForce(force, position) | World | World |
| applyWorldTorque(torque) | World | center of mass |
| applyModelForce(force, position) | Model | Model |
| applyModelTorque(torque) | Model | center of mass |
| applyWorldForceToModelPos(force, position) | World | Model |
| applyBodyForce(force, offset) | Body | body offset |
| applyBodyTorque(torque) | Body | center of mass |
| applyWorldForceToBodyPos(force, offset) | World | body offset |

The older applyRotDependent and applyInvariant methods are deprecated. Choose
both force-vector and application-position spaces explicitly.

~~~java
public final class Thruster implements ShipPhysicsListener {
    @Override public void physTick(PhysShip ship, PhysLevel level, double delta) {
        ship.applyModelForce(new Vector3d(0, 0, 4000), new Vector3d(12.5, 64.5, -3.5));
    }
    @Override public void physTick(PhysShip ship, PhysLevel level) {}
}
~~~

ShipPhysicsListener is the attachment callback for physics-tick ship work. Its
two-argument method is required; its three-argument method defaults to it.
ShipForcesInducer is deprecated at Kotlin error level. Use
ServerTickListener.onServerTick() for server-tick attachment work.

DragController manages drag/lift/rotational drag and dimension-keyed wind.
WingManager creates/deletes groups, assigns wings by model coordinate, sets
group transforms, and exposes pending changes. Both are accessed from a loaded
server ship, so follow game-thread ownership.

## Attachments and serializers

Attachments are addon-owned state on an AttachmentHolder, including
LoadedServerShip. Attachment classes must be final and registered during
initialization before use.

~~~java
var registration = api.newAttachmentRegistrationBuilder(MyAttachment.class)
        .key("example.mod.MyAttachment:2")
        .useJacksonSerializer()
        .build();
api.registerAttachment(registration);
MyAttachment state = ship.getOrPutAttachment(MyAttachment.class, MyAttachment::new);
~~~

getAttachment is exact-class lookup and returns null when absent or
deserialization failed. setAttachment and removeAttachment return the previous
value; setting after a failed deserialization throws. Access is thread-safe,
while the owner's thread contract still applies.

| Builder choice | Meaning |
| --- | --- |
| useJacksonSerializer() | Current persisted serializer; public getters/creators by default. |
| useLegacySerializer() | Compatibility serializer; source directs new projects to Jackson. |
| useTransientSerializer() | No persistence; runtime state disappears on reload. |

Keys default to fully qualified class names. Updaters migrate a deserialized
attachment to a new class; registerAttachmentForRemoval(key) deletes a key when
next loaded. Attachments are not replacements for cross-thread state models or
Minecraft persistent block/entity data.

## Minecraft utilities and lifecycle

ValkyrienSkies is the public Java facade for Kotlin extensions. It offers ship
worlds for server/client/level, shipyard and managing-ship lookup,
position/AABB conversion, ship-aware distance, and intersection queries.

~~~java
Ship ship = ValkyrienSkies.getShipManagingBlock(level, blockPos);
if (ship != null) {
    Vec3 worldPos = ValkyrienSkies.positionToWorld(ship, localPos);
}
~~~

VSGameUtilsKt provides repository-used helpers such as getLevelFromDimensionId,
getShipObjectWorld, and loaded-ship lookup; it is mod.common, not the stable
public facade. So is RaycastUtilsKt.clipIncludeShips.

BlockEntityPhysicsListener and EntityPhysicsListener receive physics ticks.
Their source explicitly prohibits reading non-volatile/non-atomic Minecraft
object fields there; use atomic state or concurrent queues.

General lifecycle operations are internal in this version: VsiServerShipWorld
adds create/delete/allocate/teleport and physics-entity operations; VsiPhysLevel
adds joint maps and add/update/remove. Keep casts to these types in a narrow
version-pinned adapter. Game lifecycle belongs on the game tick; physics worlds
and joints belong on the physics tick. Internal joint forms and backend support,
including Krunch restrictions, are not a portable API contract.

## Common pitfalls

* Treat collision contacts as physics samples, not block or damage metadata.
* Include a dimension in queries: the server ship world covers all dimensions.
* Check loaded-ship lookups for null.
* Do not store PhysShip or PhysLevel after a physics callback.
* Keep Minecraft APIs and loaded server ships off the physics thread.
* Transform points and directions differently; normalize scaled directions.
* Do not persist required state in transient attachments or connectivity augmentations.
* Do not infer identical event cadence, contact output, or joint capability across backends.

## Repository integration index

| File | Relevant VS2/Core integration |
| --- | --- |
| src/main/java/dev/kkazi/vscollisions/VSCollisions.java | API acquisition and attachment registration |
| collision/CollisionEventHandler.java | persistent collision listener |
| collision/CollisionQueue.java | physics-to-game queue |
| damage/DamageApplicator.java | dimension, ship-world, loaded-ship lookup |
| impact/ImpactZone*.java | contact consumption |
| impact/ZoneLoadCalculator.java | transform, inertia, mass, model AABB |
| stress/BlockImpactInjector.java | world-to-model conversion and shipyard checks |

## Version and provenance

Reviewed artifacts:

* org.valkyrienskies.core:api:1.1.0+cf208d8b56 and sources;
* org.valkyrienskies.core:internal:1.1.0+cf208d8b56 and sources;
* org.valkyrienskies.core:util:1.1.0+cf208d8b56 and
  org.valkyrienskies.core:impl:1.1.0+cf208d8b56;
* org.valkyrienskies:valkyrienskies-120-forge:2.4.11 and sources.

Source review covered VsCoreApi.kt, events, PhysShip.kt, PhysLevel.kt,
ship/world/query types, attachments, transforms, VSGameUtils.kt,
ValkyrienSkies.kt, and VsiServerShipWorld.kt. javap verification covered public
and internal method signatures, including events, force methods, serializers,
utility facades, and lifecycle operations.
