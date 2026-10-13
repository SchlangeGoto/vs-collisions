# Block mutation and destruction

Related: [Documentation index](../index.md) · [Coordinate spaces](coordinate-spaces.md) · [Ship lifecycle](ship-lifecycle-and-fracturing.md) · [Stress and material pipeline](stress-and-material-pipeline.md)

---

## 1. The Cardinal Rule of Ship Blocks

> [!important] Ship blocks are *just blocks in a `ServerLevel`*
> VS2 ships store their voxel data in normal chunks, in a far-away coordinate region called the **shipyard** (the ship's `ChunkClaim`). You mutate ship blocks with the **same vanilla APIs** as world blocks — `level.getBlockState`, `level.setBlock`, `level.destroyBlock` — but always at **shipyard coordinates** (the result of a `worldToShip` transform), never at world coordinates.

A position is shipyard-space iff it lies inside a ship chunk claim in that dimension. Both VS2 facades offer the check (the codebase uses each):

```kotlin
// 1. org.valkyrienskies.mod.common (VSGameUtilsKt — used by BlockImpactInjector)
val inShipyard = VSGameUtilsKt.isBlockInShipyard(level, pos)

// 2. org.valkyrienskies.mod.api.ValkyrienSkies (top-level Level extension
//    over vsApi.isChunkInShipyard — used by BlockDamageResolver / StressPropagator)
val inShipyard2 = ValkyrienSkies.isBlockInShipyard(level, pos)
```

Both answer "is this *chunk* claimed by a ship in this dimension" — they agree on allocation boundaries. Pick one style per call site for readability; don't mix them in a single expression.

> [!warning] Dimension matters
> `isBlockInShipyard(x, y, z, dimensionId)` is dimension-scoped. A ship in `minecraft:overworld` shares no shipyard with `minecraft:the_nether`. Always use the `ServerLevel` resolved from `collision.dimensionId` (as `DamageApplicator` does via `getLevelFromDimensionId`), not some ambient level.

## 2. Coordinate Duality in Practice

```kotlin
// Physics gives world space...
val contactWorld: Vector3dc = contactPoint.position

// ...damage deals in shipyard space:
val local = ship.transform.worldToShip.transformPosition(Vector3d(contactWorld))
val pos = BlockPos(floor(local.x).toInt(), floor(local.y).toInt(), floor(local.z).toInt())

if (!level.isBlockInShipyard(pos)) return       // not on any ship — world-side damage
if (level.getBlockState(pos).isAir)  return     // empty voxel — nothing to break
```

See [coordinate spaces](coordinate-spaces.md) for full transform semantics,
including why directions use transformDirection (no translation) and positions
use transformPosition.

## 3. Destruction Modes Used by the Mod

### 3.1 Full destruction — `BlockDamageResolver.evaluate`

```kotlin
// vonMises >= effectiveUltimate → structural failure
level.destroyBlock(pos, /* dropBlockDrops */ true)
damageData.remove(pos)
```

- `dropBlockDrops = true` spawns item drops inside the shipyard — VS2 ships them to the world. For combat flavor this is desired (cannonballs scatter scrap).
- Swap to `level.removeBlock(pos, false)` or `level.setBlock(pos, Blocks.AIR.defaultBlockState(), Block.UPDATE_ALL)` for surgical removal without drops (e.g., scripted scuttling).

### 3.2 Progressive damage — crack overlays

```kotlin
val crackStage = (newAccumulated * 9).toInt()        // vanilla stages 0..9
level.destroyBlockProgress(pos.hashCode(), pos, crackStage)
```

> [!warning] `destroyBlockProgress` semantics
> - First argument is an *arbitrary breaker id*. `pos.hashCode()` results in one overlay slot per position, which is fine here — but `BlockPos.hashCode()` is **not unique** across positions; two different blocks may rarely share a breaker id, clobbering each other's overlay. Cosmetic only in this design.
> - Cracks are **not self-clearing**. When the block is destroyed, healed, or the stress dissipates, you must send `-1` progress (`level.destroyBlockProgress(id, pos, -1)`), or clients keep seeing cracks. Currently the mod clears the fatigue entry (`damageData.remove(pos)`) but never sends the `-1` — a known cosmetic leak to fix.
> - Related: `ShipDamageData.prune(existingBlocks)` is defined but never invoked — wire it to a periodic game-thread sweep over `ship.shipAABB` to reclaim memory and reset overlays in one pass.

## 4. Failure Model (how a block decides to break)

From `BlockDamageResolver` + `CollisionMaterialRegistry`:

**Material thresholds** (`CollisionMaterialRegistry.get`):

```kotlin
val props = CbcToughnessHelper.getBlockProps(level, state, pos)   // CBC toughness, or
                                                                  // explosionResistance fallback
val miningHardness = state.getDestroySpeed(level, pos)            // -1 = indestructible

if (miningHardness < 0)                                           // bedrock & co.
    return CollisionMaterialProps(Double.MAX_VALUE, Double.MAX_VALUE)

CollisionMaterialProps(
    yieldThreshold    = miningHardness * 0.4,   // YIELD_FACTOR
    ultimateThreshold = props.toughness() * 1.0 // ULTIMATE_FACTOR, honoring CBC overrides
)
```

Datapack/pack-dev overrides land in `CollisionMaterialRegistry.registerOverride(id, yield, ultimate)` (call during datapack reload; `clearOverrides()` on un-load).

**Stress evaluation** — Von Mises equivalence with shear weighted ×3 (`SHEAR_VON_MISES`), physics units bridged to the CBC toughness scale by `STRESS_SCALE = 1.0 / 100000.0`:

```kotlin
val accumulated = damageData.getAccumulated(pos)
val fatigueMultiplier = max(0.05, 1.0 - accumulated)    // floor keeps late-game behavior sane
val effectiveYield    = props.yieldThreshold    * fatigueMultiplier
val effectiveUltimate = props.ultimateThreshold * fatigueMultiplier

val vonMises = sqrt(
    (stress.compression * STRESS_SCALE).pow(2) +
    3.0 * (stress.shear * STRESS_SCALE).pow(2)
)

when {
    vonMises < effectiveYield     -> Unit                        // elastic: block springs back
    vonMises >= effectiveUltimate -> { destroyBlock; fatigue.remove }  // failure
    else -> {                                                    // plastic zone
        val damageRatio = (vonMises - effectiveYield) / (effectiveUltimate - effectiveYield)
        val newAcc = min(1.0f, accumulated + damageRatio.toFloat() * 0.3f /*FATIGUE_RATE*/)
        damageData.setAccumulated(pos, newAcc)
        level.destroyBlockProgress(pos.hashCode(), pos, (newAcc * 9).toInt())
    }
}
```

Regimes:

| Regime | Condition | Effect |
|---|---|---|
| Elastic | `σ_vm < yield` | nothing permanent |
| Plastic | `yield ≤ σ_vm < ultimate` | fatigue += ratio × 0.3, crack overlay deepens |
| Failure | σ_vm ≥ ultimate | destroyBlock, fatigue reset — VS2 split handler may fire; see [fracturing](ship-lifecycle-and-fracturing.md#5-fracturing-via-vs2s-own-splitter). |

## 5. A Batch Destruction Template

Putting together [ship lifecycle](ship-lifecycle-and-fracturing.md): for a
catastrophic hit that must remove a region atomically:

```kotlin
import org.valkyrienskies.mod.common.util.SplittingDisablerAttachment

fun breachHull(ship: LoadedServerShip, level: ServerLevel, blastRegion: List<BlockPos>) {
    val gate = SplittingDisablerAttachment(true).also { it.disableSplitting() }
    ship.setAttachment(gate)
    try {
        blastRegion.forEach { pos ->
            if (level.isBlockInShipyard(pos) && !level.getBlockState(pos).isAir) {
                level.destroyBlock(pos, true)
            }
        }
    } finally {
        gate.enableSplitting()
        ship.setAttachment(gate)   // one structural re-evaluation at the end
    }
}
```

## 6. Neighbor & Update Rules

- `destroyBlock` runs the full vanilla path (neighbor updates, `onRemove`, drops, block-entity unload). Cheap enough at the current zone counts (≤ 6 zones × ≤ ~125 voxels), but a magazine detonation iterating thousands of voxels in one tick **will** hitch the game thread — chunk it over multiple ticks if you add big explosions.
- Blocks *inside* the shipyard obey identical neighbor-update semantics as normal ones — this is precisely why breaking a hull strips attached torches/rails and triggers realistic secondary collapse.
- Never mutate blocks during iteration over a stress map that was derived from blocks you are also destroying; the resolver already reads state lazily per block (`level.getBlockState(pos)`), which tolerates mid-loop destruction because each block is evaluated once.

## 7. Housekeeping

Two latent leaks in the current code worth a maintenance task:

1. **Fatigue entries survive block removal in non-destroyBlock paths** (e.g., player mining a damaged block) — `ShipDamageData.prune` exists for exactly this; schedule it.
2. **Crack overlays survive** fatigue reset — send `destroyBlockProgress(id, pos, -1)` wherever fatigue is cleared (`damageData.remove`, `setAccumulated(pos, 0f)`).
