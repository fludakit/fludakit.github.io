# Conditional Beans

A portable CDI extension that enables a bean **only when a condition matches the current deployment**.

| | |
|---|---|
| **Documentation** | [fludakit.github.io/conditional-beans](https://fludakit.github.io/conditional-beans/) |
| **Source** | [github.com/fludakit/conditional-beans](https://github.com/fludakit/conditional-beans) |
| **Artifacts** | `fluda-conditional-beans-api`, `fluda-conditional-beans-full`, `fluda-conditional-beans-lite` |

The reference documentation for this module lives in the module repository and is published with GitHub Pages.
The extension is registered through `META-INF/services/jakarta.enterprise.inject.spi.Extension`, so it is active in every deployment that has the artifact on the classpath. When a condition does not match, the bean is vetoed during bootstrap: it is not injectable, it does not show up in `BeanManager.getBeans(...)`, and it produces no `ProcessBean` event.
## Quick example

```java
public class ChinaCondition implements Condition<Conditional> {

    @Override
    public boolean matches(ConditionContext context, Conditional annotation) {
        return "CN".equals(context.getProperty("app.country").orElse(null));
    }
}
```

## Sections

- [Getting Started](https://fludakit.github.io/conditional-beans/getting-started) — add the artifact, declare your first condition, and pick the runtime that fits your deployment.
