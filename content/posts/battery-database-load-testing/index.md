---
title: "Battery: A Workload-Driven Database Load Testing Tool"
date: 2026-10-10
draft: false
slug: "battery-database-load-testing"
description: "Model realistic OLTP workloads using YAML, SQL and scripting."
tags:
  - Database Benchmarking
  - Load Testing
  - OLTP
  - PostgreSQL
  - CockroachDB
  - Java
categories:
  - Databases
cover:
  image: "cover.webp"
  alt: "Industrial control panel with pressure and temperature gauges"
  caption: 'Photo by [Chris J. Walker](https://unsplash.com/photos/a-bunch-of-clocks-that-are-on-a-pole-E2zPzEzIiQ8) on Unsplash'
  relative: true
  hiddenInList: true
---

Standardized benchmarks such as TPC-C, YCSB and sysbench are useful for comparing database systems under defined
workloads. But those workloads rarely match an application's specific transaction patterns, data access paths,
contention characteristics or concurrency model. Reproducing those interactions under load provides a different
perspective: how the database behaves under traffic shaped like your application.

[Battery](https://github.com/kai-niemi/battery) is an open-source load-testing tool built for that purpose. You describe
the workload using YAML, SQL and a small scripting language, then run it with potentially thousands of concurrent 
virtual users (VUs), optionally distributed across multiple hosts.

This post covers how Battery works and some of the design decisions behind it.

## The shape of a load test

A Battery test is a YAML file with four main parts:

* **before** steps that run once to prepare a schema or load reference data

* **phases** that control how fast virtual users arrive over time

* **scenarios**, the sequence of steps each virtual user runs

* **after** steps for cleanup

Here's a complete CRUD workload against a single table:

```yaml
battery:
  baseDir: config/demo

  before:
    steps:
      - name: Prepare Schema
        sql: |-
          create schema if not exists demo;

          create table if not exists demo.customer
          (
              id         uuid        not null default gen_random_uuid(),
              email      varchar(64) not null,
              first_name varchar(64),
              last_name  varchar(64),

              primary key (id)
          );

  phases:
    - name: Warm up phase
      duration: 60s
      users: 20
    - name: Peak load phase
      duration: 30s
      startRate: 4
      maxRate: 50
      maxConcurrency: 64
    - name: Sustained peak phase
      duration: 15s

  scenarios:
    - name: Customer CRUD
      duration: 20s
      steps:
        - name: Create Customer
          capture: true
          sql: |-
            insert into demo.customer (id, email, first_name, last_name) 
            values (gen_random_uuid(), 
                    '#{ gen.randomEmail() }',
                    '#{ gen.randomFirstName() }', 
                    '#{ gen.randomLastName() }' )
            returning id;

        - name: Retrieve Customer
          sql: |-
            select * from demo.customer where id = :id;
          params:
            id: id

        - name: Update Customer
          sql: |-
            update demo.customer
            set first_name=:first_name, last_name=:last_name
            where id=:id;
          params:
            id: id
            first_name: gen.randomFirstName()
            last_name: gen.randomLastName()

        - name: Delete Customer
          sql: |-
            delete from demo.customer
            where id=:id;
          params:
            id: id
```

Two features are doing most of the work here.

**Templating.** `#{ ... }` expressions are evaluated and spliced into the SQL, and `params:` binds named `:placeholders`
to expressions that are evaluated on every execution. Each execution can therefore get fresh generated data.

Unlike templated substitutions, named parameters use JDBC parameter binding, keeping values separate from the SQL statement.

**State capture.** Setting `capture: true` on a step passes its result to the steps that follow. If one row comes back,
each column becomes a variable, so the `id` returned by the insert is available as `id` in the next step. If several
rows come back, they're bound as a list (`result` by default) that later steps can index into or sample from. **This is
what turns a sequence of SQL statements into an application-like workflow:** create an entity, capture its identity,
then read, modify and eventually delete that same entity without writing glue code.

Captured state belongs to one virtual user and resets at the start of each iteration. State captured in `before` steps
is handed to every VU as its initial state, which is an easy way to share lookup data such as a list of valid account
IDs.

## A realistic example: transferring money

The bundled `bank` configuration shows where this gets interesting. A transfer scenario first picks two accounts in a
randomly selected city, then performs a transfer that inserts a transfer record, writes two ledger entries with running
balances, and updates both account balances:

```yaml
battery:
  baseDir: config/bank

  scenarios:

    - name: Transfer Funds (explicit + optimistic)
      alias: a
      duration: 2m
      transactional: true
      steps:
        - name: Find Accounts
          capture: true
          sql: |-
            SELECT id, balance, city
            FROM bank.account
            WHERE city = :city
            ORDER BY id
            LIMIT 2;
          params:
            city: std.selectRandom(L["london","new york","stockholm"]);

        - name: Transfer (explicit)
          capture: true
          path: transfer-explicit.sql

          params:
            city: std.selectRandom(result).get("city");
            transfer_id: transfer_id
            account_id1: result.get(0).get("id");
            account_id2: result.get(1).get("id");
            amount: gen.randomBigDecimal(10.00,500.00)

    - name: Transfer Funds (implicit - modifying CTE)
      alias: c
      duration: 2m
      transactional: false
      steps:
        - name: Find Accounts
          capture: true
          sql: |-
            SELECT id, balance, city
            FROM bank.account
            WHERE city = :city
            ORDER BY id
            LIMIT 2;
          params:
            city: std.selectRandom(L["london","new york","stockholm"]);

        - name: Transfer
          capture: true
          path: transfer-implicit.sql
          params:
            city: std.selectRandom(result).get("city");
            transfer_id: transfer_id
            account_id1: result.get(0).get("id");
            account_id2: result.get(1).get("id");
            amount: gen.randomBigDecimal(10.00,500.00)
```

The complete configuration explores several ways of executing the same logical transfer. In the explicit variants,
Battery wraps the scenario steps in a transaction, allowing accounts to be locked with `FOR UPDATE` before the transfer
is performed. 

In the implicit variant, the transfer is expressed as a single SQL statement using data-modifying CTEs.
The database executes that statement atomically within an implicit transaction, while the preceding account-selection
step remains outside that transaction.

Accounts are deliberately drawn from a small set of cities, increasing the probability of concurrent transactions
contending on the same data. This makes it possible to compare how different transaction and locking strategies behave
as contention increases.

When Battery encounters a retryable transaction failure, such as a serialization failure (SQLSTATE `40001`), the VU backs
off with jitter and restarts the scenario from the beginning rather than retrying only the failed statement. Captured
state is rebuilt, and expressions are evaluated again, so a retry may select different accounts or generate different
values. This models a fresh execution of the workflow rather than a byte-for-byte replay of the original SQL statements.
Connection failures require additional care because the outcome of a transaction may be ambiguous, particularly if the
connection is lost during commit.

In the implicit variant, the transfer itself is a single atomic SQL statement, so there is no partially completed
multi-statement transaction to recover.

## When SQL isn't enough: Battery Script

Templated SQL handles a lot, but real applications branch. For that, Battery includes **Battery Script**, a small
procedural language.

It has dynamic typing, `if`/`while`/`foreach`/`for ... from ... to` loops, collection literals, high-precision decimals
(`12.50bd`), and a namespaced function library for SQL (`jdbc`), random data (`gen`), and general utilities (`std`).

Here's a simplified example where the next database operation depends on state read from the database:

```java
account = jdbc.queryForMap(
    "select id, balance from account where id=?",
    L[accountId]
);

if (account.get("balance") < amount) {
    // Reject transfer
} else {
    jdbc.update(
        "update account set balance=balance-? where id=?",
        L[amount, accountId]
);
```

The important part isn't the SQL itself, but that the workload can make application-side decisions between database
operations. That makes it possible to model workflows where subsequent SQL depends on state observed earlier in the
request.

It also supports `fork`/`join` blocks for firing concurrent statements from within a single virtual user, which is how
you'd model an application issuing parallel queries to render one page.

Battery Script is meant to be simple and readable by someone who mostly writes SQL. You can try expressions
interactively with the shell's `execute` command and list every available function with `functions`.

## Design decisions

### Virtual threads instead of a thread pool

Each VU runs on a Java 21 virtual thread, and so does each `fork` block.

A database load generator spends much of its time waiting on blocking I/O, which makes virtual threads a natural fit. 
A single Battery process can support thousands of concurrent VUs without requiring thousands of dedicated 
operating-system threads or asynchronous callback code.

### An open workload model that drops instead of queueing

Phases are defined by arrivals rather than a fixed number of continuously looping users. You can specify either a total
number of arrivals using `users`, distributed across the phase duration, or an arrival rate using `startRate` and `maxRate`.
Each arrival creates a virtual user that executes a scenario for its configured duration. Phases can overlap: a phase
doesn't wait for its users to finish before the next one begins.

The subtle part is `maxConcurrency`. When the cap is reached, new arrivals are **dropped and counted**, not queued. That
keeps the arrival rate and phase duration honest. If the system under test falls behind, the test still runs on
schedule, and the dropped-user count tells you that demand went unmet.

Queueing arrivals would quietly stretch the test and make a struggling system look healthier than it is. The load
generator would effectively adapt to the slowdown of the system it is supposed to measure. Battery instead keeps offered
load independent of how quickly the database can service it.

This helps avoid coordinated omission, where a slowing system causes the load generator to produce less work precisely
when latency is increasing. By scheduling arrivals independently of completion times, Battery maintains the intended
arrival rate rather than adapting to the system's response times. When the concurrency limit is reached, new arrivals
are dropped and counted instead of being queued. Latency percentiles should therefore be interpreted alongside the
dropped-arrival count, particularly when the system is overloaded.

The same thinking applies to the connection pool. When VUs outnumber connections, the time spent waiting for a
connection is **included** in the measured latency. That wait is part of what your users would experience.

Battery deliberately measures this as client-observed latency rather than database execution time alone. If a VU spends
200 ms waiting for a connection before the database executes a query in 10 ms, the application still experienced roughly
210 ms of latency.

Battery can size its connection pool according to the maximum configured concurrency across the test phases, 
subject to a configurable `maxSize` limit.

### Findings, not just numbers

When a run finishes, Battery prints a summary with throughput, latency percentiles, VU and pool statistics, and a list
of **findings**: plain-language diagnoses derived from the data.

Examples:

* *Connection pool saturated: up to N users waited for a connection... Lower maxConcurrency or raise
  battery.connectionPool.maxSize.*

* Throughput leveling off: offered load increased between phases but achieved throughput didn't keep up -- the "knee"
  where adding more load stops producing more useful work

* p99 latency degrading sharply from one phase to the next

* A long latency tail (p99.9 far above the median)

* Too few samples for a percentile to be trusted

The sample-size check is a small detail that matters. A p99.9 computed from only 300 iterations isn't a meaningful tail
estimate, and the tool says so instead of presenting it with false precision. Summaries are kept in memory, exposed
through the API, and saved as JSON under `.log/runs/` so you can compare runs later.

### Distributed load with a consistency check

One machine won't always generate enough load. Any Battery instance can act as an agent, and a controlling instance
lists the others under `network.agents`. You then drive them with `agent run`, `agent show status`, and `agent cancel`.

The safeguard is that **every agent must run the same test configuration**. Before a distributed run starts, the
controller sends a hash of the workload model, and an agent whose own workload model hashes differently rejects the run.

## Three ways to run

Battery ships as a single executable JAR built on Spring Boot, with three interfaces:

1. **An interactive shell** with tab completion for commands, scenario names, and script functions. It's the main
   control plane: `validate model`, `show db`, `run`, `cancel`, `show errors`.

2. **A web UI** on port 9090 with live charts streamed over WebSockets, a runs history page, and a script playground.

3. **A hypermedia REST API** for automation. Responses expose state-dependent links, so automation can discover which
   operations are currently valid.

Prometheus metrics are exposed at `/api/actuator/prometheus` if you'd rather watch runs in your existing Grafana
dashboards.

## Getting started

You need JDK 21+ and a local PostgreSQL or CockroachDB:

```bash
createdb battery
git clone https://github.com/kai-niemi/battery.git && cd battery
./mvnw clean install
./run.sh            # pick the demo-crud profile
```

Then, in the shell:

```plaintext
battery:$ validate model
battery:$ show db
battery:$ run
```

Open http://localhost:9090 to watch it live. PostgreSQL and CockroachDB drivers are bundled. MySQL and Oracle drivers
are available through the `jdbc-drivers` build profile.

## Wrapping up

Battery isn't intended to produce a universal benchmark score. Its purpose is to answer a more application-specific
question: what happens when a database is subjected to traffic that behaves like my application?

By combining SQL, YAML and lightweight scripting, Battery makes it possible to reproduce stateful workflows, transaction
boundaries, contention and concurrency under controlled load. It handles the mechanics of generating that load,
distributing it across hosts and collecting the results.

If you're evaluating an OLTP database, testing a schema or transaction change, or investigating where your current setup
stops keeping up with demand, give it a try. The [README](https://github.com/kai-niemi/battery)
and [Battery Script guide](https://github.com/kai-niemi/battery/blob/main/SCRIPT.md) are good places to start.
