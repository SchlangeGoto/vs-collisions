# Stress and material pipeline

Related: [Documentation index](../index.md) · [Collision data flow](collision-data-flow.md) · [Coordinate spaces](coordinate-spaces.md) · [Block mutation and destruction](block-mutation-and-destruction.md)

This page documents the mod-internal physics math pipeline: how raw contact
points become per-block structural stress, and how that stress is compared with
material strengths. All stages execute on the game thread after the
[handoff](../guides/threading-and-event-handoffs.md).

---

## 1. Impact Zone Clustering — `ImpactZoneClusterer`

Physics emits N raw `ContactPoint`s per ship pair. These are merged into ≤ 6 spatially coherent **impact zones**, so that a hull press produces one "big dent" rather than N microscopic ones.

```kotlin
// Constants (stress/impact/ImpactZoneClusterer.java)
private const val CLUSTER_RADIUS = 2.0      // max center→point distance to join a zone
private const val NORMAL_THRESHOLD = 0.8    // min cos(angle) between contact normal & zone normal
private const val MAX_ZONES = 6
```

Algorithm:

1. **Severity-sort** all points descending by `|separation| × |velocity|` — the most violent contact seeds first.
2. **Greedy assignment**: each point joins the first zone whose running centroid is within `CLUSTER_RADIUS` *and* whose normalized running normal `dot ≥ 0.8`; otherwise it seeds a new zone.
3. **Cap**: if more than `MAX_ZONES`, repeatedly absorb the zone with lowest `severitySum` into its nearest neighbor (by centroid distance).
4. **Finalize** each zone:
   - `position = Σ pos / count`
   - `normal = normalize(Σ normals)`
   - `averageVelocity = Σ vel / count`, `direction = normalize(avgVelocity)`
   - `area = π × r²` where `r` = max contact distance from centroid (min 1.0)
   - `averagePenetration = mean(|separation|)`
   - `impulse = severitySum / totalSeverity` — *fraction of this collision's violence held by this zone* (note: this is a share, not a Newton-second impulse; `ZoneLoadCalculator` recomputes a physical one)

`ImpactZone.inverted()` produces the B-side view: negated normal, average velocity, and direction — a *deep-copied* zone, so the two sides never alias JOML state.

## 2. Load Calculation — `ZoneLoadCalculator`

Physics quantities per zone, per victim ship.

**Effective mass** — two-body reduced mass for ship↔ship; the single ship's mass for ship↔world:

```kotlin
val effectiveMass =
    if (isWorldCollision) if (massA > 0) massA else massB
    else (massA * massB) / (massA + massB)          // guard: totalMass > ε
```

**Velocity decomposition** — importance-weighted RMS normal/tangential speeds, with weight `|separation| × speed` (deep *and* fast contacts dominate):

```kotlin
for (contact in zone.contacts) {
    val v = Vector3d(contact.velocity)
    val speedSq = v.lengthSquared(); if (speedSq < EPSILON) continue
    var weight = abs(contact.separation) * sqrt(speedSq)
    if (weight < EPSILON) weight = sqrt(speedSq)

    val vDotN = v.dot(normal)
    weightedNormalVelocitySq    += vDotN * vDotN * weight
    weightedTangentialVelocitySq += max(0.0, speedSq - vDotN * vDotN) * weight
    totalWeight += weight
}
// fallback (all weights ≈ 0): project zone.averageVelocity onto the normal
val normalVelocity     = sqrt(weightedNormalVelocitySq / totalWeight)
val tangentialVelocity = sqrt(weightedTangentialVelocitySq / totalWeight)
```

**Energy accounting:**

```kotlin
val impulse       = effectiveMass * normalVelocity                        // J
val normalEnergy  = 0.5 * effectiveMass * normalVelocity.pow(2)
val shearEnergy   = 0.5 * effectiveMass * tangentialVelocity.pow(2) * 0.4 // FRICTION_FACTOR
val kineticEnergy = normalEnergy + shearEnergy
```

**Torque correction** (world contact point → ship-space lever arm about center of mass):

```kotlin
val r        = localZonePos - ship.inertiaData.centerOfMass
val leverArm = r cross localForceDir
val leverRatio    = |leverArm| / shipRadius        // shipRadius from shipAABB diagonal / 2
val torqueFactor  = leverRatio / (leverRatio + 1.0)
val damageEnergy  = kineticEnergy * (1 - torqueFactor * 0.3)   // SPIN_ENERGY_LOSS_FACTOR
```

`ImpactZoneResult` bundles `normalVelocity, tangentialVelocity, effectiveMass, impulse, normalEnergy, shearEnergy, kineticEnergy, damageEnergy, torqueFactor, impactDirection` for the injection stage.

> [!note] Tuning TODOs carried from source
> `SPIN_ENERGY_LOSS_FACTOR`, `FRICTION_FACTOR`, effective-mass model, and the energy terms are all annotated `TODO: check against real physics`. They're isolated in this one class — a single place to swap in a validated contact-mechanics model later.

