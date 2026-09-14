# Concurrency Standard

This document defines rules for multithreading, thread pools, asynchronous execution, shared state, and concurrency control.

It answers:

> Is concurrency worth introducing, how should threads and resources be managed, how should context and failures propagate, and how should shared state remain correct?

For transaction consistency, read:

- [Transactions](transactions.md)

Core principle:

> Do not introduce concurrency for speculative performance gains. First prove that operations are independent and that parallel execution provides real value, then accept the complexity of threads, context, exceptions, and resource contention.

---

# 1. Before Introducing Concurrency

Confirm:

* whether the operations are truly independent;
* whether they require a shared transaction or consistent snapshot;
* whether they depend on current-thread context;
* whether mutable state is shared;
* whether resources such as database and HTTP connection pools can absorb additional concurrency;
* whether parallelism benefits exceed scheduling and resource-contention costs.

Do not automatically run operations in parallel merely because they "can execute at the same time."

---

# 2. CompletableFuture / @Async

Evaluate them when all of these conditions hold:

```text
tasks are independent
+
do not share a transaction
+
context propagation is explicit
+
resource capacity is sufficient
+
there is real benefit
```

In particular, do not split Mapper operations across multiple threads inside an existing database transaction for speed and assume they remain in the same transaction.

For Spring `@Async` proxy behavior, read `spring.md`.

---

# 3. Executor and Thread Pools

Business code must not arbitrarily use:

```java
new Thread(...)
```

or create a dedicated thread pool mechanically for a single feature.

Prefer project-managed:

```text
Executor
ThreadPoolTaskExecutor
```

A thread pool should have explicit:

* corePoolSize;
* maxPoolSize;
* queueCapacity;
* keepAlive;
* threadNamePrefix;
* rejectionPolicy.

Use convenience factories such as `Executors.newFixedThreadPool` and `newCachedThreadPool` cautiously in production because they hide important capacity semantics.

---

# 4. Queues and Rejection Policies

Task queues must have deliberate capacity. Avoid accidental unbounded backlog:

```text
production rate > consumption rate
        ↓
queue keeps growing
        ↓
latency / memory becomes uncontrolled
```

A rejection policy must match business semantics:

```text
reject immediately
run in caller thread
explicit degradation
business compensation
alerting
```

Important work must never be silently discarded.

---

# 5. Thread Naming

Use responsibility-oriented thread names, for example:

```text
place-stats-
case-import-
notification-
```

so logs, Thread Dumps, and incident diagnosis remain understandable.

---

# 6. Thread Context

After a thread switch, do not assume these propagate automatically:

* Spring Transaction;
* SecurityContext;
* ThreadLocal;
* current logged-in user;
* data permissions;
* tenant;
* MDC;
* TraceId.

If a task needs them, confirm that the project already has a reliable propagation mechanism.

Do not copy every ThreadLocal manually without understanding ownership and lifecycle.

---

# 7. ThreadLocal Lifecycle

Custom ThreadLocal state must have explicit set and cleanup boundaries.

Because pool threads are reused, use patterns such as:

```java
contextHolder.set(context);
try {
    process();
} finally {
    contextHolder.remove();
}
```

Avoid exception paths that leak one user's, tenant's, or MDC state into another task.

Reuse the project's existing context mechanism when one exists. Do not build a second parallel Context system.

---

# 8. Async Failures Need Explicit Semantics

An asynchronous task failure must not become invisible.

Define:

* whether the caller waits for the result;
* whether failure affects the main business outcome;
* how the exception propagates;
* whether business degradation is allowed;
* whether compensation, bounded retries, or alerting are needed.

An exception in a child thread is not automatically caught by an ordinary `try/catch` in the caller. It must propagate through a Future, async framework, or another explicit failure channel.

## 8.1 `exceptionally()` Is Recovery

`CompletableFuture.exceptionally(...)` does not merely "handle an exception." If the callback returns a normal value, it converts exceptional completion into successful completion.

For example:

```java
future.exceptionally(ex -> Collections.emptyList());
```

means:

```text
async failure
→ convert to empty collection
→ downstream sees a successful result
```

This is correct only when the business contract explicitly says that failure may degrade to an empty collection.

Do not mechanically return:

```text
null
empty list
0
false
default object
default state
```

merely to keep the flow running and thereby disguise a system failure as a normal business result.

If failure should affect the caller, preserve failure semantics—for example by propagating it correctly when waiting on the Future, or by observing/logging it without converting the result without justification.

If degradation is allowed, be able to explain:

```text
why the failure can be ignored
what the fallback means to the business
how the caller distinguishes normal and degraded results, if needed
whether logs / metrics / alerts are required
```

Principle:

> Async recovery must come from the business contract; `exceptionally` is not a template for swallowing exceptions.

---

# 9. Waiting on Futures

