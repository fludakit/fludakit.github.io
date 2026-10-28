# Getting Started

This guide covers adding FluDa JMS Client to your project and exchanging your first messages, whether in a plain Java SE application, a CDI application, or a Jakarta EE server.

## Requirements

- JDK 21 or later.
- A Jakarta Messaging 3.1 broker. The test suite and examples use [Apache Artemis](https://activemq.apache.org/components/artemis/) 2.40; any compliant provider works, and Qpid JMS 2.7 is supported for AMQP 1.0.
- The `core` module is usable in plain Java SE: it depends only on the JDK and the Jakarta Messaging API.
- The `cdi` and `config` modules need CDI 4.1 and MicroProfile Config — a Jakarta EE 11 runtime such as GlassFish 8 or WildFly 42, or a CDI SE container (Weld SE with SmallRye Config) for Java SE.

## Adding dependencies

All modules are published under the `io.github.fludakit` group with the same version. Replace the snapshot version shown below with the release you use.

### Plain Java SE

Use the core module when you only need the fluent API and provide the connection factory yourself:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jms-core</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### CDI and MicroProfile Config

Add the CDI module to have `JmsClient` produced for injection, and the config module to build the connection factory from `fluda.jms.*` properties:

```xml
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jms-cdi</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>io.github.fludakit</groupId>
    <artifactId>fluda-jms-config</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

`cdi` depends on `core`, and `config` depends on `core` as well — add `cdi` explicitly when you want the producers. The config module treats the provider clients as optional dependencies, so add the one you use:

```xml
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>artemis-jakarta-client</artifactId>
    <version>2.40.0</version>
</dependency>
```

Use `org.apache.qpid:qpid-jms-client` (2.7.x) instead for AMQP 1.0. On a Jakarta EE server the `JMSContext` comes from the container, so no provider client is needed in the application.

## Sending your first message

### In a Java SE application

Create the provider's connection factory, wrap it in a `CachingConnectionFactory` so the thread reuses one context, and build the client:

```java
import io.github.fludakit.jms.CachingConnectionFactory;
import io.github.fludakit.jms.JmsClient;
import org.apache.activemq.artemis.jms.client.ActiveMQJMSConnectionFactory;

import java.io.Serializable;
import java.time.Duration;

record Order(String id, String item, int quantity) implements Serializable {
}

try (var factory = new ActiveMQJMSConnectionFactory("tcp://localhost:61616", "artemis", "artemis");
     var caching = new CachingConnectionFactory(factory)) {

    JmsClient client = JmsClient.builder(caching).build();

    client.destination("orders").send(new Order("order-1", "keyboard", 2));

    Order received = client.destination("orders").receive(Order.class, Duration.ofSeconds(5));
    System.out.println(received);
}
```

Two things are worth noting:

- **Close the caching factory before the raw factory.** `JmsClient` never closes the `JMSContext` it is given; the caching decorator owns the contexts it creates.
- **`send` converts the payload.** The default converter sends `Serializable` objects as an `ObjectMessage`, `String` as a `TextMessage`, `byte[]` as a `BytesMessage` and a `Map` as a `MapMessage`.

Without the caching decorator, every `JmsClient.builder(factory)` call opens a new context — and therefore a new connection — because nothing closes it. Either wrap the factory, or create the context yourself and pass it to `JmsClient.builder(context)`.

### Sending raw messages and message properties

Pass a post-processor to set properties or headers on the converted message:

```java
client.destination("orders").send(order,
        message -> JmsMessages.setStringProperty(message, "orderId", order.id()));
```

When no converter fits, build the message yourself — the creator receives the client's `JMSContext` as the message factory:

```java
client.destination("orders").send(context -> context.createTextMessage("manual payload"));
```

### Receiving messages

`receive` blocks up to the timeout and throws `JmsClientException` with the `TIMEOUT` code when no message arrives. The other variants cover the usual cases:

```java
// one message or a timeout exception
Order order = client.destination("orders").receive(Order.class, Duration.ofSeconds(5));

// one message, or empty on timeout
Optional<Order> maybe = client.destination("orders")
        .receiveOptional(Order.class, Duration.ofMillis(500));

// one message matching a JMS selector
Order urgent = client.destination("orders")
        .receiveSelected(Order.class, "priority = 'urgent'", Duration.ofSeconds(5));

// peek at the queue without consuming: browse requires a Queue destination
List<String> ids = client.destination("orders")
        .browse(message -> JmsMessages.stringProperty(message, "orderId"));
```

Each receive creates a consumer, receives and closes it, so the client stays usable for further operations.

## CDI in Java SE

With `fluda-jms-cdi` and `fluda-jms-config` on the classpath, the connection factory, its caching decorator and the client are all produced. Describe the broker in `META-INF/microprofile-config.properties`:

```properties
fluda.jms.artemis.broker-url=tcp://localhost:61616
fluda.jms.artemis.username=artemis
fluda.jms.artemis.password=artemis

# Used by client.destination() - a destination name, not a Destination.
fluda.jms.default-destination=orders
```

Then inject `JmsClient` and use it:

```java
import io.github.fludakit.jms.JmsClient;
import io.github.fludakit.jms.support.JmsMessages;
import jakarta.enterprise.context.Dependent;
import jakarta.inject.Inject;

import java.time.Duration;

@Dependent
public class OrderService {

    @Inject
    JmsClient client;

    public void place(Order order) {
        client.destination()
                .send(order, message -> JmsMessages.setStringProperty(message, "orderId", order.id()));
    }

    public Order receive(Duration timeout) {
        return client.destination().receive(Order.class, timeout);
    }
}
```

The produced `JmsClient` is `@Dependent` on purpose: a client is not thread-safe, so it must not be shared by concurrent callers. Inject it into a request-scoped or dependent bean, or obtain a fresh instance per operation from `Instance<JmsClient>`. Injecting it into an `@ApplicationScoped` bean pins one client — and one cached context — for the application's lifetime.

## In a Jakarta EE environment

Nothing is configured and nothing is created in the application: the container provides a `JMSContext` from `java:comp/DefaultJMSConnectionFactory`, and the `cdi` module builds the client from it. Declare the destination with `@JMSDestinationDefinition` and inject it, because Jakarta EE does not let an application create destinations at runtime:

```java
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.jms.JMSDestinationDefinition;

@JMSDestinationDefinition(
        name = "java:comp/jms/orders",
        interfaceName = "jakarta.jms.Queue",
        destinationName = "orders"
)
@ApplicationScoped
public class JmsResources {
}
```

```java
import jakarta.annotation.Resource;
import jakarta.enterprise.context.Dependent;
import jakarta.inject.Inject;
import jakarta.jms.Queue;

@Dependent
public class OrderService {

    @Inject
    JmsClient client;

    @Resource(lookup = "java:comp/jms/orders")
    Queue orders;

    public void place(Order order) {
        client.destination(orders).send(order);
    }
}
```

!!! warning "Do not produce a second `JMSContext` bean"
    The server already exposes the container-managed `JMSContext` as a bean. A producer of your own makes the `Instance<JMSContext>` lookup ambiguous and deployment fails with `WELD-001335: Ambiguous dependencies`.

Pass the injected `Destination` rather than a name: on a server the deployment decides the address, the queue-or-topic kind and the provider settings, while a name is resolved through `JMSContext.createQueue`/`createTopic` and ignores the resource definition. The `fluda.jms.default-destination` property has the same caveat — it is always a *name*.

## How the CDI module resolves resources

`JmsClientProducer` uses the first of these that exists:

1. **a `JMSContext` bean** — on Jakarta EE, the container-managed context. The client uses it as is and never closes it; credentials and session mode belong to the container.
2. **a `ConnectionFactory` bean** — wrapped in the `CachingConnectionFactory` bean, so all clients on a thread share a context and a connection.
3. **a configured provider section** (`fluda.jms.artemis.*` or `fluda.jms.qpid.*`) whose client is on the classpath — the config module builds the factory, wrapped the same way. Artemis is tried before Qpid when both are configured.

When none exists, production fails fast with an `IllegalStateException` naming the three options, rather than handing out a client that only fails on the first send.

An optional `JmsConfig` bean — produced by the config module from `fluda.jms.*` — is applied to whichever spec is used. For a factory it contributes the credentials, the acknowledge mode and the default destination; for an injected `JMSContext` only the default destination is applicable, and configuring credentials or a non-default acknowledge mode makes production fail loudly instead of silently ignoring them.

Optional `MessageConverter` and `DestinationResolver` beans are applied the same way, after the config: define one and every client the module produces uses it instead of the defaults, for example a `JsonbMessageConverter`:

```java
@ApplicationScoped
public class JmsBeans {

    @Produces
    @ApplicationScoped
    MessageConverter messageConverter() {
        return new JsonbMessageConverter();
    }
}
```

Keep one default-qualified bean per SPI — with more than one candidate the container reports the ambiguity, so qualify the extras. The beans replace the defaults application-wide; for a per-client choice use the builder in plain Java SE instead.

## Configuration

Client-wide properties:

| Property | Default | Meaning |
|---|---|---|
| `fluda.jms.username` | – | broker username for factory-based clients; unset or blank means anonymous access |
| `fluda.jms.password` | – | broker password for factory-based clients; unset or blank means no password |
| `fluda.jms.acknowledge-mode` | `auto` | `auto`, `client` or `dups-ok` |
| `fluda.jms.default-destination` | – | destination **name**, resolved through the `DestinationResolver` when `client.destination()` is called |

Provider sections — a section is active once its URL is set, and needs that provider's client on the classpath:

| Property | Meaning |
|---|---|
| `fluda.jms.artemis.broker-url` | Artemis core URL, e.g. `tcp://localhost:61616` |
| `fluda.jms.artemis.username` / `.password` | fall back to `fluda.jms.username` / `.password` |
| `fluda.jms.qpid.remote-uri` | Qpid JMS URI, e.g. `amqp://localhost:5672` |
| `fluda.jms.qpid.username` / `.password` | fall back to `fluda.jms.username` / `.password` |

`acknowledge-mode` uses the same short values as Spring Boot's `spring.jms.template.session.acknowledge-mode`: `auto` (`Session.AUTO_ACKNOWLEDGE`, the default), `client` (`Session.CLIENT_ACKNOWLEDGE`) and `dups-ok` (`Session.DUPS_OK_ACKNOWLEDGE`). Values are matched case-insensitively and `_` is accepted for `-`, so `CLIENT` and `DUPS_OK` both work. `transacted` is rejected deliberately, exactly as Spring Boot keeps it a separate flag: the fluent client never commits, so a transacted context would roll back on close — wrap an injected `JMSContext` in a transaction instead.

`client-id` and a connection-factory JNDI lookup are not mapped in v1: the client has no client-id support, and the provider sections configure a factory directly.

## Message converters

The default converter covers the usual payload types:

| Payload | JMS message |
|---|---|
| `String` | `TextMessage` |
| `byte[]` | `BytesMessage` |
| `Map` | `MapMessage` |
| `Serializable` | `ObjectMessage` |

For JSON payloads, opt into the JSON-B converter, which sends the payload as the body of a `TextMessage`:

```java
JmsClient client = JmsClient.builder(connectionFactory)
        .messageConverter(new JsonbMessageConverter())
        .build();
```

The converter publishes the payload class in the `_type` message property, so a consumer that is not told the type can still rebuild it; a typed `receive(Order.class)` prefers the requested type. JSON-B is an optional dependency: the class loads without it, and only constructing the converter without a provider fails, with a `JmsClientException` naming the missing piece.

A payload type the converter does not know makes `send` fail with the `CONVERSION` code. Implement `MessageConverter` and pass it to `messageConverter(...)` for anything else — under CDI a single `MessageConverter` bean is picked up automatically, as described in [How the CDI module resolves resources](#how-the-cdi-module-resolves-resources).

## Destinations

`client.destination("orders")` resolves a name through the client's `DestinationResolver`. `SimpleDestinationResolver` creates a queue per name, treats a `topic:` prefix as a topic, and caches each resolution so repeated lookups return the same destination:

```java
client.destination("orders").send(order);          // queue "orders"
client.destination("topic:events").send(event);    // topic "events"
```

Pass a `Destination` you already hold — an injected `Queue`, a provider object — to `destination(Destination)` to bypass resolution entirely, and supply your own `DestinationResolver` (for example a JNDI lookup) with `destinationResolver(...)` when names must follow server conventions. Under CDI a single `DestinationResolver` bean is applied to every produced client, so the same rule covers `fluda.jms.default-destination` and any name passed to `client.destination(name)`.

## Errors

Every failure surfaces as `JmsClientException`, with a `Code` telling the stages apart:

| Code | Raised when |
|---|---|
| `CONNECTION` | creating the `JMSContext` from a factory fails |
| `SEND` | sending a message fails |
| `RECEIVE` | receiving or browsing fails, or browsing a non-queue destination |
| `CONVERSION` | no converter handles the payload, or conversion fails |
| `TIMEOUT` | no message arrived within the receive timeout |

## Examples

The [fludakit/examples](https://github.com/fludakit/examples) repository contains four runnable modules:

| Example | Shows |
|---|---|
| [`jms-client-vanilla`](https://github.com/fludakit/examples/tree/main/jms-client-vanilla) | plain Java SE, no CDI |
| [`jms-client-artemis`](https://github.com/fludakit/examples/tree/main/jms-client-artemis) | CDI + config against Artemis |
| [`jms-client-qpid`](https://github.com/fludakit/examples/tree/main/jms-client-qpid) | the same over AMQP 1.0 with Qpid JMS |
| [`jms-client-jakartaee`](https://github.com/fludakit/examples/tree/main/jms-client-jakartaee) | Jakarta EE, on GlassFish and WildFly |

The examples share a broker from the repository's `docker-compose.yml`:

```bash
docker compose up -d artemis
```
