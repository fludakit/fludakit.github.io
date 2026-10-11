# Advanced Topics

This page covers modeling with aggregate roots, value objects and cross-aggregate associations, and gives the current status of two planned features — optimistic locking and domain events — that are **not ready yet**.

## Aggregate Root

An aggregate is a cluster of objects that is modified as a unit through a single entry point: the **aggregate root**. Mark the root type with `@AggregateRoot`. Everything the root owns — embedded value objects, composed child entities and element collections — is persisted and loaded with it, and a `save` of the root cascades to the objects it owns.

```java
import io.github.fludakit.data.annotation.AggregateRoot;
import io.github.fludakit.data.annotation.Identity;

@AggregateRoot
public class Profile {
    @Identity Long id;
    String fullName;

    Address address;                       // value object  -> flattened into this table
    List<WorkExperience> workHistory;      // entity collection -> own table, fk profile_id
    List<Education> education;             // entity collection -> own table, fk profile_id
    List<ContactPoint> contacts;           // value-object element collection -> child table
    // getters and setters ...
}
```

The owner's fields are classified by the type they reference:

| Field | Type | Storage |
|-------|------|---------|
| `address` | `@ValueObject` | **Embedded** — its leaves are flattened into the root's own table (`address.city`, `address.zip`, …). A value object has no identity of its own. |
| `contacts` | `List<` `@ValueObject` `>` | **Element collection** — a child table (`profile_contacts`) keyed by the root's foreign key plus the flattened value-object columns. |
| `workHistory`, `education` | `List<` `@Entity` `>` | **Composition** — each target is an entity with its own identity and its own table, carrying a foreign key back to the root (`profile_id`). |

### Value objects

A `@ValueObject` is immutable in spirit and defined entirely by its attributes — it has no identity. Embedded value objects may nest (an `Address` may hold a `GeoPoint`), and their leaves are flattened into the owning table. Because they are part of the root's row, you can sort and filter through them with a dot path:

```java
@Find
@OrderBy("address.city")
List<Profile> orderedByCity();
```

### Composition children

A composed `@Entity` (such as `WorkExperience`) manages its own table and identity, and holds a reference back to the root. The child's bare inverse field maps to the same foreign key the aggregate loader walks:

```java
@Entity
public class WorkExperience {
    @Identity Long id;
    String company;
    int yearStart;
    int yearEnd;
    Profile profile;   // inverse reference -> profile_id
    // ...
}
```

Querying **through** a composition collection renders an `EXISTS` semi-join, so the root never fans out into duplicate rows and the count stays correct:

```java
@Find
List<Profile> findByWorkHistoryCompany(@By("workHistory.company") String company);

long countByWorkHistoryCompany(String company);   // query-by-method-name equivalent
```

### Associations: referencing another aggregate

Not every relationship is composition. When a field points at an entity that belongs to a **different** aggregate (typically another bounded context), mark it `@Association`. An association is persisted as a **foreign key only**:

- On write, the referenced entity's identifier is extracted into the `<field>_id` column; the referenced entity itself is **never cascaded**.
- On read, the reference is **not** eagerly resolved — the field is left `null` and the application resolves it lazily when it needs to.

```java
@Entity
public class Order {
    @Identity Long id;
    BigDecimal amount;
    @Association Customer customer;   // persisted as customer_id, not cascaded
    List<OrderItem> orderItems;       // composition, owned by this aggregate
    // ...
}
```

This keeps aggregate boundaries explicit: `Order` owns its `orderItems`, but only *points at* a `Customer` that some other aggregate owns and saves.

## Optimistic locking with `@Version` — not ready

!!! warning "Not implemented yet"
    Optimistic locking is on the roadmap but is **not supported** in the current version. There is no version attribute recognition, and concurrent updates are last-write-wins — a stale update is not detected or rejected. The Jakarta Data TCK's `@Version` suites are excluded from the conformance run for this reason.

The intended design is a version column on the entity that the generated `UPDATE` both increments and guards with a `WHERE … AND version = :expected` predicate, failing the update when the row changed underneath you. Until this lands, do not rely on a version field for concurrency control. Track progress in the [issue tracker](https://github.com/fludakit/data/issues).

## Domain Events — not ready

!!! warning "Not implemented yet"
    Domain events are on the roadmap but there is **no API for them today**. Lifecycle operations do not raise events, and nothing is dispatched to the CDI container.

The intended design is to let an aggregate record domain events as its state changes (for example `OrderPlaced` when a new aggregate is saved) and have the generated repository publish them around the lifecycle operation, so application code can react through CDI observers instead of polling for changes. This is not part of the current release; watch the [issue tracker](https://github.com/fludakit/data/issues) for progress.
