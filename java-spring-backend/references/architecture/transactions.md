# Transaction Standard

This document defines transaction necessity, consistency scope, transaction boundaries, rollback, isolation, propagation, and database-concurrency semantics.

It answers:

> Which operations must commit or roll back together, what should the transaction boundary cover, and which database consistency mechanisms should guarantee correctness?

For Spring Proxy, `@Transactional`, and `TransactionTemplate` framework mechanics, read:

- [Spring](../coding/spring.md)

Core principles:

> Transactions are determined by data-consistency requirements, not by method names, Service / Manager layers, Mapper count, or annotation style.

> First determine the consistency boundary, then choose `@Transactional` or `TransactionTemplate`; both are implementation mechanisms, not architecture layers.

> Ordinary snapshot reads do not explicitly start transactions by default. When a transaction is required, keep its holding time as short as possible.

---

# 1. When a Transaction Is Needed

Transactions are typically required when:

* multiple writes must all succeed or all roll back;
* changes to multiple tables together express one atomic business action;
* read-then-write has a concurrency-consistency requirement;
* database row locks are used;
* multiple queries must share a specific consistent view;
* the business explicitly requires database operations to share one commit boundary.

Do not automatically start a transaction merely because of:

* one ordinary query;
* multiple independent ordinary queries;
* ordinary statistics;
* multiple Mapper calls;
* a method living in Service / Manager;
* a single INSERT / UPDATE / DELETE.

Principle:

> "Accesses the database" is not a transaction requirement; "must remain consistent together" is.

---

# 2. The Consistency Scope Determines the Transaction Boundary

First answer:

```text
Which database operations must succeed or fail as one unit?
```

Then decide where the transaction belongs.

Common examples:

```text
Manager
→ one reusable atomic database capability
```

or:

```text
Service
→ a complete business transaction spanning multiple Managers
```

Do not mechanically require:

```text
all transactions live in Service
```

or:

```text
all transactions are pushed down into Manager
```

Principle:

> Whoever owns the complete consistency boundary owns the transaction; the layer name itself is not evidence.

---

# 3. Use `@Transactional` Deliberately

Do not add this merely because writes exist:

```java
@Transactional(rollbackFor = Exception.class)
```

Before introducing a transaction, confirm at least:

```text
Which operations must commit / roll back together?
Where does the transaction start and end?
Does it contain remote calls, file IO, waits, or long computation?
Does read-then-write have races?
Are locks, conditional updates, or unique constraints needed?
Which failures should trigger rollback?
```

`@Transactional` implements an already-defined transaction boundary; it does not replace consistency design.

---

# 4. Service Prepares Data; Manager May Encapsulate an Atomic Capability

When a transaction corresponds to a reusable atomic database capability, data preparation that does not depend on the transaction may stay in Service while Manager owns the atomic part.

For example:

```java
public void createPlace(PlaceCreateRequest request, Operator operator) {
    PlaceDO place = buildPlace(request, operator);
    placeManager.create(place);
}
```

Manager:

```java
@Transactional(rollbackFor = Exception.class)
public void create(PlaceDO place) {
    placeMapper.insert(place);
    auditRecordMapper.insert(buildCreateRecord(place));
}
```

This expresses:

```text
Service
→ business preparation / flow that does not need the transaction

Manager
→ reusable atomic database capability
```

But this is not a mandatory template.

If the complete business use case requires writes from multiple Managers to commit / roll back together:

```java
@Transactional(rollbackFor = Exception.class)
public void registerCase(...) {
    caseManager.create(...);
    materialManager.register(...);
    personManager.bind(...);
}
```

then the transaction belongs at the Service boundary that owns the complete consistency scope.

Do not create a responsibility-free Manager merely to host a transaction annotation.

---

# 5. Work Outside and Inside the Transaction

Work that can often happen outside the transaction includes:

* parameter preparation independent of current transactional database state;
* pure in-memory transformations;
* external reads that can safely happen early and need not commit atomically with the database;
* object assembly unrelated to the current consistency boundary.

Work that may need to remain inside the transaction includes:

* business decisions based on data read inside the transaction;
* decisions that rely on row locks;
* decisions that rely on a consistent snapshot;
* validations that must be atomic with the write;
* necessary database writes.

Recommended thought process:

```text
outside transaction: preparation / nonessential IO
        ↓
inside transaction: necessary reads → consistency decision → necessary writes
        ↓
commit
        ↓
outside transaction: follow-up work not requiring atomic database commit
```

Principle:

> Shorten unnecessary transaction time; do not move necessary business rules outside the consistency boundary.

---

# 6. Reduce Database Round Trips When Semantics Remain Correct

Within one atomic operation, when semantics are unchanged, evaluate transformations such as:

