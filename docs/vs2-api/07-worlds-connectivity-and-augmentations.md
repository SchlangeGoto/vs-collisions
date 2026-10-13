# Worlds, Connectivity, and Augmentations

## World types

| Type | Annotation | Scope |
| --- | --- | --- |
| `ShipWorld` | — | Common ship-world queries (base interface) |
| `ServerShipWorld` | `@GameTickOnly` | Global server world across **all** dimensions |
| `ClientShipWorld` | `@GameTickOnly` | Client ship state |
| `PhysLevel` | `@VsBeta @PhysTickOnly` | One dimension's physics world |
| `VsiServerShipWorld` | internal `@GameTickOnly` | Server implementation |
| `VsiClientShipWorld` | internal | Client implementation |
| `VsiPhysLevel` | internal `@PhysTickOnly` | Physics implementation |

The **server ship world is global**: `allShips` and `loadedShips` are not
dimension-filtered. Include the dimension in chunk and AABB queries. `PhysLevel`
is dimension-specific.

## `ShipWorld` queries

Source: `org/valkyrienskies/core/api/world/ShipWorld.kt`. Verified public
members:

```java
QueryableShipData<Ship>       getAllShips();
QueryableShipData<LoadedShip> getLoadedShips();

boolean isChunkInShipyard(int chunkX, int chunkZ, String dimensionId);
boolean isBlockInShipyard(int x, int y, int z, String dimensionId);

List<LoadedShip> getShipObjectsIntersecting(AABBdc aabb);   // deprecated

ConnectionStatus isConnectedByAir(int x1,int y1,int z1,int x2,int y2,int z2,String dim);
ConnectionStatus isConnectedBySolid(int x1,int y1,int z1,int x2,int y2,int z2,String dim);
ConnectionStatus isIsolatedAir(int x,int y,int z,String dim);
ConnectionStatus isIsolatedSolid(int x,int y,int z,String dim);

long getSolidComponentSize(int x,int y,int z,String dim);
long getAirComponentSize(int x,int y,int z,String dim);

Set<Vector3ic> getAllSolidComponentsFromClaim(String dim, ChunkClaim claim);
Set<Vector3ic> getAllAirComponentsFromClaim(String dim, ChunkClaim claim);

void traverseSolidComponent(int x,int y,int z, Function4<Integer,Integer,Integer,Long,Unit> action, String dim);
void traverseAirComponent  (int x,int y,int z, Function4<Integer,Integer,Integer,Long,Unit> action, String dim);

Set<Vector3ic> indexSolidComponent(int x,int y,int z,String dim);
Set<Vector3ic> indexSolidComponent(int x,int y,int z,Set<? extends Vector3ic> out,String dim);
Set<Vector3ic> indexAirComponent  (int x,int y,int z,String dim);
Set<Vector3ic> indexAirComponent  (int x,int y,int z,Set<? extends Vector3ic> out,String dim);

Set<SparseVoxelPosition> indexSolidComponentVoxels(int x,int y,int z,String dim);
Set<SparseVoxelPosition> indexSolidComponentVoxels(int x,int y,int z,Set<SparseVoxelPosition> out,String dim);
Set<SparseVoxelPosition> indexAirComponentVoxels  (int x,int y,int z,String dim);
Set<SparseVoxelPosition> indexAirComponentVoxels  (int x,int y,int z,Set<SparseVoxelPosition> out,String dim);
```

Everything from `isConnectedByAir` down is `@Experimental`. `getShipObjectsIntersecting`
and `queryableShipData` are deprecated; use `getIntersecting`/`allShips`.

## Shipyard and chunk claims

Ship transforms map model (shipyard) coordinates to world coordinates. A
`ChunkClaim` identifies the 2-D region of the shipyard that owns a ship's
blocks. Key facts from `org/valkyrienskies/core/api/ships/properties/ChunkClaim.kt`:

* `ChunkClaim.DIAMETER` is `256` chunks, hard-coded; every ship is given a
  `256 × 256` chunk claim.
* `xIndex`/`zIndex` identify the claim; `xStart`/`xEnd`/`zStart`/`zEnd` are
  inclusive block bounds; `xMiddle`/`zMiddle` are the centres; `size` is the
  number of owned chunks.
* `contains(x, z)`, `toLong()`, `getCenterBlockCoordinates(yRange)`,
  `getBlockSize(yRange)`, and `getTotalVoxelRegion(yRange)` describe the claim.
* Companion helpers: `getClaimXIndex(chunkX)`, `getClaimZIndex(chunkZ)`,
  `claimToLong(claimX, claimZ)`, `getClaimThenToLong(chunkX, chunkZ)`.

`IShipActiveChunksSet` tracks the ship chunks that are loaded/ticking:

```java
int size();
boolean add(int chunkX, int chunkZ);
boolean remove(int chunkX, int chunkZ);
boolean contains(int chunkX, int chunkZ);
void forEach(IntBinaryConsumer func);
void getMinMaxWorldPos(Vector3i min, Vector3i max, LevelYRange yRange);
```

Long-encoding helpers `chunkPosToLong`, `longToChunkX`, `longToChunkZ` are
provided. Several older `*ChunkPos`/`getTotalChunks`/`iterateChunkPos` members
are deprecated in favour of the above.

