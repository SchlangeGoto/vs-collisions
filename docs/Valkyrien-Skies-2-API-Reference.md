# Valkyrien Skies 2 and VS Core API Reference

## Purpose

This document is a repository-level reference for the Valkyrien Skies 2 (VS2)
Minecraft integration and the Valkyrien Skies Core API used by this project. It
describes the API model, thread ownership, data contracts, coordinate systems,
events, attachments, ship lookup, physics access, and server ship lifecycle.

The project source is used only as an index of integration points and examples.
The behavior described here belongs to VS2 or VS Core unless a section is
explicitly labelled **Repository usage**. This document is not a description of
the collision or damage algorithm implemented by this repository.

The reference was verified against:

* Minecraft 1.20.1;
* Forge 47.4.10;
* VS2 `2.4.11`;
* VS Core `1.1.0+cf208d8b56`;
* Java 17 API signatures and Kotlin source JARs resolved by Gradle.

Dependency versions are defined in `gradle.properties`.

## Architecture

VS2 is composed of two relevant layers:

1. **VS Core** provides the platform-independent physics and ship APIs.
2. **VS2 Minecraft integration** connects those APIs to Minecraft dimensions,
   shipyards, levels, Forge lifecycle events, and server/client state.

The most important conceptual split is between two simulation domains:

```text
Game/server thread
  LoadedServerShip, ServerShipWorld, attachments, Minecraft levels

Physics thread
  PhysShip, PhysLevel, CollisionEvent, ContactPoint
```

These domains are intentionally separate. A physics callback must not directly
perform game-thread work. Values needed by game logic must cross an explicit,
thread-safe handoff.

## API package map

| Package | Responsibility |
|---|---|
| `org.valkyrienskies.core.api.events` | Physics and server lifecycle events |
| `org.valkyrienskies.core.api.physics` | Contact and collision-related physics data |
| `org.valkyrienskies.core.api.ships` | Ship interfaces and ship queries |
| `org.valkyrienskies.core.api.ships.properties` | IDs, transforms, inertia, and ship properties |
| `org.valkyrienskies.core.api.bodies.properties` | General body transforms and kinematics |
| `org.valkyrienskies.core.api.attachment` | Registered per-object addon state |
| `org.valkyrienskies.core.api.world` | Physics and server ship worlds |
| `org.valkyrienskies.core.api.util` | Thread ownership and utility contracts |
| `org.valkyrienskies.core.internal.world` | Internal server-world implementation interfaces |
| `org.valkyrienskies.mod.common` | Minecraft-facing utilities and API access |

`org.valkyrienskies.core.internal.*` is not a stable public API. Code using
internal interfaces should pin the VS Core version, isolate the dependency
behind a small adapter, and expect migration work when VS2 changes.

## Thread ownership and annotations

### `@PhysTickOnly`

`org.valkyrienskies.core.api.util.PhysTickOnly` marks a class, method, property,
field, or constructor managed by the physics thread. It carries a Kotlin
opt-in warning but does not provide runtime synchronization.

Physics-only types include `CollisionEvent`, `ContactPoint`, `PhysShip`, and
`PhysLevel`. Do not call game APIs, inspect Minecraft worlds, mutate
attachments, or retain references to physics-owned objects from a game-thread
callback.

### `@GameTickOnly`

`org.valkyrienskies.core.api.util.GameTickOnly` marks a class, method, property,
field, or constructor managed by the game thread. It is a compile-time contract,
not a runtime lock.

Game-thread-only types include `LoadedServerShip`, `ServerShipWorld`,
`VsiServerShipWorld`, game-thread ship queries, lifecycle methods, and
attachment access through loaded ships.

### Correct handoff pattern

VS2's `CollisionEvent` documentation explicitly recommends a
`ConcurrentLinkedQueue` for transferring events to the game tick:

```java
private final ConcurrentLinkedQueue<CollisionEvent> pending =
        new ConcurrentLinkedQueue<>();

collisionEvent.on(pending::add); // physics thread

// Game/server tick:
CollisionEvent event;
while ((event = pending.poll()) != null) {
    processOnGameThread(event);
}
```

The queue makes publication and polling safe. It does not make arbitrary
objects safe to mutate concurrently. If event lifetime or vector ownership is
unclear across a dependency update, copy the required scalar and vector values
into an immutable application-owned record at the handoff boundary.

## Collision events

### `CollisionEvent`

Source: `org.valkyrienskies.core.api.events.CollisionEvent`.

The interface is `@VsBeta` and `@PhysTickOnly`:

