# Getting Started

This guide covers adding FluDa SQL Init to your project and running your first database migration, whether programmatically in a Java SE application or automatically in a Jakarta EE/CDI environment.

## Requirements

- JDK 21 or later.
- A CDI-compatible runtime is required for the `cdi` and `config` modules, such as GlassFish 8, WildFly 41, or Weld SE.
- The `core` module is usable in plain Java SE applications because it depends only on `javax.sql.DataSource`.

## Adding dependencies

Choose the dependency that matches your runtime. The `core` module is sufficient for programmatic use; the `cdi` module adds automatic startup migration in a Jakarta EE/CDI environment.

### Using the core migrator

For programmatic control over migration execution:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-sql-init-core</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### Automatic startup in CDI

To apply migrations automatically when the application starts, add the CDI module:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-sql-init-cdi</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

This pulls in `core` transitively. The CDI module observes the container's `Startup` event and runs all pending migrations before the application begins serving requests.

### MicroProfile Config integration

The `fluda-sql-init-config` module is an optional addon to the CDI module that provides property-based configuration. It reads configuration properties from MicroProfile Config and produces a `SqlInitConfig` bean.

Add this dependency when you want to configure script locations, separator, or database type from application properties:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-sql-init-config</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

This module depends on the CDI module and is intended for a CDI environment with MicroProfile Config support.

All modules are published with the same version. Replace the snapshot version shown above with the release version used by your application.

#### Configuration properties

When using the `config` module, the following properties can be set in `META-INF/microprofile-config.properties`:

| Property                          | Default                  | Description                                                                                |
|-----------------------------------|--------------------------|--------------------------------------------------------------------------------------------|
| `fluda.sqlinit.script-locations`  | `classpath:db/migration` | Comma-separated list of script locations.                                                   |
| `fluda.sqlinit.separator`         | `;`                      | The statement separator.                                                                    |
| `fluda.sqlinit.db-type`           | *(empty — auto-detect)*  | The database type name. Empty means auto-detect from the JDBC connection.                   |
| `fluda.sqlinit.version-strategy`  | `integer`                | How migration versions are parsed and ordered. See [Version strategy](#version-strategy).   |

Example `microprofile-config.properties`:

```properties
fluda.sqlinit.script-locations=classpath:db/migration,classpath:db/extra
fluda.sqlinit.separator=;
fluda.sqlinit.db-type=postgresql
fluda.sqlinit.version-strategy=integer
```

For programmatic configuration in Java SE, use the `SqlInitConfig` builder as shown in the [custom configuration](#custom-configuration) section below.

#### Statement separator

The default separator is `;`. Scripts can override it inline with the `DELIMITER` directive, which is how MySQL and MariaDB scripts wrap stored-procedure bodies:

```sql
DELIMITER //
CREATE PROCEDURE reset_counts()
BEGIN
    UPDATE counters SET value = 0;
END//
DELIMITER ;
```

#### Database type

When `db-type` is empty (the default), the migrator auto-detects the database from the JDBC connection metadata. Set it explicitly when auto-detection is ambiguous or when running against an unsupported database with compatible SQL:

| Value        | Aliases                                  |
|--------------|------------------------------------------|
| `h2`         |                                          |
| `postgresql` | `pg`, `postgres`                         |
| `mysql`      | `mariadb`                                |
| `mssql`      | `sqlserver`, `sql_server`                |
| `oracle`     |                                          |

#### Version strategy

The `fluda.sqlinit.version-strategy` property controls how migration versions are parsed and ordered. The built-in values are matched case-insensitively:

| Value                    | Strategy                   | Version format                                                                       |
|--------------------------|----------------------------|--------------------------------------------------------------------------------------|
| `int`, `integer` *(default)* | `IntegerVersionStrategy` | `V1`, `V2`, `V3` — compared as integers.                                              |
| `dotted`                 | `DottedVersionStrategy`    | `V1.0`, `V1.2.3` — split on dots and compared component by component; missing components count as zero (`1` = `1.0` = `1.0.0`). |
| `semantic`, `semver`     | `SemanticVersionStrategy`  | `V1.2.3`, `V1.0.0-alpha` — `major.minor.patch[-prerelease]`; release versions rank above prereleases. |

For a custom strategy, set the property to the fully qualified class name of a `VersionStrategy` implementation. The class is loaded with `Class.forName(...)` and must provide a public no-argument constructor:

```properties
fluda.sqlinit.version-strategy=com.example.MyVersionStrategy
```

See [advanced topics](advanced.md) for writing a custom version strategy and resource resolvers.

## Creating migration scripts

Place versioned SQL scripts in `src/main/resources/db/migration/`. Each file must follow the naming convention `V<version>__<description>.sql`:

```
src/main/resources/
└── db/
    └── migration/
        ├── V1__create_engineers.sql
        └── V2__seed_engineers.sql
```

Example scripts:

```sql
-- V1__create_engineers.sql
CREATE TABLE engineers (
    id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    name VARCHAR(255) NOT NULL
);
```

```sql
-- V2__seed_engineers.sql
INSERT INTO engineers (name) VALUES ('Ada Lovelace');
INSERT INTO engineers (name) VALUES ('Alan Turing');
```

The version number determines execution order. Versions must be consecutive starting from 1, with no gaps.

## Running migrations programmatically

With only `fluda-sql-init-core` on the classpath, create a `DbMigrator` and call `migrate()`:

```java
import io.github.fludakit.sqlinit.DbMigrator;

import javax.sql.DataSource;

DataSource dataSource = ...; // obtain a DataSource

new DbMigrator(dataSource).migrate();
```

This resolves scripts from the default location (`classpath:db/migration`), detects the database type from the connection, creates the `db_migrations` history table if needed, and applies all pending migrations.

### Custom configuration

```java
import io.github.fludakit.sqlinit.DbMigrator;
import io.github.fludakit.sqlinit.SqlInitConfig;

SqlInitConfig config = SqlInitConfig.builder()
        .scriptLocations(List.of("classpath:db/migration", "classpath:db/extra"))
        .separator(";")
        .dbType("postgresql")
        .build();

new DbMigrator(dataSource, config).migrate();
```

## Automatic startup in CDI

Add `fluda-sql-init-cdi` and ensure a `DataSource` bean is available. The `SqlInitBootstrapper` observes the `Startup` event and runs migrations automatically — no code required.

```java
import javax.sql.DataSource;
import jakarta.annotation.Resource;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

@ApplicationScoped
public class DataSourceProducer {

    @Resource(lookup = "java:comp/MyDS")
    private DataSource dataSource;

    @Produces
    @ApplicationScoped
    public DataSource expose() {
        return dataSource;
    }
}
```

### Selecting a specific DataSource

When the application has multiple `DataSource` beans, use the `@SqlInit` qualifier to mark the one that migrations should run against:

```java
@Produces
@ApplicationScoped
@SqlInit
public DataSource migrationDataSource() {
    return dataSource;
}
```

When no `@SqlInit`-qualified `DataSource` exists, the default unqualified bean is used.

## Next steps

See [advanced topics](advanced.md) for custom version strategies, custom resource resolvers, and other advanced configuration options.