## 3. Stress Injection — `BlockImpactInjector`

Turns a zone's `damageEnergy` into per-block `InjectedStress(compression, shear, direction)` in **shipyard space**.

1. Transform the zone centroid + impact direction into ship space (`worldToShip` → `transformPosition` / `transformDirection`), snap to a block seed.
2. `findNearestSolidBlock(seed, level, 3)` — shell-search outward up to radius 3 for the first non-air, in-shipyard block (miss ⇒ empty map + warn).
3. **Radius**: `r = clamp(√(area/π) + torqueFactor × 3.0, 2.0, 6.0)`.
4. **Weighting** per in-radius, non-air ship block at offset `d`:
   - `falloff = 1 − |d| / r` (linear cone)
   - `alignment = max(0, dir̂ · d̂)` — bias blocks *ahead* of the blow
   - `directionalWeight = 0.5 + 0.5 × alignment` — never zero-attacks the back side
   - `weight = falloff × directionalWeight`
5. **Exact energy distribution**, split into compression/shear by the zone's energy shares:

```kotlin
val compression = result.damageEnergy * (weight / totalWeight) * (normalEnergy / kineticEnergy)
val shear       = result.damageEnergy * (weight / totalWeight) * (shearEnergy  / kineticEnergy)
```

`injectWorld` is the mirror for terrain voxels (world-space seed, `!isBlockInShipyard` filter, no ship transform).

`InjectedStress.add(...)` merges multiple zone contributions onto one block with **weight-averaged direction** and additive compression/shear.

## 4. Propagation — `StressPropagator`

Damage shouldn't stop at the surface shell; stress spreads into the structure to crack internal supports:

```kotlin
private const val MAX_DEPTH = 3
private const val MIN_STRESS_THRESHOLD = 10.0
private const val MAX_TOUGHNESS = 100.0
```

Layered BFS, at most 3 hops from the injected blocks:

- A block spreads only if its `totalStress = √(compression² + shear²) ≥ 10.0`.
- **Transfer factor** scales with source material toughness: `min(0.5, toughness/100 × 0.5)` — steel conducts stress further than dirt.
- **Directional gating**: neighbors receive stress only if the hop direction has positive dot with the weighted-average stress direction (`alignment = max(0, n̂·d̂)`; ⊥/backward ⇒ skip). No stress flows back toward the impact.
- Contributions accumulate into the *next* layer only (already-processed layers are final; seeded blocks never receive propagation) — wavefront semantics, no revisits, so a 3-depth pass is bounded by `O(blocks × 6 × depth)`.
- Propagation is confined by the same `validBlock` predicate: ship propagation never leaks into the world and vice versa.

Output: `Map<BlockPos, BlockStressState>` with `compression`, `shear`, normalized average `direction`, and derived `totalStress` — the input to failure resolution.

## 5. Material Model — `CollisionMaterialRegistry` + `CbcToughnessHelper`

```kotlin
record CollisionMaterialProps(double yieldThreshold, double ultimateThreshold)
```

Resolution order per block:

1. Static `OVERRIDES` map (datapack/reload-driven, via `registerOverride(id, yield, ultimate)`).
2. Indestructible vanilla blocks (`getDestroySpeed < 0`, e.g. bedrock) → `(MAX_VALUE, MAX_VALUE)`.
3. Otherwise:
   - `yield = miningHardness × 0.4`
   - `ultimate = CBC toughness × 1.0` where CBC = Create Big Cannons' `BlockArmorPropertiesHandler` (soft-loaded; absent CBC ⇒ `state.getExplosionResistance(...)` as toughness, `1.0` hardness).

This deliberately bridges two eras of Minecraft combat mods: vanilla scraps break on `hardness`, CBC-era armor plate breaks on `toughness`.

## 6. Failure Resolution — `BlockDamageResolver`

Full model including constants and regimes is tabulated in the
[failure model](block-mutation-and-destruction.md#4-failure-model-how-a-block-decides-to-break).
The elastic/plastic/failure state machine, Von Mises scalar, fatigue feedback
into effective thresholds, crack overlay stages, and block destruction on
ultimate are all game-thread work.

## 7. Data Class Glossary

| Type | Shape | Mutable? | Producer → Consumer |
|---|---|---|---|
| `ContactPoint` | world-space pos/normal/separation/velocity (JOML c-interfaces) | by phys engine | physics → handler |
| `ImpactZone` | sums + finalized aggregates + `contacts` list | internally | clusterer → calculator/injector |
| `ImpactZoneResult` | energy/impulse/torque/direction result | no (fields final) | calculator → injector |
| `InjectedStress` | compression, shear, weighted dir | yes (`add`) | injector → accumulator/merger |
| `BlockStressState` | compression, shear, dir, totalStress | no | accumulator/propagator → resolver |
| `CollisionMaterialProps` | yield, ultimate | record | registry → resolver |
| `ShipDamageData` | `Map<BlockPos, Float>` fatigue | yes | attachment, resolver-owned |
