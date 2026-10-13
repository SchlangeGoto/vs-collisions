# Architecture and Thread Ownership

## System layers

VS2 is built from two layers that are resolved as separate artifacts:

1. **VS Core** (`org.valkyrienskies.core.*`) is a Minecraft-independent physics
   and ship backend. It owns bodies, ships, physics worlds, transforms, events,
   attachments, connectivity, and collision data.
2. **The Minecraft integration** (`org.valkyrienskies.mod.*`, artifact
   `valkyrienskies-120-forge`) wires VS Core into Minecraft: levels, shipyards,
   chunk claims, entities, block entities, the client/server lifecycle, and
   rendering.

The Core API is what an addon should build against when it does not need
Minecraft classes. The Mod API (`org.valkyrienskies.mod.api`) extends the Core
API with Minecraft helpers.

## The three runtime domains

| Domain | Representative types | Owning thread |
| --- | --- | --- |
| Game/server | `LoadedServerShip`, `ServerShipWorld`, `ServerShip`, Minecraft `Level`/`Entity`/`BlockEntity`, attachments | Game (server) thread |
| Physics | `PhysShip`, `PhysLevel`, `CollisionEvent`, `ContactPoint`, joints, ray casts | Physics thread |
| Client/render | `ClientShip`, `ClientShipWorld`, `renderTransform`, render events | Client thread |

These are **ownership contracts, not locks**. An annotation does not queue work,
make an object immutable, or make a reference safe to retain. In particular:

* A `PhysShip` may be removed by the physics thread at any time. Do not store it
  in a field that the game thread reads.
* The same ship ID can be represented by several distinct objects (a server
  ship, a physics ship, and a client ship). They are related by ID, not by
  identity, and are not interchangeable.
* Backend-owned vectors (`Vector3dc`, `Matrix3dc`, ...) are frequently live views
  into physics state. Copy any value that must outlive the callback.

## Thread annotations

VS Core marks ownership with two Kotlin opt-in annotations. Both have
`AnnotationRetention.BINARY` and target classes, functions, properties,
constructors, and fields. They are compile-time warnings only — they do not
enforce anything at runtime.

### `@PhysTickOnly`

`org.valkyrienskies.core.api.util.PhysTickOnly`:

> "This is a Physics Thread only class or method. Ensure you are not accessing
> it, directly or by reference, from the Game Thread!"

Do not, from the game thread:

* retain or dereference a `PhysShip` or `PhysLevel`;
* call Minecraft world APIs from a physics callback;
* mutate server attachments from physics code;
* assume a `Vector3dc` obtained during a physics callback stays stable after the
  callback returns.

### `@GameTickOnly`

`org.valkyrienskies.core.api.util.GameTickOnly`:

> "This is a Game Thread only class or method. Ensure you are not accessing it,
> directly or by reference, from the Physics Thread!"

Game-thread-only work includes:

* resolving `LoadedServerShip` / `ServerShip`;
* querying `ServerShipWorld`;
* reading or mutating Minecraft levels, blocks, and entities;
* creating, deleting, teleporting, renaming, scaling, or splitting server ships;
* accessing game-owned attachment contents.

### `@VsBeta`

`org.valkyrienskies.core.api.VsBeta` is `@ApiStatus.Experimental` plus a
`@RequiresOptIn(Level.WARNING)`. Most events and several properties are
`@VsBeta`. Opt in deliberately and be ready to catch `LinkageError` or
`NotImplementedError` if the symbol disappears in a future version.

## Physics and game ticks

The game tick (20 Hz) and the physics tick are independent loops. The Core
server config (`VSCoreConfig.SERVER.pt`) controls the relationship:

| Setting | Default | Meaning |
| --- | --- | --- |
| `physicsTicksPerGameTick` | `3` | Physics ticks per game tick (3 × 20 = 60/s) |
| `synchronizePhysics` | `false` | Couples physics stepping to the game frame |
| `physicsSpeed` (Krunch config) | `1.0` | Simulation speed multiplier |

