# Advanced Topics

This page covers advanced transaction scenarios, including working with multiple data sources and mixing JPA with JDBC operations.

## Multiple DataSources

A single transaction can touch more than one `DataSource`. The primary manager drives the transaction; additional `TransactionAwareDataSourceProxy` instances join best-effort. This is **not XA**: each `Connection` commits independently, so work across `DataSource`s is not atomic.

### Example: Working with two DataSources

```java
@ApplicationScoped
public class MultiResourceService {

    @Inject
    @Named("orderDataSource")
    private DataSource orderDs;

    @Inject
    @Named("customerDataSource")
    private DataSource customerDs;

    @Transactional
    public void createOrderAndCustomer(int orderId, int customerId) throws Exception {
        // First DataSource
        try (Connection conn = orderDs.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "INSERT INTO orders (id, info) VALUES (?, ?)")) {
            ps.setInt(1, orderId);
            ps.setString(2, "order");
            ps.executeUpdate();
        }
        
        // Second DataSource - joins the same transaction
        try (Connection conn = customerDs.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "INSERT INTO customers (id, name) VALUES (?, ?)")) {
            ps.setInt(1, customerId);
            ps.setString(2, "customer");
            ps.executeUpdate();
        }
    }
}
```

Both DataSources participate in the same transaction boundary, but if one fails, the other may still commit. For true distributed transactions, consider using JTA with XA resources.

## Mixing JPA and JDBC

Because the same physical `Connection` backs both the `EntityManager` and the `TransactionAwareDataSourceProxy`, JPA and JDBC operations share a single transaction:

```java
@ApplicationScoped
public class OrderService {

    @Inject
    EntityManager em;

    @Inject
    JdbcClient client;

    @Transactional
    public void createOrder(Order order) {
        em.persist(order);                                    // JPA
        client.sql("INSERT INTO audit_log (msg) VALUES (:m)") // JDBC
              .param("m", "Order created: " + order.getId())
              .update();
    }
}
```

This allows you to leverage the strengths of both APIs: JPA for entity management and relationships, JDBC for bulk operations or complex queries that benefit from direct SQL.
