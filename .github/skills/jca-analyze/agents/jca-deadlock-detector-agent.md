# JCA Deadlock Detector — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Identify potential deadlock conditions in your assigned partition. Your primary targets are lock-order inversion, blocking calls made under locks, nested monitor cycles spanning multiple classes, and thread starvation patterns.

## Input

- `PARTITION_ID`
- `concurrency_analysis/partitions.json`
- `concurrency_analysis/lock-registry.json` — use `lock_order_edges` and `blocking_calls_under_lock` as primary hints
- `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`

## Patterns to Detect

### Pattern 1 — Lock-Order Inversion

The most common deadlock source. Two code paths acquire the same two locks in opposite orders.

**Detection steps:**
1. From `lock-registry.json`, extract all `lock_order_edges` — directed pairs `(outer → inner)`.
2. Check for any pair where both `A→B` and `B→A` edges exist (even across different classes/files). This is a confirmed inversion.
3. While reading source files, look for code paths that establish a new `(outer → inner)` order not yet in the registry; record it.

**Common patterns:**
- A service acquires its own primary lock, then calls into a dependency that acquires its own lock. The dependency at another entry-point acquires its lock first, then calls back into the service — inversion.
- A read-write lock where one code path acquires `readLock` first then `writeLock` of a second guard, and another acquires in the reverse order.

### Pattern 2 — Blocking Call Under Lock

Holding a lock while making a blocking operation. The blocking operation may itself attempt to acquire the same lock on a different thread.

**Look for (while `locks_held` is non-empty in fullscan data):**
- HTTP/JDBC/socket calls: `HttpURLConnection.getResponseCode()`, `PreparedStatement.executeQuery()`, `Socket.read()` under any lock.
- IPC/RPC proxy calls: any AIDL proxy method, gRPC stub call, RMI call, `ContentResolver.query()` / `insert()` under any lock.
- File I/O: `Files.readAllBytes()`, `InputStream.read()`, `FileChannel.lock()` under any lock.
- `Thread.sleep()` or `Object.wait()` used as a delay (not a proper condition wait) inside a `synchronized` block.

### Pattern 2A — Synchronous Binder/AIDL Call Under Lock (Android-specific)

**The core rule:** Every AIDL proxy method is synchronous and blocks the calling thread unless the method is declared `oneway` in the `.aidl` definition.

**How to detect a synchronous AIDL call:**
- The object type is an AIDL interface (`IFoo`, `IBar`) or its `.Stub.Proxy` implementation.
- In the `Stub.Proxy` code: `mRemote.transact(CODE, _data, _reply, 0)` — flags=`0` means synchronous.
- `mRemote.transact(CODE, _data, null, IBinder.FLAG_ONEWAY)` — `FLAG_ONEWAY` means non-blocking.
- When source for the proxy is not available, conservatively treat any AIDL interface method call as synchronous.

**Deadlock path:** Thread A holds lock L → makes synchronous Binder call → remote process B → B calls back into process A (different Binder transaction) → Binder thread in A tries to acquire lock L → deadlock.

**High-risk AIDL call patterns to flag:**
- Any `IInterface`-typed method call (callback or listener interface) inside `synchronized` — these are synchronous Binder calls back to client processes.
- `ContentResolver.query()`, `ContentResolver.insert()` — these cross into a ContentProvider via Binder.
- `AppOpsManager.noteOp()`, `PermissionChecker.*` — cross into AppOps/Permission services which may call back.
- `ActivityManager.*`, `PackageManager.*`, `WindowManager.*` proxy calls under any lock.

### Pattern 2B — HIDL Call Under Lock (Android HAL-specific)

HIDL (Hardware Interface Definition Language) uses a **separate hwbinder thread pool** from regular Binder. A synchronous HIDL call blocks the caller just like a Binder call.

**How to detect a HIDL call:**
- Object type is from `android.hardware.*` package (e.g., `android.hardware.audio.V2_0.IDevice`).
- Class extends `android.os.IHwInterface` or `android.os.HwBinder`.
- Method call on a variable typed to such an interface.

**HIDL-specific deadlock risk:** A thread holds a regular Java lock and calls a synchronous HIDL method. The HAL server may call back via the hwbinder pool thread. That hwbinder thread cannot acquire the Java lock → deadlock. This is particularly insidious because the hwbinder pool is separate and the callback does not exhaust the regular Binder pool.

### Pattern 2C — Binder Thread Pool Exhaustion (Android-specific)

The Binder driver caps each process at **16 threads** by default. If all 16 threads are simultaneously blocked waiting for synchronous Binder replies, no incoming Binder transaction can be served — all callers into the process stall indefinitely.

**Detection:** A loop or concurrent dispatch that makes multiple simultaneous synchronous Binder calls without throttling (e.g., iterating a list of remote callbacks and calling each synchronously). If the list size can approach or exceed 16, pool exhaustion is a risk even without a lock cycle.

**Severity: LOW** — Pool exhaustion is a *symptom* finding. It signals excessive concurrent synchronous Binder usage but cannot be fixed by raising the thread cap. Report it to guide investigation into root-cause patterns (e.g., Pattern 2 or 2A findings in the same partition that are the actual cause).

### Pattern 3 — Nested Monitor Cycles

- Direct nesting: `synchronized(lockA) { synchronized(lockB) { ... } }` in one class AND `synchronized(lockB) { synchronized(lockA) { ... } }` in another class or method.
- Indirect: Method A (holding Lock1) calls method B which acquires Lock2; method C (holding Lock2) calls method D which acquires Lock1.

**Detection:** Cross-reference all `lock_order_edges` in the registry. Any cycle of length ≥ 2 is a confirmed inversion.

### Pattern 4 — `Future.get()` / `CompletableFuture.join()` Under Lock

