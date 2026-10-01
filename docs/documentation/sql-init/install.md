# SQL Init Installation

Choose the dependency that matches your runtime. The `core` module is sufficient for programmatic use;
the `cdi` module adds automatic startup migration in a Jakarta EE / CDI environment.

## Requirements

- JDK 21 or later.
- A Jakarta EE 11 / CDI-compatible runtime is required for the `cdi` and `config` modules.
- The `core` module is usable in plain Java SE applications because it depends only on `javax.sql.DataSource`.

## Using the core migrator

For programmatic control over migration execution:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-sql-init-core</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

## Automatic startup in CDI

To apply migrations automatically when the application starts, add the CDI module:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-sql-init-cdi</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

This pulls in `core` transitively. The CDI module observes the container's `Startup` event and runs all
pending migrations before the application begins serving requests.

## MicroProfile Config integration

To configure script locations, separator, or database type from application properties, add the optional
config module:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-sql-init-config</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

See [configuration](configuration.md) for the available properties.
