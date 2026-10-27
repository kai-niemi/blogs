---
title: "Write Performance with Spring Data"
series_weight: 4
date: 2022-07-24
lastmod: 2026-10-27
draft: false
series:
  - data-access
categories:
  - Databases
description: "JDBC batching, identifier generation, persistence context management and bulk inserts with Spring Data JPA and PostgreSQL."
tags: ["Spring Data", "PostgreSQL", "JPA", "Hibernate", "SQL", "Performance"]
cover:
  image: "cover.webp"
  caption: 'Photo by [Patrick Pahlke](https://unsplash.com/@p_pixels_p?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/black-and-red-computer-part-0kwAZN1hzGI?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)'
  relative: true
  hiddenInList: true
---

This article is the fourth in a series about data access with Spring Data.

<!--more-->

{{< series-nav >}}

This time, we'll look at how to improve write throughput when an application needs to insert, update, or delete large
numbers of records.

Persisting a single entity with Spring Data JPA is straightforward. Persisting thousands of entities efficiently
requires more care.

The application may spend considerable time issuing JDBC statements, synchronizing the persistence context, generating
identifiers, and waiting for database round trips. These costs can become significant even when individual SQL
statements execute quickly.

The goal is not simply to execute fewer statements. It is to reduce unnecessary overhead while preserving the
transaction semantics and consistency guarantees required by the application.

The examples use PostgreSQL, Spring Data JPA, and Hibernate 6.

# 1. Understanding Write Throughput

Consider an application that needs to insert 10,000 records.

A straightforward implementation might look like this:

```java
@Transactional
public void createEvents(List<Event> events) {
    for (Event event : events) {
        eventRepository.save(event);
    }
}
```

Assuming each entity is new, Spring Data JPA delegates persistence to Hibernate. Hibernate tracks the entities in its
persistence context and schedules the necessary SQL operations. Without JDBC batching, the resulting database
interaction may involve executing a separate `INSERT` statement for each entity.

Conceptually:

```sql
INSERT INTO event (id, type, payload) VALUES (?, ?, ?);
INSERT INTO event (id, type, payload) VALUES (?, ?, ?);
INSERT INTO event (id, type, payload) VALUES (?, ?, ?);
```

Repeated thousands of times. Even when the database can execute each insert efficiently, the cumulative cost of 
JDBC calls, network communication, and statement execution can limit throughput.

There are several distinct sources of overhead:

- **Database round trips:** Communication between the application and PostgreSQL.
- **Statement execution:** Parsing, planning, and executing SQL statements.
- **Persistence context management:** Tracking managed entities and their state.
- **Identifier generation:** Obtaining identifiers for newly persisted entities.
- **Transaction processing:** Maintaining indexes, constraints, and transactional state.

Optimizing one of these costs does not necessarily eliminate the others.

# 2. JDBC Batching

JDBC provides a batching mechanism that allows an application to group multiple statement executions.
Rather than executing each statement independently, the application can add parameter sets to a batch and execute them
together.

Hibernate supports this mechanism for entity inserts, updates, and deletes.

## 2.1 Enabling Hibernate batching

JDBC batching can be configured through Spring Boot:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
```

This configures Hibernate to group compatible SQL statements into batches of up to 50 executions.

Consider an entity:

```java
@Entity
@Table(name = "event")
public class Event {

    @Id
    private UUID id;

    private String type;

