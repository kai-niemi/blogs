---
title: "Transaction Management with Spring Data"
author: "Kai Niemi"
slug: "transaction-management"
date: 2026-10-25
draft: false
series:
  - data-access
tags:
  - spring-boot
  - spring-data
  - transactions
  - postgresql
categories:
  - Databases  
description: "Transaction management with Spring Boot and Spring Data, covering transaction boundaries, isolation, concurrency, and failure handling."
cover:
  image: "cover.webp"
  caption: 'Photo by [Jimmy  Chang](https://unsplash.com/@photohunter?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/low-angle-photography-of-building-at-daytime-hDOnQGPofuU?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)'
  relative: true
  hiddenInList: true
---

Transactions are fundamental to reliable data access. They group database operations into a unit of work that either
commits or rolls back, preserving consistency when applications encounter concurrent requests, database errors, or
infrastructure failures.

Spring Boot and Spring Data make transaction management relatively straightforward through declarative annotations and
repository abstractions. However, the simplicity of `@Transactional` can obscure important details about transaction
boundaries, database isolation, connection management, and failure handling.

This article examines transaction management in Spring Boot applications using Spring Data JPA and Spring Data JDBC,
with PostgreSQL as the reference database. The emphasis is on understanding what happens beneath the abstractions and
how transaction design affects application correctness and performance.

## 1. Keeping Transactions Short

A database transaction holds resources for its entire lifetime. Depending on the operations performed, these may include
connections, locks, transaction metadata, and database versions required by concurrent readers.

Long-running transactions increase contention and can reduce throughput, particularly in applications with many
concurrent requests.

### 1.1 Transaction Boundaries

Consider a service that transfers money between accounts:

```java
@Service
@RequiredArgsConstructor
public class TransferService {
    
    private final AccountRepository accounts;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void transfer(
            Long sourceId,
            Long targetId,
            BigDecimal amount) {

        Account source = accounts.findById(sourceId)
                .orElseThrow();

        Account target = accounts.findById(targetId)
                .orElseThrow();

        source.debit(amount);
        target.credit(amount);

        accounts.save(source);
        accounts.save(target);
    }
}
```

Both account modifications participate in the same transaction. If either operation fails with an exception that
triggers rollback, neither modification is committed.

The service method provides a natural transaction boundary because it represents a complete business operation.

Placing transactions at the service layer also makes their scope explicit. Individual repository operations should not
determine the atomicity of a workflow involving multiple repositories.

Spring's default `REQUIRED` propagation allows a method to either start a transaction or participate in an existing one.
While convenient, this makes transaction ownership dependent on the calling context.

One architectural convention is to use `REQUIRES_NEW` for application services that own transaction boundaries and
`MANDATORY` for control services that must participate in an existing transaction. This distinguishes transaction
ownership from participation and prevents control services from implicitly establishing their own transactions.

`REQUIRES_NEW` suspends any existing transaction and establishes an independent one. Transaction-owning services should
therefore not be composed within other transactions unless independent commit semantics are intentional.

Spring Data JPA repository methods may already have transactional behavior. For example, inherited CRUD methods in
`SimpleJpaRepository` are transactional, with read operations typically marked read-only.

Spring Data JDBC follows different repository transaction conventions, but the same principle applies: use a
service-level transaction when several database operations must succeed or fail together.

### 1.2 Avoid External Calls Inside Transactions

A common mistake is performing remote operations while holding a database transaction open:

```java
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);

    paymentClient.authorize(order);

    inventoryClient.reserve(order);
}
```

The database transaction remains active while waiting for external services.

This creates several problems:

- Database connections may remain occupied during network calls, particularly after SQL statements have been executed.
- Locks may be held longer than necessary.
- External operations cannot be rolled back by the database.
- Timeouts can leave the outcome of remote operations uncertain.

A better design separates database transactions from external interactions:

```java
public void processOrder(Order order) {
    paymentClient.authorize(order);

    orderTransactionService.createOrder(order);
}
```

```java
@Service
@RequiredArgsConstructor
public class OrderTransactionService {

    private final OrderRepository orders;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void createOrder(Order order) {
        orders.save(order);
    }
}
```

This reduces transaction duration, although it does not make the overall workflow atomic. If the external operation
succeeds and the database transaction fails, reconciliation or compensation may still be necessary.

