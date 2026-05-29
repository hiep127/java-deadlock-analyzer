# Java Concurrency — Known Edge Cases & Hazard Patterns

This document catalogues known or commonly observed concurrency hazard patterns in Java applications. JCA detector agents must actively search for these patterns in any Java codebase.

---

## 1. Lock Architecture Anti-Patterns

### 1.1 God Lock / Coarse-Grained Primary Lock

A single lock (e.g., `private final Object mLock = new Object()`) is used to protect the entirety of a large component's state. Hazards:

- Any thread that needs any state must contend for the one lock, creating a bottleneck.
- Long-running operations (I/O, IPC calls) held under the lock block all other operations.
- As the class grows, it becomes easy to accidentally call external code while holding the lock.

**Detection:** A single lock field whose acquisition sites span more than 20 distinct methods, or where external calls (IPC, I/O) appear inside `synchronized(mLock)`.

### 1.2 Lock-Order Inversion (Classic Deadlock)

Two locks `A` and `B` exist. Thread 1 acquires `A` then `B`. Thread 2 acquires `B` then `A`. Both threads can stall forever.

**Detection:** From the lock graph, find any pair where both `A→B` and `B→A` edges exist across any two code paths (even across different classes or modules). This is a confirmed inversion.

### 1.3 Missing `unlock()` in Non-`finally` Code

`ReentrantLock.unlock()` called outside a `finally` block means an exception will leave the lock permanently held, preventing all future acquisitions.

**Required pattern:**
```java
lock.lock();
try {
    // work
} finally {
    lock.unlock();
}
```

---

## 2. Callback and Listener Re-Entrancy

### 2.1 Listener Dispatch Under Lock

A common pattern: a component holds a lock while iterating over registered listeners and calling them:

```java
synchronized (mLock) {
    for (Listener l : mListeners) {
        l.onEvent(event);   // DANGEROUS — external call under lock
    }
}
```

If any listener calls back into the component (e.g., to unregister itself or query state), it will deadlock on `mLock`.

**Safe alternative:** Copy the listener list under the lock, release the lock, then invoke callbacks on the copy.

### 2.2 Concurrent Modification During Iteration

`mListeners` (a `List` or `Set`) iterated in one thread while another thread adds/removes from it without consistent synchronization → `ConcurrentModificationException` at runtime (a symptom of the underlying race).

### 2.3 Synchronous IPC Callback Under Lock

If a component holds lock `L` and makes a synchronous RPC/Binder call to a remote component, and the remote component's handler calls back into the originating component requiring lock `L`, a cross-process deadlock occurs.

---

## 3. Blocking Calls Under Locks

### 3.1 I/O Under Lock

File I/O, network calls, or database queries made inside a `synchronized` block or while holding a `ReentrantLock` block the holding thread for an unbounded time, preventing all other threads from entering the lock.

**Detection:** Any call to `InputStream.read()`, `OutputStream.write()`, `Socket.*`, `HttpURLConnection.*`, `JDBC.*`, or `Files.*` inside a `synchronized` block.

### 3.2 Native / JNI Calls Under Lock

A JNI call made while holding a Java lock may block on a native mutex. If the native side then attempts to call back into Java (via `CallVoidMethod` etc.) and that callback needs the same Java lock, a cross-language deadlock occurs.

**Detection:** `native` method calls made inside `synchronized` blocks.

### 3.3 `Object.wait()` / `Condition.await()` Without While-Loop Guard

`wait()` can return spuriously without being notified. Code that uses `if` instead of `while` as the guard:

```java
synchronized (lock) {
    if (!condition) lock.wait();  // BAD — spurious wakeup bypasses the condition
    // proceed as if condition is true
}
```

**Correct pattern:** Use `while (!condition) lock.wait();`

---

## 4. Executor and Thread Pool Hazards

### 4.1 Thread Pool Deadlock (Task Starvation)

A fixed-size thread pool where all running tasks submit new tasks to the same pool and block waiting for them to complete. If the pool is full, the new tasks never start, and the running tasks never finish — deadlock.

**Detection:** `ExecutorService.submit()` or `executor.execute()` calls inside a task body, followed by `.get()` or `.join()` on the result, where both use the same pool.

### 4.2 `Future.get()` / `CompletableFuture.join()` Under Lock

```java
synchronized (mLock) {
    result = future.get();   // blocks while holding mLock
}
```

If the future's completion action needs `mLock`, this deadlocks.

### 4.3 `SingleThreadExecutor` False Thread-Safety

Code that assumes a `newSingleThreadExecutor()` makes shared state safe to access from other threads. Other threads that directly access the same shared state (without going through the executor) race with the executor's tasks.

---

## 5. `volatile` and Atomic Misuse

### 5.1 Compound `volatile` Operations

```java
volatile int mCounter;
mCounter++;   // NOT atomic: read → increment → write as three separate operations
```

Use `AtomicInteger.incrementAndGet()` instead.

### 5.2 Non-Atomic Check-Then-Act on `volatile`

```java
if (mVolatileFlag) {         // read
    mVolatileFlag = false;   // write (gap between read and write — another thread can interleave)
    doWork();
}
```

**Fix:** Use `AtomicBoolean.compareAndSet(true, false)`.

### 5.3 `volatile` Object Reference with Non-Atomic State

A `volatile` reference to a mutable object provides visibility of the reference itself but not of the object's fields. Mutations to the referenced object's fields are not automatically visible.

---

## 6. `static` Initializer and Class-Loading Deadlocks

### 6.1 Cross-Class Static Initialization Cycle

If class A's `static {}` block references class B, and class B's `static {}` block references class A, and both are initialized simultaneously from different threads, the JVM's class-loading lock produces a deadlock.

**Detection:** Circular static field references or method calls during class initialization.

---

## 7. Lifecycle and Registration Races

### 7.1 Registration-Before-Start / Deregistration-After-Stop Gaps

```java
IBinder binder = client.asBinder();
if (binder != null) {                 // check outside lock
    // gap — binder could die here
    binder.linkToDeath(handler, 0);   // registration outside lock
}
```

The correct pattern holds the lock around both the null check and `linkToDeath`.

### 7.2 Listener Cleanup Race

A listener is invoked on a background thread at the same moment the registering component calls `removeListener()` on the main thread. If `removeListener()` releases resources the listener's callback depends on, a use-after-free or NPE occurs.

### 7.3 `ThreadLocal` Leaks in Pooled Threads

`ThreadLocal` values set in pooled threads (e.g., `ExecutorService` workers, servlet containers) are not automatically cleaned up between tasks. A task may see stale `ThreadLocal` values left by a previous task that ran on the same thread.

---

## 8. Cross-Component Lock Cycles

If two components each hold their own lock while calling into each other, a mutual lock inversion can deadlock both:

- Component A holds `lockA` and calls `ComponentB.method()` which acquires `lockB`.
- Component B holds `lockB` and calls `ComponentA.method()` which acquires `lockA`.

**Detection:** Any call from a `synchronized` block in one class to a method in a different class or module that is itself `synchronized` or acquires a different known lock.