    private String payload;
}
```

And a service:

```java
@Transactional
public void createEvents(List<Event> events) {
    for (Event event : events) {
        entityManager.persist(event);
    }
}
```

Assuming the identifiers have already been assigned, Hibernate can batch the inserts when the persistence context is
flushed.

The SQL remains conceptually equivalent to:

```sql
INSERT INTO event (id, type, payload)
VALUES (?, ?, ?);
```

The difference is how Hibernate submits repeated executions through JDBC. With a batch size of 50, inserting 
10,000 entities may require approximately 200 JDBC batch executions rather than 10,000 individual executions.

A JDBC batch execution is not necessarily a single network round trip, nor does it imply that PostgreSQL processes one
SQL statement. The number of round trips and server-side statements depends on the JDBC driver and its configuration.

The PostgreSQL driver's optional batch rewriting is discussed in section 6.2.

## 2.2 Batching is not the same as saveAll()

Spring Data JPA provides `saveAll()`:

```java
eventRepository.saveAll(events);
```

Despite its name, `saveAll()` does not inherently perform a database bulk insert. Spring Data JPA processes the 
supplied entities individually, delegating each operation to the underlying persistence provider.
Whether Hibernate batches the resulting SQL depends on its configuration, identifier generation strategy, and the SQL
statements being executed.

For example:

```java
@Transactional
public void createEvents(List<Event> events) {
    eventRepository.saveAll(events);
}
```

With JDBC batching enabled, this can be an effective approach for moderate workloads.
Without batching, however, it may still result in thousands of separate JDBC statement executions.
The important distinction is that `saveAll()` is a repository API for saving multiple entities, while JDBC batching is
an execution optimization.

There is another consideration when identifiers are assigned by the application. 

Spring Data JPA must determine whether an entity is new to decide whether to persist or merge it. By default, it 
checks a nullable, non-primitive version property first, if one exists, and otherwise
checks whether the identifier is `null`. 

An entity with an assigned `UUID` and no version property may therefore be treated
as existing, causing `save()` to call `merge()` rather than `persist()`. This can introduce extra database reads and
entity-copying overhead.

For insert-only operations where the application knows that the entities are new, explicit `entityManager.persist()`
avoids this ambiguity. Alternatively, entities with assigned identifiers can implement Spring Data's `Persistable`
interface to provide their own new-entity detection. Neither approach changes the need to configure JDBC batching.

## 2.3 Ordering inserts and updates

Hibernate can also reorder certain pending statements to improve batching efficiency.

For example:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
```

Suppose an application persists entities of different types:

```java
entityManager.persist(customer1);
entityManager.persist(order1);
entityManager.persist(customer2);
entityManager.persist(order2);
```

Without ordering, the generated statements may alternate between tables.

Because JDBC batching generally groups executions of the same SQL statement, alternating statement types can reduce
batching opportunities. With insert ordering enabled, Hibernate can group compatible inserts more effectively.

However, ordering introduces additional work within Hibernate and can affect the order in which statements are executed.
It should be evaluated against the application's mappings and workload rather than enabled without measurement.

## 2.4 Batching entity updates

JDBC batching is not limited to inserts. Hibernate can also batch compatible `UPDATE` statements when several managed
entities are modified in one transaction.

```java
@Transactional
public void changeStatus(List<UUID> orderIds) {
    List<Order> orders = orderRepository.findAllById(orderIds);

    for (Order order : orders) {
        order.setStatus(OrderStatus.PROCESSED);
    }
}
```

Hibernate detects the changes and issues updates when the persistence context is flushed. With batching enabled,
compatible statements can be grouped into JDBC batches. The actual grouping depends on the generated SQL, including
which columns are updated and whether optimistic locking is used.

For versioned entities, Hibernate normally includes the version in the `UPDATE` predicate and increments it. Batched
execution must still detect optimistic-lock failures, which requires reliable update counts from the JDBC driver. If the
driver reports `SUCCESS_NO_INFO` instead of individual update counts, Hibernate cannot reliably determine whether each
versioned update affected the expected row. This behavior should be tested with the PostgreSQL driver and the
application's mappings.

If every matching row needs the same change, a bulk JPQL or SQL update may be simpler and more efficient than loading
entities. Bulk updates have different persistence-context and optimistic-locking semantics, as discussed in section 5.

# 3. Identifier Generation and Batching

Identifier generation has a significant influence on insert performance.
Different generation strategies interact with Hibernate batching in different ways.

## 3.1 Identity columns

Consider an entity using an identity column:

```java
@Entity
public class Event {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String type;
}
```

With PostgreSQL, this commonly maps to an identity-backed column.
The database generates the identifier when the row is inserted.
Hibernate generally needs to execute the insert to obtain the generated identifier.
As a result, Hibernate disables JDBC insert batching for entities using identity identifier generation.

Even if the application configures:

```yaml
hibernate:
  jdbc:
    batch_size: 50
```

Inserts for identity-generated entities will not benefit from Hibernate's normal JDBC insert batching.

## 3.2 Sequence-based identifiers

A sequence-based strategy allows Hibernate to obtain identifiers before executing the corresponding inserts.

For example:

