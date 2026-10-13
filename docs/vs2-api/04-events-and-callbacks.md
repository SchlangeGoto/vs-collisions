# Events and Callbacks

## Event plumbing

Source: `org/valkyrienskies/core/api/event/*.kt`.

```kotlin
fun interface EventConsumer<in T> {
    fun accept(event: T, handler: RegisteredListener)
}

fun interface RegisteredListener {
    fun unregister()
}

interface EmittableEvent<in T> {
    fun emit(value: T)
}

interface ListenableEvent<out T> {
    fun on(cb: EventConsumer<T>): RegisteredListener
    fun on(cb: Consumer<in T>): RegisteredListener
    fun once(cb: Consumer<in T>): RegisteredListener
    fun once(predicate: Predicate<in T>, cb: Consumer<in T>): RegisteredListener
}

interface SingleEvent<T> : ListenableEvent<T>, EmittableEvent<T>
```

* `on(cb)` stays registered until `RegisteredListener.unregister()` is called.
* `once(cb)` unregisters after its first invocation; `once(predicate, cb)`
  unregisters after the first invocation for which the predicate is `true`.
* Registration does not change the emitter thread. The callback runs on the
  emitting thread and must obey that event's annotation.
* The concrete implementation is
  `org.valkyrienskies.core.util.events.EventEmitterImpl` (in the `util`
  artifact). It implements `SingleEvent<T>`.

A deprecated `org.valkyrienskies.core.impl.hooks.VSEvents` object still exposes
legacy `ShipLoadEvent`, `ShipLoadEventClient`, `ShipUnloadEventClient`,
`StartUpdateRenderTransformsEvent`, and `TickEndEvent` singletons. Do not use
it; use the `VsCoreApi` properties.

## Event inventory

| Event | Annotation | Payload |
| --- | --- | --- |
| `CollisionEvent` | `@VsBeta @PhysTickOnly` | `dimensionId`, `physLevel`, `shipIdA`, `shipIdB`, `contactPoints` |
| `PhysTickEvent` | `@VsBeta @PhysTickOnly` | `world: PhysLevel`, `delta: Double` |
| `ShipLoadEvent` | `@VsBeta @GameTickOnly` | `ship: LoadedServerShip` |
| `ShipLoadEventClient` | `@VsBeta @GameTickOnly` | `ship: ClientShip` |
| `ShipUnloadEventClient` | `@VsBeta @GameTickOnly` | `ship: ClientShip` |
| `TickEndEvent` | `@VsBeta @GameTickOnly` | `world: ServerShipWorld` |
| `StartUpdateRenderTransformsEvent` | `@VsBeta @GameTickOnly` | `shipWorld: ClientShipWorld` |
| `MergeEvent` | `@VsBeta @GameTickOnly` | connectivity merge (see below) |
| `SplitEvent` | `@VsBeta @GameTickOnly` | connectivity split (see below) |
| `JointSerializedEvent` | `@VsBeta @PhysTickOnly` | joint serialization bookkeeping |
| `JointDeserializedEvent` | `@VsBeta @PhysTickOnly` | joint deserialization outcome |

Note that `StartUpdateRenderTransformsEvent` is annotated `@GameTickOnly` and
carries a `ClientShipWorld`; it is a client-side transform-update phase, not a
server hook.

## Collision events

`CollisionEvent` is `@VsBeta @PhysTickOnly` and is emitted from three streams:
`collisionStartEvent`, `collisionEndEvent`, and `collisionPersistEvent`. Its
own documentation recommends a `ConcurrentLinkedQueue` to hand events to the
game tick.

```kotlin
interface CollisionEvent {
    val dimensionId: DimensionId        // String
    val physLevel: PhysLevel
    val shipIdA: ShipId                 // Long
    val shipIdB: ShipId                 // Long
    val contactPoints: Collection<ContactPoint>
}
```

`CollisionEvent` has **no** block, material, impulse, or damage information.
Both IDs may refer to the ground/world body, which is not in `loadedShips`.
Resolve game objects later, on the game thread. Tolerate repeated persistent
events and empty contact collections; emission cadence and manifold policy can
differ between physics backends.

`ContactPoint` is `@VsBeta @PhysTickOnly`:

