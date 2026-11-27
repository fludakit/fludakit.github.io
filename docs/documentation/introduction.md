# Introduction

FluDa (**Flu**ent **Da**ta Toolkit for Jakarta EE/CDI) is a lightweight, modular data-access suite designed for standard Jakarta EE and CDI environments. It provides a developer-friendly, fluent programming model that mirrors modern data APIs like Spring's `JdbcClient` — without requiring Spring or introducing bulky dependencies.

FluDa is organized into seven standalone modules. Each module ships its own reference documentation, published from the module repository:
## JDBC Client

A lightweight, type-safe, fluent query engine built directly over JDBC. The core module has zero framework dependencies — just JDBC and a `DataSource`.

- Fluent API with named and positional parameters
- POJO mapping with custom row mappers
- Generated key support
- CDI integration for dependency injection

[Get started with JDBC Client →](https://fludakit.github.io/jdbc-client/getting-started/)
## SQL Initialization

Automates database schema management and data population on application startup. Perfect for development and testing scenarios where you need to set up database state automatically.

- Schema creation from SQL scripts
- Data population from SQL files
- CDI integration for automatic execution

[Get started with SQL Init →](https://fludakit.github.io/sql-init/getting-started/)
## Transaction Support

Container-agnostic declarative and programmatic transaction boundaries backed directly by a standard `DataSource` or JPA. Works in Java SE, Servlet containers, and Jakarta EE environments.

- `@Transactional` interceptor for CDI beans
- Resource-local transactions without JTA
- Transaction propagation control
- Transaction-phase callbacks and events

[Get started with Transaction Support →](https://fludakit.github.io/tx/getting-started/)
## Mail

A simple mail abstraction with built-in Jakarta Mail (SMTP) support and pluggable providers for third-party services like SendGrid.

- `MailSender` interface with `JakartaMailSender` implementation
- Fluent `MailBuilder` for composing messages
- Template support with FreeMarker and simple placeholder processing
- CDI integration with MicroProfile Config properties
- Pluggable providers — implement `MailSender` for any email service

[Get started with Mail →](https://fludakit.github.io/mail/getting-started/)
## Data

A Jakarta Data implementation for Jakarta EE and CDI. You declare a `@Repository` interface, and an annotation processor generates the implementation at compile time — no runtime proxies, no classpath scanning.

- Lifecycle annotations (`@Insert`, `@Update`, `@Save`, `@Delete`)
- Built-in repository methods, `@Find` / `@By` / `@OrderBy`, and `@Query`
- Optional query-by-method-name parser
- Generated code executes SQL through the FluDa JDBC Client

[Get started with Data →](https://fludakit.github.io/data/getting-started/)
## JMS Client

A fluent Jakarta Messaging client in the style of Spring's `JmsClient`, with CDI wiring and MicroProfile Config for Java SE and Jakarta EE.

- `JmsClient` with fluent send, receive, receive-selected and browse operations
- `MessageConverter` SPI — `SimpleMessageConverter` by default, optional JSON-B
- `CachingConnectionFactory` reuses one `JMSContext` per thread
- CDI producers that resolve a `JMSContext`, a `ConnectionFactory` or a configured Artemis/Qpid section

[Get started with JMS Client →](https://fludakit.github.io/jms-client/getting-started/)
## Conditional Beans

A portable CDI extension that enables a bean **only when a condition matches the current deployment**.

- `@Conditional` with repeatable support and stereotype composition
- `Condition` and `ConditionContext` for deployment-time decisions
- Built-in `@ConditionalOnBean`, `@ConditionalOnClass`, `@ConditionalOnProperty` and their missing counterparts
- Two runtimes — CDI Full (portable extension) and CDI Lite (build compatible extension)

[Get started with Conditional Beans →](https://fludakit.github.io/conditional-beans/getting-started/)
