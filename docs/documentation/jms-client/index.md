# JMS Client

A fluent Jakarta Messaging client in the style of Spring's `JmsClient`, with CDI wiring and MicroProfile Config.

| | |
|---|---|
| **Documentation** | [fludakit.github.io/jms-client](https://fludakit.github.io/jms-client/) |
| **Source** | [github.com/fludakit/jms-client](https://github.com/fludakit/jms-client) |
| **Artifacts** | `fluda-jms-core`, `fluda-jms-config`, `fluda-jms-cdi`, `fluda-jms-artemis`, `fluda-jms-qpid` |

The reference documentation for this module lives in the module repository and is published with GitHub Pages.

## Quick example

```java
import io.github.fludakit.jms.CachingConnectionFactory;
import io.github.fludakit.jms.JmsClient;
import org.apache.activemq.artemis.jms.client.ActiveMQJMSConnectionFactory;

import java.io.Serializable;

record Order(String id, String item, int quantity) implements Serializable {
}

try (var factory = new ActiveMQJMSConnectionFactory("tcp://localhost:61616", "artemis", "artemis");
     var caching = new CachingConnectionFactory(factory)) {

    JmsClient client = JmsClient.builder(caching).build();

    client.destination("orders").send(new Order("order-1", "keyboard", 2));
}
```

## Sections

- [Getting Started](https://fludakit.github.io/jms-client/getting-started) — add the dependency and exchange your first messages.
- [Jakarta EE](https://fludakit.github.io/jms-client/jakarta-ee) — using the container-provided `JMSContext`.
- [Message Converter](https://fludakit.github.io/jms-client/message-converter) — the `MessageConverter` SPI and payload conversion.
- [Destination Resolver](https://fludakit.github.io/jms-client/destination-resolver) — resolving destinations by name.