For workflows requiring reliable coordination between database changes and message publication, the transactional outbox
pattern is generally more appropriate.

See [Best-Effort One-Phase Commit and the Dual-Write Problem](/posts/best-effort-1pc/) for a discussion of these tradeoffs.

### 1.3 Spring Transaction Proxies

Spring typically implements declarative transactions using AOP proxies.

When a caller invokes a transactional method through the Spring proxy, the transaction interceptor starts or joins a
transaction before invoking the target method.

The transaction interceptor commits when the method completes successfully or rolls back when an exception requires it.

{{< mermaid >}}
sequenceDiagram
    participant C as Caller
    participant P as Transaction Proxy
    participant S as Service
    participant R as Repository
    participant D as Database

    C->>P: transfer()
    P->>D: BEGIN
    P->>S: transfer()
    S->>R: findById(), save()
    R->>D: SQL statements
    D-->>R: Results
    R-->>S: Return
    S-->>P: Return
    P->>D: COMMIT
    P-->>C: Return
{{< /mermaid >}}

An important consequence is that **self-invocation does not pass through the proxy**.

```java
@Service
public class TransferService {

    public void execute() {
        transfer(); // Bypasses transactional proxy
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void transfer() {
        // ...
    }
}
```

Here, the annotation on `transfer()` does not establish a new transaction when called directly from `execute()`.

Moving the transactional operation into another Spring-managed service avoids this problem.

For programmatic control, Spring also provides `TransactionTemplate`:

```java
transactionTemplate.executeWithoutResult(status -> {
    accountRepository.debit(sourceId, amount);
    accountRepository.credit(targetId, amount);
});
```

This can be useful when transaction boundaries must be controlled independently of method boundaries.

## 2. Explicit and Implicit Transactions

Understanding how transactions are started is important when working with JDBC, connection pools, and Spring Data abstractions.

### 2.1 Explicit Transactions

An explicit SQL transaction groups multiple statements:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

If an error occurs, the transaction can be rolled back:

```sql
ROLLBACK;
```

In PostgreSQL, a statement error generally leaves an explicit transaction in an aborted state. Subsequent statements
fail until the transaction is rolled back, unless the application uses savepoints to recover.

### 2.2 JDBC Auto-Commit

JDBC connections default to auto-commit mode unless configured otherwise.

With auto-commit enabled, each statement executes as its own transaction when no explicit transaction is active.

```java
connection.setAutoCommit(true);

statement.executeUpdate(
    "UPDATE accounts SET balance = balance - 100 WHERE id = 1"
);

statement.executeUpdate(
    "UPDATE accounts SET balance = balance + 100 WHERE id = 2"
);
```

These statements are not atomic as a group. The first may commit even if the second fails.

Disabling auto-commit allows several statements to participate in one transaction:

```java
connection.setAutoCommit(false);

try {
    // Execute statements

    connection.commit();
} catch (SQLException e) {
    connection.rollback();
    throw e;
}
```

Spring transaction managers normally handle these operations on behalf of the application.

### 2.3 Spring Data and Connection Management

In a typical Spring Boot application, transaction management involves several components:

- Spring's transaction manager determines transaction boundaries.
- The JPA provider or Spring Data JDBC performs persistence operations.
- HikariCP manages the underlying JDBC connection pool.
- The database provides isolation, locking, and commit semantics.

With JPA, the persistence provider may defer acquiring a physical database connection until it is needed. SQL execution
may also be deferred until the persistence context is flushed.

Consequently, the beginning of a Spring transaction does not necessarily correspond to an immediate SQL `BEGIN`
statement.

The exact behavior depends on the persistence provider, transaction manager, connection handling mode, and connection
pool configuration.

When using Hibernate with a connection pool configured to disable auto-commit, the following settings may be relevant:

```yaml
spring:
  datasource:
    hikari:
      auto-commit: false

  jpa:
    properties:
      hibernate:
        connection:
          provider_disables_autocommit: true
```

The Hibernate property tells the provider that connections are already configured with auto-commit disabled.

**These settings must agree.** Setting `provider_disables_autocommit` to `true` without actually disabling auto-commit
can cause incorrect transaction behavior.

This is a configuration choice rather than a universal optimization. It should be evaluated against the application's
connection handling and transaction requirements.

## 3. Transaction Isolation and Concurrency

