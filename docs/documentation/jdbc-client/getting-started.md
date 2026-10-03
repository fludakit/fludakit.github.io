# Getting Started

This guide covers adding FluDa JDBC Client to your project and executing your first queries, whether in a plain Java SE application or a Jakarta EE/CDI environment.

## Requirements

- JDK 21 or later.
- A CDI-compatible runtime is required for the `cdi` and `config` modules, such as GlassFish 8, WildFly 41, or Weld SE.
- The `core` module is usable in plain Java SE applications because it depends only on `javax.sql.DataSource`.

## Adding dependencies

Choose the dependency that matches your runtime. The core module is intended for Java SE applications, while the CDI and configuration integrations are for Jakarta EE-compatible runtimes.

### Using the core `JdbcClient`

Use the following dependency when only the core fluent `JdbcClient` API is required:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-client-core</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

The core module has no runtime dependency beyond the JDK. It expects a `DataSource` supplied by the application or runtime environment.

### Integrating with CDI

For a CDI-managed `JdbcClient`, add the CDI artifact:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-client-cdi</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

This artifact depends on the core module and is intended for a CDI environment.

### Integrating with MicroProfile Config

The `fluda-jdbc-client-config` module is an optional addon to the CDI module that provides property-based configuration. It reads configuration properties from MicroProfile Config and produces a `JdbcConfig` bean, which the CDI module uses to configure the `JdbcClient`.

Add this dependency when you want to configure the client via properties:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-client-config</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

This module depends on the CDI module and is intended for a CDI environment with MicroProfile Config support.

All modules are published with the same version. Replace the snapshot version shown above with the release version used by your application.

#### Configuration properties

When using the `config` module, the following properties can be set in `META-INF/microprofile-config.properties`:

| Property                   | Default | Description                                                       |
|----------------------------|--------:|-------------------------------------------------------------------|
| `jdbcclient.placeholder`   |     `?` | Placeholder used when rewriting named parameters such as `:name`. |
| `jdbcclient.query-timeout` |     `0` | Default query timeout in seconds. `0` means no timeout.           |
| `jdbcclient.fetch-size`    |     `0` | Default fetch-size hint. `0` uses the driver default.             |

##### Placeholder rewriting

Named parameters are rewritten according to the configured placeholder:

| Placeholder | Rewritten SQL   | Typical database        |
|-------------|-----------------|-------------------------|
| `?`         | `?`             | MySQL and standard JDBC |
| `$`         | `$1`, `$2`, ... | H2/PostgreSQL           |
| `:`         | `:1`, `:2`, ... | Oracle                  |
| `@`         | `@name`         | SQL Server              |

When `jdbcclient.placeholder=$`, the following SQL:

```sql
SELECT id FROM engineers WHERE id = :id AND active = :active
```

is rewritten before it is sent to the driver as:

```sql
SELECT id FROM engineers WHERE id = $1 AND active = $2
```

## Creating a `JdbcClient`

### In a Java SE application

After adding `fluda-jdbc-client-core` to your project, create a `JdbcClient` directly from a `DataSource`:

```java
import io.github.fludakit.jdbc.JdbcClient;

import javax.sql.DataSource;

DataSource dataSource = ...; // obtain a DataSource from the application or runtime

JdbcClient client = new JdbcClient(dataSource);
```

The builder API also supports explicit configuration:

```java
JdbcClient client = JdbcClient.builder(dataSource)
        .placeholder("?")
        .queryTimeout(30)
        .fetchSize(100)
        .converters(registry)
        .build();
```

### In a CDI environment

In a Jakarta EE/CDI environment, add `fluda-jdbc-client-cdi` and expose a `DataSource` as a CDI bean:

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

The `JdbcClient` is then available for injection:

```java
import io.github.fludakit.jdbc.JdbcClient;
import jakarta.inject.Inject;

@Inject
JdbcClient client;
```

## Executing queries

Once the client is available, use the fluent SQL API for both reading and writing data. The library supports both named and positional parameters in a single SQL statement.

### Named parameters

Use named parameters with `param(...)`:

```java
List<DevSummary> devs = client
        .sql("SELECT id, dev_name FROM engineers WHERE id = :id")
        .param("id", 1L)
        .query(DevSummary.class)
        .list();
```

### Positional parameters

The client also accepts positional parameters:

```java
List<DevSummary> devs = client
        .sql("SELECT id, dev_name FROM engineers WHERE id = ?")
        .param(1L)
        .query(DevSummary.class)
        .list();
```

A single SQL specification cannot combine named and positional parameters.

### Overriding configuration for a query

The `queryTimeout(...)` and `fetchSize(...)` methods on an individual SQL specification override the client defaults for that statement.

```java
List<DevSummary> devs = client
        .sql("SELECT id, dev_name FROM engineers")
        .fetchSize(100)
        .queryTimeout(30)
        .query(DevSummary.class)
        .list();
```

In practice, these defaults are useful for keeping database behavior consistent across an application while still allowing particular queries to opt into stricter or more permissive settings when needed.

## Next steps

See [querying and mapping](querying.md) for more examples of result mapping, row mappers, and streaming. Explore [updates and generated keys](updates.md) for INSERT, UPDATE, and DELETE operations.
