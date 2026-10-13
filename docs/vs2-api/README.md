# Valkyrien Skies 2 and VS Core API reference

This directory is a standalone, version-scoped reference for the **Valkyrien
Skies 2 (VS2)** Minecraft integration and **Valkyrien Skies Core (VS Core)**.
It exists because there is no official, complete API manual, and it is kept
independent from this repository's own collision implementation.

Everything in this directory is written from the actual artifacts resolved by
this project: the Gradle dependency cache, the published source JARs, and the
compiled bytecode. Where a statement could not be verified, it is marked as
unverified or omitted. See [10-api-inventory.md](10-api-inventory.md) for the
exact artifacts and how they were inspected.

> This reference documents VS2 and VS Core. It does **not** document this
> repository's damage, stress, or collision-energy code. Those are application
> behaviour and live in the [`../architecture`](../architecture) pages.

## Verified version scope

| Component | Version |
| --- | --- |
| Minecraft | 1.20.1 |
| Forge | 47.4.10 |
| Valkyrien Skies 2 Forge integration | 2.4.11 (`org.valkyrienskies:valkyrienskies-120-forge`) |
| VS Core `api` / `internal` / `util` / `impl` | 1.1.0+cf208d8b56 (`org.valkyrienskies.core`) |
| Java target | 17 |
| Kotlin (project build) | 2.0.0 (Kotlin for Forge 4.11.0 at runtime) |

The API is primarily Kotlin. Java callers see Kotlin properties as
`getX()`/`setX()` methods, Kotlin default arguments as synthetic `$default`
overloads, and top-level functions as `*Kt` static methods unless the file
declares a `@JvmName` (for example, `ValkyrienSkies.kt` is exposed as the Java
class `ValkyrienSkies`). When calling from Java, always prefer the explicit
overloads; do not call the generated `$default` methods.

## Reading order

1. [Architecture and thread ownership](01-architecture-and-threading.md)
2. [Core API and lifecycle](02-core-api-and-lifecycle.md)
3. [Ships, bodies, transforms, and forces](03-ships-bodies-and-transforms.md)
4. [Events and callbacks](04-events-and-callbacks.md)
5. [Attachments and serialization](05-attachments-and-serialization.md)
6. [Physics, shapes, contacts, and joints](06-physics-and-collision-api.md)
7. [Worlds, connectivity, and augmentations](07-worlds-connectivity-and-augmentations.md)
8. [Minecraft integration](08-minecraft-integration.md)
9. [Internal APIs and compatibility](09-internal-apis-and-compatibility.md)
10. [API inventory and provenance](10-api-inventory.md)
11. [Advanced integration facilities](11-advanced-integration-facilities.md)
12. [Practical examples](12-examples.md)
13. [Additional API surface](13-additional-api-surface.md)
14. [Tooling, publishing, and generated API docs](14-tooling-and-publishing.md)

The API is also published as a generated **Dokka** site. It is not committed
(the output is gitignored); build it locally or read the published version:

* Build locally: `./gradlew dokkaGeneratePublicationHtml`, then open
  `docs/vs2-api/api/index.html`.
* Published reference: <https://schlangegoto.github.io/vs-collisions/>

See [14-tooling-and-publishing.md](14-tooling-and-publishing.md) for details.

## How to depend on VS2 and VS Core

An addon normally declares the Minecraft-facing artifact as a mod dependency and
the Core artifacts as ordinary Java/Kotlin libraries:

```groovy
dependencies {
    modApi("org.valkyrienskies:valkyrienskies-120-forge:${vs2_version}") { transitive = false }
    implementation("org.valkyrienskies.core:api:${vs_core_version}") { transitive = false }
    // Optional: only if you use internal/implementation types.
    implementation("org.valkyrienskies.core:internal:${vs_core_version}") { transitive = false }
}
```

At runtime, obtain the API from the integration. Prefer the public facade:

```java
import org.valkyrienskies.mod.api.ValkyrienSkies;
import org.valkyrienskies.mod.api.VsApi;

VsApi api = ValkyrienSkies.api();
```

`VsApi extends VsCoreApi`, so a single reference gives access to both the
platform-neutral Core API and the Minecraft helpers. `ValkyrienSkies.api()`
resolves `ValkyrienSkiesMod.getApi()` reflectively and throws
`IllegalStateException` if VS2 is not installed.

## API status terminology

* **Public**: intended addon-facing API, under `org.valkyrienskies.core.api.*`
  or `org.valkyrienskies.mod.api.*`. Core promises backward compatibility for
  `org.valkyrienskies.core.api.*` where it can.
* **Beta**: public but annotated `@VsBeta`
  (`org.valkyrienskies.core.api.VsBeta`). It is opt-in and may change or be
  removed without notice; production code should be prepared to catch a
  `LinkageError` or `NotImplementedError`.
* **Experimental**: annotated `org.jetbrains.annotations.ApiStatus.Experimental`
  or `@ApiStatus.Experimental` in Kotlin. Not covered by compatibility promises.
* **Deprecated**: retained for compatibility. Use the replacement named in the
  `@Deprecated(replaceWith = ...)` annotation; some are `DeprecationLevel.ERROR`
  (Kotlin will refuse to compile) or `@JvmSynthetic` (hidden from Java).
* **Internal**: implementation API under `org.valkyrienskies.core.internal.*` or
  `org.valkyrienskies.core.impl.*`. Usable, but not a compatibility promise.
  Isolate and pin it (see
  [09-internal-apis-and-compatibility.md](09-internal-apis-and-compatibility.md)).

## Package map at a glance

| Package | Responsibility |
| --- | --- |
| `org.valkyrienskies.core.api` | Facade (`VsCoreApi`), `@VsBeta`, exceptions |
| `org.valkyrienskies.core.api.event` | Event plumbing (`ListenableEvent`, `SingleEvent`, ...) |
| `org.valkyrienskies.core.api.events` | Concrete event interfaces (collision, tick, lifecycle, ...) |
| `org.valkyrienskies.core.api.attachment` | Addon-owned state and serializers |
| `org.valkyrienskies.core.api.bodies` | Generic bodies (`VsBody`, `PhysicsVsBody`, ...) |
| `org.valkyrienskies.core.api.bodies.properties` | IDs, transforms, kinematics, inertia, pose |
| `org.valkyrienskies.core.api.bodies.shape` | Body collision shape descriptors |
| `org.valkyrienskies.core.api.physics` | Contact points, ray-cast results, block-state shapes |
| `org.valkyrienskies.core.api.ships` | Ship interfaces, listeners, wings, drag, queries |
| `org.valkyrienskies.core.api.ships.properties` | Ship IDs, transforms, inertia, claims, active chunks |
| `org.valkyrienskies.core.api.world` | Ship worlds, physics levels, Y ranges |
| `org.valkyrienskies.core.api.world.connectivity` | Connectivity graph, statuses, augmentations |
| `org.valkyrienskies.core.api.util` | Thread annotations, identifiers, aerodynamics, helpers |
| `org.valkyrienskies.mod.api` | Minecraft-facing facade (`VsApi`, `ValkyrienSkies`), listeners, events |
| `org.valkyrienskies.mod.common` | VS2 common implementation and `VSGameUtils` extensions |
| `org.valkyrienskies.core.internal.*` / `org.valkyrienskies.core.impl.*` | Internal Core implementation |