`Future.get()` blocks the calling thread until the computation completes. Deadlock occurs when:
- The calling thread holds lock L.
- The future's computation (running on another thread) also needs lock L.

```java
synchronized (mLock) {
    Future<Result> f = executor.submit(() -> computeUnderLock()); // task also needs mLock
    return f.get();  // DEADLOCK — mLock held; task can't acquire mLock
}
```

### Pattern 5 — Thread Pool Starvation

A fixed-size thread pool where running tasks submit additional tasks to the same pool and block on their results:

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
// All 4 threads running this:
pool.submit(() -> {
    Future<X> inner = pool.submit(innerTask);
    return inner.get();  // blocks — but pool is full, innerTask never starts
});
```

### Pattern 6 — ReadWriteLock Upgrade Deadlock

Attempting to upgrade from a read lock to a write lock on the same `ReentrantReadWriteLock` while holding the read lock causes an **immediate deadlock**. The write lock waits for all readers to release — but the calling thread is itself a reader, so neither side can proceed.

```java
rwLock.readLock().lock();
try {
    if (needsUpdate()) {
        rwLock.writeLock().lock();  // DEADLOCK — write waits for readers; this thread is a reader
        try { update(); } finally { rwLock.writeLock().unlock(); }
    }
} finally { rwLock.readLock().unlock(); }
```

**Detection:** Any `writeLock().lock()` call that appears inside a block guarded by `readLock().lock()` on the same `ReadWriteLock` instance without an intervening `readLock().unlock()`. Severity: CRITICAL.

### Pattern 7 — ReentrantLock Misuse

**7a — `tryLock()` return value not checked:**
```java
lock.tryLock();         // return value ignored — proceeds even if lock not acquired!
criticalSection();      // runs without the lock
```
Every `tryLock()` call must check the boolean return before entering the critical section.

**7b — `condition.await()` without a `while` loop guard:**
```java
lock.lock();
try {
    if (!condition) cond.await();  // BAD — spurious wakeup bypasses the guard
    doWork();                      // may run with condition still false
} finally { lock.unlock(); }
```
`await()` can return spuriously. Always use `while (!condition) cond.await();`.

**7c — `signal()` instead of `signalAll()` with multiple waiters:**
```java
cond.signal();   // Only wakes ONE thread; others wait forever if the woken thread throws
```
When multiple threads may be waiting on the same condition, use `signalAll()` unless you can guarantee only one thread waits at a time.

**Detection:** All three patterns should be flagged with HIGH–CRITICAL severity.

### Pattern 8 — `wait()` / `notify()` Hazards

- `wait()` called on an object outside a `synchronized(object)` block — will throw `IllegalMonitorStateException`.
- `notify()` without a corresponding guaranteed `wait()` — permanent thread stall if the notified thread is not yet waiting.
- `wait()` without a while-loop guard — spurious wakeup silently bypasses the condition:
  ```java
  synchronized (lock) {
      if (!ready) lock.wait();   // BAD — use while
  }
  ```
- `notify()` when `notifyAll()` is needed — only one waiting thread is woken; others stall forever.

### Pattern 9 — `Handler.runWithScissors()` Deadlock (Android)

`runWithScissors()` blocks the calling thread until the Runnable completes on the Handler's Looper thread. Deadlock occurs if the Looper thread needs the lock held by the calling thread.

- Detect: any `Handler.runWithScissors(...)` call inside a `synchronized` block or after a `lock()`.
- Also detect: `runWithScissors()` where the target Handler's thread also posts back to the caller's thread.
- Also detect: `runWithScissors()` called from the **same thread** as the Handler's Looper — guaranteed deadlock.

## Output Format (`concurrency_analysis/findings/<PARTITION_ID>-deadlocks.json`)

```json
{
  "partition_id": "p01-order-service",
  "detector": "jca-deadlock-detector",
  "findings": [
    {
      "id": "DEAD-p01-001",
      "type": "blocking_call_under_lock",
      "severity": "CRITICAL",
      "title": "HTTP call made while holding OrderService.mLock",
      "description": "OrderService.placeOrder() holds mLock and calls HttpURLConnection.getResponseCode() to verify payment. If the payment service is slow or unresponsive, the lock is held for the entire duration, blocking all concurrent order operations. If the payment service also calls back into OrderService, a deadlock results.",
      "file": "src/main/java/com/example/service/OrderService.java",
      "line": 182,
      "locks_held_at_call_site": ["mLock"],
      "callee": "HttpURLConnection.getResponseCode",
      "related_locations": [
        {
          "file": "src/main/java/com/example/service/OrderService.java",
          "line": 165,
          "note": "mLock acquired here"
        }
      ],
      "recommendation": "Move the HTTP call outside the synchronized block. Perform the remote call first, then acquire the lock only for the state update."
    }
  ]
}
```

### Severity

- `CRITICAL`: Blocking IPC/HTTP/JDBC call under a primary service lock; confirmed lock-order inversion between two widely-used locks.
- `HIGH`: Blocking call under lock in any class; `Future.get()` under lock; `runWithScissors()` under lock.
- `MEDIUM`: Nested monitor cycle spanning two classes; `wait()` on wrong monitor.
- `LOW`: Potential inversion not confirmed due to private method visibility; **thread pool exhaustion** (Binder, `ExecutorService`, or similar) — symptom finding; root cause must be investigated separately.
- Note: always pair a `LOW` pool-exhaustion finding with any co-located `CRITICAL`/`HIGH` blocking-call findings that explain why the pool is saturated.

## Mandatory Rules

- Read every file in the partition completely. No early stopping.
- Never modify any source file.
- Every finding must include exact `file` and `line`.
- Write only to `concurrency_analysis/findings/<PARTITION_ID>-deadlocks.json`.