```java
@Entity
public class Event {

    @Id
    @GeneratedValue(
            strategy = GenerationType.SEQUENCE,
            generator = "event_seq"
    )
    @SequenceGenerator(
            name = "event_seq",
            sequenceName = "event_seq",
            allocationSize = 50
    )
    private Long id;

    private String type;
}
```

PostgreSQL sequences provide identifiers independently of row insertion.
Hibernate can obtain identifiers and defer the inserts until the persistence context is flushed.
This makes JDBC insert batching possible.

However, obtaining one sequence value for every inserted entity can introduce additional database communication.
The `allocationSize` setting allows Hibernate to allocate identifiers in groups using its configured sequence optimizer.
With an appropriate optimizer, Hibernate can reduce the frequency of sequence access.

For example, with an allocation size of 50 and a compatible pooled optimizer, Hibernate can reserve identifiers for
approximately 50 entities with one sequence access, instead of fetching a sequence value for each entity. The exact
allocation behavior depends on the optimizer.

The sequence definition and Hibernate allocation settings must be compatible. In particular, an existing PostgreSQL
sequence may need an increment that matches the allocation size used by the selected optimizer. Hibernate can validate
this relationship at startup.

`allocationSize` controls identifier allocation. It does **not** control the number of JDBC statements in a batch;
`hibernate.jdbc.batch_size` is a separate setting.

For workloads inserting large numbers of entities, sequence-based identifiers with pooled allocation can be
substantially more efficient than identity generation.

## 3.3 UUID identifiers

UUID identifiers are another option:

```java
@Entity
public class Event {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    private String type;
}
```

Hibernate can generate UUID identifiers without inserting the row first. Consequently, UUID generation does not 
impose the same JDBC batching limitation as identity columns.

An application can also assign identifiers explicitly:

```java
Event event = new Event();
event.setId(UUID.randomUUID());

entityManager.persist(event);
```

The tradeoffs differ from sequence-based identifiers. Random UUIDs can affect index locality and storage behavior,
particularly when used as primary keys in large PostgreSQL
tables.

Hibernate 6's default UUID generator typically produces random UUIDv4 values. Time-ordered identifiers such as UUIDv7
can improve index insertion locality, although their suitability depends on the application's requirements and the UUID
generator being used. `GenerationType.UUID` should not be assumed to produce UUIDv7.

Identifier generation should therefore be considered part of the write-performance design rather than merely an
entity-mapping detail.

# 4. Managing the Persistence Context

JDBC batching reduces statement execution overhead, but it does not eliminate the cost of managing entities.

Consider:

```java
@Transactional
public void importEvents(List<Event> events) {
    for (Event event : events) {
        entityManager.persist(event);
    }
}
```

If the method persists 100,000 entities, Hibernate may retain all of them in the persistence context until the
transaction completes.

The persistence context maintains references to managed entities and coordinates pending database operations.

As the number of managed entities grows, memory consumption and persistence-context processing overhead can become
significant.

## 4.1 Flushing and clearing in batches

For large imports, a common approach is to flush and clear the persistence context periodically.

```java
@Transactional
public void importEvents(List<Event> events) {
    int batchSize = 50;

    for (int i = 0; i < events.size(); i++) {
        entityManager.persist(events.get(i));

        if ((i + 1) % batchSize == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
}
```

`flush()` synchronizes pending changes with the database, while `clear()` detaches all entities from the persistence
context. Together, these operations prevent the persistence context from retaining every imported entity throughout the
transaction. The final incomplete batch is flushed when the transaction commits, assuming the normal Hibernate flush
behavior remains enabled.

There are two important distinctions. First, flushing does not commit the transaction. All inserts still belong to the
surrounding transaction and can be rolled back together. Second, clearing the persistence context detaches every managed
entity, not just those inserted by the loop. If the transaction contains other managed entities that the application
still needs, calling `clear()` may have unintended consequences. This technique is therefore best suited to dedicated
import operations with a well-defined persistence-context scope.

Notice that `events` is still a `List<Event>` holding references to every input entity. Clearing the persistence context
reduces Hibernate's retained managed state, but does not free the input list or the objects it references. For very
large imports, read and process records incrementally in chunks rather than materializing the entire input in memory.

## 4.2 Batch size and transaction size

