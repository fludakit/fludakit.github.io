# Jakarta EE

In a Jakarta EE environment, nothing is configured and nothing is created in the application: the container provides a `JMSContext` from `java:comp/DefaultJMSConnectionFactory`, and the `cdi` module builds the client from it.

## Declaring destinations

Jakarta EE does not let an application create destinations at runtime. Declare the destination with `@JMSDestinationDefinition` and inject it:

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

The annotation is repeatable — a second queue would be another annotation on the same class.

## Using the client

Inject `JmsClient` and the `Queue`, then pass the injected `Destination` rather than a name:

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

On a server the deployment decides the address, the queue-or-topic kind and the provider settings, while a name is resolved through `JMSContext.createQueue`/`createTopic` and ignores the resource definition. The `fluda.jms.default-destination` property has the same caveat — it is always a *name*.

!!! warning "Do not produce a second `JMSContext` bean"
    The server already exposes the container-managed `JMSContext` as a bean. A producer of your own makes the `Instance<JMSContext>` lookup ambiguous and deployment fails with `WELD-001335: Ambiguous dependencies`.

## How the CDI module resolves resources

`JmsClientProducer` uses the first of these that exists:

1. **a `JMSContext` bean** — on Jakarta EE, the container-managed context. The client uses it as is and never closes it; credentials and session mode belong to the container.
2. **a `ConnectionFactory` bean** — wrapped in the `CachingConnectionFactory` bean, so all clients on a thread share a context and a connection.
3. **a configured provider section** (`fluda.jms.artemis.*` or `fluda.jms.qpid.*`) whose client is on the classpath — the config module builds the factory, wrapped the same way. Artemis is tried before Qpid when both are configured.

When none exists, production fails fast with an `IllegalStateException` naming the three options, rather than handing out a client that only fails on the first send.

An optional `JmsConfig` bean — produced by the config module from `fluda.jms.*` — is applied to whichever spec is used. For a factory it contributes the credentials, the acknowledge mode and the default destination; for an injected `JMSContext` only the default destination is applicable, and configuring credentials or a non-default acknowledge mode makes production fail loudly instead of silently ignoring them.

Optional `MessageConverter` and `DestinationResolver` beans are applied the same way, after the config: define one and every client the module produces uses it instead of the defaults. See [Message Converter](message-converter.md#cdi-integration) and [Destination Resolver](destination-resolver.md#cdi-integration) for details.

## Example

The [`jms-client-jakartaee`](https://github.com/fludakit/examples/tree/main/jms-client-jakartaee) example runs on GlassFish 8 and WildFly 42.
