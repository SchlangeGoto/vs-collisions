# Practical Examples

All examples state the thread they run on. Imports are abbreviated; see the
chapter for each type for its package.

## 1. Obtain the API (any thread, after VS2 loads)

```java
import org.valkyrienskies.mod.api.ValkyrienSkies;
import org.valkyrienskies.mod.api.VsApi;

VsApi api = ValkyrienSkies.api();   // throws IllegalStateException if VS2 is absent
```

Kotlin:

```kotlin
import org.valkyrienskies.mod.api.vsApi
val api = vsApi
```

## 2. Register a physics attachment and apply force (game setup + physics tick)

```java
import org.valkyrienskies.core.api.ships.*;
import org.valkyrienskies.core.api.world.PhysLevel;
import org.joml.Vector3d;

public final class Thruster implements ShipPhysicsListener {
    private volatile double throttle;   // game writes, physics reads

    public void setThrottle(double value) { this.throttle = value; }

    @Override
    public void physTick(PhysShip ship, PhysLevel level) {
        if (throttle <= 0.0) return;
        // Model force at a model-space position (the thruster's block position).
        ship.applyModelForce(new Vector3d(0, 0, 4000.0 * throttle),
                             new Vector3d(12.5, 64.5, -3.5));
    }
}

// During mod initialisation (game thread):
api.registerAttachment(Thruster.class);

// When a ship is loaded (game thread, e.g. in shipLoadEvent):
api.getShipLoadEvent().on(event -> {
    LoadedServerShip ship = event.getShip();
    ship.getOrPutAttachment(Thruster.class, Thruster::new).setThrottle(0.5);
});
```

Kotlin:

```kotlin
class Thruster : ShipPhysicsListener {
    @Volatile var throttle = 0.0
    override fun physTick(physShip: PhysShip, physLevel: PhysLevel) {
        if (throttle <= 0.0) return
        physShip.applyModelForce(Vector3d(0.0, 0.0, 4000.0 * throttle), Vector3d(12.5, 64.5, -3.5))
    }
}
api.registerAttachment(Thruster::class.java)
```

## 3. Capture collisions safely (physics -> game)

```java
import java.util.concurrent.ConcurrentLinkedQueue;
import org.joml.Vector3d;

record ContactSnapshot(String dimension, long shipA, long shipB,
                       Vector3d position, Vector3d normal,
                       float separation, Vector3d velocity) {}

private final ConcurrentLinkedQueue<ContactSnapshot> pending = new ConcurrentLinkedQueue<>();

// Physics thread
void onStart(CollisionEvent event) {          // collisionStartEvent
    for (ContactPoint p : event.getContactPoints()) {
        pending.add(new ContactSnapshot(event.getDimensionId(),
                event.getShipIdA(), event.getShipIdB(),
                new Vector3d(p.getPosition()), new Vector3d(p.getNormal()),
                p.getSeparation(), new Vector3d(p.getVelocity())));
    }
}

// Game thread: TickEndEvent
api.getTickEndEvent().on(event -> {
    ContactSnapshot snap;
    while ((snap = pending.poll()) != null) {
        ServerShipWorld world = event.getWorld();
        LoadedServerShip a = world.getLoadedShips().getById(snap.shipA()); // may be null (ground)
        // ... act on the game thread
    }
});
```

## 4. Query ships and transforms (game thread)

```java
import org.valkyrienskies.mod.api.ValkyrienSkies;
import org.valkyrienskies.core.api.ships.Ship;
import org.joml.Vector3d;
import org.joml.primitives.AABBd;

Ship ship = ValkyrienSkies.getShipManagingBlock(level, blockPos);
if (ship != null) {
    // world -> model (shipyard) space
    Vector3d model = ship.getTransform().getToModel()
            .transformPosition(new Vector3d(worldX, worldY, worldZ), new Vector3d());

    // model -> world, using the Minecraft helper
    var worldPos = ValkyrienSkies.positionToWorld(ship, model);

    // All ships whose world AABB intersects a region, in one dimension:
    ShipWorld shipWorld = ValkyrienSkies.getShipWorld(level);
    for (Ship s : shipWorld.getAllShips()
            .getIntersecting(new AABBd(minX, minY, minZ, maxX, maxY, maxZ),
                             ValkyrienSkies.getDimensionId(level))) {
        // ...
    }
}
```

## 5. Apply a world force at a model position (physics thread)

```java
// Balloon: always push "up" in world space, applied at a model-space block position.
ship.applyWorldForceToModelPos(new Vector3d(0, 5000, 0), balloonModelPos);
```

## 6. Read live physics state (physics thread only)