Transactions provide atomicity, but atomicity alone does not prevent concurrency anomalies.

Isolation determines how concurrent transactions interact and which intermediate or committed changes they can observe.

### 3.1 Isolation Levels

The SQL standard defines four isolation levels:

| Isolation level  | General behavior                                     |
|------------------|------------------------------------------------------|
| Read Uncommitted | Permits the weakest isolation                        |
| Read Committed   | Prevents dirty reads                                 |
| Repeatable Read  | Provides stronger consistency across repeated reads  |
| Serializable     | Requires results equivalent to some serial execution |

Database implementations differ in how they provide these guarantees.

PostgreSQL maps `READ UNCOMMITTED` to `READ COMMITTED` because it does not permit dirty reads. CockroachDB follows the
same convention. Oracle supports neither `READ UNCOMMITTED` nor `REPEATABLE READ`, while MySQL InnoDB supports all four
standard isolation levels, including dirty reads under `READ UNCOMMITTED`.

The SQL standard's isolation levels do not capture every possible concurrency anomaly. For a more detailed examination
of their limitations, see [A Critique of ANSI SQL Isolation Levels](https://blog.acolyer.org/2016/02/24/a-critique-of-ansi-sql-isolation-levels/),
a review of the influential 1995 paper by Berenson et al.

`SERIALIZABLE` is intended to provide the strongest isolation guarantee, but its implementation and guarantees vary
between databases. In PostgreSQL, concurrent transactions may encounter serialization failures and require retries.
These failures are a consequence of enforcing serializability, not a violation of it.

The key takeaway is to understand the actual isolation guarantees provided by your database and whether they are
sufficient to protect your application's invariants. Isolation-level names alone do not tell the whole story.

Spring exposes isolation settings through `@Transactional`:

```java
@Transactional(
        propagation = Propagation.REQUIRES_NEW,
        isolation = Isolation.SERIALIZABLE
)
public void transfer(
        Long sourceId,
        Long targetId,
        BigDecimal amount) {
    // ...
}
```

The setting applies when Spring starts a new physical transaction. A method joining an existing transaction generally
inherits that transaction's isolation level.

### 3.2 Lost Updates

Consider two concurrent transactions updating the same account balance.

Both read an initial balance of 100.

Transaction A subtracts 20, while transaction B subtracts 30.

If each calculates a new balance in application code and writes it without concurrency protection, the final balance may
reflect only one update.

{{< mermaid >}}
sequenceDiagram
    participant A as Transaction A
    participant D as Database
    participant B as Transaction B

    A->>D: SELECT balance
    D-->>A: 100
    B->>D: SELECT balance
    D-->>B: 100

    A->>D: UPDATE balance = 80
    D-->>A: COMMIT

    B->>D: UPDATE balance = 70
    D-->>B: COMMIT

    Note over D: Final balance = 70<br/>Expected balance = 50
{{< /mermaid >}}

Under PostgreSQL's default `READ COMMITTED` isolation, this can happen when an application reads a value and later
writes an absolute replacement value.

Several approaches can prevent it.

**Atomic SQL updates** avoid the separate read-modify-write cycle:

```sql
UPDATE accounts
SET balance = balance - 20
WHERE id = 1 AND balance >= 20;
```

The application must check the affected row count to determine whether the operation succeeded.

**Pessimistic locking** explicitly locks a row before modifying it:

```sql
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
```

Spring Data JPA supports pessimistic locking through `@Lock`:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findForUpdate(Long id);
```

**Optimistic locking** detects conflicting updates using a version column:

```java
@Entity
public class Account {

    @Id
    private Long id;

    @Version
    private Long version;

    private BigDecimal balance;
}
```

The JPA provider checks the entity's version when updating it. If another transaction has modified the entity in the
meantime, the update fails with an optimistic locking exception.

```sql
UPDATE accounts
SET balance = ?, version = version + 1
WHERE id = ? AND version = ?;
```

These mechanisms have different performance and contention characteristics. The appropriate choice depends on the access
pattern, expected conflicts, and business invariants.

A separate article in this series will examine concurrency control in greater detail.

## 4. Read-Only Transactions

Spring supports read-only transaction declarations:

```java
@Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
)
public AccountSummary getAccount(Long id) {
    return accountRepository.findById(id)
            .map(AccountSummary::from)
            .orElseThrow();
}
```

The meaning of `readOnly = true` depends on the transaction manager and persistence provider.

With Hibernate as the JPA provider, Spring can optimize read-only transactions by adjusting the flush mode and disabling
dirty checking for loaded entities.

This may improve performance for read-heavy workloads involving managed entities.

However, **Spring's read-only declaration is not necessarily enforced by the database**.

A read-only transaction can therefore be a performance hint rather than a strict guarantee that no modifications occur.

PostgreSQL provides database-enforced read-only transactions:

```sql
BEGIN READ ONLY;

