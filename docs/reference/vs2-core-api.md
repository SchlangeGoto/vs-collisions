# Valkyrien Skies 2 / VS Core API reference

> This page is a short pointer kept for backwards compatibility. The full,
> maintained VS2 and VS Core reference now lives in
> [`../vs2-api/`](../vs2-api/README.md).

The canonical reference is the [`vs2-api/`](../vs2-api/README.md) directory. It
covers architecture and thread ownership, the Core and Mod API facades, ships,
bodies, transforms and forces, events, attachments and serializers, physics and
collision, worlds/connectivity/augmentations, Minecraft integration, internal
APIs and compatibility, a full API inventory with provenance, advanced
facilities, and practical Java/Kotlin examples.

## Verified version scope

| Component | Resolved version |
| --- | --- |
| Minecraft / Forge | 1.20.1 / 47.4.10 |
| Valkyrien Skies Forge | 2.4.11 (`org.valkyrienskies:valkyrienskies-120-forge`) |
| VS Core `api` / `internal` / `util` / `impl` | 1.1.0+cf208d8b56 (`org.valkyrienskies.core`) |
| Java target | 17 |

Names, signatures, annotations, and deprecations in the `vs2-api/` reference
were verified against the resolved source JARs and bytecode. See
[`../vs2-api/10-api-inventory.md`](../vs2-api/10-api-inventory.md) for the exact
artifacts and inspection method.
