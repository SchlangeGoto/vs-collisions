# Ship state and persistence

Related: [Documentation index](../index.md) · [Collision data flow](collision-data-flow.md) · [VS2/Core API reference](../reference/vs2-core-api.md)

---

## 1. What an Attachment Is

VS2 ships implement `AttachmentHolder`, a typed key-value container bound to the ship object. It exists so that addons (like this mod) can hang custom per-ship state — health, stress fields, fatigue — on a ship without touching VS2 internals.

Verified contract (`org.valkyrienskies.core.api.attachment.AttachmentHolder`):

```kotlin
@VsBeta
interface AttachmentHolder {
    fun <T : Any> getAttachment(clazz: Class<T>): T?
    fun <T : Any> getOrPutAttachment(clazz: Class<T>, supplier: Supplier<T>): T
    fun <T : Any> setAttachment(value: T): T?      // returns previous instance
    fun <T : Any> removeAttachment(clazz: Class<T>): T?
    fun getAllAttachments(): Iterable<Any>
}

// Kotlin reified sugar shipped in the same file:
inline fun <reified T : Any> AttachmentHolder.getAttachment(): T?
inline fun <reified T : Any> AttachmentHolder.getOrPutAttachment(supplier: Supplier<T>): T
inline fun <reified T : Any> AttachmentHolder.removeAttachment(): T?
```

Hard rules from the API docs:

1. **Attachment classes must be `final`** (Kotlin classes are final by default — do not mark them `open`).
2. **Registration is mandatory before first use**, during mod initialization like any other registry.
3. Lookup keys are the **exact declared class**, no inheritance matching — `getAttachment<Any>()` never returns anything.
4. *"Setting and retrieving attachments is thread-safe."* — the map handle is; your object's internals are your own responsibility.

## 2. Registration

The Java entrypoint registers `ShipDamageData` with a **transient serializer** (in-memory only, never written to disk):

```kotlin
import org.valkyrienskies.mod.common.ValkyrienSkiesMod
import net.minecraftforge.fml.event.lifecycle.FMLCommonSetupEvent

class VSCollisions {
    init {
        MOD_BUS.addListener(::init)
    }

    private fun init(event: FMLCommonSetupEvent) {
        event.enqueueWork {
            val api = ValkyrienSkiesMod.getApi()
            api.registerAttachment(
                api.newAttachmentRegistrationBuilder(ShipDamageData::class.java)
                    .useTransientSerializer()   // not persisted to disk
                    .build()
            )
        }
        CollisionEventHandler.register()
    }
}
```

`AttachmentRegistration.Builder` serializer options (verified in `AttachmentRegistration.kt`):

| Builder call | Persistence | Use for |
|---|---|---|
| `useTransientSerializer()` | Never serialized | Pure runtime state (current choice) |
| `useJacksonSerializer()` | Jackson; public getters/creators by default | State that must survive world save/load (e.g., persistent ship HP) |
| `useLegacySerializer()` | Old VS2 format | Compatibility only — avoid for new code |
| `serializer(mySerializer)` | Custom `AttachmentSerializer` | Full control (e.g., binary) |

The **registration key** defaults to the fully-qualified class name. If you version the schema, append a suffix (`"dev.kkazi.vscollisions.damage.ShipDamageData:2"`); you can schedule the old key for cleanup via `api.registerAttachmentForRemoval(oldKey)`.

> [!warning] One registration per `Class`
> Attachments are keyed by their **declared class**, so a ship may hold *one* `ShipDamageData` instance. Design the attachment as an aggregate (a map of all your per-ship state), not one attachment per datum.

## 3. The Runtime Attachment: `ShipDamageData`

Mirrors `damage/ShipDamageData.java`. `final` as required; internal map is plain and game-thread-confined:

```kotlin
import net.minecraft.core.BlockPos
import java.util.HashMap

/** Accumulated fatigue per block. 0.0 = pristine, 1.0 = about to fail. */
final class ShipDamageData {

    private val accumulated = HashMap<BlockPos, Float>()

    fun getAccumulated(pos: BlockPos): Float = accumulated.getOrDefault(pos, 0.0f)

    fun setAccumulated(pos: BlockPos, value: Float) {
        if (value <= 0.0f) accumulated.remove(pos)
        else accumulated[pos] = minOf(1.0f, value)
    }

    fun remove(pos: BlockPos) = accumulated.remove(pos)

    /**
     * Drop entries for blocks that no longer exist — call periodically
     * (e.g., once per N ticks per loaded ship) to prevent a slow memory leak.
     */
    fun prune(existingBlocks: Set<BlockPos>) {
        accumulated.keys.retainAll(existingBlocks)
    }
}
```

## 4. Retrieval Pattern (Game Thread)

`BlockDamageResolver` receives the attachment from `DamageApplicator.getOrCreateDamageData`. Idiomatic Kotlin using the reified extension:

```kotlin
import org.valkyrienskies.core.api.attachment.getAttachment
import org.valkyrienskies.core.api.attachment.getOrPutAttachment
import org.valkyrienskies.core.api.ships.LoadedServerShip
import java.util.function.Supplier

fun attach(ship: LoadedServerShip?): ShipDamageData? =
    ship?.getOrPutAttachment(Supplier { ShipDamageData() })

fun existingOnly(ship: LoadedServerShip?): ShipDamageData? =
    ship?.getAttachment<ShipDamageData>()   // null until first damage event
```

Note the semantics:

- `getOrPutAttachment` **creates** the attachment on first contact damage — no data on untouched ships, zero memory cost in steady state.
- `getAttachment` is read-only-null — use it for queries that must not mutate (e.g., a HUD render of "has this ship ever been hit?").

## 5. What Belongs Where

| State | Storage | Rationale |
|---|---|---|
| Per-block fatigue | `ShipDamageData` attachment (currently transient) | ship-scoped, lazily created |
| Transient hit staging | `DamageApplicator.worldDamageData` HashMap keyed by dimension id | world blocks have no ship to attach to |
| Persistent ship HP (future) | attachment with `useJacksonSerializer()` | survives restart |
| Structural stress (current tick) | local computation only (`Map<BlockPos, BlockStressState>`) | derived per collision, must not persist |

> [!warning] Serialization gotchas
> - With the **transient** serializer, all fatigue is lost on server restart. If fatigue should persist (it arguably should — a ship that limped into port shouldn't repair by relog), switch to `useJacksonSerializer()` and give `ShipDamageData` Jackson-visible getters (VS2 ships `BlockPos` Jackson serializers internally — see `ValkyrienSkiesMod.init` registering `BlockPosSerializer`/`BlockPosKeySerializer`).
> - `setAttachment` on a failed-deserialization attachment slot throws. If you change Jackson schema, bump the key.
> - `prune()` is currently never called — see
>   [housekeeping](block-mutation-and-destruction.md#7-housekeeping) for where
>   to wire it.

## 6. Thread Safety of Attachments — Precisely

From the `AttachmentHolder` javadoc: *"Setting and retrieving attachments is thread-safe."* This means the holder's internal concurrent map is synchronized. It does **not** make `ShipDamageData.accumulated` thread-safe.

```kotlin
// ❌ FORBIDDEN on the physics thread: mutating attachment contents
@OptIn(PhysTickOnly::class)
api.collisionPersistEvent.on { ev ->
    val ship: LoadedServerShip = ???   // you can't even GET this here legally
    ship.getOrPutAttachment(Supplier { ShipDamageData() })
        .setAccumulated(pos, 0.5f)       // races with game-thread resolver
}

// ✅ CORRECT: only ever touch contents on the game thread
@OptIn(GameTickOnly::class)
fun onServerTick(event: TickEvent.ServerTickEvent) {
    ...
    val data = getOrCreateDamageData(shipA)  // game thread
    evaluate(propagatedA, shipA, level, data)
}
```

`LoadedServerShip` itself is `@GameTickOnly` (verified — the interface is annotated), which is the compiler's way of enforcing this for you.
