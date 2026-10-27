---
title: "Write Patterns with Spring Data"
weight: 2
date: 2026-10-26
draft: false
series:
  - data-access
categories:
  - Databases
description: "Atomic updates, conditional writes, locking and upserts with Spring Data and PostgreSQL."
tags: ["Spring Data", "PostgreSQL", "JPA", "JDBC", "SQL", "Transactions"]
cover:
  image: "cover.webp"
  caption: 'Photo by [Debby Hudson](https://unsplash.com/@hudsoncrafted?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/purple-flowers-on-paper-DR31squbFoA?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)'
  relative: true
  hiddenInList: true
---

# Introduction

This article is the second in a series about data access with Spring Data.

{{< series-nav >}}

This time, we'll focus on how to structure write operations to reduce contention, avoid unnecessary database round
trips, and preserve application invariants.

A common approach when working with JPA is to load an entity, modify its state, and let the persistence provider
synchronize the changes with the database. This is convenient and often appropriate, but it isn't always the most
efficient way to express an operation.

In some cases, we can achieve better concurrency and performance by expressing the modification directly in SQL.

The examples use PostgreSQL, Spring Data JPA, and Spring JDBC. Most principles apply to relational databases in general,
although their concurrency behavior may differ.

# 1. Avoiding Read-Modify-Write

A common persistence pattern is to read data from the database, modify it in application memory, and write the changes
back.

For example:

```java
Account account = accountRepository.findById(accountId)
        .orElseThrow();

account.setBalance(account.getBalance().add(amount));
```

With JPA, the persistence provider tracks changes to managed entities and eventually issues an `UPDATE`, typically when
the persistence context is flushed.

Without optimistic locking, the resulting SQL conceptually looks like this:

```sql
SELECT id, balance
FROM account
WHERE id = ?;

UPDATE account
SET balance = ?
WHERE id = ?;
```

This is commonly referred to as the *read-modify-write* pattern. It provides a convenient programming model, but
introduces a dependency between the value previously read and the value subsequently written.

## 1.1 The concurrency problem

Consider an account with an initial balance of 100. Two concurrent transactions attempt to withdraw 30 and 50,
respectively.

The following example illustrates how concurrent read-modify-write operations can lead to a lost update.

{{< mermaid >}}
sequenceDiagram
participant T1 as Transaction 1
participant DB as PostgreSQL (READ COMMITTED)
participant T2 as Transaction 2

    Note over DB: Initial balance = 100

    T1->>DB: SELECT balance
    DB-->>T1: 100

    T2->>DB: SELECT balance
    DB-->>T2: 100

    Note over T1: Calculate 100 - 30 = 70
    Note over T2: Calculate 100 - 50 = 50

    T1->>DB: UPDATE balance = 70
    T1->>DB: COMMIT

    T2->>DB: UPDATE balance = 50
    T2->>DB: COMMIT

    Note over DB: Final balance = 50 (lost update)
{{< /mermaid >}}

The expected balance is 20, but the second transaction overwrites the first transaction's modification. This is a **lost
update**.

Whether this sequence can commit depends on the database's isolation level and concurrency-control implementation.

For example, PostgreSQL's default `READ COMMITTED` isolation does not automatically prevent this form of lost update
when the application writes a previously calculated value without additional concurrency control. Under PostgreSQL's
`REPEATABLE READ` or `SERIALIZABLE` isolation, a conflicting concurrent update can instead cause a transaction to abort.

JPA optimistic locking using `@Version` would also detect this conflict and prevent the second transaction from silently
overwriting the first.

The problem is not that reading an entity before updating it is inherently incorrect. The problem is that the
application makes a decision using a value that may no longer represent the current database state when the modification
takes place.

## 1.2 Use atomic updates

When the operation can be expressed as a modification of an existing value, the initial read may be unnecessary.

Instead of:

```sql
SELECT balance
FROM account
WHERE id = :id;

UPDATE account
SET balance = :newBalance
WHERE id = :id;
```

Use:

```sql
UPDATE account
SET balance = balance + :amount
WHERE id = :id;
```

The database evaluates the expression as part of the `UPDATE` statement.

In PostgreSQL, concurrent updates to the same row are coordinated through row locking. Under `READ COMMITTED`, a
transaction waiting for another update can apply its expression to the updated row after the preceding transaction
commits.

{{< mermaid >}}
sequenceDiagram
participant T1 as Transaction 1
participant DB as PostgreSQL (READ COMMITTED)
participant T2 as Transaction 2

    Note over DB: Initial balance = 100

    T1->>DB: UPDATE balance = balance - 30
    Note over DB: Uncommitted balance = 70
    Note over T1,DB: T1 holds the row lock

    T2->>DB: UPDATE balance = balance - 50
    Note over T2,DB: T2 waits for the row lock

    T1->>DB: COMMIT
    Note over DB: T2 resumes using balance = 70

    DB-->>T2: UPDATE succeeds
    T2->>DB: COMMIT

    Note over DB: Final balance = 20
{{< /mermaid >}}

