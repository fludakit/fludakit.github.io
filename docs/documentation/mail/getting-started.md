# Getting Started

This guide covers adding FluDa Mail to your project and sending your first email, whether in a plain Java SE application or a Jakarta EE/CDI environment.

## Requirements

- JDK 21 or later.
- A Jakarta EE 11 / CDI-compatible runtime is required for the `cdi` and `config` modules, such as GlassFish 8 or WildFly 41.
- The `core` module is usable in plain Java SE applications because it depends only on the JDK and Jakarta Mail API.

## Adding dependencies

Choose the dependency that matches your runtime. The core module is intended for Java SE applications, while the CDI and configuration integrations are for Jakarta EE-compatible runtimes.

### Using the core `MailSender`

Use the following dependency when only the core fluent `MailSender` API is required:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-mail-core</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

The core module provides `JakartaMailSender` for SMTP. You must supply a `jakarta.mail.Session` configured for your mail server.

### Integrating with CDI

For a CDI-managed `MailSender` and `MailBuilder`, add the CDI artifact:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-mail-cdi</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

This artifact depends on the core module and is intended for a CDI environment. The CDI module produces `MailSender` and `MailBuilder` beans, but requires either a `MailConfig` bean or a `jakarta.mail.Session` bean to be available in the container.

### Integrating with MicroProfile Config

The `fluda-mail-config` module is an optional addon to the CDI module that provides property-based configuration. It reads `fluda.mail.*` properties from MicroProfile Config and produces a `MailConfig` bean, which the CDI module uses to create a `JakartaMailSender`.

Add this dependency when you want to configure mail settings via properties instead of providing your own `Session` bean:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-mail-config</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

This module depends on the CDI module and is intended for a CDI environment with MicroProfile Config support.

All modules are published with the same version. Replace the snapshot version shown above with the release version used by your application.

## Sending your first email

### In a Java SE application

In a plain Java SE application, construct a `JakartaMailSender` from a `jakarta.mail.Session`:

```java
import io.github.fludakit.mail.JakartaMailSender;
import io.github.fludakit.mail.MailMessage;
import io.github.fludakit.mail.MailSender;

import jakarta.mail.Session;
import java.util.Properties;

// Configure your SMTP session
Properties props = new Properties();
props.put("mail.smtp.host", "smtp.example.com");
props.put("mail.smtp.port", "587");
props.put("mail.smtp.auth", "true");
props.put("mail.smtp.starttls.enable", "true");

Session session = Session.getInstance(props);
MailSender sender = new JakartaMailSender(session);

// Build and send a message
MailMessage message = new MailMessage();
message.setFrom("sender@example.com");
message.addTo("recipient@example.com");
message.setSubject("Hello from FluDa Mail");
message.setHtmlBody("<p>This is a test email sent with FluDa Mail.</p>");

sender.send(message);
```

### Using the fluent `MailBuilder`

The `MailBuilder` provides a more fluent API for constructing and sending messages:

```java
import io.github.fludakit.mail.MailBuilder;
import io.github.fludakit.mail.JakartaMailSender;

import jakarta.mail.Session;
import java.util.Properties;

Session session = Session.getInstance(new Properties());
MailSender sender = new JakartaMailSender(session);

new MailBuilder(sender)
    .from("sender@example.com")
    .to("recipient@example.com")
    .subject("Hello from FluDa Mail")
    .htmlBody("<p>This is a test email sent with FluDa Mail.</p>")
    .send();
```

The builder also supports attachments:

```java
new MailBuilder(sender)
    .from("sender@example.com")
    .to("recipient@example.com")
    .subject("Report attached")
    .htmlBody("<p>Please find the report attached.</p>")
    .attachment("report.pdf", "application/pdf", () -> new FileInputStream("report.pdf"))
    .send();
```

### In a CDI environment

In a Jakarta EE/CDI environment, the `fluda-mail-cdi` module automatically produces a `MailSender` and `MailBuilder` bean. However, the `MailSender` requires either a `MailConfig` bean or a `jakarta.mail.Session` bean to be available in the CDI container.

#### Using `fluda-mail-config` for property-based configuration

The `fluda-mail-config` module provides a `MailConfig` bean that is configurable via MicroProfile Config properties. Add both dependencies:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-mail-cdi</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-mail-config</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

Then configure your SMTP settings in `META-INF/microprofile-config.properties`:

```properties
fluda.mail.host=smtp.example.com
fluda.mail.port=587
fluda.mail.auth-enabled=true
fluda.mail.starttls=true
fluda.mail.username=your-username
fluda.mail.password=your-password
fluda.mail.from=noreply@example.com
```

The `config` module reads these properties and produces a `MailConfig` bean, which the `cdi` module uses to create a `JakartaMailSender`.

#### Injecting and using the mail beans

Once configured, inject `MailSender` or `MailBuilder` directly:

```java
import io.github.fludakit.mail.MailBuilder;
import io.github.fludakit.mail.MailSender;
import jakarta.inject.Inject;

@ApplicationScoped
public class NotificationService {

    @Inject
    MailSender sender;

    @Inject
    MailBuilder mailBuilder;

    public void sendNotification(String to, String subject, String body) {
        mailBuilder
            .from("noreply@example.com")
            .to(to)
            .subject(subject)
            .htmlBody(body)
            .send();
    }
}
```

The `MailBuilder` is produced as `@Dependent`, so each injection point receives a fresh instance. The `MailSender` is `@ApplicationScoped` and backed by `JakartaMailSender` using the configuration from MicroProfile Config.

#### Alternative: providing your own `Session` bean

If you prefer to use a container-managed JavaMail session (via JNDI) instead of property-based configuration, you can produce your own `Session` bean. See the [Jakarta EE environment](#in-a-jakarta-ee-environment) section for details.

#### Configuration properties

The following properties are supported by the `config` module:

| Property                    | Description                          | Default     |
|-----------------------------|--------------------------------------|-------------|
| `fluda.mail.host`           | SMTP server hostname                 | `localhost` |
| `fluda.mail.port`           | SMTP server port                     | `25`        |
| `fluda.mail.username`       | SMTP username                        | (empty)     |
| `fluda.mail.password`       | SMTP password                        | (empty)     |
| `fluda.mail.auth-enabled`   | Enable SMTP authentication           | `false`     |
| `fluda.mail.starttls`       | Enable STARTTLS                      | `false`     |
| `fluda.mail.protocol`       | Mail protocol (`smtp` or `pop3`)     | `smtp`      |
| `fluda.mail.from`           | Default from address                 | (empty)     |

## In a Jakarta EE environment

Jakarta EE application servers (GlassFish, WildFly, Payara, etc.) provide built-in JavaMail session management via JNDI. Instead of configuring SMTP properties in `microprofile-config.properties`, you can use the container-managed mail session.

### Using the default mail session

Most Jakarta EE servers provide a default mail session at `java:comp/DefaultMail`. You can inject it directly and expose it as a CDI bean:

```java
import jakarta.annotation.Resource;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.mail.Session;

@ApplicationScoped
public class MailSessionProducer {

    @Resource(lookup = "java:comp/DefaultMail")
    private Session defaultSession;

    @Produces
    @ApplicationScoped
    public Session mailSession() {
        return defaultSession;
    }
}
```

When a `Session` bean is available in the CDI container, the `MailSenderProducer` in `fluda-mail-cdi` automatically uses it instead of creating a session from `MailConfig` properties.

### Defining a custom mail session

You can also define a custom mail session using `@MailDefinition` (Jakarta EE 11+):

```java
import jakarta.mail.MailDefinition;
import jakarta.annotation.Resource;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.mail.Session;

@ApplicationScoped
@MailDefinition(
    host = "smtp.example.com",
    port = 587,
    from = "noreply@example.com",
    enableStartTls = true,
    auth = true,
    username = "your-username",
    password = "your-password"
)
public class MailSessionProducer {

    @Resource(lookup = "java:comp/DefaultMail")
    private Session mailSession;

    @Produces
    @ApplicationScoped
    public Session session() {
        return mailSession;
    }
}
```

The `@MailDefinition` annotation configures the mail session at the class level, and the container creates the corresponding JNDI resource. You then inject it via `@Resource` and expose it as a CDI bean.

### How it works

The `MailSenderProducer` in `fluda-mail-cdi` resolves the `Session` in the following order:

1. If a `MailConfig` bean is available (from the `config` module), it creates a session from those properties.
2. Otherwise, if a `Session` bean exists in the CDI container (produced by your code as shown above), it uses that session.
3. As a fallback, it creates a default session with `localhost:25` and no authentication.

By exposing a JNDI-managed `Session` as a CDI bean, you leverage the container's mail configuration and avoid duplicating SMTP settings in your application.

See [advanced topics](advanced.md) for custom `MailSender` implementations and replacing the default `JakartaMailSender` with third-party providers.
