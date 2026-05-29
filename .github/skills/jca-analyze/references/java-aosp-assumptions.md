# Java Concurrency Analysis Assumptions

This document defines the foundational assumptions all JCA agents must apply when analyzing Java source code.

---

## 1. Thread Model Assumptions

### 1.1 Thread Taxonomy

JCA assumes the following thread categories may exist in any Java application:

| Thread Type | Description | Lock Behavior |
|---|---|---|
| **Main Thread** | The thread that starts the application (`main()`) or initializes a component. | May hold primary locks during initialization or coordinated shutdown. |
| **Thread Pool Workers** | Threads managed by `ExecutorService`, `ForkJoinPool`, or custom `ThreadPoolExecutor`. | Must not assume any lock is held on entry. Must be thread-safe. |
| **Scheduled Threads** | Threads fired by `ScheduledExecutorService` or `Timer`/`TimerTask`. | Run independently; must not assume any application lock is held. |
| **Event Dispatch Thread (EDT)** | Used in UI frameworks (Swing/JavaFX). | All UI mutations must occur here; calling blocking operations on EDT is a hazard. |
| **Handler/Looper Thread** | Single-threaded message loop (common in Android). | Runs messages sequentially; does not hold the primary lock unless explicitly acquired. |
| **Callback/Listener Thread** | Thread that delivers async notifications (e.g., from a library or IPC layer). | Must not assume any application lock is held. |
| **Background/Worker Thread** | Ad-hoc threads for I/O or computation. | Must not assume any service lock is held. |

### 1.2 Thread Entry Points

The following method signatures are considered thread entry points and must be treated as potentially concurrent with all other threads:

- `Runnable.run()` / `Callable.call()` passed to any `Executor`, `Thread`, or thread pool.
- `Thread.run()` overrides (subclassing `Thread`).
- `TimerTask.run()` overrides.
- `Handler.Callback.handleMessage()` overrides (called on the Handler's Looper thread).
- IPC/RPC stub methods (e.g., generated AIDL stubs, gRPC service implementations) — called on the framework's thread pool.
- `IBinder.DeathRecipient.binderDied()` — called on a background thread, not the service's primary thread.
- `CompletableFuture` completion callbacks when no explicit executor is supplied (may run on the completing thread).
- `ContentObserver.onChange()` — called on the ContentResolver thread (Android).
- `BroadcastReceiver.onReceive()` — called on the main thread or the Handler's thread (Android).

---

## 2. Synchronization Assumptions

### 2.1 Lock Granularity Convention

Well-structured Java services typically follow this lock hierarchy (coarse to fine):

1. **Primary/service lock** (`mLock`, `lock`, `stateLock`) — protects the bulk of component state.
2. **Sub-component locks** — protect a specific subsystem or collection.
3. **Fine-grained / per-entry locks** — protect individual records in a map or list.

JCA assumes: a lock acquired at a coarser level must be acquired **before** any finer-level lock on the same code path. Acquiring in reverse order is an inversion.

### 2.2 `@GuardedBy` Annotations

When a field is annotated `@GuardedBy("lockName")` (from JSR-305 or `androidx.annotation`), JCA treats any access to that field without holding the named lock as a **confirmed violation**, not just a potential one.

### 2.3 `final` Fields

Fields declared `final` and initialized in the constructor are thread-safe after construction (Java Memory Model guarantee). JCA must not flag `final` field reads as races unless the object reference itself is shared unsafely before construction completes.

### 2.4 `volatile` Semantics

- A `volatile` field guarantees visibility (reads see the latest write) and ordering (no reordering around volatile accesses), but **not atomicity** for compound operations.
- A `volatile boolean` used only as a stop flag (write once in one thread, read in another) is safe.
- A `volatile` field used for any compound operation (`++`, CAS without `AtomicXxx`) is not safe.

---

## 3. IPC / RPC Assumptions

### 3.1 Synchronous vs. Asynchronous Calls

- **Synchronous** remote calls (blocking RPC, synchronous Binder calls, JDBC calls, blocking HTTP) hold the calling thread until the remote side returns. Any lock held during such a call is held across the entire remote operation.
- **Asynchronous** calls (`CompletableFuture`, `oneway` Binder, non-blocking I/O) return quickly. However, the completion callback still runs on some thread and may attempt to acquire locks.

### 3.2 Callback Re-Entrance

Remote call frameworks are generally **not re-entrant** by design. If thread A holds a lock and makes a synchronous IPC call to component B, and B calls back into component A via a different method, the callback is served by a **different thread** in A's thread pool. That thread will deadlock if it tries to acquire the same lock that thread A is holding.

### 3.3 Thread Pool Exhaustion Deadlock

If all threads in a fixed-size pool are blocked waiting for a task that itself needs a thread from the same pool, the pool deadlocks. JCA should flag patterns where tasks submitted to a pool may themselves block indefinitely on the same pool.

---

## 3A. Android Binder / AIDL / HIDL Deep Dive

This section defines the precise semantics agents must apply when analyzing Android IPC code. These rules apply to any codebase using `android.os.Binder`, AIDL-generated stubs, or HIDL interfaces.

### 3A.1 Binder Call Synchrony — The Core Rule

**Every AIDL-generated proxy method is SYNCHRONOUS and BLOCKS the calling thread by default**, unless the method is declared with the `oneway` keyword in the `.aidl` file.

How to determine if a call is synchronous at a Java call site:
1. Look at the interface type of the object being called (e.g., `IFooService`).
2. Find the corresponding `.aidl` file or the generated `Stub.Proxy` class.
3. In the `Stub.Proxy` implementation, a synchronous call passes `0` as the flags argument to `mRemote.transact(CODE, data, reply, 0)`. A `oneway` call passes `IBinder.FLAG_ONEWAY` = `1`.
4. If the flags argument is `0` (or the flag constant is absent), **the call blocks the caller**.

Any lock held when such a call is made is held for the entire round-trip across the process boundary.

### 3A.2 `oneway` Semantics — What It Does and Does NOT Guarantee

`oneway` methods:
- Return to the caller **immediately** — the calling thread does not block.
- Are **queued** in the Binder driver and delivered to the remote process asynchronously.
- Guarantee delivery ordering **per-interface per-caller**, but NOT across different interfaces or callers.
- The completion callback (if any) still runs on a Binder thread pool thread in the remote process.

**Hazards with `oneway`:**
- Code that assumes a `oneway` call has been processed before a subsequent synchronous call on the same interface — this ordering is NOT guaranteed if the callee is busy.
- A `oneway` callback dispatched from the server while holding a lock (even though the server call returns immediately, the Binder thread delivering it to the client may try to acquire the client's lock re-entrantly).
- Mixing `oneway` and synchronous calls on the same interface and assuming ordering between them.

**Detection signature for `oneway` confusion:**
```java
// In the generated Stub.Proxy:
mRemote.transact(TRANSACTION_foo, _data, null, IBinder.FLAG_ONEWAY); // oneway — non-blocking
mRemote.transact(TRANSACTION_bar, _data, _reply, 0);                // synchronous — BLOCKS
```
Flag code that mixes both and assumes strict ordering without an explicit synchronization barrier.

### 3A.3 Binder Thread Pool Limits

The Android Binder driver maintains a thread pool per process:
- **Default maximum: 16 threads** per process (`/proc/sys/android/binder/max_threads` or set via `ProcessState::setThreadPoolMaxThreadCount()`).
- If all 16 threads are blocked (e.g., all waiting for synchronous Binder replies), the process cannot handle any incoming Binder calls — all callers into that process stall.
- This is a **thread pool exhaustion deadlock**: no lock cycle needed; the process simply has no available threads to serve the callback.

**Detection:** Flag any code path where multiple concurrent synchronous Binder calls are made that could collectively saturate the thread pool (e.g., in a loop or from many concurrent workers without throttling).

### 3A.4 Binder Re-Entrance (Cross-Process)

Binder IPC is **not re-entrant across process boundaries** by default:
- Process A holds lock L and makes synchronous Binder call → Process B.
- Process B calls back into Process A synchronously (a different Binder transaction).
- The callback arrives on a **different Binder thread** in Process A's pool.
- That thread attempts to acquire lock L → **deadlock**.

This is distinct from in-process re-entrance. The JVM's `synchronized` is re-entrant within the same thread, but the callback arrives on a *different* thread, so re-entrance does not apply.

**Detection signature:**
```java
synchronized (mLock) {
    mRemoteService.doSomething(); // synchronous Binder call — Process B may call back
    // If Process B calls e.g. ILocalCallback.onResult() synchronously,
    // and onResult() needs mLock → deadlock
}
```

### 3A.5 HIDL (Hardware Interface Definition Language)

HIDL is Android's IPC mechanism for Hardware Abstraction Layer (HAL) communication, used alongside (and being gradually replaced by) AIDL. Key differences from regular Binder:

| Property | Binder (AIDL) | HIDL (hwbinder) |
|---|---|---|
| Transport | `/dev/binder` | `/dev/hwbinder` |
| Thread pool | Regular Binder pool (max 16) | Separate hwbinder pool |
| Java stubs | `IFoo.Stub` / `IFoo.Stub.Proxy` | `IFoo` (via generated Java wrappers) |
| Sync default | Synchronous unless `oneway` | Synchronous unless `oneway` |
| Callback style | `IBinder.linkToDeath` | `IBase.linkToDeath` equivalent |

**HIDL Java interface detection:**
- Generated HIDL Java classes live in packages like `android.hardware.*` (e.g., `android.hardware.audio.V2_0.IDevice`).
- They extend `android.os.IHwInterface` or `android.os.HwBinder`.
- Methods are called via `mHwService.methodName(...)` where `mHwService` is typed to a HIDL interface.

**HIDL-specific deadlock hazards:**
- A Java thread holds a regular Binder lock and calls a synchronous HIDL method. The HIDL method may callback into the Java process via the hwbinder pool — a different thread that needs the same lock → deadlock.
- HIDL death notifications (`serviceDied()`) arrive on a hwbinder thread, not a regular Binder thread — if `serviceDied()` needs a lock held by a Binder thread, deadlock.
- Mixing HIDL `oneway` callbacks with synchronous HIDL calls and assuming ordering.

**Detection signature:**
```java
synchronized (mLock) {
    mHwAudioDevice.setParameters(params); // HIDL synchronous call via hwbinder
    // hwbinder callback thread cannot acquire mLock → deadlock
}
```

### 3A.6 AIDL Callback Interface Re-Entrancy Pattern

The most common Android deadlock involves a service dispatching callbacks to registered clients while holding its own lock:

```java
// SERVER SIDE — DANGEROUS PATTERN:
synchronized (mLock) {
    for (IClientCallback cb : mCallbacks) {
        cb.onEvent(event); // synchronous Binder call back to client process
        // Client's Binder thread for this call cannot acquire mLock
        // if client immediately calls back into this service
    }
}
```

**Detection:** Any call to a method on an `IInterface`-typed object (i.e., an AIDL proxy) inside a `synchronized` block, where the interface type is a registered callback or listener (named `ICallback`, `IListener`, `IObserver`, `IDispatcher`, or similar).

---

## 4. Executor / Handler Assumptions

### 4.1 Single-Threaded Executor Affinity

Code running inside a `newSingleThreadExecutor()` or a `Handler` with a single `Looper` thread can safely assume single-threaded sequential access **with respect to other tasks on the same executor/handler**. It cannot assume safety with respect to other executors or threads that directly access shared state.

### 4.2 `Handler.runWithScissors()` Contract (Android)

`Handler.runWithScissors(Runnable r, long timeout)`:
- Blocks the **calling thread** until `r` completes on the Handler's Looper thread.
- Is **always a deadlock risk** if the calling thread holds any lock that the Looper thread might need, directly or transitively.
- `timeout == 0` means infinite wait — code that passes `0` while holding a lock is particularly dangerous.

### 4.3 `CompletableFuture.join()` / `Future.get()` Under Lock

Calling `.join()` or `.get()` on a future while holding a lock blocks the thread until the future completes. If the future's completion depends on acquiring the same lock, a deadlock occurs.

---

## 5. Java Memory Model Assumptions

### 5.1 Happens-Before Rules JCA Relies On

- **Thread start:** all actions in thread A before `t.start()` happen-before any action in thread `t`.
- **Lock release/acquire:** all actions in thread A before releasing lock L happen-before all actions in thread B after acquiring lock L.
- **`volatile` write/read:** a volatile write happens-before a subsequent volatile read of the same variable.
- **`final` field:** all writes to `final` fields in a constructor are safely published once the constructor returns.
- **`Thread.join()`:** all actions in thread `t` happen-before `t.join()` returns in the joining thread.

### 5.2 Long/Double Non-Atomicity

64-bit `long` and `double` reads/writes are **not** guaranteed atomic on 32-bit JVMs without `volatile` or synchronization. Flag any unsynchronized `long`/`double` field access shared across threads.

---

## 6. False Positive Suppression Rules

The following patterns are safe and must **not** be flagged:

- A field initialized in the constructor and never written after construction (effectively final), as long as the object is safely published.
- Double-checked locking on a `volatile` field following the correct Java 5+ idiom.
- `ConcurrentHashMap`, `CopyOnWriteArrayList`, and other `java.util.concurrent` thread-safe collections — their internal operations are safe; only flag misuse of the collection reference itself or compound operations on its contents.
- `AtomicInteger.incrementAndGet()`, `AtomicReference.compareAndSet()`, and other fully-atomic operations.
- Fields accessed only within a `static { }` initializer (JVM class-loading provides single-threaded execution of static initializers).
- Fields wrapped in `Collections.unmodifiableX()` with no write sites outside construction.