Before using:

```java
join()
get()
```

define:

* whether the wait can become long;
* whether a timeout is needed;
* how exceptions propagate;
* whether a request thread will be occupied for too long;
* whether any real parallelism remains.

Avoid:

```text
execute asynchronously
→ immediately join
```

when there is no parallel benefit.

---

# 10. Coordination Tools

`CountDownLatch`, Semaphore, Barrier, and similar tools must advance coordination state correctly on every exit path.

For example:

```java
try {
    process();
} finally {
    latch.countDown();
}
```

Waiting logic must also define timeout, interruption, and child-task failure semantics.

Do not wrap a simple synchronous flow in complex coordination primitives.

---

# 11. Retries

Do not automatically retry every exception.

Before retrying, confirm:

* whether the failure is transient;
* whether the operation is idempotent;
* whether retries amplify database / downstream pressure;
* whether retry count, interval, and total wait are bounded.

Normally do not automatically retry:

* invalid parameters;
* authorization failures;
* business-validation failures;
* non-idempotent writes;
* database write failures with unknown causes.

Do not introduce infinite retries or arbitrary fixed retry counts without a basis.

---

# 12. Shared State

Prefer:

* method-local variables;
* immutable objects;
* stateless Service / Manager;
* clearly thread-safe data structures;
* atomic database operations.

Spring Beans are typically singletons, so do not store request-scoped temporary state in mutable Bean fields.

Avoid, for example:

```java
@Service
public class PlaceService {
    private String currentPlaceId;
}
```

---

# 13. synchronized

`synchronized` only provides mutual exclusion inside the current JVM. It does not automatically solve consistency for shared database data across multiple application instances.

Lock only real shared state and avoid holding a lock around an entire long business process.

Inside locks, avoid when possible:

* HTTP / RPC;
* slow database calls;
* file IO;
* long computations;
* unbounded waits.

---

# 14. Lock

After a lock is acquired, release it reliably in `finally`:

```java
lock.lock();
try {
    doSomething();
} finally {
    lock.unlock();
}
```

With `tryLock()`, release only after confirming acquisition succeeded.

When multiple locks are required, keep a stable and consistent acquisition order to reduce deadlock risk.

---

# 15. volatile and Atomic Types

`volatile` mainly guarantees visibility and some ordering; it does not make compound operations atomic.

For example:

```java
count++
```

is not atomic even if `count` is volatile.

For atomic updates, choose according to the actual scenario:

```text
AtomicInteger / AtomicLong
locks
concurrent collections
atomic database operations
```

Do not mechanically replace every field with an atomic type in the name of "thread safety."

---

# 16. Prefer Database Consistency Mechanisms for Database Concurrency

For shared database state, prefer mechanisms appropriate to the actual case:

* conditional UPDATE;
* UNIQUE Constraint;
* optimistic locking;
* pessimistic locking;
* suitable transaction isolation.

Do not default to JVM locks for multi-instance database consistency.

Read detailed rules in `transactions.md`.

---

# 17. Resource Capacity

Before increasing business concurrency, inspect downstream capacity:

```text
thread-pool concurrency
↓
database connection pool
HTTP connection pool
third-party rate limits
CPU / memory
```

More threads do not automatically mean more throughput.

If the bottleneck is a database connection pool or third-party rate limit, more threads may only increase waiting and timeouts.

---

# 18. Avoid Speculative Concurrency Optimization

Without benchmarks, monitoring, or clear evidence of a serial bottleneck, do not rewrite simple synchronous code into CompletableFuture merely because it "looks parallelizable."

Performance optimization must also account for:

* readability;
* failure behavior;
* context;
* thread-pool capacity;
* database connections;
* testing complexity.

---

# 19. Codex Concurrency Checklist

When concurrency is involved, check:

1. Whether there is real concurrency value rather than speculative optimization.
2. Whether operations are independent and whether they incorrectly share a transaction or snapshot.
3. Whether a controlled Executor is reused and whether pool capacity and rejection policy are explicit.
4. Whether SecurityContext, tenant, data permissions, MDC, and similar context are propagated reliably.
5. Whether ThreadLocal is cleaned up on every path.
6. Whether async exceptions are visible and whether `exceptionally` turns failures into normal fallback without business justification.
7. Whether Future waits, timeouts, and exception propagation are explicit.
8. Whether retries are idempotent, bounded, and do not amplify failures.
9. Whether singleton Beans store request-level mutable state.
10. Whether Lock / synchronized are released reliably and kept sufficiently narrow.
11. Whether JVM locks are incorrectly used for multi-instance database consistency.
12. Whether downstream resources such as database and HTTP connection pools can support the added concurrency.

Final principle:

> Concurrency is not a default optimization. Introduce it only when independence, benefit, context, failure semantics, and resource capacity are all clear.
