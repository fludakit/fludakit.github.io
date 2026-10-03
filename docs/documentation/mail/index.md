# FluDa Mail: Lightweight Mail Abstraction for Jakarta EE/CDI

FluDa Mail provides a simple, framework-agnostic mail abstraction for Jakarta EE and CDI applications. It defines a clean `MailSender` interface with built-in Jakarta Mail (SMTP) support, and makes it straightforward to integrate third-party providers like SendGrid.

The project is organized into three core modules:

| Module   | Artifact              | Description                                                                                                   |
|----------|-----------------------|---------------------------------------------------------------------------------------------------------------|
| `core`   | `fluda-mail-core`     | Contains the `MailSender` API, `MailMessage`, `MailBuilder`, and the default `JakartaMailSender` implementation. Depends only on the JDK and Jakarta Mail API. |
| `config` | `fluda-mail-config`   | Integrates with MicroProfile Config to read `fluda.mail.*` properties and expose a `MailConfig` bean.         |
| `cdi`    | `fluda-mail-cdi`      | Provides CDI producers for `MailSender` and `MailBuilder`, wiring the core beans into a Jakarta EE environment. |

## Key concepts

- **`MailSender`** — the core interface with a single `send(MailMessage)` method. The `core` module provides `JakartaMailSender` for SMTP; you can implement your own for third-party services.
- **`MailMessage`** — a simple POJO holding `from`, `to`, `subject`, `htmlBody`, and `attachments`.
- **`MailBuilder`** — a fluent builder that wraps a `MailSender` and optionally a `TemplateProcessor` for rendering email templates.
- **`MailConfig`** — configuration POJO with SMTP/POP3 properties, produced from MicroProfile Config in the `config` module.

## Reading order

- Start with [getting started](getting-started.md) to add dependencies and send your first email.
- Explore [advanced topics](advanced.md) for custom `MailSender` implementations and CDI integration patterns.
