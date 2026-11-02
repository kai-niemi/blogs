---
title: "Designing Idempotent REST APIs: Stateless Preconditions and Idempotency Keys"
description: "Explore two approaches to making REST API operations safer to retry: stateless resource-state preconditions and idempotency keys. Learn their guarantees, trade-offs, and concurrency considerations."
summary: "How do you safely retry a POST request when the outcome is unknown? Compare stateless preconditions and idempotency keys through funds transfers and order creation."
date: 2022-10-19
lastmod: 2026-11-02
draft: false
tags:
  - REST
  - HTTP
  - API Design
  - Idempotency
  - Distributed Systems
categories:
  - Software Engineering
showToc: true
TocOpen: false
aliases:
  - /designing-idempotent-rest-apis/  
cover:
  image: "cover.webp"
  caption: 'Photo by [Growtika](https://unsplash.com/@growtika?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/diagram-k5AxNUn4c5o?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)'
  relative: true
  hiddenInList: true
---

*Substantially revised October–November 2026, including a hands-on idempotency key implementation using Java, 
Spring Boot, and PostgreSQL.*

What happens when a client submits a payment or funds transfer, the server successfully processes it, but the response
is lost?

From the client's perspective, the outcome is unknown. Retrying the request might be necessary, but it could also
execute the same operation twice.

This is where **idempotency** becomes an important property of API design. An idempotent operation can be repeated
without producing additional intended side effects beyond those of the first execution.

HTTP methods such as GET, PUT, and DELETE are defined as idempotent, but POST is not. Nevertheless, APIs can implement
safeguards that make POST operations safer to retry.

In this article, we'll explore two approaches:

1. **Stateless preconditions**, where a token derived from resource versions determines whether an operation may
   execute.
2. **Idempotency keys**, where the server records a unique operation identifier to recognize and deduplicate repeated
   requests.

We'll use a funds-transfer example to demonstrate both approaches, then examine a simpler idempotency-key implementation
using order creation. We'll compare their guarantees, trade-offs, and concurrency considerations.

## HTTP Safety and Idempotency

HTTP defines two important properties for request methods: **safety** and **idempotency**.

A method is *safe* when its defined semantics are essentially read-only. Safe requests should not cause state changes
requested by the client, although incidental effects such as logging and metrics are permitted.

A method is *idempotent* when multiple identical requests have the same intended effect on the server as a single
request.

The common HTTP methods have the following properties:

| Method | Safe | Idempotent |
|---|---|---|
| GET | Yes | Yes |
| HEAD | Yes | Yes |
| OPTIONS | Yes | Yes |
| PUT | No | Yes |
| DELETE | No | Yes |
| POST | No | No |
| PATCH | No | No |

These are the methods' defined semantics. Individual POST or PATCH operations can still be implemented idempotently.

Although PATCH is not inherently idempotent, some patch operations are. For example, setting a field to a specific value
can be idempotent, while incrementing a counter generally is not.

For example, consider deleting a resource:

```http
DELETE /accounts/123
```

The first request might return:

```http
HTTP/1.1 204 No Content
```

A subsequent request might return:

```http
HTTP/1.1 404 Not Found
```

The responses differ, but the intended effect is the same: the resource no longer exists.

**Idempotency concerns the intended effect of an operation, not whether repeated requests return identical responses.**

This distinction becomes particularly important when implementing idempotent POST operations.

## Why Idempotency Matters

Distributed systems communicate over networks, and networks are [unreliable](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing).
A client can send a request without knowing whether the server received it, processed it, or successfully committed its changes.

Consider a funds transfer between two accounts:

{{< mermaid >}}
sequenceDiagram
participant C as Client
participant S as Server

    C->>S: POST /transfers
    activate S

    Note right of S: Validate transfer
    Note right of S: Debit account A
    Note right of S: Credit account B
    Note right of S: Commit transaction

    S--xC: 201 Created (response lost)
    deactivate S

    Note over C: Timeout
{{< /mermaid >}}

The transfer completed successfully, but the response was lost.
The client now faces an ambiguous outcome.

- Did the server fail before processing the request? 
- Did the transaction commit? 
- Was only the response lost?