`PhysTickEvent` is emitted at the **start** of a physics tick, after updates
from the game-frame queue have been applied. Its `delta` is the time since the
previous physics tick **in milliseconds**, and is normally exactly `1/60`
second unless synchronized physics is enabled. Never hard-code a physics tick
rate in simulation logic; derive per-second quantities from `delta`.

Because the loops are independent:

* a game tick may observe zero, one, or several physics ticks;
* a sleeping body may produce no physics work;
* events queued from the preceding physics frame may be delivered later.

## Safe handoff between physics and game

`CollisionEvent`'s own documentation recommends a `ConcurrentLinkedQueue` for
physics-to-game transfer. Copy the scalars and vectors you need into an
immutable snapshot, publish the snapshot through the queue, and resolve game
objects on the game thread:

```java
record ContactSnapshot(String dimension, long shipA, long shipB,
        org.joml.Vector3d position, org.joml.Vector3d normal,
        float separation, org.joml.Vector3d velocity) {}

private final java.util.concurrent.ConcurrentLinkedQueue<ContactSnapshot> pending =
        new java.util.concurrent.ConcurrentLinkedQueue<>();

// Physics thread
void onCollision(org.valkyrienskies.core.api.events.CollisionEvent event) {
    for (org.valkyrienskies.core.api.physics.ContactPoint p : event.getContactPoints()) {
        pending.add(new ContactSnapshot(
            event.getDimensionId(), event.getShipIdA(), event.getShipIdB(),
            new org.joml.Vector3d(p.getPosition()),
            new org.joml.Vector3d(p.getNormal()),
            p.getSeparation(),
            new org.joml.Vector3d(p.getVelocity())));
    }
}

// Game thread
void onServerTick() {
    ContactSnapshot snapshot;
    while ((snapshot = pending.poll()) != null) {
        processOnGameThread(snapshot);
    }
}
```

A concurrent queue safely **publishes** the immutable record. It does **not**
make a mutable `CollisionEvent`, `PhysShip`, Minecraft object, or backend-owned
vector safe for concurrent use.

For the reverse direction (game to physics), VS2 itself uses a
`GameToPhysicsAdapter` that records requested forces/joint changes on the game
thread and replays them during `PhysTickEvent`. See
[11-advanced-integration-facilities.md](11-advanced-integration-facilities.md).

## Listener thread semantics

`ListenableEvent.on(...)` invokes the callback **on the thread that emits the
event**. Registering a listener does not move work onto another thread and does
not change ownership. A callback must obey the annotation of the event it
receives. `RegisteredListener.unregister()` is the only lifecycle control; there
is no "off-thread" variant.

## Choosing a callback

Use **physics callbacks** (`ShipPhysicsListener`, `PhysTickEvent`,
`CollisionEvent`, `BlockEntityPhysicsListener`, `EntityPhysicsListener`) for:

* applying force and torque;
* reading live physics mass, pose, velocity, and contact data;
* updating physics-only control state.

Use **game callbacks** (`TickEndEvent`, `ServerTickListener`, `ShipLoadEvent`,
Minecraft tick hooks) for:

* Minecraft block, entity, and inventory access;
* `LoadedServerShip` / `ServerShipWorld` lookup;
* attachment creation and game-state mutation;
* ship lifecycle and connectivity response;
* persistence and networking decisions.

Use **client/render callbacks** (`StartUpdateRenderTransformsEvent`,
`PreRenderShipEvent`, `PostRenderShipEvent`) only for client visuals. Never
mutate server ships or persistent state from them.

## Common race conditions and pitfalls

1. Looking up a `LoadedServerShip` from a collision callback (physics thread).
2. Storing a `PhysShip`/`PhysLevel` in a field read by a server tick.
3. Mutating a backend-owned vector in place while the physics engine still owns
   it — copy first.
4. Treating an attachment container as if thread safety made its *contents*
   thread-safe, or as if it overrode the owner's thread annotation.
5. Deleting a server ship while a physics callback still holds the old physics
   reference.
6. Assuming a listener fires once because it was registered once.
7. Assuming a collision participant is a ship — the other side may be the
   ground/world body, whose ID is not in `loadedShips`.
8. Assuming events are delivered in a fixed order or cadence across backends.
