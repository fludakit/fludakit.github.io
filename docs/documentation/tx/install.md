# Installation

## Prerequisites

- Java 21 or later
- A CDI container (Weld SE for Java SE, Weld Servlet for Servlet containers, or a full Jakarta EE server)
- A JDBC `DataSource` (typically a connection pool like HikariCP)

## Maven repository

FluDa publishes SNAPSHOT artifacts to GitHub Packages. Add the following repository to your `pom.xml`:

```xml
<repositories>
    <repository>
        <id>github-fludakit</id>
        <url>https://maven.pkg.github.com/fludakit/*</url>
        <releases><enabled>false</enabled></releases>
        <snapshots><enabled>true</enabled></snapshots>
    </repository>
</repositories>
```

Authenticate by adding a server entry in `~/.m2/settings.xml` with a [Personal Access Token](https://github.com/settings/tokens) that has `read:packages` scope:

```xml
<servers>
    <server>
        <id>github-fludakit</id>
        <username>YOUR_GITHUB_USERNAME</username>
        <password>YOUR_PERSONAL_ACCESS_TOKEN</password>
    </server>
</servers>
```

## Maven dependencies

For Java SE or Servlet environments, add the three tx modules:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-tx-core</artifactId>
    <version>${fluda.version}</version>
</dependency>
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-tx-cdi</artifactId>
    <version>${fluda.version}</version>
</dependency>
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-tx-jdbc</artifactId>
    <version>${fluda.version}</version>
</dependency>
```

You also need the JDBC Client modules:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-client-core</artifactId>
    <version>${fluda.version}</version>
</dependency>
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jdbc-client-cdi</artifactId>
    <version>${fluda.version}</version>
</dependency>
```

The tx modules declare the CDI API and `jakarta.transaction-api` as `provided`: supply a CDI container (Weld SE, or a Servlet container with Weld) plus `jakarta.transaction-api` for the `@Transactional` annotation.

## Jakarta EE environments

On a full Jakarta EE server JTA is built in, so `jakarta.transaction.Transactional` is handled by the container — **no `fluda-tx` dependency is needed**. Just use the JDBC Client modules and let the container manage transactions.

## Using the BOM

If you use the FluDa BOM, the JDBC Client module versions are managed automatically:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.github.fludakit</groupId>
            <artifactId>fluda-bom</artifactId>
            <version>${fluda.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

The tx modules are not yet included in the BOM — specify their version explicitly.