A timeout cannot answer these questions, and this is a [fundamental](https://en.wikipedia.org/wiki/Two_Generals%27_Problem) problem in distributed systems: 
**the absence of a response does not imply the absence of an effect**.

The client might retry:

```http
POST /transfers
Content-Type: application/json

{
  "from": "A",
  "to": "B",
  "amount": 100
}
```

Without additional safeguards, the server could process the same transfer again, moving another 100 units between the
accounts.

A reliable API should allow clients to recover from uncertain outcomes without accidentally duplicating business
operations.

There are different ways to address this, each with its own guarantees.

## Example: Transferring Funds Between Accounts

Our running example is a transfer of 100 units from account A to account B:

```json
{
  "from": "A",
  "to": "B",
  "amount": 100
}
```

The operation must satisfy several invariants:

- The source account must have sufficient funds.
- The total amount debited must equal the total amount credited.
- The debit and credit must commit atomically.
- Retrying the same logical transfer must not create additional transfers.

The first three requirements concern the correctness of the transfer itself.
The fourth concerns **request idempotency**.

We'll examine two approaches to addressing it.

## Approach 1: Stateless Preconditions

A stateless precondition allows the server to determine whether an operation may execute based on the current state of
the affected resources, without storing a history of previous requests.

The idea is similar to optimistic concurrency control. HTTP provides standardized conditional requests through headers
such as `If-Match`, which compares an entity tag (ETag)
against the selected representation of the target resource.

For a transfer involving multiple accounts, however, we want to validate the combined state of several resources.

Rather than treating a multi-resource token as an ETag for the `/transfers` collection, we'll use an application-defined
precondition header.

Unlike `If-Match`, this header defines an application-specific precondition over multiple resources. The API uses `412
Precondition Failed` to indicate that the supplied token no longer matches the current resource versions.

Unlike idempotency keys, stateless preconditions do not identify individual operations. They prevent execution against
unexpected resource state, which can protect against duplicate effects, but cannot establish whether a particular
request has already succeeded.

### Generating a Resource-State Token

Suppose two accounts have the following state:

```text
Account A:
  Balance: 500
  Version: 12

Account B:
  Balance: 200
  Version: 8
```

Each account has a version number that changes whenever its relevant state is modified.

The client first requests a precondition token:

```http
POST /transfer-preconditions
Content-Type: application/json

{
  "from": "A",
  "to": "B"
}
```

The server reads the current account versions and derives a token:

```text
token = H(accountA.id, accountA.version,
          accountB.id, accountB.version)
```

Here, `H` represents a deterministic hash function over an unambiguous, canonical encoding of the values.

The server returns:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token": "a8f3c9..."
}
```

The token represents the observed versions of both accounts.
It is not a record of an operation, nor does it prove that a transfer has previously occurred.
Because the server can recompute the token from current resource versions, it does not need to store the token itself.

The token is a concurrency-control mechanism, not an authorization credential. A deterministic hash does not prevent
clients from constructing tokens if the underlying values are predictable. If the token must be tamper-resistant, the
server can use a keyed construction such as HMAC, but authorization must still be enforced independently.

### Executing the Transfer

The client submits the transfer with the token:

```http
POST /transfers
Transfer-Precondition: "a8f3c9..."
Content-Type: application/json

{
  "from": "A",
  "to": "B",
  "amount": 100
}
```

`Transfer-Precondition` is an illustrative, application-defined header, not a standardized HTTP conditional header.
The server validates the token against the current account versions before executing the transfer.

Conceptually:

```text
BEGIN TRANSACTION

  Read current account versions

  expected = H(accountA.id, accountA.version,
               accountB.id, accountB.version)

  IF supplied token != expected:
      ROLLBACK
      RETURN 412 Precondition Failed

  Validate transfer invariants

  Debit account A
  Credit account B

  Increment account versions

COMMIT

RETURN 201 Created
```

The important requirement is that **precondition validation and the transfer execute atomically**.

Otherwise, two concurrent requests could both validate the same token before either updates the accounts.
The transaction must ensure that only one conflicting operation can commit against a particular set of versions.

Databases providing serializable transaction isolation, such as PostgreSQL and CockroachDB, are well suited to this pattern.
Serializable isolation ensures that concurrent transactions produce results equivalent to **some serial** execution order.
This guarantee depends on every relevant state modification advancing the corresponding resource version. Versions must
not be reused or reset in ways that could make an earlier token valid again.

Contention may cause serialization failures that require retrying the transaction. The precondition must
be re-evaluated on each attempt using the original client-supplied token, ensuring that a retry cannot silently proceed
against resource state that has since changed.

### What Happens on a Retry?

Suppose the transfer succeeds.

The accounts now have the following state:

```text
Account A:
  Balance: 400
  Version: 13

