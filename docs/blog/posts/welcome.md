---
date: 2026-09-30
categories:
  - Announcement
---

# Introducing FluDa: Fluent Data Toolkit for Jakarta EE

We're excited to announce **FluDa** — a lightweight, modular data-access toolkit designed specifically for standard Jakarta EE and CDI environments.

<!-- more -->

## The problem

Modern Java applications often reach for Spring's `JdbcClient` or `JdbcTemplate` for fluent database access. But what if you're building on standard Jakarta EE without Spring? You're left with raw JDBC — verbose, error-prone, and lacking the developer experience that modern APIs provide.

## What FluDa offers

FluDa fills this gap with three standalone pillars:

1. **JDBC Client** — A fluent, type-safe query engine built directly over JDBC. No framework dependencies in the core module.

2. **SQL Initialization** *(coming soon)* — Automated schema management and data population on startup.

3. **Transaction Support** *(coming soon)* — Declarative and programmatic transaction boundaries that work with any Jakarta EE server.

## Get started

The JDBC Client is available now. Check out the [quickstart guide](../../docs/jdbc-client/quickstart.md) to get up and running in minutes.

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-client-core</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

We'd love your feedback — head over to [GitHub](https://github.com/fludakit) to explore the source, report issues, or contribute.
