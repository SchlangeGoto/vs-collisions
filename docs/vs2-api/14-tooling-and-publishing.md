# Tooling, Publishing, and Generated API Docs

This chapter explains how the reference is organised, how the Dokka API site is
built, and how it is published.

## Two layers

1. **Hand-written guides** (this directory, `README.md` … `13-*`) explain
   *concepts*: architecture, threads, transforms, lifecycle, pitfalls. They are
   short, reviewed by humans, and cover things a generator cannot (thread
   ownership, coordinate spaces, backend differences).
2. **A generated Dokka site** renders the actual VS2/VS Core sources into a
   browsable HTML API reference with KDoc, deprecations, and opt-in annotations.

This is the professional norm and follows the **Diátaxis** framework:

| Kind | Question it answers | Where it lives |
| --- | --- | --- |
| Tutorial | "Teach me by doing" | `12-examples.md` |
| How-to guide | "How do I accomplish X?" | `08`, `11`, `12` |
| Reference | "What exactly exists?" | `03`–`07`, `13`, generated Dokka site |
| Explanation | "Why is it designed this way?" | `01`, `02`, `09`, `14` |

Hand-writing the member reference goes stale the moment the dependency updates,
so it is generated instead.

## Why Dokka and not `javadoc`

Valkyrien Skies is **Kotlin**. The classic `javadoc` tool only understands
`.java` sources, so it cannot document the Kotlin API. **Dokka** is JetBrains'
Kotlin documentation engine; it reads KDoc, `@Deprecated(replaceWith = ...)`,
`@RequiresOptIn` (`@VsBeta`, `@PhysTickOnly`, `@GameTickOnly`), and nullability.

## The Dokka setup in this repository

Dokka is configured in [`build.gradle`](../../build.gradle) at the repository
root. It reuses the project's fully-resolved classpath (Minecraft, Forge, JOML,
and the VS artifacts), but documents the **VS2/VS Core sources** rather than
this mod's own code.

How it works:

1. A `vsApiSources` configuration resolves the `:sources` artifacts:

   ```groovy
   vsApiSources "org.valkyrienskies.core:api:${vs_core_version}:sources"
   vsApiSources "org.valkyrienskies.core:internal:${vs_core_version}:sources"
   vsApiSources "org.valkyrienskies:valkyrienskies-120-forge:${vs2_version}:sources"
   ```

2. `extractVsApiSources` unzips them into `build/vs-api-sources`, keeping only
   `org/valkyrienskies/**` (the VS source JARs also vendor JOML/OptiFine/etc.
   sources, which we do not want to document).

3. The `dokka { }` block adds that directory as a source root and filters
   packages:
   * documents `org.valkyrienskies.*`;
   * suppresses this repository's own `dev.kkazi.*`;
   * suppresses implementation noise (`mod.mixin`, `mod.mixinducks`,
     `mod.compat`, `mod.client`, `mod.event`, `mod.forge.mixin`,
     `mod.forge.compat`, `mod.platform`, `mod.util`).

4. The HTML publication is written to `docs/vs2-api/api`.

### Build it

```bash
./gradlew dokkaGeneratePublicationHtml
# -> docs/vs2-api/api/index.html
```

The output is **gitignored** (`.gitignore` → `docs/vs2-api/api/`); it is a build
artifact, not source.

### Scope and limitations

* Dokka can only document types that have sources. In this build the published
  source JARs are `api`, `internal`, and `valkyrienkies-120-forge`; the `util`
  and `impl` modules (and the native `physics_api_krunch` backend) have no
  source JAR, so they are covered in prose in
  [13-additional-api-surface.md](13-additional-api-surface.md).
* KDoc that contains broken `[links]` produces warnings, not failures
  (`failOnWarning` is off). The upstream sources have a few.

## Publishing to GitHub Pages

[`.github/workflows/docs.yml`](../../.github/workflows/docs.yml) builds the
Dokka site and deploys it to GitHub Pages on pushes that touch `docs/**` or the
build files (and on manual dispatch).

One-time setup: **Settings → Pages → Source: GitHub Actions**. The published
site is then served at:

```text
https://schlangegoto.github.io/vs-collisions/
```

## Adding the guides to the same site (optional)

The hand-written guides are plain Markdown and can be published alongside the
API reference with a site generator. A ready-to-edit
[`../mkdocs.yml`](../mkdocs.yml) is included; install and serve:

```bash
pip install mkdocs-material
mkdocs serve     # from the docs/ directory
mkdocs build
```

To combine both on Pages, build MkDocs into the Pages artifact root and copy the
Dokka output under `api/`, or keep them as two sites.

## Versioning

The reference is version-scoped to the artifacts pinned in `gradle.properties`
(`vs2_version`, `vs_core_version`). Keep one reference per dependency version
(for example this directory for VS2 2.4.11 / VS Core 1.1.0), or use the site
generator's versioning plugin (MkDocs `mike`, Docusaurus versioned docs). Never
mix claims from two versions on one page.

## Regeneration checklist for a VS2/Core bump

1. Update `vs2_version` / `vs_core_version` in `gradle.properties`.
2. Re-resolve the artifacts (`./gradlew --refresh-dependencies`).
3. Run `./gradlew dokkaGeneratePublicationHtml` and review the output.
4. Update the version table in [`README.md`](README.md) and
   [`10-api-inventory.md`](10-api-inventory.md).
5. Re-read [`09-internal-apis-and-compatibility.md`](09-internal-apis-and-compatibility.md)
   for internal casts and re-verify them.
6. Push; the Pages workflow republishes the site.

## Conventions

* **Generate, don't transcribe.** Member lists come from the sources.
* **One concept per page**, with a clear "Related" header.
* **State thread, coordinate space, and public/internal status** in every
  example and table.
* **Mark uncertainty explicitly** ("unspecified", "backend-dependent") instead
  of guessing.
* **Keep the provenance** (artifact coordinates, inspection method) so a reader
  can reproduce the reference.