```java
if (!ship.isSleeping()) {
    double mass = ship.getMass();
    Vector3dc com = ship.getCenterOfMass();        // model space
    Matrix3dc inertia = ship.getMomentOfInertia(); // model space
    double fluid = ship.getLiquidOverlap();        // 0..1
    Vector3dc v = ship.getVelocity();
    Vector3dc w = ship.getAngularVelocity();
    // Copy anything you need beyond this callback.
}
```

## 7. Server-tick per-ship listener (game thread)

```java
public final class FuelBurner implements ServerTickListener {
    @Override public void onServerTick() { /* game-thread per-ship work */ }
}

// Register as an attachment on a LoadedServerShip; VS2 invokes it once per game tick.
ship.getOrPutAttachment(FuelBurner.class, FuelBurner::new);
```

## 8. Register a physics block state (initialisation)

The block-state builders use Kotlin function receivers, so this example is in
Kotlin (Java callers can pass a `Function1<SolidState.Builder, Unit>` but it is
awkward):

```kotlin
import org.valkyrienskies.mod.api.events.RegisterBlockStateEvent

api.registerBlockStateEvent.on { event ->
    val solid = event.buildSolidState {
        elasticity(0.3)
        friction(0.5)
        boxesShape {
            addPositiveBox(AABBi(0, 0, 0, 15, 15, 15))
            addCollisionPoint(8f, 8f, 8f, 4f)
        }
    }
    event.register(MY_BLOCK.defaultBlockState(), solid)
}
```

## 9. Connectivity queries (game thread, experimental)

```java
import org.valkyrienskies.core.api.world.connectivity.ConnectionStatus;

ConnectionStatus status = world.isIsolatedSolid(x, y, z, dimensionId);
if (status == ConnectionStatus.DISCONNECTED) {
    long size = world.getSolidComponentSize(x, y, z, dimensionId);
    var voxels = world.indexSolidComponentVoxels(x, y, z, dimensionId);
    // ... e.g. fracture this component
} else if (status == ConnectionStatus.UNKNOWN) {
    // Required chunks are not loaded; do not treat as disconnected.
}
```

## 10. Client render transform (client thread only)

```java
import org.valkyrienskies.core.api.ships.ClientShip;

ClientShip ship = (ClientShip) ValkyrienSkies.getShipManagingBlock(clientLevel, pos);
if (ship != null) {
    // Interpolated transform for rendering; NOT authoritative physics state.
    var renderToWorld = ship.getRenderTransform().getToWorld();
    var renderAABB = ship.getRenderAABB();
}
```

## 11. Entity physics listener (physics thread)

```java
import org.valkyrienskies.mod.api.EntityPhysicsListener;
import org.valkyrienskies.core.api.world.PhysLevel;

public final class MyEntity extends Entity implements EntityPhysicsListener {
    private volatile double cachedMass;   // thread-safe handoff only
    private String dimension;             // set by the implementer

    @Override public void physTick(PhysLevel level) {
        // Do NOT read ordinary Minecraft fields here. Use atomics/volatiles,
        // then schedule Minecraft mutations back to the game thread.
    }

    @Override public String getDimension() { return dimension; }
    @Override public void setDimension(String value) { dimension = value; }
}
```

## 12. Route work out of a physics callback

```java
// In a physics callback, schedule game-thread work instead of touching the world:
org.valkyrienskies.mod.common.VSGameUtilsKt.executeOrSchedule(level, () -> {
    level.setBlockAndUpdate(pos, blockState);
});
```

## 13. Game-thread ship creation (internal API, game thread)

```java
import org.valkyrienskies.core.internal.world.VsiServerShipWorld;
import org.valkyrienskies.core.api.ships.ServerShip;
import org.valkyrienskies.mod.api.ValkyrienSkies;
import org.joml.Vector3i;

VsiServerShipWorld world = (VsiServerShipWorld) ValkyrienSkies.getShipWorld(level);
ServerShip ship = world.createNewShipAtBlock(
        new Vector3i(x, y, z), true, 1.0,
        ValkyrienSkies.getDimensionId(level));
// ...
world.deleteShip(ship);
```

Keep casts like these inside one version-pinned adapter; they are internal API
and must be re-verified on every VS2/Core update.

## 14. Transform points and directions correctly

```java
import org.joml.Vector3d;

// Point: include translation.
Vector3d modelPoint = ship.getTransform().getToModel()
        .transformPosition(new Vector3d(worldPoint), new Vector3d());

// Direction: rotation only, no translation or scaling.
Vector3d worldDir = ship.getTransform().getRotation()
        .transform(new Vector3d(modelDir), new Vector3d()).normalize();

// Direction with scaling: use the matrix.
Vector3d scaledWorldDir = ship.getTransform().getToWorld()
        .transformDirection(new Vector3d(modelDir), new Vector3d());
```
