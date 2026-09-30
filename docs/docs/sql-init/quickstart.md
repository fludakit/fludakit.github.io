# SQL Init Quickstart

The fastest path to running your first migration.

## Migration scripts

Place versioned SQL scripts in `src/main/resources/db/migration/`. Each file must follow the naming
convention `V<version>__<description>.sql`:

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

## Programmatic usage

With only `fluda-sql-init-core` on the classpath, create a `DbMigrator` and call `migrate()`:

```java
import io.github.fludakit.sqlinit.DbMigrator;

import javax.sql.DataSource;

DataSource dataSource = ...; // obtain a DataSource

new DbMigrator(dataSource).migrate();
```

This resolves scripts from the default location (`classpath:db/migration`), detects the database type
from the connection, creates the `db_migrations` history table if needed, and applies all pending
migrations.

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

See [configuration](configuration.md) for all options.

## Automatic startup in CDI

Add `fluda-sql-init-cdi` and ensure a `DataSource` bean is available. The `SqlInitBootstrapper` observes
the `Startup` event and runs migrations automatically — no code required.

```java
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

When the application has multiple `DataSource` beans, use the `@SqlInit` qualifier to mark the one
that migrations should run against:

```java
@Produces
@ApplicationScoped
@SqlInit
public DataSource migrationDataSource() {
    return dataSource;
}
```

When no `@SqlInit`-qualified `DataSource` exists, the default unqualified bean is used.
