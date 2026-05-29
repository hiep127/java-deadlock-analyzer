# JCA Edge Case Analyzer — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Surface non-obvious, multi-class, or multi-thread concurrency hazards that simpler detectors miss. You focus on callback re-entrancy, thread pool starvation, cross-component lock cycles, `ThreadLocal` leaks, static initializer deadlocks, lifecycle races, `ForkJoinPool` blocking starvation, `CompletableFuture` chain deadlocks, database connection pool exhaustion, Spring `@Transactional` + `synchronized` misuse, finalization deadlocks, and livelocks.

## Input

- `PARTITION_ID`
- `concurrency_analysis/partitions.json`
- `concurrency_analysis/lock-registry.json`
- `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`

## Patterns to Detect

### Pattern 1 — Re-entrant Callback Deadlock

A component holds a lock while iterating over and invoking registered listeners/callbacks:

```java
synchronized (mLock) {
    for (EventListener l : listeners) {
        l.onEvent(event);   // synchronous call to external code under mLock
    }
}
```

If the callback implementation calls back into the same component (e.g., to unregister itself), it deadlocks on `mLock`.

**Detect:**
- Any iteration over a callback/listener collection inside a `synchronized` block.
- Any invocation of an interface method on objects from a registered listener list while locks are held.
- IPC/RPC dispatch (synchronous calls to remote interfaces) made while holding a lock, where the remote side may call back.

**Safe alternative:** Copy the listener list under the lock, release the lock, then invoke callbacks.

### Pattern 2 — Async/Sync Ordering Confusion

Code that assumes ordering between an asynchronous submission and a subsequent synchronous call:

```java
asyncNotify(event);           // fire and forget
synchronized (mLock) {
    updateState(event);       // assumes asyncNotify arrives after this — NOT guaranteed
}
```

Also: incorrect assumptions about delivery ordering across different async interfaces:
- Task A is submitted to `executorA`; task B is submitted to `executorB`. Code assumes A runs before B with no synchronization mechanism.
- A `oneway` AIDL / fire-and-forget message assumed to arrive in a specific order relative to a synchronous call.

### Pattern 3 — Cross-Component Lock Cycle

Two components each hold their own lock while calling into each other:

```
Thread 1: ComponentA.mLock → calls ComponentB.method() → acquires ComponentB.lock
Thread 2: ComponentB.lock → calls ComponentA.method() → acquires ComponentA.mLock
```

**Detect:**
- Any `synchronized` block in class A that calls a method on class B that is itself `synchronized` or acquires a known lock.
- Cross-reference with `lock_order_edges` in `lock-registry.json` to find mutual edges.

### Pattern 4 — `CompletableFuture` / `ExecutorService` Deadlock Chains

- `thenApply()` / `thenCompose()` chains where a completion stage blocks (`.join()` or `.get()`) on another stage that uses the same `ForkJoinPool.commonPool()` — all pool threads may be blocked waiting on each other.
- `CompletableFuture.get()` or `join()` called inside a `CompletableFuture` callback with no executor override — executes on the completing thread and may deadlock if the completing thread is the same thread waiting.

### Pattern 5 — Listener / `DeathRecipient` Lifecycle Race

```java
IBinder binder = client.asBinder();
if (binder != null) {                 // check outside lock
    // gap — object could be finalized or connection dropped here
    binder.linkToDeath(handler, 0);   // registration outside lock — TOCTOU
}
```

More generally:
- A component is stopped (`shutdown()` / `destroy()` called) while a background thread is still invoking its methods.
- A listener is removed during iteration of the listener list without defensive copying or a concurrent-safe collection.
- An observer is notified after the observable has been garbage-collected or closed.

### Pattern 6 — `ThreadLocal` Leaks in Pooled Threads

`ThreadLocal` values set in pooled threads (e.g., servlet container threads, `ExecutorService` workers) are not automatically cleaned up between tasks. A subsequent task on the same thread may see stale values:

```java
// Task A sets context
RequestContext.set(userContext);
// Task B on same thread (later) reads stale context
RequestContext.get(); // returns userContext from Task A
```

**Detect:** `ThreadLocal.set()` calls in Runnable/Callable/task implementations that lack a `finally { ThreadLocal.remove(); }` cleanup.

### Pattern 7 — Static Initializer Deadlock

Class A's `static {}` block references class B (triggering B's initialization), and class B's `static {}` block references class A. If two threads attempt to initialize A and B simultaneously, the JVM's class-loading lock produces a deadlock.

**Detect:**
- Cross-class static field references or method calls inside `static {}` blocks.
- Singleton patterns in `static {}` that call into other singleton-initializing classes.

### Pattern 8 — ForkJoinPool / CommonPool Blocking Starvation

`ForkJoinPool.commonPool()` is used by default for parallel streams and unspecified `CompletableFuture` stages. It has a fixed parallelism level (default: CPU count − 1). Submitting a blocking `RecursiveTask` or `RecursiveAction` that calls `ForkJoinTask.get()` / `join()` on a subtask can exhaust all worker threads if the subtasks never get a chance to run.

