# Version Strategy

The version strategy controls how migration versions are parsed from filenames and how they are ordered. The default is `IntegerVersionStrategy` which expects simple numeric versions (`V1`, `V2`, `V3`).

## Built-in strategies

| Strategy | Example filenames | Ordering |
|----------|-------------------|----------|
| `IntegerVersionStrategy` | `V1__init.sql`, `V2__add_users.sql` | Numeric: 1, 2, 3, ... |
| `DottedVersionStrategy` | `V1.0__init.sql`, `V1.1__fix.sql` | Dotted: 1.0, 1.1, 1.2, 2.0 |
| `SemanticVersionStrategy` | `V1.0.0__init.sql`, `V1.1.0__fix.sql` | SemVer: 1.0.0, 1.1.0, 2.0.0 |

Use a built-in strategy via the builder:

```java
SqlInitConfig config = SqlInitConfig.builder()
        .scriptLocations(List.of("classpath:db/migration"))
        .versionStrategy(SemanticVersionStrategy.INSTANCE)
        .build();
```

The built-in strategies are exposed as `INSTANCE` singletons (`IntegerVersionStrategy.INSTANCE`, `DottedVersionStrategy.INSTANCE`, `SemanticVersionStrategy.INSTANCE`). In a CDI environment with the `config` module, you can select them declaratively instead via the `fluda.sqlinit.version-strategy` property — see [getting started](getting-started.md#version-strategy).

## Implementing a custom strategy

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
