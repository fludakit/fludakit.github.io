# SQL Init Configuration

Configuration controls where migration scripts are loaded from, the statement separator, and the
database type used for the history table DDL.

## Programmatic configuration

The `core` module provides an immutable `SqlInitConfig` with a builder:

```java
SqlInitConfig config = SqlInitConfig.builder()
        .scriptLocations(List.of("classpath:db/migration"))
        .separator(";")
        .dbType("h2")
        .build();
```

| Property           | Default                      | Description                                                                                   |
|--------------------|------------------------------|-----------------------------------------------------------------------------------------------|
| `scriptLocations`  | `classpath:db/migration`     | One or more resource locations to scan for `V<version>__<description>.sql` files.             |
| `separator`        | `;`                          | The statement separator passed to the SQL script parser.                                      |
| `dbType`           | `null` (auto-detect)         | The database type for the `db_migrations` DDL. When `null`, detected from the JDBC metadata.  |
| `versionStrategy`  | `IntegerVersionStrategy`     | The strategy for parsing and ordering migration versions.                                     |

### Script locations

Locations use a protocol prefix:

| Prefix          | Example                                    | Description                                    |
|-----------------|--------------------------------------------|------------------------------------------------|
| `classpath:`    | `classpath:db/migration`                   | Scans the classpath (directories and JARs).    |
| `filesystem:`   | `filesystem:/opt/app/migrations`           | Scans the operating system file system.        |
| `file:`         | `file:/opt/app/migrations`                 | Alias for `filesystem:`.                       |
| *(none)*        | `db/migration`                             | Defaults to `classpath:`.                      |

Ant-style patterns are supported: `classpath:db/**/*.sql`.

When multiple locations are configured, scripts are merged by filename — the first location wins on
name collisions.

### Statement separator

The default separator is `;`. Scripts can override it inline with the `DELIMITER` directive, which is
how MySQL and MariaDB scripts wrap stored-procedure bodies:

```sql
DELIMITER //
CREATE PROCEDURE reset_counts()
BEGIN
    UPDATE counters SET value = 0;
END//
DELIMITER ;
```

### Database type

When `dbType` is `null` (the default), the migrator auto-detects the database from the JDBC connection
metadata. Set it explicitly when auto-detection is ambiguous or when running against an unsupported
database with compatible SQL:

| Value        | Aliases                                  |
|--------------|------------------------------------------|
| `h2`         |                                          |
| `postgresql` | `pg`, `postgres`                         |
| `mysql`      | `mariadb`                                |
| `mssql`      | `sqlserver`, `sql_server`                |
| `oracle`     |                                          |

## Declarative configuration in CDI

In a Jakarta EE / CDI environment, the optional `fluda-sql-init-config` module populates `SqlInitConfig`
from MicroProfile Config properties.

| Property                           | Default                      | Description                                                                    |
|------------------------------------|------------------------------|--------------------------------------------------------------------------------|
| `jdbcclient.init.script-locations` | `classpath:db/migration`     | Comma-separated list of script locations.                                      |
| `jdbcclient.init.separator`        | `;`                          | The statement separator.                                                       |
| `jdbcclient.init.db-type`          | *(empty — auto-detect)*      | The database type name. Empty means auto-detect from the JDBC connection.      |

Example `microprofile-config.properties`:

```properties
jdbcclient.init.script-locations=classpath:db/migration,classpath:db/extra
jdbcclient.init.separator=;
jdbcclient.init.db-type=postgresql
```

## Version strategy

The version strategy controls how migration versions are parsed from filenames and how they are ordered. The default is `IntegerVersionStrategy` which expects simple numeric versions (`V1`, `V2`, `V3`).

### Built-in strategies

| Strategy | Example filenames | Ordering |
|----------|-------------------|----------|
| `IntegerVersionStrategy` | `V1__init.sql`, `V2__add_users.sql` | Numeric: 1, 2, 3, ... |
| `DottedVersionStrategy` | `V1.0__init.sql`, `V1.1__fix.sql` | Dotted: 1.0, 1.1, 1.2, 2.0 |
| `SemanticVersionStrategy` | `V1.0.0__init.sql`, `V1.1.0__fix.sql` | SemVer: 1.0.0, 1.1.0, 2.0.0 |

Use a custom strategy via the builder:

```java
SqlInitConfig config = SqlInitConfig.builder()
        .scriptLocations(List.of("classpath:db/migration"))
        .versionStrategy(new SemanticVersionStrategy())
        .build();
```

### Custom version strategy

Implement `VersionStrategy` for specialized versioning schemes. For example, date-based versions (`V20240101`, `V20241231`):

```java
public class DateVersionStrategy implements VersionStrategy {

    private static final DateTimeFormatter FORMAT = DateTimeFormatter.BASIC_ISO_DATE;

    @Override
    public String parse(String version) {
        LocalDate date = LocalDate.parse(version, FORMAT);
        return date.format(FORMAT);
    }

    @Override
    public int compare(String v1, String v2) {
        return v1.compareTo(v2);
    }
}
```

## Custom resource resolvers

The built-in resolvers handle `classpath:`, `file:`, and `filesystem:` locations. To load migrations from other sources (e.g. AWS S3, HTTP, a database), implement `ResourceResolver` and register it with a `ResourceResolverRegistry`:

```java
ResourceResolverRegistry registry = new ResourceResolverRegistry();
registry.register("s3", new S3ResourceResolver(s3Client));

DbMigrator migrator = new DbMigrator(dataSource, config, registry);
migrator.migrate();
```

The location prefix in `scriptLocations` must match the registered protocol:

```java
SqlInitConfig config = SqlInitConfig.builder()
        .scriptLocations(List.of("s3://my-bucket/migrations/"))
        .build();
```

### Implementing a ResourceResolver

A `ResourceResolver` handles a single protocol and provides two methods:

```java
public interface ResourceResolver {

    String protocol();

    Resource getResource(String location);

    List<Resource> getResources(String locationPattern);
}
```

Each `Resource` provides an `InputStream`, filename, content length, URL, and existence check.

### S3 example

The following resolver loads SQL scripts from AWS S3 using the `s3://` protocol:

```java
public class S3ResourceResolver implements ResourceResolver {

    private final S3Client s3Client;

    public S3ResourceResolver(S3Client s3Client) {
        this.s3Client = s3Client;
    }

    @Override
    public String protocol() {
        return "s3";
    }

    @Override
    public Resource getResource(String location) {
        // parse s3://bucket/key → bucket + key
        return new S3Resource(bucket, key);
    }

    @Override
    public List<Resource> getResources(String locationPattern) {
        // list objects under s3://bucket/prefix, filter .sql files
        ListObjectsV2Response response = s3Client.listObjectsV2(
            ListObjectsV2Request.builder()
                .bucket(bucket).prefix(prefix).build());
        // map each S3Object to an S3Resource
    }
}
```

The `S3Resource` inner class implements `Resource` by delegating to `GetObjectRequest` for content and `HeadObjectRequest` for metadata.