Account B:
  Balance: 300
  Version: 9
```

The original token no longer matches the current account versions.

If the client retries using the same token:

```http
POST /transfers
Transfer-Precondition: "a8f3c9..."
Content-Type: application/json

{
  "from": "A",
  "to": "B",
  "amount": 100
}
```

The server rejects the request:

```http
HTTP/1.1 412 Precondition Failed
```

No additional transfer is performed. However, this response does not prove that the original transfer succeeded.
Another operation could have modified either account before the first transfer was executed.

**A failed precondition means the relevant resource state has changed, not that a particular request was previously processed.**

This is the fundamental distinction between state-based preconditions and request deduplication.

### Resource Versions and the ABA Problem

Why use account versions rather than hashing account balances directly?

Consider:

```text
Initial balance: 100
Token: H(100)

Transfer A: -20
Balance: 80

Transfer B: +20
Balance: 100

Token: H(100)
```

The account has returned to its original balance, making the original token valid again.

This is known as the **ABA problem**, named after the sequence `A → B → A`, where a value changes and subsequently
returns to its original state, making an intervening modification undetectable through value comparison alone.

A monotonically increasing resource version avoids this particular issue:

```text
Balance 100, version 1
Balance  80, version 2
Balance 100, version 3
```

Even though the balance returns to 100, the original version is no longer current.

For concurrency-sensitive operations, versions are therefore generally preferable to hashes derived solely from mutable
business values.

### Guarantees and Limitations

Stateless preconditions offer several advantages:

- No need to store idempotency keys or previous responses.
- Straightforward validation against current resource versions.
- Protection against executing operations against stale resource state.
- Compatibility with optimistic concurrency control.

However, there are important limitations:

- The client must obtain a token before executing the operation.
- The token must cover all relevant resource state.
- Validation and modification must be atomic.
- A failed precondition does not reveal whether the original operation succeeded.
- The server cannot replay the original response without separately recording the operation's outcome.

There is also an important distinction between **preventing a duplicate effect** and **making an operation idempotent**.
A state-based precondition can prevent a repeated transfer from executing after the first transfer changes the protected versions.

However, the mechanism is fundamentally conditional execution, not recognition of a previously submitted logical operation.
For clients that need to recover the outcome of a request after a timeout, this limitation matters.

## Approach 2: Idempotency Keys

An idempotency key identifies a logical operation independently of how many times the client submits it.

Instead of obtaining a token derived from resource state, the client generates a unique identifier and includes it with the request:

```http
POST /transfers
Idempotency-Key: "8d27c0f1-7b24-4f9e-8d38-0e7b6d0d18a4"
Content-Type: application/json

{
  "from": "A",
  "to": "B",
  "amount": 100
}
```

The server associates the key with the operation and its outcome.
If the same request is submitted again using the same key, the server recognizes it as a retry rather than a new transfer.

Unlike stateless preconditions, idempotency keys identify **the operation itself**, not the current state of the affected resources.

### Request Deduplication

An idempotency record that retains completed HTTP responses might contain:

```text
Idempotency record
------------------
Key
Request fingerprint
Status
Response status
Response body
Created at
Expires at
```

The request fingerprint represents the relevant parameters of the original operation.
The key should also be scoped appropriately, for example to a particular client or tenant, so unrelated clients cannot
accidentally share the same namespace.

When a request arrives, the server distinguishes four situations:

| Situation                           | Server behavior                        |
|-------------------------------------|----------------------------------------|
| New key                             | Accept the operation for execution     |
| Existing key, different request     | Reject conflicting key reuse           |
| Existing key, operation in progress | Wait or return an in-progress response |
| Existing key, operation completed   | Return the previously recorded result  |

This distinction matters because duplicate requests can arrive **while the original request is still executing**, not
only after it has completed.

### Atomic Execution

A common mistake is to check whether an idempotency key exists before executing the business operation:

```text
IF key does not exist:
    Execute transfer
    Store key