| Property | JVM type | Description |
|---|---|---|
| `dimensionId` | `String` | Dimension in which the physics event occurred |
| `physLevel` | `PhysLevel` | Physics-side level associated with the event |
| `shipIdA` | `long` | First colliding body ID |
| `shipIdB` | `long` | Second colliding body ID |
| `contactPoints` | `Collection<ContactPoint>` | Contacts reported for the event |

The interface is a data carrier. It does not expose solver impulse, mass,
material, or block information. Those values must be obtained from the
appropriate ship or Minecraft APIs on the owning thread.

The resolved implementation stores the supplied dimension, physics level, IDs,
and contact collection and returns them directly. It does not defensively copy
the collection.

### Collision event streams

The available event streams depend on the physics backend and VS2 version. A
persist event may be emitted repeatedly while contact remains active; a start
event represents the beginning of a contact where supported.

Choose the stream according to the requirement:

* start events for one-time impact initialization;
* persist events for continuous contact information;
* coalescing or rate limiting before expensive game-thread work.

Do not assume that every backend emits identical event types or frequencies.

### `ContactPoint`

Source: `org.valkyrienskies.core.api.physics.ContactPoint`.

`ContactPoint` is `@VsBeta` and `@PhysTickOnly`:

| Property | JVM type | Description |
|---|---|---|
| `position` | `Vector3dc` | Contact position in world space |
| `normal` | `Vector3dc` | Contact normal |
| `separation` | `float` | Signed separation; negative means penetration |
| `velocity` | `Vector3dc` | Relative velocity: A's point velocity minus B's point velocity |

The interface represents a sample in the collision manifold. It does not
calculate the manifold; narrow-phase contact generation is performed by the
active physics backend.

The concrete `ContactPointImpl` contains the four supplied values. In the
resolved Krunch integration, the event encoder serializes position, normal,
separation, and relative velocity for each contact.

## Ship representations

### `PhysShip`

Source: `org.valkyrienskies.core.api.ships.PhysShip`.

`PhysShip` is the physics-thread representation of a ship and is
`@PhysTickOnly`. VS Core explicitly warns that references must not be stored or
accessed from the game thread because the physics thread may remove them.

Important properties:

| Property | Description |
|---|---|
| `isStatic` | Whether the body is kinematic/static rather than dynamic |
| `buoyantFactor` | Buoyancy factor |
| `doFluidDrag` | Whether fluid drag is applied |
| `mass` | Body mass with scaling applied |
| `centerOfMass` | Center of mass in model space |
| `momentOfInertia` | Moment of inertia tensor in model space |
| `liquidOverlap` | Fraction of blocks overlapping fluid |
| `isSleeping` | Whether simulation is sleeping |

Force and torque methods include world, model, and body-space variants such as
`applyWorldForce`, `applyWorldTorque`, `applyModelForce`,
`applyModelTorque`, `applyBodyForce`, and `applyBodyTorque`. They must be
called from the physics thread.

### `LoadedServerShip`

Source: `org.valkyrienskies.core.api.ships.LoadedServerShip`.

`LoadedServerShip` is the game-thread representation of a loaded server ship.
It extends `AttachmentHolder`, `LoadedShip`, and `ServerShip`, and is
`@GameTickOnly`.

It exposes loaded-server state such as transforms, inertia data, AABB, ship ID,
and attachment access through its parent interfaces. `wingManager` and
`dragController` are also available where configured.

A loaded server ship is not interchangeable with a `PhysShip`. Use
`LoadedServerShip` for game-thread state and `PhysShip` for physics-thread
force and torque operations.

## Ship worlds and queries

### `ServerShipWorld`

Source: `org.valkyrienskies.core.api.world.ServerShipWorld`.

`ServerShipWorld` is `@GameTickOnly` and global on the server rather than one
instance per dimension. Its primary collections are:

```java
QueryableShipData<ServerShip> allShips;
QueryableShipData<LoadedServerShip> loadedShips;
```

The collections cover all dimensions. Dimension-aware query methods must be used
when querying by chunk or AABB.

### `VsiServerShipWorld`

Source: `org.valkyrienskies.core.internal.world.VsiServerShipWorld`.

This internal interface extends `ServerShipWorld` and adds:

```java
ServerShip createNewShipAtBlock(
    Vector3ic blockPosInWorldCoordinates,
    boolean createShipObjectImmediately,
    double scaling,
    String dimensionId
);

void deleteShip(ServerShip ship);
long allocateShipId(String dimensionId);
```

It also exposes all/loaded ship data, physics entity creation and deletion,
physics entity retrieval, player and chunk-watch task management, ship and
physics-entity teleportation, and the dimension-to-ground-body map.

All methods are game-thread operations. The interface is internal and should
be isolated behind a version-specific adapter.

### `QueryableShipData`

