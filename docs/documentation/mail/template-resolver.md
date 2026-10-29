# Template Resolver

The `MailBuilder` optionally integrates with a `TemplateProcessor` for rendering email templates.

## Template resolution in CDI

In a CDI environment, the `MailBuilderProducer` resolves the `TemplateProcessor` in the following order:

1. If a `TemplateProcessor` bean is available in the CDI container, it uses that bean.
2. Otherwise, it falls back to `TemplateProcessor.DEFAULT`, which:
   - Checks if FreeMarker is on the classpath (via `Class.forName("freemarker.template.Configuration")`). If found, uses the built-in `FreeMarkerTemplateProcessor`.
   - Otherwise, falls back to `SimpleTemplateProcessor`, a lightweight implementation that performs `:name` placeholder substitution.

This means you can use templates out of the box without any additional configuration — just add FreeMarker to your classpath if you want `.ftl` template support, or use the simple placeholder-based processor by default.

## Built-in processors

The `core` module ships with two built-in processors:

- **`FreeMarkerTemplateProcessor`** — used automatically when FreeMarker is on the classpath. Loads `.ftl` templates from the classpath under `templates/`.
- **`SimpleTemplateProcessor`** — a lightweight fallback that loads `.subject` and `.body` templates from the classpath and performs `:name` placeholder substitution.

## Using templates

To use templates, inject a `TemplateProcessor` bean or rely on the default:

```java
new MailBuilder(sender, templateProcessor)
    .from("noreply@example.com")
    .to("user@example.com")
    .locale(Locale.FRENCH)
    .template("welcome", Map.of("name", "Alice"))
    .send();
```

Calling `.template("welcome", ...)` renders two templates — `welcome.subject` and `welcome.body` — using the supplied context map. Both processors load files from the classpath under `templates/`, try a locale-specific file first (suffix `_fr` for `Locale.FRENCH`), and fall back to the un-suffixed file:

| Processor | Subject file | Body file |
|---|---|---|
| `SimpleTemplateProcessor` (default) | `templates/welcome.subject_fr.txt` → `templates/welcome.subject.txt` | `templates/welcome.body_fr.html` → `templates/welcome.body.html` |
| `FreeMarkerTemplateProcessor` | `templates/welcome.subject_fr.ftl` → `templates/welcome.subject.ftl` | `templates/welcome.body_fr.ftl` → `templates/welcome.body.ftl` |

`SimpleTemplateProcessor` substitutes `:name` placeholders (e.g. `:name` is replaced by the `name` entry in the context map). `FreeMarkerTemplateProcessor` uses standard FreeMarker syntax (e.g. `${name}`).

## Custom `TemplateProcessor`: Quarkus Qute integration

If you prefer to use Quarkus Qute as your template engine, you can implement a custom `TemplateProcessor` and expose it as a CDI bean. The `MailBuilderProducer` will automatically pick it up.

First, add the Qute dependency:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-qute</artifactId>
    <version>3.15.1</version>
</dependency>
```

Then implement `TemplateProcessor`:

```java
import io.github.fludakit.mail.template.TemplateProcessor;
import io.quarkus.qute.Engine;
import io.quarkus.qute.Template;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Locale;
import java.util.Map;

@ApplicationScoped
public class QuteTemplateProcessor implements TemplateProcessor {

    @Inject
    Engine engine;

    @Override
    public String render(String templateName, Map<String, Object> context, Locale locale) {
        // Qute uses template files from resources/templates/ by default
        // You can implement locale-aware resolution here
        String templatePath = templateName + ".html";
        Template template = engine.getTemplate(templatePath);
        
        if (template == null) {
            throw new IllegalArgumentException("Template not found: " + templatePath);
        }
        
        return template.data(context).render();
    }
}
```

Once this bean is available in the CDI container, the `MailBuilderProducer` will inject it automatically, and all `MailBuilder` instances will use your Qute-based processor for template rendering.