| Property | Type | Meaning |
| --- | --- | --- |
| `position` | `Vector3dc` | World-space contact point |
| `normal` | `Vector3dc` | Contact normal (orientation convention is not documented) |
| `separation` | `Float` | Signed separation; negative means penetration |
| `velocity` | `Vector3dc` | Relative point velocity: A's velocity minus B's |

## Physics tick event

```kotlin
@VsBeta @PhysTickOnly
interface PhysTickEvent {
    val world: PhysLevel
    val delta: Double   // milliseconds since the previous physics tick
}
```

Emitted at the start of a physics tick, right after game-frame queue updates are
processed. `delta` is normally `1/60` second but changes when synchronized
physics is enabled. Use it to derive per-second quantities and never assume a
fixed rate.

## Ship lifecycle events

* `ShipLoadEvent.ship: LoadedServerShip` — a server ship was loaded (game
  thread).
* `ShipLoadEventClient.ship: ClientShip` / `ShipUnloadEventClient.ship:
  ClientShip` — the client representation was created or removed (client
  thread).
* `TickEndEvent.world: ServerShipWorld` — emitted at the end of a server game
  tick.

`TickEndEvent` is the usual place to drain a physics-to-game queue and to
perform game-thread ship work once per tick.

## Connectivity events

`MergeEvent` and `SplitEvent` describe changes to the connectivity graph, not
collision impulses. Both carry a `voxelType: Int` documented as `1` for solids
and `0` for air, and a mutable `var dimensionId: DimensionId?`.

```kotlin
interface MergeEvent {
    val stillPocket: Boolean   // true if neither component was connected to the world
    val oldRootA: Vector3ic
    val oldRootB: Vector3ic
    val newRoot: Vector3ic
    val voxelType: Int
    var dimensionId: DimensionId?
}

interface SplitEvent {
    val wasPocket: Boolean      // true if the parent was an isolated component
    val oldRoot: Vector3ic
    val newRootA: Vector3ic
    val newRootB: Vector3ic
    val voxelType: Int
    var dimensionId: DimensionId?
}
```

They identify graph roots, not complete block lists. Query the world/component
APIs (see [07-worlds-connectivity-and-augmentations.md](07-worlds-connectivity-and-augmentations.md))
if you need component contents.

## Joint events

```kotlin
@VsBeta @PhysTickOnly
interface JointSerializedEvent {
    val jointId: Int
    val shipId0: ShipId?
    val shipId1: ShipId?
    val jointType: String
    val dimensionId: DimensionId
    val operation: Operation   // CREATED, UPDATED, RESTORED, REMOVED
}

@VsBeta @PhysTickOnly
interface JointDeserializedEvent {
    val jointId: Int
    val shipId0: ShipId?
    val shipId1: ShipId?
    val jointType: String
    val dimensionId: DimensionId?
    val outcome: Outcome       // RESTORED, DISCARDED
    val reason: String?
}
```

These describe the persisted joint store's bookkeeping. Joint support is
backend- and version-dependent; do not assume every internal joint class is
portable. See [06-physics-and-collision-api.md](06-physics-and-collision-api.md).

## Client/render events

* `StartUpdateRenderTransformsEvent` (`shipWorld: ClientShipWorld`) — the client
  transform-update phase.
* `PreRenderShipEvent` / `PostRenderShipEvent` — `@Experimental` marker
  interfaces on the `VsApi`; registered through `preRenderShipEvent` /
  `postRenderShipEvent`.

Client ships and render state must remain on the client side. Do not use render
events to mutate server ships or persistent state.

## Choosing the right event

| Goal | Event / listener |
| --- | --- |
| Apply per-ship force every physics tick | `ShipPhysicsListener` attachment |
| Read live physics state once per physics tick | `PhysTickEvent` |
| Detect/observe ship contacts | `collisionStartEvent` / `collisionPersistEvent` / `collisionEndEvent` |
| Per-ship game-tick work | `ServerTickListener` |
| Global once-per-game-tick work | `TickEndEvent` |
| React to a ship becoming loaded (server) | `ShipLoadEvent` |
| React to a client ship appearing/leaving | `ShipLoadEventClient` / `ShipUnloadEventClient` |
| React to splitting/merging | `SplitEvent` / `MergeEvent` |
| Render a ship | `PreRenderShipEvent` / `PostRenderShipEvent` |
