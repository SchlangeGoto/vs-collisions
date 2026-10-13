# Threading and event handoffs

Related: [Documentation index](../index.md) · [VS2/Core API reference](../reference/vs2-core-api.md) · [Collision data flow](../architecture/collision-data-flow.md)

VS2 collision callbacks run on the physics tick. Minecraft block, entity, and
loaded server ship operations run on the game/server tick. Crossing that
boundary without a handoff is unsafe.

## Ownership

| Physics tick | Game/server tick |
| --- | --- |
| CollisionEvent, ContactPoint, PhysLevel, PhysShip | ServerLevel, LoadedServerShip, ServerShipWorld, blocks, entities |
| Read physics state; apply physics forces | Resolve dimensions and loaded ships; mutate blocks and attachments |
| Push copied data to a concurrent queue | Drain the queue and perform game work |

GameTickOnly and PhysTickOnly are ownership contracts, not synchronization.
Attachment lookup is thread-safe, but attachment contents still need their own
ownership rules.

## Capture a snapshot

The resolved Core documentation recommends ConcurrentLinkedQueue for collision
handoff. Copy only the data needed later; do not queue CollisionEvent,
ContactPoint, PhysLevel, or PhysShip objects.

~~~java
record ContactSnapshot(
        String dimensionId,
        long shipIdA,
        long shipIdB,
        Vector3d position,
        Vector3d normal,
        float separation,
        Vector3d relativeVelocity) {}

private final ConcurrentLinkedQueue<ContactSnapshot> pending =
        new ConcurrentLinkedQueue<>();

private void onCollision(CollisionEvent event) { // physics tick
    for (ContactPoint point : event.getContactPoints()) {
        pending.add(new ContactSnapshot(
                event.getDimensionId(), event.getShipIdA(), event.getShipIdB(),
                new Vector3d(point.getPosition()), new Vector3d(point.getNormal()),
                point.getSeparation(), new Vector3d(point.getVelocity())));
    }
}
~~~

JOML vectors are mutable. Copying them makes the application own the values
that cross the queue.

## Consume on the server tick

Drain the queue from a server tick listener. Resolve the level from the snapshot
dimension and resolve loaded ships at consumption time. A missing ship is valid:
it may be unloaded, removed, or represent the dimension ground body.

~~~java
private void drain(MinecraftServer server) { // server tick
    for (ContactSnapshot snapshot; (snapshot = pending.poll()) != null;) {
        ServerLevel level = VSGameUtilsKt.getLevelFromDimensionId(
                server, snapshot.dimensionId());
        if (level == null) continue;

        VsiServerShipWorld world = VSGameUtilsKt.getShipObjectWorld(level);
        LoadedServerShip shipA = world.getLoadedShips().getById(snapshot.shipIdA());
        LoadedServerShip shipB = world.getLoadedShips().getById(snapshot.shipIdB());
        // Perform game-thread work only after these checks.
    }
}
~~~

## Rules for repository changes

* Keep physics callbacks short: copy, enqueue, return.
* Do not read or mutate Minecraft state from a collision callback.
* Do not retain physics objects after their callback returns.
* Keep mutable attachment contents confined to the game thread unless the
  attachment itself provides explicit synchronization.
* Coalesce or rate-limit snapshots on the game thread if expensive processing
  is required.
* Convert contact positions from world space to shipyard/model space only after
  resolving the target loaded ship. See [coordinate transforms](../architecture/coordinate-spaces.md).

The current CollisionQueue queues CollisionEvent instances rather than snapshots.
That does not match the stricter lifetime-safe pattern above; changes to that
queue should migrate it to application-owned snapshots.