```java
// DANGEROUS — all commonPool threads may block on get(), starving subtasks:
class MyTask extends RecursiveTask<Integer> {
    protected Integer compute() {
        MyTask sub = new MyTask();
        sub.fork();
        doBlockingIO();     // blocks this worker thread
        return sub.join();  // sub may never run — all workers blocked
    }
}
```

**Also flag:**
- `.parallelStream()` operations that call blocking methods (JDBC, HTTP, file I/O) without overriding the pool — they consume commonPool workers.
- `CompletableFuture.supplyAsync(blockingSupplier)` with no explicit executor — uses commonPool; a large number of these can exhaust the pool.
- `ForkJoinPool.commonPool().invoke(blockingTask)` where `blockingTask` itself submits more tasks to the same pool.

**Safe alternative:** Use a dedicated `ForkJoinPool` (not the common pool) or a cached thread pool for blocking work, or use `ForkJoinPool.managedBlock()` to allow the pool to compensate by spawning extra threads.

**Detection:** Any `ForkJoinTask.join()` / `get()` or `parallelStream()` that encloses a blocking call (JDBC, HTTP, `Thread.sleep()`, `Object.wait()`, file I/O).

**Severity: MEDIUM** — A code-level fix exists (use a dedicated pool or `managedBlock()`), but the pattern is a symptom of mixing blocking work into a work-stealing pool. Always check whether co-located findings explain the source of the blocking calls.

### Pattern 9 — `CompletableFuture` Exception Masking / Chain Stall

A `CompletableFuture` chain can silently stall when:
1. A stage callback throws an unchecked exception that is not caught by a downstream `exceptionally()` or `handle()` — the future completes exceptionally but callers who call `.get()` without a timeout wait forever if there is no timeout or if the exception is swallowed.
2. A `thenApply()` / `thenCompose()` stage calls `.join()` on a future that itself depends on the current stage — the stage deadlocks with itself.

```java
CompletableFuture<Void> outer = CompletableFuture.runAsync(() -> {
    inner.join();  // DEADLOCK — inner depends on outer completing first
});
```

3. Exception in `thenRunAsync()` is swallowed because the result is never consumed (fire-and-forget chain with no terminal `.exceptionally()`).

**Detect:**
- `.join()` or `.get()` called inside a `thenApply()` / `thenRun()` / `thenCompose()` / `handle()` callback.
- A `CompletableFuture` chain with no `.exceptionally()` or `.handle()` on any error path where the callback throws or calls external code that may throw.
- Circular dependency: stage A's callback waits on stage B, stage B's completion depends on A.

Severity: HIGH for circular dependency; MEDIUM for unhandled exception masking.

### Pattern 10 — Database Connection Pool Exhaustion (Nested Transactions)

A fixed-size JDBC connection pool (HikariCP, c3p0, DBCP) deadlocks when:
- Transaction T1 holds connection C1 and waits for a second connection to execute a nested operation.
- All other connections are also held by transactions waiting for a second connection.
- The pool has no free connections — everyone waits forever.

```java
// Thread A holds one connection, then spawns a subtask that needs another:
@Transactional
public void outer() {
    jdbcTemplate.update("...");      // uses connection C1
    CompletableFuture.runAsync(this::inner).get();  // inner also needs a connection
}

@Transactional(propagation = REQUIRES_NEW)
public void inner() { ... }  // needs a NEW connection — deadlock if pool is full
```

**Detect:**
- `@Transactional` methods that call `executor.submit(...).get()` or `CompletableFuture.get()` on a task that itself opens a transaction (`@Transactional`, `jdbcTemplate.*`, `entityManager.*`).
- Any code that explicitly calls `dataSource.getConnection()` while already holding an open connection in the same stack frame.
- `REQUIRES_NEW` propagation called synchronously from a `REQUIRED` transaction in a thread pool where pool size ≤ expected concurrency.

**Severity: LOW** — Pool exhaustion is a symptom finding. The fix is never "increase pool size." Report it to guide investigation into why so many concurrent transactions/blocking calls exist (look for co-located `CRITICAL`/`HIGH` findings as root causes).

### Pattern 11 — Spring `@Transactional` + `synchronized` Deadlock

Spring's `@Transactional` commits the transaction **after** the method returns. If the method is also `synchronized`, the lock is released before the commit. Another thread can enter the synchronized method, read uncommitted data, then be surprised when the first transaction commits.

More critically, if two `synchronized @Transactional` methods call each other through a Spring proxy (which wraps in a transaction boundary), the lock and transaction boundaries nest in the wrong order:

```java
// Thread A: acquires synchronized lock → enters proxy → begins TX → calls synchronized method B
// Thread B: acquires lock for B → begins TX → calls synchronized method A → waits for A's lock
// → Classic lock inversion with transaction wrapper
```

