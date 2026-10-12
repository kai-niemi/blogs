---
title: "Cluster Singleton Scheduling with Spring Boot and ShedLock"
date: 2026-10-12
draft: false
description: "Coordinating scheduled tasks across Spring Boot instances using ShedLock and database-backed locks, including configuration, failure handling, and execution guarantees."
tags:
  - spring-boot
  - java
  - distributed-systems
  - scheduling
  - shedlock
cover:
  image: "cover.webp"
  caption: 'Photo by <a href="https://unsplash.com/@parsoakhorsand?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Parsoa Khorsand</a> on <a href="https://unsplash.com/photos/padlock-on-black-metal-fence-Dd6n63H9szw?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>'
  relative: true
  hiddenInList: true
---

Spring Boot makes it easy to schedule recurring tasks using the `@Scheduled` annotation. This works well for a single
application instance, but what happens when the application is deployed across multiple servers or regions?

Each instance runs its own scheduler, meaning the same task may execute concurrently on several instances. For some
workloads, this is harmless. For others, such as importing data, publishing notifications, or performing periodic
cleanup, concurrent execution can cause duplicate processing or conflicting updates.

What we need is a *cluster singleton*: a scheduled task that runs on only one application instance at a time, regardless
of how many instances are deployed. One straightforward approach is to coordinate scheduled execution through a shared
database using [ShedLock](https://github.com/lukas-krecan/ShedLock).

## The Problem with @Scheduled

Consider a Spring Boot application with a scheduled cleanup task:

```java
@Component
public class CleanupJob {

    @Scheduled(cron = "0 */5 * * * *")
    public void cleanup() {
        // Remove expired records
    }
}
```

The task executes every five minutes. With a single application instance, this behaves as expected. However, when the
application is deployed across three instances, all three independently execute the same task.

```text
                  Every 5 minutes
                         |
              +----------+----------+
              |          |          |
          Instance A Instance B Instance C
              |          |          |
           Cleanup    Cleanup    Cleanup
```

Spring's scheduling mechanism has no built-in coordination between application instances. One option is to designate a
specific instance to run scheduled tasks, but this introduces an operational dependency on that instance and complicates
failover.

A more flexible approach is to allow every instance to schedule the task while coordinating which one is permitted to
execute it.

## Introducing ShedLock

ShedLock is a lightweight Java library that provides distributed locking for scheduled tasks. It works alongside
Spring's scheduling mechanism rather than replacing it:

- **Spring `@Scheduled`** determines when a task should run.
- **ShedLock** coordinates which application instance may execute it.
- **A shared database** stores the lock state.

When a scheduled task is triggered, each instance attempts to acquire a lock identified by a unique name. Only one
instance can acquire the lock, while the others skip that execution rather than waiting for the lock to become available.

When the task completes, the lock is released. If the instance crashes before releasing it, the lock eventually expires,
allowing another instance to take over. This approach requires no dedicated scheduling service and works with many
relational databases through JDBC.

## Configuring ShedLock

The following example uses Spring Boot, Spring's built-in scheduling, and ShedLock's JDBC provider.

### Maven Dependencies

Add the ShedLock dependencies to your `pom.xml`:

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>${shedlock.version}</version>
</dependency>

<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>${shedlock.version}</version>
</dependency>
```

Use the same ShedLock version for both dependencies. ShedLock 5.x and later require Java 17 or newer, while ShedLock 7.x
supports modern Spring Boot versions. Older Spring Boot 2.x applications require an earlier ShedLock generation. Check
the [compatibility matrix](https://github.com/lukas-krecan/ShedLock#compatibility-matrix) before selecting a version.

The example assumes that a JDBC `DataSource` is already configured.

### Database Schema

ShedLock's JDBC provider uses a table to store lock information. The following schema works with PostgreSQL. Other
databases may require different timestamp types, and ShedLock provides database-specific DDL in its
[documentation](https://github.com/lukas-krecan/ShedLock#jdbctemplate).

```sql
CREATE TABLE shedlock (
    name VARCHAR(64) PRIMARY KEY,
    lock_until TIMESTAMP NOT NULL,
    locked_at TIMESTAMP NOT NULL,
    locked_by VARCHAR(255) NOT NULL
);
```

Each row represents a named lock. The `lock_until` column determines when the lock expires, while `locked_at` and
`locked_by` record when and where it was acquired. The primary key prevents duplicate lock records with the same name,
and ShedLock uses conditional database operations to coordinate lock acquisition between competing application
instances.

### Spring Boot Configuration

Next, enable Spring scheduling and configure ShedLock to use the application's existing database connection:

```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class SchedulingConfiguration {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime()
                .build()
        );
    }
}
```

The `@EnableScheduling` annotation activates Spring's scheduling support, while `@EnableSchedulerLock` enables ShedLock
and defines a default maximum lock duration.

The `usingDbTime()` option uses the database server's clock for lock timestamps rather than relying on individual
application clocks. It also uses database-specific SQL that helps avoid insert conflicts during lock acquisition.
ShedLock recommends enabling this option for supported databases.

This does not synchronize Spring's scheduled triggers across application instances. ShedLock still assumes reasonably
synchronized application clocks, and all instances must use the same logically consistent database and lock table.

## Scheduling a Cluster Singleton

With ShedLock configured, we can modify the cleanup task:

```java
@Component
public class CleanupJob {

