# Transaction Support

Container-agnostic declarative and programmatic transaction boundaries backed directly by a standard `DataSource` or JPA.

| | |
|---|---|
| **Documentation** | [fludakit.github.io/tx](https://fludakit.github.io/tx/) |
| **Source** | [github.com/fludakit/tx](https://github.com/fludakit/tx) |
| **Artifacts** | `fluda-tx-core`, `fluda-tx-cdi`, `fluda-tx-jdbc`, `fluda-tx-jpa` |

The reference documentation for this module lives in the module repository and is published with GitHub Pages.

## Quick example

```java
import io.github.fludakit.tx.PlatformTransactionManager;
import io.github.fludakit.tx.jdbc.DataSourceTransactionManager;
import io.github.fludakit.tx.jdbc.TransactionAwareDataSourceProxy;

import javax.sql.DataSource;

DataSource pool = ...; // your raw pool
TransactionAwareDataSourceProxy dataSource = new TransactionAwareDataSourceProxy(pool);
PlatformTransactionManager transactionManager = new DataSourceTransactionManager(pool);
```

## Sections

- [Getting Started](https://fludakit.github.io/tx/getting-started) — add dependencies and set up resource-local transactions in a CDI environment.
- [JDBC Implementation](https://fludakit.github.io/tx/jdbc) — resource-local transactions over a plain `DataSource`.
- [JPA Implementation](https://fludakit.github.io/tx/jpa) — transactions driven by an `EntityManager`.
- [Mixing JDBC and JPA](https://fludakit.github.io/tx/advanced) — multiple data sources and sharing one transaction between JPA and JDBC.