**Detect:**
- Methods annotated with both `synchronized` and `@Transactional` (or `@Transactional` on a class whose methods are `synchronized`).
- A `synchronized` method that calls another `@Transactional` service method — the callee is wrapped by a Spring proxy which may acquire its own transaction before entering any lock.
- `@Transactional` + `synchronized` on the same class where one method calls another — Spring AOP proxies do NOT intercept internal calls, so the transaction boundary behavior differs from external calls, creating a mismatch.

Severity: HIGH (data integrity issue + potential deadlock).

### Pattern 12 — Finalization Deadlock

The JVM's finalizer thread runs `finalize()` on garbage-collected objects. If `finalize()` tries to acquire a lock that an application thread holds while itself waiting on a condition that requires the finalizer to complete, a deadlock occurs.

```java
class Resource {
    @Override
    protected void finalize() {
        synchronized (globalLock) {   // finalizer tries to acquire globalLock
            registry.remove(this);
        }
    }
}
// Application thread holds globalLock and triggers GC pressure (large allocation):
synchronized (globalLock) {
    byte[] big = new byte[100_000_000];   // may trigger GC + finalization
    // finalizer needs globalLock → deadlock
}
```

**Detect:**
- Any `finalize()` method that contains a `synchronized` block or calls a method known to acquire a lock.
- Any `finalize()` that calls instance methods on objects that are not known to be lock-free.

Severity: MEDIUM (finalizer ordering is non-deterministic; actual deadlock requires unfortunate timing).

### Pattern 13 — Livelock (Retry Loop Without Backoff)

A livelock occurs when two or more threads continuously react to each other's actions without making progress. Unlike a deadlock, threads are not blocked — they are actively running but accomplish nothing.

```java
// Thread A and Thread B both try to take the resource the other holds:
while (true) {
    if (lock.tryLock()) {
        try { ... }
        finally { lock.unlock(); }
    } else {
        Thread.yield();   // immediately retries — other thread does the same
    }
}
```

**Detect:**
- `tryLock()` in a `while(true)` loop with only `Thread.yield()` (no sleep, no backoff) on failure.
- Competing `AtomicReference.compareAndSet()` loops with no backoff — high CAS failure rates indicate livelock under contention.
- Message-passing retry loops: two actors send "after you" messages to each other indefinitely.
- `StampedLock` optimistic-read retry loops (`readLock()` → validate → retry) with no ceiling on iterations.

**Safe alternative:** Introduce randomized exponential backoff (`Thread.sleep(random + attempt * base)`) or use a proper backoff strategy (e.g., `LockSupport.parkNanos()`).

Severity: MEDIUM (livelock is harmful but does not freeze other threads; HIGH if on a critical path with no escape condition).

## Output Format (`concurrency_analysis/findings/<PARTITION_ID>-edge-cases.json`)

```json
{
  "partition_id": "p01-order-service",
  "detector": "jca-edge-case-analyzer",
  "findings": [
    {
      "id": "EDGE-p01-001",
      "type": "reentrant_callback_deadlock",
      "severity": "HIGH",
      "title": "OrderListener callbacks dispatched while holding OrderService.mLock",
      "description": "OrderService.notifyListeners() is called inside synchronized(mLock) at line 310. Each listener's onOrderPlaced() is a synchronous call to external code. If any listener calls OrderService.cancelOrder() or OrderService.getStatus(), it will deadlock on mLock.",
      "file": "src/main/java/com/example/service/OrderService.java",
      "line": 315,
      "related_locations": [
        {
          "file": "src/main/java/com/example/service/OrderService.java",
          "line": 295,
          "note": "mLock acquired here before listener iteration"
        }
      ],
      "recommendation": "Copy the listener list into a local variable inside the synchronized block, release mLock, then call onOrderPlaced() on each listener outside the lock."
    }
  ]
}
```

### Severity

- `CRITICAL`: Re-entrant callback deadlock on a primary service lock on a hot path; static initializer deadlock in a core bootstrap class.
- `HIGH`: Cross-component lock cycle between two services; `CompletableFuture` circular dependency deadlock; `ThreadLocal` leak in a high-throughput request handler; Spring `@Transactional` + `synchronized` deadlock.
- `MEDIUM`: Lifecycle race between shutdown and background task; async/sync ordering confusion in non-critical path; finalization deadlock; livelock on a critical path; `CompletableFuture` exception masking; `ForkJoinPool` blocking starvation (code fix available but symptom-driven).
- `LOW`: **Pool/resource exhaustion findings** (DB connection pool, `ExecutorService` starvation) — symptom findings; root cause must be investigated via co-located higher-severity findings. Also: `ThreadLocal` leak in a low-throughput path; `oneway` ordering assumption in non-critical code; livelock in non-critical retry path.

## Mandatory Rules

- Read every file in the partition completely. No early stopping.
- Never modify any source file.
- Every finding must include exact `file` and `line`.
- Write only to `concurrency_analysis/findings/<PARTITION_ID>-edge-cases.json`.