    @Scheduled(cron = "0 */5 * * * *")
    @SchedulerLock(
        name = "cleanupJob",
        lockAtLeastFor = "1m"
    )
    public void cleanup() {
        LockAssert.assertLocked();

        // Remove expired records
    }
}
```

ShedLock's default `PROXY_METHOD` mode wraps the method in a Spring AOP proxy. Scheduled methods should be public and
non-final. The lock applies to calls intercepted by the proxy, including direct calls through that proxy, not just
scheduled invocations. Calls from within the same class bypass the proxy and therefore bypass ShedLock.

The `@Scheduled` annotation triggers the task every five minutes on every application instance, while `@SchedulerLock`
coordinates execution through the shared database.

The lock parameters control its behavior:

- **`name`** identifies the shared lock. Every instance must use the same name for the same task, while different tasks
  should use distinct names.
- **`lockAtMostFor`** defines when the lock expires if it is not released. Here, it inherits the ten-minute default
  from `@EnableSchedulerLock`.
- **`lockAtLeastFor`** keeps the lock held for a minimum duration, even if the task finishes earlier. This is useful
  for short-running tasks because scheduled triggers on different instances may not fire at precisely the same time.

The `LockAssert.assertLocked()` call verifies that the method is executing under a ShedLock lock, helping detect
configuration problems. It will also throw when the method is called directly in unit tests. ShedLock provides
`LockAssert.TestHelper.makeAllAssertsPass(true)` to disable these assertions on the current thread during testing.

In this example, the lock remains held for at least one minute and expires after ten minutes at the latest. If three
application instances trigger the task simultaneously, only one acquires the lock and executes the method. The others
skip execution.

Once the task completes and the minimum lock duration has elapsed, the lock becomes available for the next scheduled
execution.

**ShedLock does not queue skipped executions.** If an instance fails to acquire the lock, that scheduled invocation is
simply skipped.

## What Happens When an Instance Fails?

Distributed locks need a way to recover when the instance holding a lock becomes unavailable. Suppose Instance A
acquires the cleanup lock and then crashes. Without an expiration mechanism, the lock could remain held indefinitely,
preventing other instances from executing the task.

ShedLock addresses this using time-limited locks, sometimes called *leases*. The `lockAtMostFor` setting establishes an
upper bound on how long a lock remains valid. If Instance A crashes, the lock eventually expires, allowing another
instance to acquire it on a subsequent scheduled trigger.

This provides recovery without requiring an external process to detect and release abandoned locks.

### When a Task Exceeds Its Lock Duration

Consider the following sequence:

1. Instance A acquires a lock with a ten-minute expiration.
2. Instance A begins executing the scheduled task.
3. The task takes longer than expected, or the instance experiences a long pause.
4. After ten minutes, the lock expires.
5. Instance B acquires the lock and starts executing the same task.
6. Instance A continues or resumes its original execution.

Both instances may now execute the task concurrently. The database has not granted the same unexpired lock to two
instances. Rather, Instance A has continued executing after losing ownership of its lock.

**Exclusive lock ownership does not necessarily guarantee exclusive task execution.**

Increasing `lockAtMostFor` reduces the likelihood of this scenario but does not eliminate it. ShedLock also provides a
`KeepAliveLockProvider` that can periodically extend the lock while a task is running. However, lease renewal cannot
guarantee exclusivity if an instance becomes paused or disconnected long enough for the lock to expire.

For this reason, `lockAtMostFor` should be set comfortably above the expected maximum execution time, and scheduled
tasks should tolerate occasional duplicate or overlapping execution.

## Production Considerations

### Idempotency

Where possible, scheduled tasks should be [idempotent](/posts/designing-idempotent-rest-apis/). An idempotent operation
can be executed repeatedly without producing additional effects beyond the first successful execution.

For example, deleting expired records is generally safe to repeat, provided the deletion logic does not introduce other
side effects. Operations such as charging accounts or sending notifications require additional care because repeating
them may produce duplicate business effects.

### Lock Duration

Choose `lockAtMostFor` based on the task's expected worst-case execution time, including unusually slow database
operations and external dependencies. A duration that is too short increases the risk of overlapping execution, while
one that is unnecessarily long delays recovery after a crashed instance.

### Database Operations

ShedLock relies on atomic conditional updates to coordinate lock acquisition and does not require `serializable`
transaction isolation. PostgreSQL's default `READ COMMITTED` isolation is sufficient. Higher isolation levels provide no
additional benefit for lock acquisition and may introduce serialization errors under contention.

The database's replication guarantees also matter. With [asynchronous replication](/posts/multi-active-architectures/), 
an acknowledged lock acquisition may be lost during failover, allowing another instance to acquire the same lock while the original task is still running.
This applies to any primary-replica setup with asynchronous failover, regardless of geographic distribution. Properly
configured synchronous or consensus-based replication can prevent this loss of acknowledged writes.

ShedLock also depends on the availability of its shared database. If the database becomes unavailable, instances may be
unable to acquire new locks, preventing scheduled tasks from running.

Avoid manually deleting rows from the lock table while applications are running. ShedLock caches lock-row existence and
may not recreate deleted rows until the application restarts.

### Monitoring

Monitor task duration, failures, skipped executions, and whether important tasks complete within their expected
intervals. ShedLock also offers a Micrometer integration for lock-related metrics.

## When Should You Use ShedLock?

ShedLock is a good fit when multiple application instances need to coordinate periodic tasks without introducing a
dedicated distributed scheduler.

Typical examples include:

- Periodic cleanup and data housekeeping.
- Cache refreshes.
- Scheduled imports and synchronization.
- Maintenance operations.
- Batch jobs that tolerate occasional duplicate execution.

ShedLock is not an exactly-once execution mechanism. If a task exceeds its lock duration, another instance may begin
executing it before the original instance has finished. Workloads requiring stronger guarantees may need fencing tokens,
transactional ownership validation, or idempotency.

ShedLock is intentionally lightweight and does not provide durable job scheduling or execution queues. If every
scheduled execution must be tracked and eventually processed, consider a distributed scheduler such as
[db-scheduler](https://github.com/kagkarlsson/db-scheduler) or [JobRunr](https://www.jobrunr.io/).

## Conclusion

ShedLock provides a straightforward way to coordinate scheduled tasks across Spring Boot instances using a shared
database, without introducing a dedicated scheduling service. It works well for periodic tasks that tolerate occasional
overlapping execution, but operations where duplicate effects are unacceptable require additional correctness
mechanisms at the application or resource level.
