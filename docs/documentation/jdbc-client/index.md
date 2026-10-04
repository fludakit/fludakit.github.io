# FluDa: Fluent Data Toolkit for Jakarta EE/CDI

FluDa is a lightweight, framework-agnostic JDBC client for Jakarta EE and CDI applications. It provides the
fluent developer experience of Spring's `JdbcClient` without introducing a Spring dependency.

The project is organized into three core modules:

| Module   | Artifact                  | Description                                                                                                |
|----------|---------------------------|------------------------------------------------------------------------------------------------------------|
| `core`   | `fluda-jdbc-client-core`  | Contains the plain `JdbcClient` API and helper types. It depends only on the JDK's `javax.sql.DataSource`. |
| `config` | `fluda-jdbc-client-config`| Integrates with MicroProfile Config and exposes a configurable `JdbcConfig` bean.                          |
| `cdi`    | `fluda-jdbc-client-cdi`   | Provides CDI producers for `JdbcClient` and the CDI-discovered `ConverterRegistry`.                        |

The repository also includes Arquillian integration tests for GlassFish and WildFly. These examples provide a practical
reference for validating Jakarta EE/CDI applications in real runtimes.

Follow this reading order:

- Start with [getting started](getting-started.md) to add the dependency, run your first query, and review configuration options such as placeholders, timeouts, and defaults.
- Explore [querying and result mapping](querying.md) and [updates](updates.md) for detailed behavior.