```

**This is unsafe under concurrency.**
Two requests can both observe that the key is absent and proceed to execute the transfer.

The server needs an atomic mechanism for claiming the key, typically backed by a database uniqueness constraint or
equivalent concurrency control.

The following pseudocode assumes that atomic key acquisition resolves concurrent claims by waiting for the competing
transaction or returning an explicit in-progress outcome. The exact behavior depends on the persistence strategy.

Conceptually:

```text
FUNCTION submitTransfer(key, request):

  fingerprint = fingerprint(request)

  BEGIN TRANSACTION

    result = atomicallyClaimKey(key, fingerprint)

    IF result == IN_PROGRESS:
        ROLLBACK
        RETURN documentedInProgressResponse
    
    IF result == FINGERPRINT_MISMATCH:
        ROLLBACK
        RETURN 422 Unprocessable Content

    IF result == ALREADY_COMPLETED:
        ROLLBACK
        RETURN previouslyRecordedResponse

    Validate transfer invariants

    Debit source account
    Credit destination account

    response = createTransferResult()

    Store completed response with key

  COMMIT

  RETURN response
```

The atomic claim must serialize concurrent requests using the same key. If another transaction currently holds the
claim, the server can wait for its outcome or return a documented in-progress response. If that transaction aborts, a
subsequent attempt may acquire the key and execute the operation.

This is illustrative pseudocode rather than a complete database algorithm.

The critical requirement is that **the transfer and its completed idempotency record commit atomically**.

A database transaction can provide this guarantee when both the business operation and idempotency record reside in the
same transactional database.

The `IN_PROGRESS` state requires additional care. An implementation might register an in-progress operation before starting the business transaction, allowing concurrent
requests to detect that execution is underway.

If the server crashes, that registration must not permanently prevent retries. A lease, timeout, or recovery mechanism
is needed.

Alternatively, an implementation can rely on transactional key claiming and database concurrency control without
persisting a separate in-progress record. Concurrent requests may block or conflict until the original transaction
commits or aborts.

The choice depends on the desired API behavior and persistence model.

In either case, the server must distinguish between:

- An operation that is still executing.
- An operation that completed successfully.
- An operation that failed before committing any business effects.

A failed attempt must not permanently consume an idempotency key unless the API deliberately records that failure as the
operation's final outcome.

### Implementation A: Storing and Replaying Responses

One implementation strategy is to store the original HTTP response alongside the idempotency record. When the same
request is retried, the server returns the recorded response without executing the operation again.

Suppose the original request succeeds:

```http
HTTP/1.1 201 Created
Location: /transfers/789
Content-Type: application/json

{
  "id": "789",
  "status": "completed"
}
```

The response is lost.

The client retries with the same idempotency key:

```http
POST /transfers
Idempotency-Key: "8d27c0f1-7b24-4f9e-8d38-0e7b6d0d18a4"
Content-Type: application/json

{
  "from": "A",
  "to": "B",
  "amount": 100
}
```

The server finds the completed operation and returns its recorded result:

```http
HTTP/1.1 201 Created
Location: /transfers/789
Content-Type: application/json

{
  "id": "789",
  "status": "completed"
}
```

No additional transfer is executed.

Unlike resource-state preconditions, idempotency keys allow the server to identify a previously completed operation and
recover its outcome.

An API could instead return `200 OK` with the existing result. Both behaviors are possible, but the response semantics
should be documented and consistent.

### Implementation B: Referencing the Created Resource

Storing the original HTTP response is not always necessary.

For operations that create persistent resources, the server can instead associate the idempotency key with the
identifier of the created resource.

This approach is illustrated by [Order System](https://github.com/kai-niemi/order-system), a Java and Spring Boot
application demonstrating REST architectural principles.

Consider creating an order:

```http
POST /api/orders
Idempotency-Key: "8d27c0f1-7b24-4f9e-8d38-0e7b6d0d18a4"
Content-Type: application/json

