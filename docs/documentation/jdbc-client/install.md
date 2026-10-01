# Installation

Choose the dependency that matches your runtime. The core module is intended for Java SE applications, while the CDI and
configuration integrations are for Jakarta EE-compatible runtimes.

## Requirements

- JDK 21 or later.
- A Jakarta EE 11 / CDI-compatible runtime is required for the `cdi` and `config` modules, such as GlassFish 8 or
  WildFly 41.
- The `core` module is usable in plain Java SE applications because it depends only on `javax.sql.DataSource`.

## Maven repository

FluDa publishes SNAPSHOT artifacts to GitHub Packages. Add the following repository to your `pom.xml`:

```xml
<repositories>
    <repository>
        <id>github-fludakit</id>
        <url>https://maven.pkg.github.com/fludakit/*</url>
        <releases><enabled>false</enabled></releases>
        <snapshots><enabled>true</enabled></snapshots>
    </repository>
</repositories>
```

Authenticate by adding a server entry in `~/.m2/settings.xml` with a [Personal Access Token](https://github.com/settings/tokens) that has `read:packages` scope:

```xml
<servers>
    <server>
        <id>github-fludakit</id>
        <username>YOUR_GITHUB_USERNAME</username>
        <password>YOUR_PERSONAL_ACCESS_TOKEN</password>
    </server>
</servers>
```

## Using the core `JdbcClient`

Use the following dependency when only the core fluent `JdbcClient` API is required:

```xml

<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-client-core</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

The core module has no runtime dependency beyond the JDK. It expects a `DataSource` supplied by the application or
runtime environment.

## Integrating with CDI

For a CDI-managed `JdbcClient`, add the CDI artifact:

```xml

<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-client-cdi</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

This artifact depends on the core module and is intended for a Jakarta EE 11 runtime that provides the Jakarta EE APIs.

## Integrating with MicroProfile Config

To configure the library from MicroProfile Config properties, add the optional configuration module:

```xml

<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-client-config</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

All modules are published with the same version. Replace the snapshot version shown above with the release version used
by the application.

After adding the appropriate module, start using the fluent API for executing queries and updates.
See [quickstart](quickstart.md) for the shortest path to using the client,
and [querying and result mapping](querying.md) for more detailed behavior.
