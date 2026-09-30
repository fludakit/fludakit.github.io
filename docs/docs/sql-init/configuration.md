# SQL Init Configuration

Configuration controls where migration scripts are loaded from, the statement separator, and the
database type used for the history table DDL.

## Programmatic configuration

The `core` module provides an immutable `SqlInitConfig` with a builder:

```java
SqlInitConfig config = SqlInitConfig.builder()
        .scriptLocations(List.of("classpath:db/migration"))
        .separator(";")
        .dbType("h2")
        .build();
```

| Property           | Default                      | Description                                                                                   |
|--------------------|------------------------------|-----------------------------------------------------------------------------------------------|
| `scriptLocations`  | `classpath:db/migration`     | One or more resource locations to scan for `V<version>__<description>.sql` files.             |
| `separator`        | `;`                          | The statement separator passed to the SQL script parser.                                      |
| `dbType`           | `null` (auto-detect)         | The database type for the `db_migrations` DDL. When `null`, detected from the JDBC metadata.  |

### Script locations

Locations use a protocol prefix:

| Prefix          | Example                                    | Description                                    |
|-----------------|--------------------------------------------|------------------------------------------------|
| `classpath:`    | `classpath:db/migration`                   | Scans the classpath (directories and JARs).    |
| `filesystem:`   | `filesystem:/opt/app/migrations`           | Scans the operating system file system.        |
| `file:`         | `file:/opt/app/migrations`                 | Alias for `filesystem:`.                       |
| *(none)*        | `db/migration`                             | Defaults to `classpath:`.                      |

Ant-style patterns are supported: `classpath:db/**/*.sql`.

When multiple locations are configured, scripts are merged by filename — the first location wins on
name collisions.

### Statement separator

The default separator is `;`. Scripts can override it inline with the `DELIMITER` directive, which is
how MySQL and MariaDB scripts wrap stored-procedure bodies:

```sql
DELIMITER //
CREATE PROCEDURE reset_counts()
BEGIN
    UPDATE counters SET value = 0;
END//
DELIMITER ;
```

### Database type

When `dbType` is `null` (the default), the migrator auto-detects the database from the JDBC connection
metadata. Set it explicitly when auto-detection is ambiguous or when running against an unsupported
database with compatible SQL:

| Value        | Aliases                                  |
|--------------|------------------------------------------|
| `h2`         |                                          |
| `postgresql` | `pg`, `postgres`                         |
| `mysql`      | `mariadb`                                |
| `mssql`      | `sqlserver`, `sql_server`                |
| `oracle`     |                                          |

## Declarative configuration in CDI

In a Jakarta EE / CDI environment, the optional `fluda-sql-init-config` module populates `SqlInitConfig`
from MicroProfile Config properties.

| Property                           | Default                      | Description                                                                    |
|------------------------------------|------------------------------|--------------------------------------------------------------------------------|
| `jdbcclient.init.script-locations` | `classpath:db/migration`     | Comma-separated list of script locations.                                      |
| `jdbcclient.init.separator`        | `;`                          | The statement separator.                                                       |
| `jdbcclient.init.db-type`          | *(empty — auto-detect)*      | The database type name. Empty means auto-detect from the JDBC connection.      |

Example `microprofile-config.properties`:

```properties
jdbcclient.init.script-locations=classpath:db/migration,classpath:db/extra
jdbcclient.init.separator=;
jdbcclient.init.db-type=postgresql
```
