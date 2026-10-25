---
title: "Best-Effort One-Phase Commit and the Dual-Write Problem"
date: 2024-05-15
lastmod: 2026-10-24
draft: false
description: "Understanding best-effort one-phase commit, dual-write failures, transaction ordering, and idempotency when coordinating databases, message brokers, and remote APIs."
tags:
  - distributed-systems
  - transactions
  - spring-boot
  - java
  - messaging
  - idempotency
categories:
  - Distributed Systems  
cover:
  image: "cover.webp"
  caption: 'Photo by [Omar:. Lopez-Rincon](https://unsplash.com/@procopiopi?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/molecular-structures-are-seen-against-an-orange-backdrop-XkPNEqAhlaI?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)'
  relative: true
  hiddenInList: true
---

Distributed applications frequently need to coordinate operations across multiple transactional resources. A service
might update a database and acknowledge a message, or persist a record and invoke a remote API.

The challenge is that these operations are independent. A successful database commit does not guarantee that a message
acknowledgement or remote API call will succeed.

One commonly adopted strategy is **best-effort one-phase commit (1PC)**. Rather than introducing a distributed
transaction coordinator, it relies on careful transaction ordering and idempotent processing to reduce the risk of
inconsistent outcomes.

This approach is simpler than XA/two-phase commit (2PC), but it comes with an important trade-off: **there is no
atomicity across the participating resources**.