Do not infer that a world-space `BlockPos` is a shipyard-space position. Use the
ship transform or the Minecraft integration helpers.

## `LevelYRange`

`org.valkyrienskies.core.api.world.LevelYRange(minY, maxY)` describes a
dimension's vertical range (Minecraft dimensions need not be 0–255). It
validates that `minY ≡ 0 (mod 16)`, `maxY ≡ 15 (mod 16)`, and `maxY >= minY`.
It exposes `center`, `size`, and `forEachYChunkPos`.

## Connectivity graph

The connectivity graph can represent connected **solids** or **air pockets**.
`ConnectionStatus` is an `@Experimental` enum:

* `CONNECTED` — the blocks are connected / the block is part of the main world.
* `DISCONNECTED` — not connected / isolated (an island or air pocket).
* `UNKNOWN` — could not be determined because required chunks are unavailable.

Always handle `UNKNOWN` explicitly; it is not the same as `DISCONNECTED`.

`Component` (`org.valkyrienskies.core.api.world.connectivity.Component`)
represents a connected component:

```java
long getSize();
double collectAugmentation(DoubleAugmentation key);
```

`SparseVoxelPosition` extends JOML's `Vector3i` with an `extent` (default 1),
representing a cube AABB. It provides `contains(...)`, `overlaps(...)`,
`getVolume()`, `getContaining()`, `getContainingAsVoxel()`,
`getHigherLevelVoxel(...)`, `relativeTo(...)`, and `allRelativeTo(...)`.

### Traversal and indexing

* `isConnectedByAir` / `isConnectedBySolid` test connectivity between two
  positions.
* `isIsolatedAir` / `isIsolatedSolid` test whether a position is part of the
  main world or an isolated component.
* `getSolidComponentSize` / `getAirComponentSize` return the size of the
  component containing a position. They throw `IllegalArgumentException` if the
  position is not `DISCONNECTED`.
* `getAllSolidComponentsFromClaim` / `getAllAirComponentsFromClaim` return the
  roots of all components in a claim.
* `traverseSolidComponent` / `traverseAirComponent` walk a component and invoke
  an action for each block.
* `index*Component` return sets of block positions (or sparse voxels) belonging
  to the component containing a position, optionally filling a caller-provided
  set to reduce allocation.

`SplitHandler` in the Minecraft integration is a worked example of using these
APIs to fracture a ship; see
[11-advanced-integration-facilities.md](11-advanced-integration-facilities.md).

## Augmentations

Augmentations attach a `double` to blocks or to connected components. They are
`@Experimental` and, as the source warns repeatedly, **not currently
serialized** — they are not a substitute for persistent Minecraft data or
attachments.

| Type | Meaning |
| --- | --- |
| `DoubleAugmentation` | A named double value with `combineDouble(a, b)`; the combination must form a commutative group. |
| `DoubleComponentAugmentation : DoubleAugmentation` | Adds `splitDouble(value, component1, component2)` used when a component splits. Holding a reference to the components past the call is forbidden. |

Create and use them on `ServerShipWorld` (all `@Experimental`):

```java
DoubleAugmentation gas = world.createDoubleSumAugmentation("mymod", "gas");
DoubleComponentAugmentation gasPerComponent =
        world.createDoubleSumComponentAugmentation("mymod", "gas");

world.setSolidAugmentation(gas, x, y, z, dimension, 1.5);
double v = world.getSolidAugmentation(gas, x, y, z, dimension);       // 0.0 default
world.updateSolidAugmentation(gas, x, y, z, dimension, old -> old + 1);
world.removeSolidAugmentation(gas, x, y, z, dimension);
double total = world.collectSolidAugmentation(gas, x, y, z, dimension);

world.setSolidComponentAugmentation(gasPerComponent, x, y, z, dimension, 1.0);
double c = world.getSolidComponentAugmentation(gasPerComponent, x, y, z, dimension);
world.removeSolidComponentAugmentation(gasPerComponent, x, y, z, dimension);

Map<Triple<Integer,Integer,Integer>, Double> perSolid =
        world.getFromEachSolidComponent(gas, dimension, /* chunkClaim = */ null);
Map<Triple<Integer,Integer,Integer>, Double> perSolidRoot =
        world.getFromEachSolidComponentRoot(gasPerComponent, dimension, null);
```

The `air` variants mirror every `solid` method. The `collect*` and
`*Component*` methods throw `IllegalArgumentException` when
`isIsolatedSolid`/`isIsolatedAir` does not return `DISCONNECTED` for the
coordinates, so check connectivity status first. `getFromEach*` accept an
optional `ChunkClaim` to scope the scan.

## Dimension identity

Core represents a dimension as `DimensionId`, a **typealias for `String`**
(source: `org/valkyrienskies/core/api/world/properties/DimensionId.kt`). The
Minecraft integration derives it from a `Level` (`VsApi.getDimensionId(level)`).
Dimension identity is required for:

* server ship queries (`getByChunkPos`, `getIntersecting`);
* ship creation and ID allocation;
* connectivity and augmentations;
* mapping a `PhysLevel` to a Minecraft `Level`.

`VsiCore.isChunkInShipyard(chunkX, chunkZ)` (no dimension) is deprecated because
chunk allocation may become per-world in the future.