{
  "customerId": "550e8400-e29b-41d4-a716-446655440000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440001",
      "quantity": 2
    }
  ]
}
```

The server computes a canonical fingerprint from the HTTP method, request path, and body. It then attempts to acquire
the idempotency key within a database transaction. The fingerprint must cover all request inputs that affect the
operation's semantics. For this endpoint, the method, path, and body are sufficient.

A simplified idempotency record contains:

```text
Scope
Key
Request fingerprint
Resource ID
Created at
Expires at
```

If the key is successfully acquired, the server creates the order using the resource identifier associated with the key.

If the response is lost and the client retries using the same key and request body, the server retrieves the existing
order rather than creating another.

The important distinction is that the server does not retain the complete HTTP response.
On the initial request, the application executes the business operation and builds the resource representation. On
replay, it skips the business operation and invokes only the representation function using the previously stored
resource identifier.

#### Atomic Key Acquisition

The Order System implementation uses PostgreSQL's `INSERT ... ON CONFLICT` to acquire idempotency keys atomically.

A unique constraint on `(scope, idempotency_key)` combined with the conditional `INSERT ... ON CONFLICT` statement,
ensures that competing requests cannot independently acquire the same live key.

The acquisition operation distinguishes three cases:

- **New key:** A record is inserted with a generated resource identifier.
- **Existing, unexpired key:** Acquisition returns no identifier. The server retrieves the existing record and verifies the request fingerprint.
- **Existing, expired key:** The record is replaced with a new resource identifier and expiration time.

The relevant SQL is:

```sql
INSERT INTO idempotency_record (...)
VALUES (...)
ON CONFLICT (scope, idempotency_key) DO UPDATE
SET fingerprint = excluded.fingerprint,
    resource_id = excluded.resource_id,
    created_at  = now(),
    expires_at  = excluded.expires_at
WHERE idempotency_record.expires_at < now()
RETURNING resource_id
```

The returned resource identifier determines whether the business operation should execute.

The application separates the business operation from representation generation:

```java
public <T> IdempotencyResult<T> execute(
        IdempotencyRequest request,
        Consumer<UUID> operation,
        Function<UUID, T> representation)
