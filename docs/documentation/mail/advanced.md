# Advanced Topics

This page covers extending FluDa Mail with custom `MailSender` implementations and integrating third-party providers like SendGrid in a CDI environment.

## Creating a custom `MailSender`

The `MailSender` interface is intentionally minimal:

```java
public interface MailSender {
    void send(MailMessage message) throws MailException;
}
```

Any email provider — SendGrid, Amazon SES, Mailgun, or a proprietary API — can be integrated by implementing this single method. The `MailMessage` POJO provides access to `from`, `to`, `subject`, `htmlBody`, and `attachments`, which is sufficient for most provider APIs.

### Example: SendGrid implementation

The following example shows how to wrap the SendGrid Java SDK in a `MailSender` implementation.

Add the SendGrid dependency to your project:

```xml
<dependency>
    <groupId>com.sendgrid</groupId>
    <artifactId>sendgrid-java</artifactId>
    <version>4.10.3</version>
</dependency>
```

Then implement `MailSender`:

```java
import com.sendgrid.Method;
import com.sendgrid.Request;
import com.sendgrid.Response;
import com.sendgrid.SendGrid;
import com.sendgrid.helpers.mail.Mail;
import com.sendgrid.helpers.mail.objects.Content;
import com.sendgrid.helpers.mail.objects.Email;
import io.github.fludakit.mail.MailAttachment;
import io.github.fludakit.mail.MailException;
import io.github.fludakit.mail.MailMessage;
import io.github.fludakit.mail.MailSender;

import java.io.InputStream;
import java.util.Base64;

public class SendGridMailSender implements MailSender {

    private final SendGrid sendGrid;

    public SendGridMailSender(String apiKey) {
        this.sendGrid = new SendGrid(apiKey);
    }

    @Override
    public void send(MailMessage message) throws MailException {
        try {
            Email from = new Email(message.getFrom());
            Email to = new Email(message.getTo().getFirst());
            Content content = new Content("text/html", message.getHtmlBody());
            Mail mail = new Mail(from, message.getSubject(), to, content);

            // Add additional recipients
            for (int i = 1; i < message.getTo().size(); i++) {
                mail.getPersonalization().getFirst()
                    .addTo(new Email(message.getTo().get(i)));
            }

            // Add attachments
            for (MailAttachment attachment : message.getAttachments()) {
                try (InputStream is = attachment.toDataSource().getInputStream()) {
                    byte[] bytes = is.readAllBytes();
                    String base64 = Base64.getEncoder().encodeToString(bytes);

                    var sdkAttachment = new com.sendgrid.helpers.mail.objects.Attachments();
                    sdkAttachment.setContent(base64);
                    sdkAttachment.setType(attachment.toDataSource().getContentType());
                    sdkAttachment.setFilename(attachment.getFilename());
                    sdkAttachment.setDisposition("attachment");

                    mail.addAttachments(sdkAttachment);
                }
            }

            Request request = new Request();
            request.setMethod(Method.POST);
            request.setEndpoint("mail/send");
            request.setBody(mail.build());

            Response response = sendGrid.api(request);
            if (response.getStatusCode() < 200 || response.getStatusCode() >= 300) {
                throw new MailException("SendGrid rejected payload: " + response.getStatusCode());
            }
        } catch (MailException e) {
            throw e;
        } catch (Exception e) {
            throw new MailException("Failed to send email via SendGrid", e);
        }
    }
}
```

This implementation translates the `MailMessage` into the SendGrid SDK's `Mail` object, handles Base64 encoding for attachments, and checks the HTTP response status.

## Replacing `JakartaMailSender` in CDI

By default, the `fluda-mail-cdi` module produces a `JakartaMailSender` backed by a `jakarta.mail.Session` configured from MicroProfile Config properties. To use a different provider — such as the `SendGridMailSender` above — you need to supply your own `MailSender` bean that takes precedence.

### Option 1: CDI producer with `@Priority`

The simplest approach is to produce your own `MailSender` bean with a higher `@Priority`:

```java
import io.github.fludakit.mail.MailSender;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
@Priority(100)
public class SendGridProducer {

    @Produces
    @ApplicationScoped
    public MailSender mailSender(
            @ConfigProperty(name = "sendgrid.api.key") String apiKey) {
        return new SendGridMailSender(apiKey);
    }
}
```

The `@Priority(100)` annotation ensures this producer takes precedence over the default `MailSenderProducer` in the `fluda-mail-cdi` module. The `sendgrid.api.key` property can be defined in `META-INF/microprofile-config.properties` or as an environment variable.

### Option 2: CDI alternative bean

Alternatively, mark the default producer as an alternative and provide your own:

In a CDI extension or `beans.xml`, you can disable the default producer. However, using `@Priority` is generally simpler and more maintainable.

### Using the custom sender

Once your producer is in place, inject `MailSender` or `MailBuilder` as usual — the CDI container resolves your custom implementation:

```java
import io.github.fludakit.mail.MailBuilder;
import jakarta.inject.Inject;

@ApplicationScoped
public class NotificationService {

    @Inject
    MailBuilder mailBuilder;

    public void sendWelcomeEmail(String to) {
        mailBuilder
            .from("welcome@example.com")
            .to(to)
            .subject("Welcome!")
            .htmlBody("<p>Welcome to our platform.</p>")
            .send();
    }
}
```

No code changes are needed at the injection site. The `MailBuilder` produced by the CDI module depends on `MailSender` by type, so it automatically picks up whichever bean the container resolves — whether that is the default `JakartaMailSender` or your custom `SendGridMailSender`.

## Template support

The `MailBuilder` optionally integrates with a `TemplateProcessor` for rendering email templates. The `core` module ships with two built-in processors:

- **`FreeMarkerTemplateProcessor`** — used automatically when FreeMarker is on the classpath. Loads `.ftl` templates from the classpath under `templates/`.
- **`SimpleTemplateProcessor`** — a lightweight fallback that loads `.subject` and `.body` templates from the classpath and performs `:name` placeholder substitution.

To use templates, inject a `TemplateProcessor` bean or rely on the default:

```java
new MailBuilder(sender, templateProcessor)
    .from("noreply@example.com")
    .to("user@example.com")
    .locale(Locale.FRENCH)
    .template("welcome", Map.of("name", "Alice"))
    .send();
```

The builder looks for `templates/welcome.subject` (or `welcome_fr.subject` for the French locale) and `templates/welcome.body` (or `welcome_fr.body`), rendering them with the provided context map.