Both modifications can be preserved without an application-level read-modify-write sequence.

Atomic updates avoid lost updates caused by application-side calculations, but they do not eliminate transaction
conflicts. Under serializable isolation, concurrent modifications may still require retries.

If the row is also managed through JPA optimistic locking, the update must increment its version, as shown in the next
section.

The benefit is that the database can perform the modification without an unnecessary read round trip or a calculation
based on a potentially stale application-side copy.

## 1.3 Protect invariants with conditional updates

Atomic increments solve only part of the problem. Suppose an account must never have a negative balance unless
explicitly permitted.

A conventional approach is to read the balance, validate the withdrawal, and then update the account.

```java
Account account = accountRepository.findById(accountId)
        .orElseThrow();

BigDecimal newBalance = account.getBalance().add(amount);

if (!account.isAllowNegative() && newBalance.signum() < 0) {
    throw new InsufficientFundsException();
}

account.setBalance(newBalance);
```

The validation is performed against the value previously read. Instead, we can combine the modification and its
precondition in a single SQL statement:

```sql
UPDATE account
SET balance = balance + :amount,
    version = version + 1
WHERE id = :id
  AND closed = false
  AND (
      allow_negative = true
      OR balance + :amount >= 0
  );
```

The database updates the account only when the predicate is satisfied. The operation now expresses both the modification
and the invariant in SQL.

The version is incremented because the example assumes that the same account may also be modified through JPA entities
using `@Version`. Keeping the version consistent allows those operations to detect intervening modifications.

If the statement affects one row, the update succeeded. If it affects no rows, the account may not exist, may be closed,
or may have insufficient funds. The application must distinguish these outcomes where necessary.

With Spring's `JdbcTemplate`, we can execute the statement directly:

```java
@Repository
public class AccountWriter {

    private final JdbcTemplate jdbcTemplate;

    public AccountWriter(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Transactional(propagation = Propagation.MANDATORY)
    public void adjustBalance(UUID accountId, BigDecimal amount) {
        int updated = jdbcTemplate.update("""
                UPDATE account
                SET balance = balance + ?,
                    version = version + 1
                WHERE id = ?
                  AND closed = false
                  AND (
                      allow_negative = true
                      OR balance + ? >= 0
                  )
                """,
                amount,
                accountId,
                amount
        );

        if (updated != 1) {
            throw new BalanceUpdateRejectedException(accountId);
        }
    }
}
```

The custom exception should be unchecked so that Spring's default rollback rules apply.

The `MANDATORY` propagation setting requires an existing transaction. When the operations use the same transaction
manager and data source, they participate in the caller's transaction rather than establishing independent transaction
boundaries.

The same operation can be expressed through a Spring Data JPA repository using a modifying query:

```java
@Modifying
@Transactional(propagation = Propagation.MANDATORY)
@Query("""
    UPDATE Account a
    SET a.balance = a.balance + :amount,
        a.version = a.version + 1
    WHERE a.id = :id
      AND a.closed = false
      AND (
          a.allowNegative = true
          OR a.balance + :amount >= 0
      )
    """)
int adjustBalance(
        @Param("id") UUID id,
        @Param("amount") BigDecimal amount
);
```

The example assumes the JPA provider supports arithmetic expressions for the mapped balance and version types. A native
SQL query is another option.

One important consideration is that JPQL bulk updates bypass the persistence context and do not automatically increment
entity versions. If the entity uses optimistic locking through `@Version`, the update must account for version
management explicitly. Otherwise, concurrent modifications may go undetected.

Incrementing the version ensures that a previously loaded entity with an older version cannot silently overwrite the
modification. However, the bulk update itself does not check an expected version. It instead relies on the database's
concurrency control and the conditional update predicate.

Bulk updates also leave already managed entities potentially stale. Mixing them with ordinary JPA entity updates in the
same persistence context therefore requires care.

For operations that depend on exact SQL behavior, JDBC is often the simpler choice.

## 1.4 Keep related writes atomic

A balance adjustment rarely happens in isolation. A financial transfer may involve updating two accounts and recording
corresponding ledger entries.

These operations must execute within the same database transaction. If the source account update fails, the entire
transaction must roll back. Otherwise, the application could record a transfer or credit the destination account without
successfully debiting the source.

Following the transaction ownership conventions discussed in the previous article, an application service can establish
the transaction boundary:

```java
@Service
public class TransferService {

    private final TransferRepository transferRepository;
    private final AccountWriter accountWriter;

    public TransferService(
            TransferRepository transferRepository,
            AccountWriter accountWriter) {
        this.transferRepository = transferRepository;
        this.accountWriter = accountWriter;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void transfer(
            UUID sourceId,
            UUID destinationId,
            BigDecimal amount) {

        if (amount.signum() <= 0) {
            throw new IllegalArgumentException(
                    "Transfer amount must be positive");
        }

        accountWriter.adjustBalance(sourceId, amount.negate());
        accountWriter.adjustBalance(destinationId, amount);

        transferRepository.recordTransfer(
                sourceId, destinationId, amount);
    }
}
```

Concurrent transfers involving the same accounts can deadlock when they acquire row locks in different orders.
Production implementations should consider consistent lock ordering and retry handling for deadlocks and serialization
failures.

The example assumes the accounts exist and the transfer is otherwise valid. Production code would also need to address
currency compatibility, account ownership, idempotency, and other business rules.

The participating repository operations must use the same transaction. They must not start independent transactions that
commit separately. The transfer-recording operation should also require an existing transaction, for example through
`MANDATORY` propagation.

If any operation fails, the application should allow the exception to propagate so the transaction rolls back.

Conditional updates let the database evaluate business preconditions at the point of modification. They do not replace
transaction management when several statements must succeed or fail together.

# 2. When Read-Modify-Write Is Necessary

Not every operation can be reduced to a single SQL statement. Sometimes the application must inspect several entities,
apply complex business logic, or perform calculations that cannot reasonably be expressed in SQL.

In those cases, read-modify-write remains a valid approach, provided concurrency is handled correctly.

## 2.1 Pessimistic locking

One option is to acquire locks when reading the rows that will subsequently be modified.

In PostgreSQL:

```sql
SELECT id, balance
FROM account
WHERE id IN (:ids)
ORDER BY id
FOR UPDATE;
```

`FOR UPDATE` acquires row-level locks that prevent other transactions from concurrently modifying the selected rows
until the locks are released.

Under `READ COMMITTED`, a concurrent transaction attempting to lock the same rows will generally wait until the
preceding transaction finishes.

With Spring Data JPA:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("""
    SELECT a
    FROM Account a
    WHERE a.id IN :ids
    ORDER BY a.id
    """)
List<Account> findForUpdate(@Param("ids") Collection<UUID> ids);
```

The query must execute within an active transaction for the locks to remain held throughout the operation.

Acquiring locks in a consistent order can reduce deadlock risk when transactions modify overlapping sets of rows. The
actual locking behavior depends on the database and execution plan.

Pessimistic locking does not eliminate contention or guarantee that transactions will never fail. Deadlocks, lock
timeouts, and serialization failures remain possible.

## 2.2 Optimistic locking

Another approach is optimistic locking using a version column.

With JPA:

```java
@Entity
public class Account {

    @Id
    private UUID id;

    @Version
    private Long version;

    private BigDecimal balance;
}
```

The JPA provider includes the expected version in the update predicate.

Conceptually:

```sql
UPDATE account
SET balance = :balance,
    version = version + 1
WHERE id = :id
  AND version = :expectedVersion;
