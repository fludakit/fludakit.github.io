# Mail

A simple mail abstraction with built-in Jakarta Mail (SMTP) support and pluggable providers for third-party services.

| | |
|---|---|
| **Documentation** | [fludakit.github.io/mail](https://fludakit.github.io/mail/) |
| **Source** | [github.com/fludakit/mail](https://github.com/fludakit/mail) |
| **Artifacts** | `fluda-mail-core`, `fluda-mail-config`, `fluda-mail-cdi` |

The reference documentation for this module lives in the module repository and is published with GitHub Pages.

## Quick example

```java
import io.github.fludakit.mail.JakartaMailSender;
import io.github.fludakit.mail.MailMessage;
import io.github.fludakit.mail.MailSender;

import jakarta.mail.Session;
import java.util.Properties;

Properties props = new Properties();
props.put("mail.smtp.host", "smtp.example.com");
props.put("mail.smtp.port", "587");
props.put("mail.smtp.auth", "true");
props.put("mail.smtp.starttls.enable", "true");

Session session = Session.getInstance(props);
MailSender sender = new JakartaMailSender(session);

MailMessage message = new MailMessage();
message.setFrom("sender@example.com");
message.addTo("recipient@example.com");
message.setSubject("Hello from FluDa Mail");
message.setHtmlBody("<p>This is a test email sent with FluDa Mail.</p>");
sender.send(message);
```

## Sections

- [Getting Started](https://fludakit.github.io/mail/getting-started) — add dependencies and send your first email.
- [Mail Sender](https://fludakit.github.io/mail/mail-sender) — the `MailSender` abstraction and the built-in Jakarta Mail implementation.
- [Template Resolver](https://fludakit.github.io/mail/template-resolver) — rendering email templates with FreeMarker or simple placeholder processing.