```text
row-by-row INSERT
→ Batch

SELECT then UPDATE when the state condition can be expressed in SQL
→ conditional UPDATE

repeat the same read
→ read once and reuse
```

But do not reduce Mapper calls by:

* creating giant SQL;
* combining unrelated business behavior;
* bypassing unique constraints, locks, or state validation;
* forcing different consistency boundaries into one large transaction.

---

# 7. `@Transactional` and `TransactionTemplate`

Both are Spring transaction implementation styles. Neither determines Service / Manager responsibility.

Choose based on how the transaction boundary is best expressed, not on which API a particular layer is "supposed" to use.

## 7.1 Complete Method-Level Boundary

When a public method itself is a clear and stable complete transaction boundary, declarative transactions are usually simpler:

```java
@Transactional(rollbackFor = Exception.class)
public void create(...) {
    ...
}
```

## 7.2 Precise Local Boundary

When only part of a method requires a transaction or when the holding time should be explicitly narrowed, evaluate:

```java
transactionTemplate.execute(...)
transactionTemplate.executeWithoutResult(...)
```

For example, **the Manager that owns the atomic capability** may use:

```java
public void create(PlaceDO place) {
    transactionTemplate.executeWithoutResult(status -> {
        placeMapper.insert(place);
        auditRecordMapper.insert(buildCreateRecord(place));
    });
}
```

If Service owns the complete consistency boundary, Service may also use `TransactionTemplate` around the complete business transaction.

Therefore do not infer:

```text
TransactionTemplate → must live in Service
```

or:

```text
@Transactional → must live in Manager
```

Principle:

> First determine who owns the consistency boundary; then choose declarative or programmatic transactions for that method.

## 7.3 When Using `TransactionTemplate`

Remember:

* `rollbackFor` belongs to `@Transactional`, not `TransactionTemplate`;
* if an exception is caught and swallowed inside the callback, the transaction does not automatically know that a failure once occurred;
* when rollback is required, propagate failure correctly or use `setRollbackOnly()` only as part of a real project recovery flow;
* propagation, isolation, and timeout must still match business semantics;
* do not place unnecessary HTTP / RPC, file IO, waits, or long computation inside the callback;
* do not mechanically convert all declarative transactions into template transactions merely to bypass Proxy or layering problems.

Read Spring API and Proxy details in `spring.md`.

---

# 8. `rollbackFor` and Rollback Semantics

By default, Spring typically rolls back for `RuntimeException` and `Error`, but not automatically for ordinary checked exceptions.

The default convention in this Skill applies only when:

```text
a new business transaction boundary is being created
or
the current task explicitly changes transaction / rollback semantics
```

and the target project has no more specific convention. In that case, default to:

```java
@Transactional(rollbackFor = Exception.class)
```

This is not a migration rule for ordinary code changes.

For example, if existing code has:

```java
@Transactional
public void audit(...) {
    ...
}
```

and the user only changes ordinary business logic, do not opportunistically change it to:

```java
@Transactional(rollbackFor = Exception.class)
```

merely because this Skill uses that as a default for new boundaries. Doing so can change rollback behavior for checked exceptions.

Prefer existing project contracts when:

* the project has a unified transaction annotation;
* `rollbackFor` / `noRollbackFor` is already explicit;
* a class of checked exception explicitly must not roll back;
* a historical public method already has stable transaction semantics;
* the change would create compatibility risk.

Principle:

> Defaults are for designing new transaction boundaries, not for silently changing existing transaction semantics.

---

# 9. Ordinary Snapshot Reads

Do not explicitly start a transaction merely because a read method:

```text
queries multiple tables
calls multiple Mappers
calls multiple Managers
```

Likewise, do not mechanically add:

```java
@Transactional(readOnly = true)
```

just because a method is named:

```text
get / find / list / query / select
```

Use a read transaction only when there is a real transaction-boundary requirement.

---

# 10. Consistent Snapshots Across Multiple Queries

If multiple queries must use one consistent data view, consider together:

* the transaction boundary;
* PostgreSQL snapshot semantics;
* isolation level.

Under PostgreSQL's default `READ COMMITTED`, different SQL statements inside the same transaction may still observe data committed at different times.

Therefore:

```java
@Transactional(readOnly = true)
```

does not automatically mean every SELECT in the entire method sees exactly the same snapshot.

When a fixed snapshot is required, design it according to the actual isolation level.

---

# 11. Read-Then-Write and Races

Typical risk:

```text
SELECT
↓
Java decision
↓
UPDATE
```

Multiple requests can read the same old state concurrently.

Adding a transaction alone does not necessarily solve the race.

Evaluate according to the actual case:

* conditional UPDATE;
* optimistic locking;
* pessimistic locking;
* unique constraints;
* suitable isolation level.

For simple state races, evaluate conditional updates first:

```sql
UPDATE place
SET zt = #{targetStatus}
WHERE id = #{id}
  AND zt = #{expectedStatus}
```

then inspect the affected-row count.

---

# 12. Optimistic and Pessimistic Locking

## 12.1 Optimistic Locking

When conflicts are relatively rare and conflict detection is acceptable, mechanisms such as `version` may be appropriate.

An update count of 0 indicates a conflict. Retries must be bounded and only used when the operation's semantics permit retrying.

## 12.2 Pessimistic Locking

When database row locks are truly required, use mechanisms such as:

```sql
SELECT ... FOR UPDATE
```

inside an explicit transaction, while considering:

* lock holding time;
* deadlocks;
* lock acquisition order;
* throughput;
* whether conditional update / optimistic locking could replace it.

Do not default to pessimistic locking for every race.

---

# 13. Transaction Propagation

Prefer the ordinary Spring default:

```text
REQUIRED
```

Without clear business semantics, do not casually use:

```text
REQUIRES_NEW
NESTED
NOT_SUPPORTED
NEVER
```

`REQUIRES_NEW` in particular creates an independent transaction whose commit may survive an outer transaction rollback.

Use it only when the business truly requires an independent commit.

---

# 14. Exceptions and Transactions

Inside a transaction, do not catch a failure and swallow it:

```java
try {
    mapper.update(...);
} catch (Exception ex) {
    log.error("failed", ex);
}
```

or the transaction may continue and commit.

When catching an exception, define:

* whether recovery occurs;
* whether the exception is rethrown;
* whether it is translated while preserving the cause;
* whether the current transaction must still roll back.

Read cross-layer exception rules in `error-handling.md`.

---

# 15. Avoid Long Transactions

Inside a transaction, minimize:

* HTTP / RPC;
* file uploads;
* third-party calls;
* long computations;
* blocking waits / sleep;
* large object transformations unrelated to the consistency boundary;
* external reads that could safely happen earlier.

However, necessary decisions that depend on transactional state, locks, or a consistent view must not be moved outside mechanically.

---

# 16. Spring Proxy and Self-Invocation

Whether `@Transactional` is actually invoked through a Spring Proxy, whether self-calls bypass the Proxy, and method-visibility rules are Spring mechanics. Read:

- [spring.md](../coding/spring.md)

This transaction standard requires only:

> The presence of `@Transactional` in source code is not sufficient evidence that the actual call path runs inside the intended transaction.

Do not duplicate a full Spring Proxy tutorial here.

---

# 17. Transactions and Thread Switching

Ordinary Spring transactions are generally bound to the current thread.

Do not assume:

```text
current-thread transaction
→ automatically propagates to CompletableFuture / @Async / manually created thread
```

Do not split database operations inside a transaction across threads for performance and assume they still belong to one transaction.

For threads, Executor, and context propagation, read `concurrency.md`.

---

# 18. Forbidden Practices

Do not:

* add transactions uniformly to every Service / Manager method;
* add `readOnly` to every query method;
* start a transaction merely because multiple Mappers / Managers are called;
* expand a transaction boundary "for safety";
* orchestrate business transactions in Controller / Mapper;
* create a Manager mechanically just to host a transaction;
* treat `TransactionTemplate` as an architecture pattern;
* change an existing `rollbackFor` contract without authorization during an ordinary business change;
* swallow exceptions inside a transaction;
* assume transactions propagate across threads automatically;
* create giant SQL or giant transactions merely to reduce Mapper calls.

---

# 19. Codex Transaction Decision Flow

```text
Is there a clear consistency requirement?
    ↓ no
Do not add a transaction
```

If yes:

```text
Which operations must commit / roll back together?
        ↓
Who owns the complete consistency boundary?
        ↓
Manager atomic capability / Service complete business transaction
        ↓
Which work can safely move outside the transaction?
        ↓
Are conditional update / lock / unique constraint / specific isolation needed?
        ↓
Choose @Transactional or TransactionTemplate
        ↓
Define rollback semantics
```

Check:

* whether the transaction is truly necessary;
* whether the boundary is complete yet sufficiently small;
* whether layering was mechanically changed just to host a transaction;
* whether unnecessary remote calls, IO, or waits remain inside;
* whether critical business decisions were incorrectly moved outside;
* whether `rollbackFor` on a new boundary matches defaults and project conventions;
* whether existing transaction semantics were changed without authorization;
* whether race, locking, propagation, or snapshot issues exist;
* whether cross-thread transaction propagation is incorrectly assumed.

Final principle:

> The consistency boundary determines transaction ownership; `@Transactional` and `TransactionTemplate` only determine implementation style. Without a consistency requirement, do not add a transaction. When one is needed, cover only the operations that truly must remain consistent together.