Source: `org.valkyrienskies.core.api.ships.QueryableShipData`.

Important methods:

```java
ShipType getById(long shipId);
ShipType getByChunkPos(int chunkX, int chunkZ, String dimensionId);
Iterable<ShipType> getIntersecting(AABBdc aabb, String dimensionId);
boolean contains(long shipId);
```

`getById` returns `null` when no ship is present. The AABB overload without a
dimension is deprecated because it can return ships from unrelated dimensions.
Prefer the dimension-aware overload.

## Attachments

### `AttachmentHolder`

Source: `org.valkyrienskies.core.api.attachment.AttachmentHolder`.

An attachment holder stores addon-owned state on a VS2 object. Attachment
classes must be final and must be registered before use.

The interface guarantees thread-safe attachment setting and retrieval, but the
owning VS2 object retains its own thread annotation. Thread-safe attachment
storage does not make a `LoadedServerShip` safe to access from the physics
thread.

Methods:

```java
<T> T getAttachment(Class<T> clazz);
<T> T getOrPutAttachment(Class<T> clazz, Supplier<T> supplier);
<T> T setAttachment(T value);
<T> T removeAttachment(Class<T> clazz);
Iterable<Object> getAllAttachments();
```

Behavior:

* lookup is by the exact declared class;
* inherited-class lookup is not performed;
* `getAttachment` returns `null` when absent or deserialization failed;
* `getOrPutAttachment` creates and stores a value only when absent;
* `setAttachment` and `removeAttachment` return the previous value;
* setting an attachment that failed to deserialize throws an exception;
* registration is mandatory.

### Registration

Registration is performed through `VsCoreApi`:

```java
var registration = api
    .newAttachmentRegistrationBuilder(MyAttachment.class)
    .useTransientSerializer()
    .build();
api.registerAttachment(registration);
```

Registration belongs in the mod initialization/registry phase, before any
attachment is read or written. A transient serializer is appropriate for
runtime-only state and must not be treated as a world-save format.

## Transforms and coordinate spaces

### `BodyTransform`

Source: `org.valkyrienskies.core.api.bodies.properties.BodyTransform`.

A body transform contains:

| Property | Space/meaning |
|---|---|
| `position` | Body position in world space |
| `rotation` | Normalized world rotation quaternion |
| `scaling` | Model-to-world scaling |
| `positionInModel` | Body position in model/shipyard space |
| `toWorld` | Model/shipyard to world affine matrix |
| `toModel` | World to model/shipyard inverse matrix |

The documented construction is equivalent to:

```text
toWorld = translate(position) * rotate(rotation) * scale(scaling)
          * translate(-positionInModel)
toModel = inverse(toWorld)
```

Use `transformPosition` for points. Use `transformDirection` for vectors that
must not receive translation. Directions may still be affected by scaling;
normalize afterward when a unit direction is required.

### `ShipTransform`

Source: `org.valkyrienskies.core.api.ships.properties.ShipTransform`.

`ShipTransform` is a compatibility interface over `BodyTransform`. Important
aliases are:

| Legacy alias | Preferred property |
|---|---|
| `positionInWorld` | `position` |
| `positionInShip` | `positionInModel` |
| `shipToWorld` | `toWorld` |
| `worldToShip` | `toModel` |

New code should prefer `BodyTransform` names.

### Coordinate conversion

```java
Vector3d modelPoint = new Vector3d();
ship.getTransform().getWorldToShip()
    .transformPosition(worldPoint, modelPoint);

Vector3d modelDirection = new Vector3d();
ship.getTransform().getWorldToShip()
    .transformDirection(worldDirection, modelDirection);
modelDirection.normalize();
```

Do not transform a direction with `transformPosition`; translation would make
the result dependent on the ship origin. Do not use model/shipyard coordinates
as world coordinates when querying Minecraft blocks.

## Server ship lifecycle

`VsiServerShipWorld.createNewShipAtBlock(...)` creates server ship data centered
at a block position in world coordinates. When
`createShipObjectImmediately` is true, the corresponding ship object is created
immediately. The dimension ID is explicit because the server ship world spans
all dimensions.

`deleteShip(ServerShip)` removes a server ship. Callers must define ownership
of blocks, attachments, joints, and dependent game objects before deletion.

Creation and deletion are game-thread operations. They must not be called from
collision or other `@PhysTickOnly` callbacks.

## Events and listener registration

### `ListenableEvent`

Source: `org.valkyrienskies.core.api.event.ListenableEvent`.

The generic event interface provides:

```kotlin
fun on(cb: EventConsumer<T>): RegisteredListener
fun on(cb: Consumer<in T>): RegisteredListener
fun once(cb: Consumer<in T>): RegisteredListener
fun once(predicate: Predicate<in T>, cb: Consumer<in T>): RegisteredListener
```

