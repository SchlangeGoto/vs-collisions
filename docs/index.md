# VS Collisions documentation

This documentation is for contributors and maintainers of VS Collisions. It
separates stable external API facts from explanations of this repository's
implementation.

## Start here

1. Read the [VS2 and VS Core API reference](vs2-api/README.md) — a standalone,
   version-scoped reference for everything Valkyrien Skies 2 and VS Core expose
   to addons. It is independent of this repository's implementation.
2. Read [threading and event handoffs](guides/threading-and-event-handoffs.md)
   before changing collision handling.
3. Follow the architecture pages for the current collision-to-block-mutation
   implementation.

## VS2 and VS Core API reference

The canonical VS2/VS Core reference lives in [`vs2-api/`](vs2-api/README.md).
It covers architecture and thread ownership, the Core and Mod API facades,
ship/body/transform/force APIs, events, attachments and serializers, physics and
collision, worlds/connectivity/augmentations, Minecraft integration, internal
APIs, a full API inventory with provenance, advanced facilities, and practical
Java/Kotlin examples.

| Page | Purpose |
| --- | --- |
| [VS2 and VS Core API reference](vs2-api/README.md) | Entry point and version scope. |
| [Architecture and thread ownership](vs2-api/01-architecture-and-threading.md) | Threads, ticks, and safe handoff. |
| [Core API and lifecycle](vs2-api/02-core-api-and-lifecycle.md) | Facades, initialization, ship lifecycle. |
| [Ships, bodies, transforms, and forces](vs2-api/03-ships-bodies-and-transforms.md) | Ship types, spaces, force/torque APIs. |
| [Events and callbacks](vs2-api/04-events-and-callbacks.md) | Every event and its thread. |
| [Attachments and serialization](vs2-api/05-attachments-and-serialization.md) | Addon state and serializers. |
| [Physics, shapes, contacts, and joints](vs2-api/06-physics-and-collision-api.md) | PhysLevel, contacts, ray casts, joints. |
| [Worlds, connectivity, and augmentations](vs2-api/07-worlds-connectivity-and-augmentations.md) | Ship worlds, connectivity, augmentations. |
| [Minecraft integration](vs2-api/08-minecraft-integration.md) | Mod API, VSGameUtils, config, datagen. |
| [Internal APIs and compatibility](vs2-api/09-internal-apis-and-compatibility.md) | Internal types and migration. |
| [API inventory and provenance](vs2-api/10-api-inventory.md) | Full inventory and how it was verified. |
| [Advanced integration facilities](vs2-api/11-advanced-integration-facilities.md) | Wings, drag, networking, assembly. |
| [Practical examples](vs2-api/12-examples.md) | Java/Kotlin examples. |
| [Additional API surface](vs2-api/13-additional-api-surface.md) | Utilities, impl, native Krunch, and other public types not covered above. |
| [Tooling and publishing](vs2-api/14-tooling-and-publishing.md) | Docs structure, Dokka build, GitHub Pages. |

## Guides

| Page | Purpose |
| --- | --- |
| [Threading and event handoffs](guides/threading-and-event-handoffs.md) | Safely transfer collision data from physics callbacks to server-tick work. |
| [Development workflow](guides/development-workflow.md) | Local prerequisites, build commands, and documentation expectations. |

## Architecture

| Page | Purpose |
| --- | --- |
| [Collision data flow](architecture/collision-data-flow.md) | Current route from a collision event to block mutation. |
| [Coordinate transforms and spatial math](architecture/coordinate-spaces.md) | World, model, and shipyard coordinates. |
| [Ship state and persistence](architecture/ship-attachments-and-persistence.md) | Attachments and the repository's per-ship state. |
| [Ship lifecycle and fracturing](architecture/ship-lifecycle-and-fracturing.md) | Ship lookup, lifecycle, and connectivity implications. |
| [Block mutation and destruction](architecture/block-mutation-and-destruction.md) | Updating shipyard blocks safely. |
| [Stress and material pipeline](architecture/stress-and-material-pipeline.md) | Repository-specific impact, stress, and material calculations. |

## Conventions

The `vs2-api/` reference documents what resolved VS2/Core artifacts expose.
Architecture pages document this project and may discuss implementation choices
or open work. Examples should state their thread, coordinate space, and whether
an API is public or internal. Links use ordinary relative Markdown so they work
on GitHub, common repository browsers, and local editors.