The approach is also discussed in David Syer's
article [Distributed Transactions in Spring, with and without XA](https://www.infoworld.com/article/2077963/distributed-transactions-in-spring--with-and-without-xa.html).

## The Dual-Write Problem

A dual write occurs when an application performs two operations that must logically succeed together but cannot be
committed atomically.

Consider a service that processes an incoming message and updates a database:

{{< mermaid >}}
sequenceDiagram
participant B as Message Broker
participant A as Application
participant D as Database

    B->>A: Deliver message
    A->>D: Begin transaction
    A->>D: Update records
    A->>D: Commit transaction
    D-->>A: Commit successful
    A->>B: Acknowledge message
{{< /mermaid >}}

The database commit and message acknowledgement are separate operations. Even if both are individually transactional,
there is no shared transaction covering them.

Suppose the database commit succeeds but the application crashes before acknowledging the message. The broker may
redeliver it, causing the application to repeat the database operation.

Reversing the order creates a different problem. If the application acknowledges the message before committing the
database transaction, a subsequent database failure may cause the message to be lost without its intended effects being
persisted.

This is the fundamental problem that best-effort 1PC attempts to manage.

## Best-Effort One-Phase Commit

Best-effort 1PC avoids coordinating multiple resources through a global transaction manager. Instead, each resource
commits independently, with operations ordered to minimize the risk of inconsistency.

The term best-effort reflects that the application attempts to coordinate the completion of multiple operations without
a global atomic commit protocol. It cannot guarantee that all participating resources reach the same outcome.

The general strategy is to:

1. Perform business validation before committing.
2. Complete database operations before acknowledging or committing the incoming message.
3. Make processing idempotent so that retries do not repeat business effects.

The intention is to expose predictable failures, such as invalid input and business rule violations, before
acknowledging successful processing.

Infrastructure failures can still occur between commits, but these are handled through redelivery and idempotent
processing rather than atomic coordination.

Best-effort 1PC therefore provides a practical compromise between implementation complexity and consistency guarantees.
It does not make independent commits atomic.

## Database and Message Broker

Consider a Spring Boot service that consumes registration events from a JMS broker and persists them in a relational
database.

A simplified consumer might look like this:

```java
@Component
public class RegistrationConsumer {
    private final RegistrationRepository repository;

    public RegistrationConsumer(RegistrationRepository repository) {
        this.repository = repository;
    }

    @JmsListener(destination = "registrations")
    @Transactional
    public void receiveMessage(RegistrationEvent event) {
        repository.save(toEntity(event));
    }
}
```

The intended processing sequence is:

1. The broker delivers a message.
2. The application begins a database transaction.
3. The application validates and processes the message.
4. The database transaction commits.
5. The broker acknowledges the message.

The example assumes that the JMS listener container is configured to acknowledge or commit message consumption only
after successful processing. The exact behavior depends on the listener container, transaction manager, and
acknowledgement configuration. The `@Transactional` annotation alone does not establish a distributed transaction
between JMS and the database.

### Why Commit Ordering Matters

Suppose the database transaction fails because of a constraint violation.

If the message has not yet been acknowledged, the broker can redeliver it according to its retry and error-handling
configuration. The database changes are rolled back, and the message remains available for further processing.

Now consider a different failure:

```text
1. Receive message
2. Update database
3. Commit database       SUCCESS
4. Acknowledge message   FAILURE
5. Broker redelivers message
```

The application may process the same message again even though the database update was committed successfully.
This is why **idempotency is essential for reliable at-least-once message processing**.

For example, a registration event could contain a stable event identifier:

```java
public record RegistrationEvent(
    UUID eventId,
    UUID customerId,
    String email
) {}
```

The consumer can use that identifier to detect previously processed events.
A common approach is to maintain a table of processed message identifiers and update it in the same database transaction
as the business operation:

```sql
CREATE TABLE processed_messages (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

When processing a message, the application attempts to insert its identifier. If the identifier already exists, the
message has been processed before and its business effects should not be repeated.

The deduplication record and business update must be committed atomically in the same database transaction. Otherwise,
the application introduces another consistency gap.

### Database Flush and Commit

When using JPA, database operations may be deferred until the persistence context is flushed.

Calling `repository.save()` does not necessarily mean that the corresponding SQL statements have been executed. The JPA
provider may defer inserts, updates, and deletes until flush or transaction commit, at which point database constraints
may also be evaluated.

Explicitly flushing the persistence context can expose some failures earlier, but it does not guarantee that the
transaction will commit successfully. Deadlocks, serialization failures, connection loss, and other commit-time failures
may still occur.

The important requirement is that the database transaction commits successfully before the broker acknowledges the
message.

### What About Kafka?

The same general problem applies to Kafka consumers, although the mechanics differ.

Unlike traditional message queues, Kafka does not remove records when consumers acknowledge processing. Instead,
consumers track their progress through offsets, while records are retained according to the topic's configured retention
and cleanup policies.

A consumer that updates a database and commits its offset performs two independent operations. If the database commit
succeeds but the offset commit fails, the record may be processed again after a restart or rebalance.

Kafka provides additional guarantees through **idempotent producers** and **transactions**.

Idempotent producers prevent duplicate records caused by retries of the same produce operation, using producer
identities and sequence numbers maintained by Kafka. 

Kafka transactions extend these guarantees by allowing produced records and consumed offsets to be
committed atomically within Kafka, supporting exactly-once processing for Kafka-to-Kafka workflows when consumers are
configured appropriately.

However, neither mechanism automatically extends atomicity to an external relational database. A database commit and a
Kafka offset commit remain independent operations.

Idempotent database processing therefore remains important when consuming Kafka records and updating external
transactional resources.

## Database and Remote API

A second scenario involves updating a database and invoking a remote API.

Suppose a service needs to initiate a payment through an external provider and record the result locally.

A tempting implementation is:

```java
@Transactional
public void createTransfer(TransferRequest request) {
    paymentClient.createTransfer(request);
    transferRepository.save(toEntity(request));
}
```

This performs a remote call while holding a database transaction open.

There are several problems with this approach. Network calls can be slow or fail unpredictably, increasing transaction
duration and potentially holding database locks or connections longer than necessary.

More importantly, the remote API call is not part of the database transaction.

If the API succeeds but the database transaction rolls back, the external payment may still have been created.

### Reordering Does Not Eliminate the Problem

One improvement is to invoke the remote API outside the database transaction:

```text
1. Call remote API
2. Begin database transaction
3. Persist result
4. Commit database transaction
```

This avoids holding a database transaction open during the network call, but the two operations remain independent.

Consider:

```text
1. POST /transfers        SUCCESS
2. Database transaction   FAILURE
3. Application retries
4. POST /transfers        SUCCESS
```

Without additional safeguards, the retry may create a second transfer.

Reversing the order does not solve the problem either:

```text
1. Persist transfer       SUCCESS
2. Commit database        SUCCESS
3. POST /transfers        FAILURE
```

Now the database records a transfer that may not have been created remotely.

The correct ordering depends on the business operation and its recovery requirements. Neither ordering provides
atomicity.

### Idempotency Keys

For APIs that support idempotency, the application can associate a stable identifier with the logical operation.

For example:

```http
POST /transfers HTTP/1.1
Content-Type: application/json
Idempotency-Key: 7d9e2b8a-6f41-4e73-9a2c-814cbdf37e61

{
  "sourceAccount": "A123",
  "destinationAccount": "B456",
  "amount": 100.00,
  "currency": "EUR"
}
```

If the request is retried using the same idempotency key, the receiving service can recognize that the operation has
already been processed and avoid creating another transfer.

This also addresses an important ambiguity in distributed systems: a request may succeed even when the client never
receives its response.

{{< mermaid >}}
sequenceDiagram
participant C as Client
participant A as Remote API

    C->>A: POST /transfers
    activate A
    Note right of A: Process transfer
    Note right of A: Commit result
    A--xC: Response lost (connection failure)
    deactivate A

    C->>A: Retry with same idempotency key
    activate A
    Note right of A: Recognize duplicate
    A-->>C: Return original result
    deactivate A
{{< /mermaid >}}

The client cannot determine whether the original request succeeded merely from a timeout. A stable idempotency key
allows the remote service to associate retries with the same logical operation.

The receiving service must enforce this guarantee. Simply sending an idempotency key does not make an endpoint
idempotent. Idempotency keys must remain stable across retries of the same logical operation. Generating a new key for
each retry defeats the purpose.

For a deeper discussion, see [Designing Idempotent REST APIs](/posts/designing-idempotent-rest-apis/).

## Alternatives to Best-Effort 1PC

Best-effort 1PC is not the only approach to coordinating operations across transactional boundaries. Several patterns
provide different trade-offs between complexity, availability, and consistency.

### Transactional Outbox

The transactional outbox pattern addresses the problem of updating a database and publishing an event.

Instead of writing directly to the database and message broker, the application stores both the business update and an
outgoing event in the same database transaction.

{{< mermaid >}}
flowchart TD
A["Application"] --> T

    subgraph T["Database Transaction"]
        direction LR
        B["Business Data"]
        O["Outbox Event"]
    end

    T --> C["COMMIT"]
    C --> P["Outbox Publisher"]
    P -->|At-least-once delivery| M["Message Broker"]
{{< /mermaid >}}

A separate process publishes committed outbox events to the broker, commonly through polling or change data capture
(CDC). This removes the non-atomic database-and-broker write from the application transaction. However, publishing may
still happen more than once, so consumers must tolerate duplicate events.

### Idempotent Consumer and Inbox

The inbox pattern complements the outbox by recording incoming messages or their identifiers in the receiving database.
The consumer can atomically record that a message has been processed alongside the corresponding business update.
If the broker redelivers the same message, the consumer recognizes the duplicate and avoids repeating its effects.

{{< mermaid >}}
sequenceDiagram
participant B as Message Broker
participant C as Consumer
participant D as Database

    B->>C: Deliver message (eventId)
    C->>D: Begin transaction
    C->>D: Insert eventId into inbox

    alt New message
        D-->>C: Insert successful
        C->>D: Apply business updates
        C->>D: Commit transaction
        D-->>C: Commit successful
    else Already processed
        D-->>C: Duplicate eventId
        C->>D: Rollback transaction
    end

    C->>B: Acknowledge message
{{< /mermaid >}}

A duplicate identifier should only be treated as successfully processed once the original transaction has committed.
Concurrent attempts must be resolved through the database's uniqueness and transaction semantics.

This provides a practical way to achieve effectively-once business processing on top of at-least-once message delivery,
provided the deduplication and business updates share the same transaction.

### Saga

A saga coordinates a business process across multiple independently committed local transactions. Each participating
service maintains its own state and uses local ACID transactions, while the overall workflow progresses through a
sequence of state transitions.

Sagas are commonly used in microservice architectures where business transactions span multiple service boundaries.
Coordination can be orchestrated through a central coordinator or choreographed through asynchronous events.

For example, an order workflow might:

1. Reserve inventory.
2. Authorize payment.
3. Confirm the order.

If payment authorization fails, the system can release the inventory reservation through a compensating action.

Unlike XA/2PC, sagas do not provide atomicity across participating services. Intermediate states may be visible, and
compensation is not equivalent to an atomic rollback. Some operations cannot be fully reversed.

The benefit is greater service autonomy and reduced coupling between transactional resources. The trade-off is
additional complexity in managing distributed state transitions, retries, compensation, and failure recovery.
Asynchronous workflows can also be harder to trace and reason about than a single database transaction.

Sagas are often combined with transactional outbox and idempotent consumer patterns to make message-driven coordination
more reliable.

### XA and Two-Phase Commit

Two-phase commit (2PC) resembles a wedding ceremony, with the transaction coordinator acting as the priest.

In the prepare phase, each participant promises to commit if instructed, much like exchanging vows. Once everyone
agrees, the coordinator records the decision and instructs them to commit. If anyone refuses, the transaction is rolled
back.

But what if the priest disappears before announcing the decision? Participants may be left waiting, unable to proceed
independently. This illustrates the blocking problem of 2PC.

XA defines a standardized interface through which a transaction manager coordinates compatible transactional resources,
typically using 2PC to provide atomic commit guarantees.

Not every database, broker, or external service supports XA. In particular, ordinary HTTP API calls cannot generally
participate in such a transaction.

## When Is Best-Effort 1PC Appropriate?

Best-effort 1PC is useful when operations span independent resources, the remaining failure windows are acceptable, and
recovery through retries is practical.

It is particularly suitable for message-driven processing where the broker supports redelivery and the database
operation can be made idempotent.

However, idempotency is not a substitute for every consistency guarantee. It prevents repeated execution from creating
duplicate effects, but it does not ensure that an operation which has never completed will eventually succeed.

For critical workflows, the design must also account for retry exhaustion, permanently failing operations, ambiguous
outcomes, and reconciliation between systems.

The choice between best-effort 1PC, an outbox, a saga, or distributed transactions depends on the required guarantees
rather than implementation convenience alone.

## Conclusion

Best-effort one-phase commit is a pragmatic alternative to distributed transactions when independent resource commits
are acceptable. By committing the database before acknowledging incoming messages, applications can favor redelivery
over message loss. Idempotency makes that redelivery safe, but does not eliminate the possibility of partial failures.

For workflows requiring stronger recovery guarantees, patterns such as transactional outbox, inbox, and saga provide
additional options without necessarily introducing XA.