`on` registers a persistent listener. `once` unregisters its listener after the
predicate succeeds. Registration returns a `RegisteredListener` for explicit
unregistration.

Listener registration does not change the thread ownership of an event. The
callback executes in the thread/domain that emits the event.

## Physics ticks and force application

`PhysTickEvent` is a physics-thread event for physics-thread ship logic. Use
physics callbacks for applying forces and torques, and for reading dynamic
physics state. Use game ticks for Minecraft block access, server entities,
loaded-server ship lookup, game attachments, and ship lifecycle operations.

`PhysShip` exposes three relevant force spaces:

| Space | Origin/orientation | Typical use |
|---|---|---|
| Body | Center of mass, untransformed body basis | Raw physics force/torque |
| Model | Shipyard coordinates and ship orientation | Block-based ship machinery |
| World | Global world coordinates | External world forces |

Select the method based on both the vector space and the position space.

## Minecraft integration utilities

The Forge integration exposes `org.valkyrienskies.mod.common.VSGameUtilsKt`
and `ValkyrienSkiesMod`. The relevant operations used by this repository are:

| Utility | Purpose |
|---|---|
| `ValkyrienSkiesMod.getApi()` | Obtain the VS2/Core API facade |
| `VSGameUtilsKt.getLevelFromDimensionId(server, id)` | Resolve a Minecraft server level |
| `VSGameUtilsKt.getShipObjectWorld(level)` | Obtain the server ship world |
| `VSGameUtilsKt.isBlockInShipyard(level, pos)` | Classify a block coordinate |

`org.valkyrienskies.mod.api.ValkyrienSkies.isBlockInShipyard(...)` provides the
corresponding mod API classification. Prefer the public mod API where
available; isolate `VSGameUtilsKt` usage when a Kotlin bridge is required.

## API usage checklist

1. Identify whether every touched type is physics-thread or game-thread-only.
2. Register every attachment before first access.
3. Use a concurrent handoff for physics events consumed by game logic.
4. Copy event data when it must outlive the emitting callback or ownership is
   not guaranteed.
5. Resolve dimensions explicitly; server ship worlds span all dimensions.
6. Use `getById` for IDs and dimension-aware AABB/chunk queries.
7. Use `toWorld`/`toModel` consistently and distinguish positions from
   directions.
8. Keep `PhysShip` references on the physics thread.
9. Keep Minecraft world access and server ship lifecycle calls on the game
   thread.
10. Treat `org.valkyrienskies.core.internal` as version-specific API.

## Repository integration index

These files identify the VS2 APIs relevant to this project:

| Repository file | VS2/Core integration points |
|---|---|
| `VSCollisions.java` | API access and attachment registration |
| `collision/CollisionEventHandler.java` | Collision event listener |
| `collision/CollisionQueue.java` | Physics-to-game event handoff |
| `damage/DamageApplicator.java` | Server ship world, loaded ships, mass, attachments |
| `stress/BlockImpactInjector.java` | World-to-ship transform and shipyard classification |
| `stress/StressPropagator.java` | Shipyard classification |
| `damage/BlockDamageResolver.java` | Shipyard classification |
| `impact/ImpactZone.java` | `ContactPoint` consumption |
| `impact/ImpactZoneClusterer.java` | `ContactPoint` grouping |
| `impact/ZoneLoadCalculator.java` | Ship transform, inertia, mass, AABB |

The repository's collision, stress, material, and fatigue algorithms are
intentionally not reproduced here. Consult those source files for behavior
that is specific to this mod.

## Version and source provenance

The reference was built from:

* `org.valkyrienskies.core:api:1.1.0+cf208d8b56`;
* `org.valkyrienskies.core:internal:1.1.0+cf208d8b56`;
* `org.valkyrienskies:valkyrienskies-120-forge:2.4.11`.

Inspected source definitions include `CollisionEvent.kt`,
`ContactPoint.kt`, `LoadedServerShip.kt`, `PhysShip.kt`, `AttachmentHolder.kt`,
`BodyTransform.kt`, `ShipTransform.kt`, `ServerShipWorld.kt`,
`QueryableShipData.kt`, `ListenableEvent.kt`, `PhysTickOnly.kt`,
`GameTickOnly.kt`, and `VsiServerShipWorld.kt`.

Bytecode was inspected for `CollisionEventImpl`, `ContactPointImpl`, and
`CollisionEventEncoder`. It confirms the concrete event/contact data layout and
Krunch serialization fields. It does not expose the complete narrow-phase
manifold algorithm, which belongs to the physics backend.