The JDBC batch size and transaction size are different concepts.
A JDBC batch groups statement executions.
A transaction determines the atomic unit of work.
An application can insert 100,000 rows in a single transaction while flushing JDBC batches of 50 statements.

Alternatively, it can divide the import into multiple transactions.
These approaches have different tradeoffs.
A single transaction provides all-or-nothing semantics but may hold database resources for longer.
Multiple transactions reduce the amount of work associated with each commit but allow partial progress if a later
transaction fails.

The appropriate choice depends on whether the application requires the entire import to be atomic.
For example, processing independent events may allow transaction boundaries between chunks.
Transferring financial balances between accounts generally requires stronger atomicity guarantees.
Increasing throughput should not come at the expense of required consistency properties.

For 10,000 rows, consider three different execution models:

| Execution model                         | Transaction boundary        | Failure behavior                                       |
|-----------------------------------------|-----------------------------|--------------------------------------------------------|
| One transaction, JDBC batches of 50     | All 10,000 rows             | The transaction can roll back all rows                 |
| 100 transactions of 100 rows            | One chunk at a time         | Earlier committed chunks remain if a later chunk fails |
| One `COPY` operation in one transaction | All rows in the transaction | The transaction can roll back the entire load          |

These models may have very different performance characteristics, but batching and `COPY` do not inherently imply
separate transactions. When splitting work into chunks, the application must also decide how to retry failed chunks
without duplicating data.

# 5. Bulk Operations with Spring Data JPA

JDBC batching still involves individual entity operations.
For some workloads, it is more efficient to execute a single SQL statement that affects many rows.
JPA supports bulk updates and deletes through JPQL.

## 5.1 Bulk updates

Suppose an application needs to mark all expired orders as cancelled.
One approach would be to load the matching entities and update them individually.

```java
@Transactional
public void cancelExpiredOrders(Instant cutoff) {
    List<Order> orders = orderRepository.findExpired(cutoff);

    for (Order order : orders) {
        order.setStatus(OrderStatus.CANCELLED);
    }
}
```

Hibernate must load the matching entities and manage their state.
For large result sets, this introduces unnecessary overhead when the application only needs to update a column.

A bulk update can express the operation directly:

```java
@Transactional
@Modifying
@Query("""
    UPDATE Order o
    SET o.status = :status
    WHERE o.status = :currentStatus
      AND o.createdAt < :cutoff
    """)
int cancelExpiredOrders(
        @Param("currentStatus") OrderStatus currentStatus,
        @Param("status") OrderStatus status,
        @Param("cutoff") Instant cutoff
);
```

Assuming the Order entity maps to the orders table, the resulting SQL is conceptually:

```sql
UPDATE orders
SET status = 'CANCELLED'
WHERE status = 'PENDING'
  AND created_at < ?;
```

This avoids loading and modifying each entity individually.
The returned integer indicates the number of affected entities reported by the bulk operation.

## 5.2 Bulk operations bypass the persistence context

Bulk JPQL updates and deletes have different semantics from ordinary entity operations.
They execute directly against the database rather than modifying each managed entity.
Consequently, they bypass normal entity dirty checking and do not synchronize already-managed entity instances with the
updated database state.

Consider:

```java
@Transactional
public void example(UUID orderId) {
    Order order = orderRepository.findById(orderId)
            .orElseThrow();

    orderRepository.cancelExpiredOrders(
            OrderStatus.PENDING,
            OrderStatus.CANCELLED,
            Instant.now()
    );

    // The managed order may still contain its previous status.
}
```

Even if the bulk update modifies the row represented by `order`, the managed Java object is not automatically refreshed.
This can lead to stale application state.

Spring Data JPA supports clearing the persistence context after a modifying query:

```java
@Modifying(clearAutomatically = true)
```

It also supports flushing before executing the query:

```java
@Modifying(
    flushAutomatically = true,
    clearAutomatically = true
)
```

These options can be useful, but clearing the persistence context detaches all managed entities.
Pending changes must be handled carefully to avoid losing work that has not been flushed.

Bulk operations also bypass ordinary entity lifecycle callbacks. Applications relying on callbacks, auditing listeners,
or entity-level version checks need to account for these differences.

For bulk updates involving optimistic locking, version handling must be considered explicitly.

Suppose `Order` has a version property:

```java
@Version
private Long version;
```

