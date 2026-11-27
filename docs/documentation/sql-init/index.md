# SQL Init

Automates database schema management and data population on application startup.

| | |
|---|---|
| **Documentation** | [fludakit.github.io/sql-init](https://fludakit.github.io/sql-init/) |
| **Source** | [github.com/fludakit/sql-init](https://github.com/fludakit/sql-init) |
| **Artifacts** | `fluda-sql-init-core`, `fluda-sql-init-config`, `fluda-sql-init-cdi` |

The reference documentation for this module lives in the module repository and is published with GitHub Pages.

## Quick example

```java
import io.github.fludakit.sqlinit.DbMigrator;

import javax.sql.DataSource;

DataSource dataSource = ...; // obtain a DataSource

new DbMigrator(dataSource).migrate();
```

## Sections

- [Getting Started](https://fludakit.github.io/sql-init/getting-started) — add the dependency, run your first migration, and review configuration (script locations, separator, and database type).
- [Version Strategy](https://fludakit.github.io/sql-init/version-strategy) — how migration versions are parsed and ordered.
- [Resource Resolver](https://fludakit.github.io/sql-init/resource-resolver) — how script locations are resolved.
