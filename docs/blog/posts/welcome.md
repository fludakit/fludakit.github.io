---
date: 2026-11-19
authors:
  - hantsy
categories:
  - Announcement
  - Getting Started
---

# Introducing Fluda JdbcClient 1.0.0.Beta1

As I mentioned in my LinkedIn posts, the original [JdbcClient CDI project](https://github.com/hantsy/jdbc-client-cdi) has moved to the new [FluDaKit](https://github.com/fludakit) community and become an umbrella project with several subprojects. I have made a number of small updates to [jdbc-client](https://github.com/fludakit/jdbc-client) over the past few weeks. Today I'm excited to announce that **JdbcClient 1.0.0.Beta1** is now available from the public Maven repository. If we find no major issues, this will be the only preview before **1.0.0** final.

JdbcClient is a small, fluent JDBC client for Java 21+, Jakarta EE, and CDI applications. It gives you the query and update API you may know from Spring's `JdbcClient` or `JdbcTemplate`, without adding a Spring dependency to a standard Jakarta EE project. The core module is built directly on `javax.sql.DataSource` and has no runtime framework dependency.

<!-- more -->

## Why another JDBC client?

Plain JDBC is powerful, but everyday data access often involves a lot of ceremony:

- Open a `Connection`.
- Prepare a statement.
- Bind parameters by index.
- Map a `ResultSet` by hand.
- Close every resource correctly.

Frameworks solved this years ago. But if your application is built on Jakarta EE and CDI, adding a full Spring stack just for a better JDBC API is a poor trade-off. JPA is a good fit for rich domain models, but it can feel heavy for simple SQL.

JdbcClient is a lightweight alternative that keeps the focus on JDBC:

- **Fluent SQL API** — `client.sql(...)` guides you through parameters, queries, and updates.
- **Named and positional parameters** — use `:name` or `?`, with configurable placeholder rewriting for different databases.
- **Typed result mapping** — map rows directly to records and POJOs, or supply a `RowMapper`.
- **Generated key support** — retrieve database-generated keys with a `GeneratedKeyHolder`.
- **Batch updates** — send groups of changes as maps or arrays of parameters.
- **CDI integration** — add `fluda-jdbc-cdi` and inject a ready-to-use `JdbcClient`.
- **No framework lock-in** — you provide the `DataSource`; JdbcClient uses it through standard JDBC.

## Getting started

Make sure you have Java 21+ and Maven 3.9+ installed, then create a new Maven project from IDE or Maven archetype.

```bash 
mvn archetype:generate \
-DgroupId=com.example \
-DartifactId=jdbc-client-demo \
-DarchetypeArtifactId=maven-archetype-quickstart \
-DinteractiveMode=false
```

Open your `pom.xml`, and add the core artifact and a JDBC driver, such as H2:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-core</artifactId>
    <version>1.0.0.Beta1</version>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.5.252</version>
</dependency>
```

> [!NOTE]
> We'll cover the `fluda-jdbc-cdi` module, which provides CDI integration, later in this post. The optional `fluda-jdbc-config` module reads settings from MicroProfile Config. All JdbcClient modules share the same version.

## Using JdbcClient

To create a `JdbcClient`, provide a `DataSource` and pass it to the constructor:

```java
import io.github.fludakit.jdbc.JdbcClient;
...
var ds = Utils.newDataSource(); // an in-memory H2 DataSource in the example
JdbcClient client = new JdbcClient(ds);
```

Then use this jdbc client to execute SQL statements. The following example shows how to insert a row and retrieve the generated key, then query all rows and map them to a record:

```java
import io.github.fludakit.jdbc.support.GeneratedKeyHolder;
import io.github.fludakit.jdbc.support.KeyHolder;

import java.util.List;

public record Engineer(Long id, String name) {
}

// Insert and retrieve the generated id.
KeyHolder holder = new GeneratedKeyHolder();
client.sql("INSERT INTO engineers (name) VALUES (:name)")
        .param("name", "Ada")
        .update(holder);

long adaId = ((Number) holder.getKey()).longValue();

// Map rows directly to a record.
List<Engineer> all = client
        .sql("SELECT id, name FROM engineers ORDER BY id")
        .query(Engineer.class)
        .list();
```

Similar to the previous example, use the same client for reads, updates, and deletes:

```java
Engineer ada = client.sql("SELECT id, name FROM engineers WHERE id = :id")
        .param("id", adaId)
        .query(Engineer.class)
        .single();

client.sql("UPDATE engineers SET name = :name WHERE id = :id")
        .param("name", "Ada Lovelace")
        .param("id", adaId)
        .update();

client.sql("DELETE FROM engineers WHERE id = :id")
        .param("id", adaId)
        .update();
```

That is the whole model: a `JdbcClient` instance, a fluent specification, and standard JDBC under the covers.

This example is adapted from [`VanillaExample`](https://github.com/fludakit/examples/blob/main/vanilla/src/main/java/com/example/VanillaExample.java) in the Fludakit examples repository.

To customize client-wide defaults, create the client with the builder:

```java
JdbcClient client = JdbcClient.builder(dataSource)
        .placeholder("?")
        .queryTimeout(30)
        .fetchSize(100)
        .build();
```

## Using JdbcClient with CDI

Add the `fluda-jdbc-cdi` module to your `pom.xml`:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-cdi</artifactId>
    <version>1.0.0.Beta1</version>
</dependency>
```

In a CDI environment, this module supplies the `JdbcClient` producer. Expose a `DataSource` bean, then inject the client where you need it:

```java
import io.github.fludakit.jdbc.JdbcClient;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class EngineerService {

    @Inject
    JdbcClient client;

    public Engineer findById(Long id) {
        return client.sql("SELECT id, name FROM engineers WHERE id = :id")
                .param("id", id)
                .query(Engineer.class)
                .single();
    }
}
```

Its [`DataSourceProducer`](https://github.com/fludakit/examples/blob/main/javase/src/main/java/com/example/DataSourceProducer.java) creates a HikariCP pool and exposes it to CDI:

```java
@ApplicationScoped
public class DataSourceProducer {

    private final HikariDataSource raw;

    public DataSourceProducer() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:javase;DB_CLOSE_DELAY=-1");
        config.setUsername("sa");
        config.setPassword("");
        config.setMaximumPoolSize(10);
        this.raw = new HikariDataSource(config);
    }

    @Produces
    @Singleton
    public DataSource dataSource() {
        return raw;
    }
}
```

The service code stays the same whether the `DataSource` comes from a local pool, a servlet container, or a Jakarta EE server.

The [`javase` example](https://github.com/fludakit/examples/tree/main/javase) uses this pattern and starts CDI with Weld SE. 

## Providing a DataSource in Tomcat

The Tomcat container context manages the connection pools globally. Add a `META-INF/context.xml`, declare a `javax.sql.DataSource` resource, which can be accessed via JNDI:

```xml
<Context>
    <Resource name="jdbc/myDS"
              auth="Container"
              type="javax.sql.DataSource"
              driverClassName="org.mariadb.jdbc.Driver"
              url="jdbc:mariadb://localhost:3306/servlet"
              username="root"
              password="root"
              maxTotal="8"
              maxIdle="4"/>
</Context>
```

The following `DataSourceProducer` obtains the resource with `@Resource` and republishes it as a CDI bean:

```java
@ApplicationScoped
public class DataSourceProducer {

    @Resource(name = "jdbc/myDS")
    private DataSource dataSource;

    @Produces
    @Singleton
    public DataSource expose() {
        return dataSource;
    }
}
```

After that, the rest of the application is ordinary JdbcClient code. See the example's `EngineerService` and `EngineerServlet` for a compact CRUD flow.

The [`servlet` example](https://github.com/fludakit/examples/tree/main/servlet) runs on Tomcat 11 with Weld Servlet.

## Providing a DataSource in Jakarta EE

In Jakarta EE, you can create a custom `DataSource` with the `@DataSourceDefinition` annotation. The following example declares a PostgreSQL data source under a portable JNDI name, injects it with `@Resource`, and exposes it as a CDI bean:

```java
import javax.sql.DataSource;
import jakarta.annotation.Resource;
import jakarta.annotation.sql.DataSourceDefinition;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

@DataSourceDefinition(
        name = "java:app/MyDS",
        className = "org.postgresql.xa.PGXADataSource",
        url = "jdbc:postgresql://localhost:5432/testdb",
        user = "testuser",
        password = "testpassword"
)
@ApplicationScoped
public class DataSourceProducer {

    @Resource(lookup = "java:app/MyDS")
    private DataSource dataSource;

    @Produces
    @ApplicationScoped
    public DataSource expose() {
        return dataSource;
    }
}
```

The [`jakartaee` example](https://github.com/fludakit/examples/tree/main/jakartaee) targets GlassFish and WildFly. A real deployment can replace the annotation-defined data source with one configured by the application server. The injection point remains `@Inject JdbcClient client;`, and JdbcClient works with the container-managed connection pool.

Because this is a full Jakarta EE runtime, container-provided `@Transactional` is available for transaction boundaries. The example does not require an additional FluDa transaction dependency.

For more use cases—generated keys, batch updates, result mapping options, converters, streaming, and configuration—see the [reference documentation](https://fludakit.github.io/documentation/jdbc-client/getting-started/).

## Try the examples

The complete runnable examples are in the [`fludakit/examples`](https://github.com/fludakit/examples) repository:

- [`vanilla`](https://github.com/fludakit/examples/tree/main/vanilla) — the core client with no CDI, backed by an in-memory H2 database.
- [`javase`](https://github.com/fludakit/examples/tree/main/javase) — CDI with Weld SE and a HikariCP pool.
- [`servlet`](https://github.com/fludakit/examples/tree/main/servlet) — Tomcat 11, Weld Servlet, and a JNDI `DataSource`.
- [`jakartaee`](https://github.com/fludakit/examples/tree/main/jakartaee) — JAX-RS on GlassFish or WildFly with a container-defined `DataSource`.

## What's next

`1.0.0.Beta1` is our invitation to try JdbcClient before the final release. If you find a rough edge, have an API suggestion, or want to share how you provide a `DataSource` in your environment, open an issue on [GitHub](https://github.com/fludakit/jdbc-client). Your feedback will help shape `1.0.0` final.
