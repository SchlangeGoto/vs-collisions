# Development workflow

Related: [Documentation index](../index.md) · [VS2/Core API reference](../reference/vs2-core-api.md)

## Prerequisites

The build is configured for Java 17, Minecraft 1.20.1, Forge 47.4.10, VS2
2.4.11, and VS Core 1.1.0+cf208d8b56. The authoritative version properties are
in the repository root's gradle.properties.

Use the Gradle wrapper supplied by the repository:

~~~bash
./gradlew build
~~~

For a narrower Java compilation check:

~~~bash
./gradlew compileJava
~~~

The first run resolves plugins and dependencies from the repositories declared
in build.gradle. Do not change VS2/Core versions without refreshing the API
reference against the newly resolved source JARs and bytecode.

## Documentation changes

Use [docs/index.md](../index.md) as the navigation entry point.

* Put verified external contracts in reference.
* Put contributor procedures in guides.
* Put repository-specific mechanics and decisions in architecture.
* Use relative Markdown links, not editor-specific wiki links.
* State the relevant thread and coordinate space in examples that touch VS2.
* Label Core internal and mod.common APIs, and avoid presenting them as stable.

Before committing documentation, check links, code fences, and whitespace:

~~~bash
git diff --check
rg -n '\[\[' docs
~~~

The second command should return no results; double-bracket links depend on
specific wiki editors and do not work in normal repository renderers.
