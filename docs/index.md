# VS Collisions documentation

This documentation is for contributors and maintainers of VS Collisions. It
separates stable external API facts from explanations of this repository's
implementation.

## Start here

1. Read [threading and event handoffs](guides/threading-and-event-handoffs.md)
   before changing collision handling.
2. Consult the [VS2 and VS Core API reference](reference/vs2-core-api.md) for
   versioned API contracts, ownership, events, transforms, and attachments.
3. Follow the architecture pages for the current collision-to-block-mutation
   implementation.

## Reference

| Page | Purpose |
| --- | --- |
| [VS2 and VS Core API reference](reference/vs2-core-api.md) | Verified dependency reference for VS2 2.4.11 and VS Core 1.1.0+cf208d8b56. |

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

The reference page documents what resolved VS2/Core artifacts expose. Architecture
pages document this project and may discuss implementation choices or open work.
Examples should state their thread, coordinate space, and whether an API is
public or internal. Links use ordinary relative Markdown so they work on GitHub,
common repository browsers, and local editors.