```

When acquisition succeeds, the application executes the business operation using the allocated resource identifier.

When acquisition fails because a live record already exists, the application compares request fingerprints. A mismatch
is rejected. Otherwise, the representation of the existing resource is retrieved without executing the business
operation again.

Both key acquisition and order creation execute within the same database transaction.

This ensures that a committed order creation has a corresponding idempotency record. If order creation fails, the
transaction rolls back the key acquisition as well.

Concurrent requests cannot independently claim the same live key, although database contention may require transaction
retries.

The implementation supports both `READ COMMITTED` and `SERIALIZABLE` transaction isolation. Under `READ COMMITTED`,
competing requests are coordinated through PostgreSQL's unique constraint and atomic `INSERT ... ON CONFLICT` operation.
A waiting request can observe the committed idempotency record and retrieve the existing order.

Under `SERIALIZABLE`, a competing request may instead encounter a serialization failure (`SQLSTATE 40001`). The
application retries the entire transaction, allowing the retry to observe the committed record and return the existing
order without executing the business operation again.

#### Response Semantics

Unlike storing the original response, retrieving the existing order may return its current representation.

For example, if the order was subsequently cancelled, a retry could return the same order identifier with a cancelled
status.

The operation remains idempotent because no additional order is created.
The API must also define what happens when the referenced resource has been deleted or is no longer accessible.

This approach trades exact response replay for a smaller persistence model and a direct relationship between the
idempotency key and the created resource.

### Key Scope and Retention

Idempotency keys cannot necessarily be retained forever.
A server might retain them for a configurable period, such as 24 hours, after which the records are eligible for
deletion.

This creates an important limitation:

**The deduplication guarantee applies only while the idempotency record remains valid and available.**

If a client retries after the retention window, the server may treat the request as a new operation.
Reusing a key after its expiration is treated as a new operation and may create another order, even if the original
order still exists.

The API should therefore define:

- How keys are generated and scoped.
- How long keys remain valid.
- What happens when a key is reused with different parameters.
- How concurrent requests using the same key are handled.
- Whether completed responses are replayed.
- Which operation outcomes are retained.

The retention period should reflect realistic client retry behavior and the business consequences of duplicate
execution.

For financial operations, an application may also require durable business-level uniqueness constraints independent of
temporary HTTP idempotency records.

### Idempotency Is Not Exactly-Once Execution

Idempotency keys are sometimes described as providing 
*exactly-once semantics* by deduplication.

That terminology deserves care.
A request handler may execute multiple times because of client retries, server restarts, or database transaction retries.

What matters is that the intended business effect commits **no more than once** for a particular logical operation.

For example:

```text
Request attempt 1 -> Transaction aborted
Request attempt 2 -> Transaction committed
Request attempt 3 -> Original result returned
```

The operation was attempted three times, but its business effect committed once.

This is better described as **at-most-once committed effects**, within the defined scope and retention period.

It does not guarantee that an operation eventually succeeds. That depends on retries, availability, and other
failure-handling mechanisms.

## Comparing the Approaches

Both stateless preconditions and idempotency keys can help prevent unintended duplicate effects, but they provide different guarantees.

| Property                               | Stateless preconditions             | Idempotency keys                          |
|----------------------------------------|-------------------------------------|-------------------------------------------|
| Primary purpose                        | Optimistic concurrency control      | Request deduplication                     |
| Identifies                             | Resource state or version           | Logical operation                         |
| Requires initial request               | Usually                             | No, if client generates the key           |
| Requires server-side operation history | No                                  | Yes                                       |
| Protects against concurrent execution  | With atomic precondition validation | With atomic key claiming                  |
| Recognizes an earlier operation        | No                                  | Yes                                       |
| Can replay the original response       | Not inherently                      | Only if the response is retained          |
| Requires retention management          | Not necessarily                     | Yes                                       |
| Typical retry outcome                  | `412 Precondition Failed`           | Recorded result or existing resource      |
| Main limitation                        | Cannot establish why state changed  | Requires storage and lifecycle management |

### Idempotency-Key Implementation Trade-offs

| Property                 | Stored response               | Resource reference         |
|--------------------------|-------------------------------|----------------------------|
| Stored result            | Original HTTP response        | Created resource ID        |
| Retry behavior           | Replay recorded response      | Retrieve existing resource |
| Response consistency     | Original response             | Potentially current state  |
| Storage overhead         | Depends on response size      | Typically smaller          |
| Resource deletion        | Response can remain available | Requires explicit handling |
| Atomicity required       | Yes                           | Yes                        |
| Request fingerprint      | Recommended                   | Recommended                |
| Concurrent deduplication | Required                      | Required                   |

### When to Use Stateless Preconditions

Stateless preconditions are useful when the client intends to execute an operation against a particular version of one or more resources.

Typical examples include:

- Updating a document without overwriting another user's changes.
- Modifying account configuration only if it has not changed.
- Executing an operation against a known set of resource versions.

The primary concern is whether the relevant resource state has changed since the client last observed it.

### When to Use Idempotency Keys

Idempotency keys are a better fit when clients need to retry an operation safely after an uncertain outcome.

Typical examples include:

- Payments and funds transfers.
- Order creation.
- Booking requests.
- Job submissions.
- Other operations that must not be duplicated.

The primary concern is whether a particular logical operation has already been performed.

### Can They Be Combined?

Yes. The two mechanisms address different concerns and can complement each other.

For example:

```http
POST /transfers
Idempotency-Key: "abc-123"
Transfer-Precondition: "a8f3c9..."
Content-Type: application/json

{
  "from": "A",
  "to": "B",
  "amount": 100
}
```

The idempotency key identifies the logical transfer across retries.
The precondition ensures that the initial execution occurs only against the expected resource state.

A retry of an already completed operation can return its recorded result without re-evaluating the original precondition.

The API must define how these mechanisms interact, including which checks take precedence.

## Conclusion

Reliable APIs must account for requests whose outcomes become uncertain because of network failures.

Stateless preconditions protect operations against changes to relevant resource state. They are useful for optimistic
concurrency control but cannot identify whether a particular request has already executed.

Idempotency keys identify logical operations across retries, enabling request deduplication and recovery of previously
completed outcomes.

The server can retain the original HTTP response or store a reference to the affected resource. Both approaches require
atomic persistence and concurrency control, but they differ in their storage requirements and response semantics.

These mechanisms can also be combined when an operation needs both state validation and safe retries.

The goal isn't to prevent retries, since that's not practically possible. It's to make them safe and predictable.