An ordinary managed-entity update normally includes the expected version in its SQL predicate. A JPQL bulk update does
not automatically perform that per-entity version check or increment the version.

If the intent is to invalidate previously loaded versions after a bulk change, the query can explicitly increment a
numeric version field:

```java
@Transactional
@Modifying
@Query("""
    UPDATE Order o
    SET o.status = :status,
        o.version = o.version + 1
    WHERE o.status = :currentStatus
    """)
int updateStatus(
        @Param("currentStatus") OrderStatus currentStatus,
        @Param("status") OrderStatus status
);
```

This advances the version for affected rows, but it is **not** equivalent to checking an expected version for each
entity. If the operation requires compare-and-swap semantics, the WHERE clause must include the expected version or another
concurrency condition. The application must check the affected-row count against the expected number of updates and
handle mismatches appropriately.

## 5.3 Bulk deletes

The same principle applies to deletes.

For example:

```java
@Transactional
@Modifying
@Query("""
    DELETE FROM Event e
    WHERE e.createdAt < :cutoff
    """)
int deleteOldEvents(@Param("cutoff") Instant cutoff);
```

This avoids loading every entity before deletion.
However, bulk deletes do not invoke entity removal callbacks or automatically apply ORM-level cascading operations.
Database foreign-key constraints and database-defined cascading rules still apply.

For large deletions, a single statement may also create substantial database work.
Deleting millions of rows in one transaction can generate significant WAL activity, hold locks, and leave dead tuples
that PostgreSQL must subsequently reclaim.

Large cleanup operations may therefore benefit from processing rows in smaller chunks or using partitioning when the
data model permits it.

# 6. Bulk Inserts with PostgreSQL

JPQL supports bulk updates and deletes, but it does not provide a portable bulk `INSERT ... VALUES` mechanism equivalent
to PostgreSQL's native SQL capabilities.

For high-volume inserts, native SQL or JDBC may be more appropriate.

## 6.1 Multi-row inserts

PostgreSQL supports inserting multiple rows with a single statement:

```sql
INSERT INTO event (id, type, payload)
VALUES
    (?, ?, ?),
    (?, ?, ?),
    (?, ?, ?);
```

This differs from JDBC batching.
A multi-row insert is one SQL statement containing multiple rows.
JDBC batching groups multiple executions of a parameterized statement.
Both approaches can reduce overhead, but they operate at different levels.

Multi-row inserts can be particularly effective when the application does not require JPA entity lifecycle behavior.
However, very large statements have their own costs, including parameter handling, memory consumption, and statement
construction.

The appropriate number of rows per statement should be measured.

## 6.2 PostgreSQL JDBC batch rewriting

The PostgreSQL JDBC driver supports the `reWriteBatchedInserts` connection property.

The layers involved are distinct:

| Mechanism                       | Responsibility                                                    |
|---------------------------------|-------------------------------------------------------------------|
| Hibernate batching              | Collects compatible entity DML operations for JDBC execution      |
| JDBC batching                   | Submits repeated parameter sets through the driver                |
| PostgreSQL JDBC batch rewriting | May rewrite compatible batched inserts into multi-row SQL         |
| Native multi-row `INSERT`       | Application sends a single SQL statement containing multiple rows |
| PostgreSQL `COPY`               | Uses PostgreSQL's bulk data-loading mechanism                     |

These mechanisms are not all mutually exclusive. For example, Hibernate can submit a JDBC batch that the PostgreSQL
driver subsequently rewrites.

For example:

```properties
jdbc:postgresql://localhost:5432/app?reWriteBatchedInserts=true
```

When applicable, the driver can rewrite compatible batched inserts into multi-row insert statements.
This can reduce protocol and execution overhead beyond ordinary JDBC batching.
However, rewriting depends on the SQL statement and driver behavior.

It should not be assumed that every batched insert will be rewritten.
The generated database workload should be measured with the setting enabled and disabled.

## 6.3 Using JdbcTemplate

Spring's `JdbcTemplate` provides a direct way to execute JDBC batches without creating managed entities.

For example:

```java
@Transactional
public void insertEvents(List<EventData> events) {
    jdbcTemplate.batchUpdate(
            """
            INSERT INTO event (id, type, payload)
            VALUES (?, ?, ?)
            """,
            events,
            500,
            (ps, event) -> {
                ps.setObject(1, event.id());
                ps.setString(2, event.type());
                ps.setString(3, event.payload());
            }
    );
}
```

