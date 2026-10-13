# Attachments and Serialization

## What attachments are

An attachment is addon-owned state stored on an object that implements
`org.valkyrienskies.core.api.attachment.AttachmentHolder`. On the server side
that is a `LoadedServerShip`; attachments cannot be stored on an unloaded
`ServerShip`. Typical attachments are physics listeners (`ShipPhysicsListener`),
per-ship settings, controllers, and other mod state.

Required rules:

1. Attachment classes **must be final** (Kotlin `class`/`data class`, or Java
   `final class`).
2. Register the attachment before first use, during mod initialisation.
3. Lookup is by the **exact declared class**; inherited-class lookup never
   matches (e.g. `getAttachment(Object.class)` returns nothing).
4. The container's get/set operations are thread-safe, but that does not make a
   mutable attachment object internally thread-safe and does not override the
   thread annotation of its owner.
5. Persisted attachments must remain serializable across the save format you
   support; treat serializer changes as migrations.

## `AttachmentHolder`

Source: `org/valkyrienskies/core/api/attachment/AttachmentHolder.kt`. Verified
JVM signatures:

```java
<T extends Object> T getAttachment(Class<T> clazz);
<T extends Object> T getOrPutAttachment(Class<T> clazz, Supplier<T> supplier);
<T extends Object> T setAttachment(T value);
<T extends Object> T removeAttachment(Class<T> clazz);
Iterable<Object> getAllAttachments();
```

* `getAttachment` returns `null` when absent **or** when deserialization failed.
* `getOrPutAttachment` constructs via the supplier only when absent, and returns
  the stored value.
* `setAttachment` returns the previously attached instance, or `null`. Setting
  an attachment whose deserialization failed throws.
* `removeAttachment` returns the previously attached instance, or `null`.
* `getAllAttachments` returns every successfully loaded attachment.

### Kotlin reified helpers

The file declares `@file:JvmSynthetic`, so these are Kotlin-only. Java callers
must use the class-based methods above.

```kotlin
import org.valkyrienskies.core.api.attachment.getAttachment
import org.valkyrienskies.core.api.attachment.getOrPutAttachment
import org.valkyrienskies.core.api.attachment.removeAttachment

val state: MyAttachment? = ship.getAttachment<MyAttachment>()
val created: MyAttachment = ship.getOrPutAttachment { MyAttachment() }
val removed: MyAttachment? = ship.removeAttachment<MyAttachment>()
```

`getOrPutAttachment` accepts a `java.util.function.Supplier`, so the Kotlin
trailing lambda works and Java can pass a method reference.

### Legacy setter

`LoadedServerShip` keeps a deprecated `setAttachment(Class<T>, T?)` overload
(and a deprecated Kotlin `LoadedServerShip.setAttachment(value)` extension) for
compatibility with pre-2.5 code. New code should call
`AttachmentHolder.setAttachment(value)` directly.

## Registration

Source: `org/valkyrienskies/core/api/attachment/AttachmentRegistration.kt`.
`AttachmentRegistration<T>` holds:

* `attachmentClass: Class<T>` — must be final;
* `key: String` — unique; defaults to the fully qualified class name;
* `serializer: AttachmentSerializer`.

`AttachmentRegistration.Builder<T>` exposes `key`/`serializer` properties plus
fluent setters, and:

```java
Builder<T> useJacksonSerializer();     // default persisted serializer
Builder<T> useLegacySerializer();      // old format, compatibility only
Builder<T> useTransientSerializer();   // never persisted
Builder<T> serializer(AttachmentSerializer serializer);
Builder<T> key(String key);
<NEW> Builder<T> useUpdater(Class<NEW> newAttachmentClass, Function1<T, NEW> updater);
AttachmentRegistration<T> build();
```

Register through the API:

```java
// Java
var registration = api.newAttachmentRegistrationBuilder(MyAttachment.class)
        .key("example.mod.MyAttachment:2")
        .useJacksonSerializer()
        .build();
api.registerAttachment(registration);
```

```kotlin
// Kotlin DSL
api.registerAttachment(MyAttachment::class.java) {
    key = "example.mod.MyAttachment:2"
    useJacksonSerializer()
}
```

Key guidance: use a stable explicit key when the attachment is part of a
long-lived save format. When you change the schema and want a clean break,
append a version, e.g. `example.mod.MyAttachment:2`. Keys default to the fully
qualified class name, so renaming or moving a class silently changes its key —
set an explicit key before shipping.

## Serializer types

| Serializer | Use |
| --- | --- |
| `JacksonAttachmentSerializer` | Current persisted serializer; serialises public getters and creators by default. Use for new persisted data. |
| `LegacyAttachmentSerializer` | Compatibility with the old attachment format. Source directs new projects to Jackson. |
| `TransientAttachmentSerializer` | Never persisted. Runtime-only state disappears on reload. |
| `BinaryAttachmentSerializer<T>` | Custom binary serialization; see below. |

`AttachmentSerializer` is `@NonExtendable`, so you cannot implement it directly.
For custom binary data, implement `BinaryAttachmentSerializer<T>`:

```java
public interface BinaryAttachmentSerializer<T> extends AttachmentSerializer {
    void serialize(T attachment, DataOutput output);
    T deserialize(DataInput input);   // null => treated as deserialization failure
}
```

To delete an attachment on the next load, use
`api.registerAttachmentForRemoval(key)` rather than returning `null` from
`deserialize`.

## Migration with updaters

`useUpdater(newClass, updater)` registers a migration. On load, VS2
recursively applies registered updaters until none remain. The updater receives
a deserialized instance of the old attachment and returns an instance of the new
class; the old attachment is removed and replaced. If the updater returns
`null`, returns a non-attachment, or throws, the old attachment is **not**
replaced.

Keep migration code deterministic and versioned. Do not silently replace
malformed state with success-shaped defaults.

## Threading

Attachment get/set is thread-safe, but this is a property of the container, not
of the attachment value. A mutable attachment read or written from both the game
and physics threads is still a race. Follow the owning object's thread
annotation (`LoadedServerShip` is `@GameTickOnly`) and hand off immutable
snapshots or synchronised state.

## Examples

### A per-ship physics attachment

```java
public final class ThrusterAttachment implements ShipPhysicsListener {
    private volatile double throttle;   // written on the game thread, read on physics

    public void setThrottle(double value) { this.throttle = value; }

    @Override
    public void physTick(PhysShip ship, PhysLevel level) {
        if (throttle <= 0) return;
        ship.applyModelForce(new org.joml.Vector3d(0, 0, 4000 * throttle),
                             new org.joml.Vector3d(0, 0, 0));
    }
}

// registration (once)
api.registerAttachment(ThrusterAttachment.class);

// usage (game thread)
LoadedServerShip ship = ...;
ThrusterAttachment thruster = ship.getOrPutAttachment(ThrusterAttachment.class, ThrusterAttachment::new);
thruster.setThrottle(0.5);
```

### A transient control attachment

```java
api.registerAttachment(MyControls.class, builder -> builder.useTransientSerializer());
MyControls controls = ship.getOrPutAttachment(MyControls.class, MyControls::new);
```
