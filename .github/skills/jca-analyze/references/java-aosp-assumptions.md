# AOSP Java Analysis Assumptions

This document defines the foundational assumptions all JCA agents must apply when analyzing AOSP Android Framework Java code.

---

## 1. Thread Model Assumptions

### 1.1 Thread Taxonomy

JCA assumes the following thread categories exist in every AOSP system service:

| Thread Type | Description | Lock Behavior |
|---|---|---|
| **Main/System Server Thread** | The thread that initializes the service (may be SystemServer's main thread). | Typically holds the service's primary lock for long operations. |
| **Binder Thread Pool** | Up to 16 threads from the Binder driver thread pool. Calls `onTransact()` / AIDL stub methods. | Must not assume any lock is held on entry. Must be thread-safe. |
| **Service Handler Thread** | A `HandlerThread` created by the service for serialized async operations. | Runs messages sequentially; does not hold the primary lock unless explicitly acquired. |
| **Worker/Background Thread** | Ad-hoc threads for I/O, computation. | Must not assume any service lock is held. |

### 1.2 Thread Entry Points

The following method signatures are considered thread entry points and must be treated as potentially concurrent with all other threads:

- Any method in an AIDL-generated `*.Stub` subclass (called on Binder threads).
- `Handler.handleMessage(Message)` overrides (called on the Handler's Looper thread).
- `Runnable.run()` passed to any Executor, Handler, or Thread (called on the target thread).
- `IBinder.DeathRecipient.binderDied()` (called on a Binder thread, not the service's main thread).
- `ContentObserver.onChange()` (called on the ContentResolver thread).
- `BroadcastReceiver.onReceive()` when registered with a Handler (called on the Handler's thread).

---

## 2. Synchronization Assumptions

### 2.1 Lock Granularity Convention

AOSP services typically follow this lock hierarchy (coarse to fine):

1. **Service primary lock** (`mLock`, `mStateLock`) — protects the bulk of service state.
2. **Sub-component locks** (`mFocusLock`, `mDeviceLock`, `mSessionLock`) — protects a specific subsystem.
3. **Collection locks** — a lock protecting a specific list or map.

JCA assumes: a lock acquired at a coarser level must be acquired **before** any finer-level lock. Acquiring in the reverse order is an inversion.

### 2.2 `@GuardedBy` Annotations

When a field is annotated `@GuardedBy("mLock")` (or similar), JCA treats any access to that field without holding the named lock as a confirmed violation, not just a potential one.

### 2.3 `final` Fields

Fields declared `final` and initialized in the constructor are thread-safe after construction (Java Memory Model guarantee). JCA must not flag `final` field reads as races unless the object reference itself is shared unsafely before construction completes.

### 2.4 `volatile` Semantics

- A `volatile` field guarantees visibility (reads see the latest write) and ordering (no reordering around volatile accesses), but **not atomicity** for compound operations.
- A `volatile boolean` used only as a stop flag (write once in one thread, read in another) is safe.
- A `volatile` field used for any compound operation (`++`, CAS without `AtomicXxx`) is not safe.

---

## 3. Binder/IPC Assumptions

### 3.1 Synchronous vs. Asynchronous Binder Calls

- All AIDL-generated proxy methods without the `oneway` keyword are **synchronous**: the calling thread blocks until the remote stub returns.
- `oneway` methods are **asynchronous**: the calling thread returns immediately; the call is queued. However, `oneway` does not bypass the Binder thread pool concurrency on the receiving end.

### 3.2 Binder Re-entrance

Binder IPC in Android is **not re-entrant by default**. If thread A holds a lock and makes a synchronous Binder call to process B, and process B calls back into process A (to a different method), the callback is served by a **different Binder thread** in process A's thread pool. That Binder thread will deadlock if it tries to acquire the same lock that thread A is holding.

### 3.3 Binder Thread Count

The Binder driver maintains a maximum of 16 threads per process by default. Under heavy IPC load, a Binder call can block waiting for an available thread. Deadlocks can be caused by saturating the thread pool as well as by lock cycles.

---

## 4. Handler/Looper Assumptions

### 4.1 Looper Thread Affinity

A `Handler`'s `handleMessage()` always runs on the `Looper` thread the handler was created for. Code within `handleMessage()` can safely assume single-threaded access with respect to other messages on the same `Looper`, but not with respect to Binder threads or other Looper threads.

### 4.2 `runWithScissors()` Contract

`Handler.runWithScissors(Runnable r, long timeout)`:
- Blocks the **calling thread** until `r` completes on the Handler's Looper thread, or until `timeout` milliseconds elapse.
- Returns `false` if the Looper is quitting or the timeout expired without execution.
- Is **always a deadlock risk** if the calling thread holds any lock that the Looper thread might need, directly or transitively.
- `timeout == 0` means infinite wait — system code that passes `0` while holding a lock is particularly dangerous.

---

## 5. Java Memory Model Assumptions

### 5.1 Happens-Before Rules JCA Relies On

- Thread start: all actions in thread A before `t.start()` happen-before any action in thread `t`.
- Lock release/acquire: all actions in thread A before releasing lock L happen-before all actions in thread B after acquiring lock L.
- `volatile` write/read: a volatile write happens-before a subsequent volatile read of the same variable.
- `final` field: all writes to `final` fields in a constructor are safely published once the constructor returns.

### 5.2 Out-of-Thin-Air Values

JCA does not assume out-of-thin-air value generation (OOTA) for 32-bit primitives. 64-bit `long` and `double` are **not** guaranteed atomic on 32-bit architectures without `volatile` or synchronization; flag any unsynchronized `long`/`double` access.

---

## 6. False Positive Suppression Rules

The following patterns are safe and must **not** be flagged:

- A field initialized in the constructor and never written after construction (effectively final, even if not declared `final`), as long as the object is safely published.
- Double-checked locking on a `volatile` field following the correct Java 5+ idiom.
- `ConcurrentHashMap`, `CopyOnWriteArrayList`, and other `java.util.concurrent` thread-safe collections — their internal operations are safe; only flag misuse of the collection reference itself or compound operations on its contents.
- `AtomicInteger.incrementAndGet()`, `AtomicReference.compareAndSet()`, and other fully-atomic operations.
- `Looper.myLooper()` checks used as thread-affinity assertions before a non-synchronized access (common in Android UI framework code; these are correct by design).