This avoids the overhead of entity materialization, dirty checking, and persistence-context management.
The application also gains direct control over the SQL being executed.
The tradeoff is that it no longer benefits from JPA entity lifecycle management for these operations.

For straightforward bulk inserts, this may be desirable.
For operations involving complex entity relationships, cascades, or domain behavior, JPA may still provide a more
convenient programming model.

## 6.4 PostgreSQL COPY

For very large data imports, PostgreSQL provides the `COPY` command.

For example:

```sql
COPY event (id, type, payload)
FROM STDIN
WITH (FORMAT CSV);
```

`COPY` is designed for transferring large amounts of data between a client and PostgreSQL.

It can be substantially more efficient than executing individual inserts, even when JDBC batching is enabled.
The PostgreSQL JDBC driver exposes COPY functionality through its `CopyManager` API.
However, COPY requires a different integration approach from ordinary Spring Data repositories.
The application must manage the input format, error handling, and database interaction explicitly.

COPY also does not invoke JPA entity callbacks or manage inserted rows as entities.
It does not bypass PostgreSQL's normal transactional guarantees. Constraints, indexes, and triggers still apply, and
loading into ordinary logged tables generates WAL. The database work may remain substantial even when client-side
insertion overhead is greatly reduced.

It is particularly useful for bulk imports, data migrations, and ingestion pipelines where the application can process
records without ORM lifecycle management.

# 7. Measuring Write Performance

Write optimizations should be evaluated using representative workloads.
A larger JDBC batch size does not automatically translate into proportionally higher throughput.
Performance depends on the database schema, indexes, constraints, network latency, transaction size, and the amount of
work performed by PostgreSQL.

## 7.1 Measuring throughput

A basic throughput metric is:

```
Throughput (rows/sec) = Rows processed / Elapsed time (seconds)
```

For example, inserting 100,000 rows in 10 seconds corresponds to 10,000 rows per second.
However, throughput alone does not describe the complete workload.

A useful benchmark should also consider:

- Total transaction duration.
- Commit latency.
- Application memory consumption.
- CPU utilization.
- Database I/O and WAL generation.
- Lock contention.
- Error and retry behavior.

It is also important to distinguish the time required to queue entity operations from the time required to flush and
commit them. Measuring only the execution time of a `persist()` loop can produce misleading results because Hibernate 
may defer the actual SQL execution.

## 7.2 Comparing write strategies

For a representative insert workload, useful comparisons include:

| Strategy                      | Persistence context | JDBC batching           | Main consideration                     |
|-------------------------------|---------------------|-------------------------|----------------------------------------|
| Individual `save()` calls     | Yes                 | Configuration-dependent | Convenient but potentially expensive   |
| `saveAll()`                   | Yes                 | Configuration-dependent | Not inherently a bulk insert           |
| JPA with periodic flush/clear | Yes, bounded        | Yes, when supported     | Controls persistence-context growth    |
| `JdbcTemplate.batchUpdate()`  | No                  | Yes                     | Lower ORM overhead                     |
| Native multi-row INSERT       | No                  | Not required            | Single SQL statement for multiple rows |
| PostgreSQL COPY               | No                  | Not applicable          | Optimized for large imports            |

The relative performance of these approaches depends on the workload.
For small batches, the convenience of JPA may outweigh any measurable overhead.
For large imports, avoiding entity management entirely can provide substantial benefits.
The objective is to choose the simplest approach that meets the application's throughput and consistency requirements.

# Conclusion

Spring Data JPA provides a convenient way to persist application state, but efficient bulk writes require an
understanding of how Hibernate interacts with JDBC and PostgreSQL.

JDBC batching can reduce the overhead of executing repeated statements. Identifier generation determines whether
Hibernate can batch inserts efficiently. Persistence-context management becomes increasingly important as the number of
entities grows.

For workloads that do not require entity lifecycle management, JDBC batching, native SQL, and PostgreSQL COPY provide
alternatives that can offer better throughput.

The goal is not to replace JPA with SQL everywhere. It is to understand where the abstraction adds value, where it
introduces overhead, and when a more direct database operation is appropriate. Measure the full operation, including
flushing, committing, and database-side work, and preserve the transaction boundaries that correctness requires.