SELECT * FROM accounts WHERE id = 1;

COMMIT;
```

Operations that attempt prohibited modifications fail.

Spring's `DataSourceTransactionManager` can be configured to enforce read-only transactions at the database level, where
supported, through `setEnforceReadOnly(true)`.

This behavior should not be assumed for JPA transaction managers.

For applications requiring strict protection against writes, database privileges and explicit database-level read-only
enforcement provide stronger guarantees than application annotations alone.

## 5. Handling Transaction Failures

Transactions can fail for reasons unrelated to application validation.

Common examples include serialization conflicts, deadlocks, constraint violations, timeouts, and lost database connections.

Correct failure handling requires distinguishing errors that can be retried from errors whose outcome is uncertain.

### 5.1 Serialization Failures and Deadlocks

PostgreSQL reports serialization failures using SQLSTATE `40001`.

These can occur under `SERIALIZABLE` isolation when concurrent transactions cannot be committed in a serializable order.

Deadlocks are reported using SQLSTATE `40P01`.

For example:

{{< mermaid >}}
sequenceDiagram
participant A as Transaction A
participant D as Database
participant B as Transaction B

    A->>D: UPDATE account 1
    Note over A,D: A holds lock on account 1

    B->>D: UPDATE account 2
    Note over D,B: B holds lock on account 2

    A->>D: UPDATE account 2
    Note over A,D: A waits for B

    B->>D: UPDATE account 1
    Note over D,B: B waits for A

    Note over A,B: Deadlock detected
    D-->>B: ERROR 40P01
    B->>D: ROLLBACK
    D-->>A: Lock acquired
{{< /mermaid >}}

PostgreSQL detects the deadlock and aborts one transaction.

Both serialization failures and deadlocks may be transient. Retrying the complete transaction can succeed once the
conflicting activity has completed.

The important requirement is that **the retry must start a new transaction**.

Retrying an individual SQL statement inside a transaction that has already been aborted is generally insufficient.

### 5.2 Ambiguous Commit Outcomes

A more difficult failure occurs when the connection is lost during commit.

Consider:

{{< mermaid >}}
sequenceDiagram
    participant A as Application
    participant D as Database

    A->>D: BEGIN
    A->>D: UPDATE accounts
    D-->>A: Success
    A->>D: COMMIT
    Note over A,D: Connection lost
    Note over A: Commit outcome unknown
{{< /mermaid >}}

The application cannot determine from the connection failure alone whether the transaction committed.

Two outcomes are possible:

1. The database committed successfully, but the acknowledgment was lost.
2. The transaction did not commit.

Blindly retrying a non-idempotent operation can therefore execute it twice.

This distinction is particularly important for payments, transfers, and other operations that must not be duplicated.

The usual solution is to associate the logical operation with a stable identifier and make duplicate execution detectable.

For example:

```sql
CREATE TABLE transfers (
    id UUID PRIMARY KEY,
    source_account BIGINT NOT NULL,
    target_account BIGINT NOT NULL,
    amount NUMERIC(19, 2) NOT NULL
);
```

The transfer identifier can serve as an idempotency key when it is inserted in the same transaction as the account updates.

A retry using the same identifier must not apply the transfer again.

The database's uniqueness constraint provides the foundation for duplicate detection, while application logic determines
how an existing operation is handled.

See [Designing Idempotent REST APIs](/posts/designing-idempotent-rest-apis/) for a more detailed treatment of idempotency keys and request semantics.

### 5.3 Transaction Retry Boundaries

Spring Retry can be used to retry transient transaction failures.

A critical detail is the ordering of retry and transaction interceptors.

The retry interceptor must execute outside the transaction interceptor so that every attempt receives a fresh transaction.

{{< mermaid >}}
sequenceDiagram
    participant C as Caller
    participant R as Retry Interceptor
    participant T as Transaction Interceptor
    participant D as Database

    C->>R: execute()

    R->>T: Attempt 1
    T->>D: BEGIN
    T->>D: SQL operations
    D-->>T: SQLSTATE 40001
    T->>D: ROLLBACK
    T-->>R: Exception

    Note over R: Backoff

    R->>T: Attempt 2
    T->>D: BEGIN
    T->>D: SQL operations
    T->>D: COMMIT
    T-->>R: Success
    R-->>C: Success
{{< /mermaid >}}

One way to make the boundaries explicit is to separate the retrying service from the transactional service:

```java
@Service
@RequiredArgsConstructor
public class TransferRetryService {

