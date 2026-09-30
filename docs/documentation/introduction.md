# Introduction

FluDa (**Flu**ent **Da**ta Toolkit for Jakarta EE/CDI) is a lightweight, modular data-access suite designed for standard Jakarta EE and CDI environments. It provides a developer-friendly, fluent programming model that mirrors modern data APIs like Spring's `JdbcClient` — without requiring Spring or introducing bulky dependencies.

## Core Pillars

FluDa is organized into three standalone modules:

### JDBC Client

A lightweight, type-safe, fluent query engine built directly over JDBC. The core module has zero framework dependencies — just JDBC and a `DataSource`.

- Fluent API with named and positional parameters
- POJO mapping with custom row mappers
- Generated key support
- CDI integration for dependency injection

[Get started with JDBC Client →](jdbc-client/index.md)

### SQL Initialization

Automates database schema management and data population on application startup. Perfect for development and testing scenarios where you need to set up database state automatically.

- Schema creation from SQL scripts
- Data population from SQL files
- CDI integration for automatic execution

[Get started with SQL Init →](sql-init/index.md)

### Transaction Support

Container-agnostic declarative and programmatic transaction boundaries backed directly by a standard `DataSource` or JPA. Works in Java SE, Servlet containers, and Jakarta EE environments.

- `@Transactional` interceptor for CDI beans
- Resource-local transactions without JTA
- Transaction propagation control
- Transaction-phase callbacks and events

[Get started with Transaction Support →](tx/index.md)
