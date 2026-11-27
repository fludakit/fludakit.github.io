# Data

A Jakarta Data implementation — `@Repository` interfaces with query-by-method-name, `@Find`, and `@Query`, generated at compile time and executed through `JdbcClient`.

| | |
|---|---|
| **Documentation** | [fludakit.github.io/data](https://fludakit.github.io/data/) |
| **Source** | [github.com/fludakit/data](https://github.com/fludakit/data) |
| **Artifacts** | `fluda-data-core`, `fluda-data-qbn`, `fluda-data-processor` |

The reference documentation for this module lives in the module repository and is published with GitHub Pages.

## Quick example

```java
import io.github.fludakit.data.annotation.Entity;
import io.github.fludakit.data.annotation.Identity;

@Entity
public class Task {
    @Identity Long id;
    String title;
    Status status;
}
```

## Sections

- [Getting Started](https://fludakit.github.io/data/getting-started) — add the processor, map an entity, declare a repository, and use the lifecycle annotations and query-by-method-name.
- [Applying DDD](https://fludakit.github.io/data/ddd) — aggregate roots, value objects, and associations.
- [Optimistic Locking](https://fludakit.github.io/data/version) — the versioning roadmap.
- [Domain Events](https://fludakit.github.io/data/domain-events) — the domain event roadmap.
