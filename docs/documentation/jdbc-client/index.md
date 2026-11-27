# JDBC Client

A lightweight, type-safe, fluent query engine built directly over JDBC for Jakarta EE / CDI applications.

| | |
|---|---|
| **Documentation** | [fludakit.github.io/jdbc-client](https://fludakit.github.io/jdbc-client/) |
| **Source** | [github.com/fludakit/jdbc-client](https://github.com/fludakit/jdbc-client) |
| **Artifacts** | `fluda-jdbc-core`, `fluda-jdbc-config`, `fluda-jdbc-cdi` |

The reference documentation for this module lives in the module repository and is published with GitHub Pages.

## Quick example

```java
import io.github.fludakit.jdbc.JdbcClient;
import javax.sql.DataSource;

DataSource dataSource = ...; // obtain a DataSource from your application or runtime
JdbcClient client = new JdbcClient(dataSource);

List<Engineer> engineers = client.sql("SELECT id, name FROM engineers WHERE department = :dept")
        .param("dept", "Engineering")
        .query(Engineer.class)
        .list();
```

## Sections

- [Getting Started](https://fludakit.github.io/jdbc-client/getting-started) — add the dependency, run your first query, and review configuration options such as placeholders, timeouts, and defaults.
- [Querying and Result Mapping](https://fludakit.github.io/jdbc-client/querying) — named and positional parameters, POJO mapping, and custom row mappers.
- [Updates and Generated Keys](https://fludakit.github.io/jdbc-client/updates) — inserts, updates, batches, and generated key retrieval.