```

If the `UPDATE` affects no rows because the version no longer matches, the JPA provider detects an optimistic locking
conflict.

This prevents one transaction from silently overwriting changes made after the entity was read.

Optimistic locking is useful when the application needs to detect stale entity state, including across separate
transactions when the originally observed version is retained and checked during the subsequent modification.

Within a single transaction running under true serializable isolation, optimistic locking generally does not provide
stronger isolation guarantees. It may still be useful for detecting stale state across separate transactions.

For a more detailed discussion of transaction isolation and locking, see the previous article in this series.

# 3. Avoiding Check-Then-Insert

Read-modify-write is not the only pattern that introduces unnecessary database round trips and concurrency problems.
Another common example is checking whether a row exists before inserting it.

Consider a table containing account settings:

```sql
CREATE TABLE account_settings (
    account_id UUID PRIMARY KEY,
    notifications_enabled BOOLEAN NOT NULL,
    locked BOOLEAN NOT NULL DEFAULT false
);
```

An application might first query the table:

```sql
SELECT account_id
FROM account_settings
WHERE account_id = :accountId;
```

Then decide whether to insert or update:

```sql
INSERT INTO account_settings (
    account_id,
    notifications_enabled
)
VALUES (:accountId, :enabled);
```

The problem is that another transaction can insert the same row between the existence check and the insert.

{{< mermaid >}}
sequenceDiagram
participant T1 as Transaction 1
participant DB as PostgreSQL
participant T2 as Transaction 2

    T1->>DB: SELECT settings WHERE account_id = X
    DB-->>T1: No row

    T2->>DB: SELECT settings WHERE account_id = X
    DB-->>T2: No row

    T1->>DB: INSERT settings for X
    T1->>DB: COMMIT

    T2->>DB: INSERT settings for X
    DB-->>T2: Unique constraint violation

    Note over T1,T2: Both existence checks found no row
{{< /mermaid >}}

Even if the application checks first, the database must still enforce the primary-key constraint.

## 3.1 Use an upsert

PostgreSQL provides `INSERT ... ON CONFLICT` to express this operation in a single statement:

```sql
INSERT INTO account_settings (
    account_id,
    notifications_enabled
)
VALUES (:accountId, :enabled)
ON CONFLICT (account_id)
DO UPDATE
SET notifications_enabled = EXCLUDED.notifications_enabled;
```

The database handles the uniqueness conflict as part of the statement. This removes the application-level existence
check and its associated race condition.

For operations that should insert a row only if it does not already exist:

```sql
INSERT INTO account_settings (
    account_id,
    notifications_enabled
)
VALUES (:accountId, :enabled)
ON CONFLICT (account_id)
DO NOTHING;
```

This is useful when an existing row should be left unchanged.

An upsert makes the insert-or-update decision atomic, but it does not necessarily make the entire business operation
idempotent. That depends on the operation's semantics, the uniqueness constraints used to identify duplicates, and
whether other side effects are protected against repetition.

Likewise, an upsert does not eliminate all concurrency-related failures. Deadlocks and serialization failures may still
occur.

The benefit is that the database can resolve the existence condition atomically rather than requiring the application to
coordinate a separate read and write.

## 3.2 Conditional conflict handling

PostgreSQL also supports predicates on the conflict update.

Suppose account settings can be modified by the account owner until an administrator locks them. Once locked, ordinary
updates must be rejected.

We can express this restriction directly in SQL:

```sql
INSERT INTO account_settings (
    account_id,
    notifications_enabled,
    locked
)
VALUES (:accountId, :enabled, false)
ON CONFLICT (account_id)
DO UPDATE
SET notifications_enabled = EXCLUDED.notifications_enabled
WHERE account_settings.locked = false;
```

If the settings do not exist, the statement inserts a new row. If they exist and are unlocked, it updates the
notification preference.

If the existing row is locked, PostgreSQL skips the update without raising an error. The statement affects zero rows,
allowing the application to distinguish a rejected modification from a successful insert or update.

The condition is evaluated against the conflicting row as part of the statement, avoiding a separate application-level
check.

This illustrates the same principle as conditional balance updates: express the operation and its preconditions
together, as close to the data as possible.

# 4. Choosing the Right Write Pattern

There is no single approach that is optimal for every workload. The choice depends on the operation, its concurrency
requirements, and the database's isolation guarantees.

| Pattern                     | Appropriate use case                         | Main consideration                              |
|-----------------------------|----------------------------------------------|-------------------------------------------------|
| Atomic `UPDATE`             | Incrementing counters or balances            | Avoids unnecessary reads                        |
| Conditional `UPDATE`        | Enforcing a precondition during modification | Check affected-row counts                       |
| `SELECT FOR UPDATE`         | Business logic requiring current row state   | Locks can increase waiting and deadlock risk    |
| JPA optimistic locking      | Detecting stale entity state                 | Conflicts require application handling          |
| `INSERT ... ON CONFLICT`    | Insert-or-update operations                  | Requires an appropriate unique constraint       |
| Ordinary JPA entity updates | Domain operations involving managed entities | Understand generated SQL and isolation behavior |

A few general principles are worth keeping in mind:

- **Avoid unnecessary reads.** If the database can perform a modification directly, loading an entity first may add
  overhead without providing additional value.
- **Express preconditions in SQL where practical.** Conditional updates and upserts allow the database to evaluate
  conditions atomically with the corresponding write.
- **Check the outcome.** An `UPDATE` that affects zero rows may represent a rejected business operation rather than a
  successful no-op.
- **Keep related operations transactional.** A sequence of individually atomic SQL statements is not automatically an
  atomic business operation.
- **Understand the isolation guarantees.** The same SQL pattern can behave differently under PostgreSQL's
  `READ COMMITTED`, PostgreSQL's `SERIALIZABLE`, or another database's concurrency-control implementation.

# Conclusion

JPA's entity-oriented programming model is convenient, but not every database modification needs to start by loading an
entity. Atomic updates, conditional predicates, and upserts can eliminate unnecessary reads, reduce round trips, and
make concurrency-sensitive operations easier to reason about.

The key is to choose write patterns that preserve application invariants while minimizing unnecessary work and
contention. Correct transaction boundaries remain essential, but the SQL executed within those boundaries matters just
as much.