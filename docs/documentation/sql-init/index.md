# SQL Initialization

Automated database schema management and data population on application startup.

FluDa SQL Init resolves versioned SQL scripts from the classpath or file system, tracks which migrations
have been applied in a `db_migrations` history table, and runs pending scripts in order at startup.

## Modules

| Module   | Artifact                 | Description                                                                                   |
|----------|--------------------------|-----------------------------------------------------------------------------------------------|
| `core`   | `fluda-sql-init-core`    | Plain SQL script parser and migration runner. Depends only on `javax.sql.DataSource`.         |
| `config` | `fluda-sql-init-config`  | MicroProfile Config integration — populates `SqlInitConfig` from `fluda.sqlinit.*` properties. |
| `cdi`    | `fluda-sql-init-cdi`     | CDI bean that observes the `@Startup` event and applies pending migrations automatically.     |

## How it works

1. Scripts are resolved from the configured locations (default: `classpath:db/migration`).
2. Each file must follow the naming convention `V<version>__<description>.sql` (for example, `V1__create_engineers.sql`).
3. On startup, the migrator compares resolved scripts against the `db_migrations` history table.
4. Pending migrations are applied in version order, each in its own transaction.
5. A migration is marked `succeeded` or `failed` in the history table.

## Supported databases

The history table DDL is provided for the following databases:

| Database   | Auto-detected from JDBC product name or URL        |
|------------|----------------------------------------------------|
| H2         | `H2`                                               |
| PostgreSQL | `PostgreSQL`, `postgres`                           |
| MySQL      | `MariaDB`, `MySQL`                                 |
| MSSQL      | `Microsoft SQL Server`, `sqlserver`                |
| Oracle     | `Oracle`                                           |

The database type is auto-detected from the connection metadata. It can also be set explicitly through
[`SqlInitConfig`](getting-started.md) or the `fluda.sqlinit.db-type` configuration property.

## Reading order

- [Getting Started](getting-started.md) — add the right dependency, run your first migration, and review configuration (script locations, separator, and database type).
- [Advanced Topics](advanced.md) — custom version strategies and resource resolvers.
