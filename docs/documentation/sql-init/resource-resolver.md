# Customizing Resource Resolver

Script locations use a protocol prefix that maps to a `ResourceResolver`:

| Prefix          | Example                                    | Description                                    |
|-----------------|--------------------------------------------|------------------------------------------------|
| `classpath:`    | `classpath:db/migration`                   | Scans the classpath (directories and JARs).    |
| `filesystem:`   | `filesystem:/opt/app/migrations`           | Scans the operating system file system.        |
| `file:`         | `file:/opt/app/migrations`                 | Alias for `filesystem:`.                       |
| *(none)*        | `db/migration`                             | Defaults to `classpath:`.                      |

Ant-style patterns are supported: `classpath:db/**/*.sql`.

When multiple locations are configured, scripts are merged by filename — the first location wins on name collisions.

## Custom resource resolvers

To load migrations from other sources (e.g. AWS S3, HTTP, a database), implement `ResourceResolver` and register it with a `ResourceResolverRegistry`:

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

## Implementing a ResourceResolver

A `ResourceResolver` handles a single protocol and provides two methods:

```java
public interface ResourceResolver {

    String protocol();

    Resource getResource(String location);

    List<Resource> getResources(String locationPattern);
}
```

Each `Resource` provides an `InputStream`, filename, content length, URL, and existence check.

## S3 example

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