    private final TransferService transfers;

    @Retryable(
        retryFor = {
            CannotSerializeTransactionException.class,
            DeadlockLoserDataAccessException.class
        },
        maxAttempts = 3,
        backoff = @Backoff(
                delay = 100,
                multiplier = 2,
                maxDelay = 2000,
                random = true
        )
    )
    public void transfer(TransferCommand command) {
        transfers.execute(command);
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class TransferService {

    private final AccountRepository accounts;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void execute(TransferCommand command) {
        // Execute the complete transfer
    }
}
```

Each retry invokes the transactional service through its Spring proxy. With `REQUIRES_NEW` propagation, every attempt
starts an independent transaction, even when the caller has an existing transaction. A failed attempt is rolled back
before the retry interceptor initiates another attempt.

Actual exception types depend on the persistence framework, transaction manager, and exception translation. Retry
policies should be tested against the database errors the application encounters.

The example also assumes that the transaction does not perform irreversible external side effects.

Exponential backoff reduces retry frequency under contention, but deterministic delays can cause concurrent transactions
to retry at approximately the same time. This may lead to repeated conflicts, particularly during periods of high
concurrency.

Adding jitter introduces randomness into retry delays, spreading competing transactions over time and reducing
synchronized retry attempts. Spring Retry supports randomized exponential backoff through `@Backoff(random = true)`.

Retries should remain bounded, with a maximum delay and a limited number of attempts. Jitter reduces the probability of
repeated contention but does not eliminate the underlying conflicts.

### 5.4 Rollback Rules

By default, Spring rolls back declarative transactions for unchecked exceptions and `Error` subclasses.

Checked exceptions do not trigger rollback unless configured.

For example:

```java
@Transactional(
    rollbackFor = BusinessException.class
)
public void execute() throws BusinessException {
    // ...
}
```

This distinction matters when service methods use checked exceptions to report failures that should invalidate the transaction.

Another subtle case is catching an exception inside a transactional method:

```java
@Transactional
public void process() {
    try {
        repository.performOperation();
    } catch (DataAccessException e) {
        log.warn("Operation failed", e);
    }
}
```

Catching the exception does not necessarily restore the transaction to a usable state.

In PostgreSQL, a statement error can leave the underlying transaction aborted. Spring may also have marked the
transaction rollback-only.

If the method subsequently returns normally, the transaction may still fail during commit.

Failures that invalidate the unit of work should generally propagate to the transaction boundary, where rollback and
retry decisions can be made consistently.

## Conclusion

Spring Boot and Spring Data simplify transaction management, but they do not eliminate the underlying database semantics.

Reliable transaction design starts with choosing appropriate boundaries and keeping them short. It also requires
understanding when connections and transactions are established, how isolation affects concurrent operations, and which
failures can safely be retried.

Three principles are particularly important:

- Define transaction boundaries around complete database operations rather than individual repository calls.
- Use concurrency control appropriate to the business invariants, not merely the default isolation level.
- Treat transient transaction failures and ambiguous commit outcomes as different problems.

Transactions provide atomicity within the database. Correctness across retries, external systems, and network failures
requires additional application-level design.

## References

- [Spring Framework: Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Spring Data JPA: Transactionality](https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html)
- [Spring Data JDBC: Transaction Management](https://docs.spring.io/spring-data/relational/reference/jdbc/transactions.html)
- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [PostgreSQL: Error Codes](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [A Critique of ANSI SQL Isolation Levels](https://blog.acolyer.org/2016/02/24/a-critique-of-ansi-sql-isolation-levels/)

Part of the [Data Access](/series/data-access/) series.