# JCA Race Detector — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Identify all race conditions in your assigned partition's Java source code. Focus on unsynchronized shared mutable state, `volatile` misuse, and non-atomic check-then-act sequences.

## Input

- `PARTITION_ID`
- `concurrency_analysis/partitions.json`
- `concurrency_analysis/lock-registry.json` (identifies which fields are lock-protected)
- `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json` (pre-annotated events; use as a hint but re-read source)

## Patterns to Detect

### Pattern 1 — Unsynchronized Shared Mutable State

A field is written in one thread context without a lock **and** read in another thread context without a lock (or with inconsistent locks).

**How to detect:**
1. From the structural index, identify all instance fields of classes that are accessed from multiple thread entry-points (identified in the structure scan).
2. For each field, collect all write sites (assignment `=`, `+=`, `++`, `--`, collection mutations).
3. For each write site, check whether it is inside a `synchronized(X)` block or a `lock()` / `unlock()` pair.
4. For each read site, do the same check.
5. If the same field is written/read without consistent lock coverage, it is a race.

**Common patterns to watch:**
- Singleton fields accessed from multiple threads without synchronization.
- Collection fields (`List`, `Map`, `Set`) mutated from a background thread and iterated on the main thread without synchronization.
- Counter fields (`int count`) incremented from multiple threads without `AtomicInteger` or synchronization.
- Flag fields (`boolean done`) written in one thread and read in another without `volatile` or synchronization.

### Pattern 2 — `volatile` Misuse

Detect compound operations on `volatile` fields that are not atomic:

- Read-check-then-write: `if (mVolatileFlag) { mVolatileFlag = false; }` (two separate memory accesses — another thread can interleave).
- Increment: `mVolatileCounter++` (read-modify-write is not atomic).
- Object reference swap without CAS: `mVolatileRef = newValue; if (mVolatileRef == expected) { ... }`.

### Pattern 3 — Non-Atomic Check-Then-Act

- Condition on a shared field checked outside a lock, followed by acting on it outside a lock:
  `if (state != RUNNING) { start(); }` — if `start()` is not synchronized with the check.
- `AtomicReference.get()` followed by a non-CAS modification: `ref.set(ref.get().withField(x))`.
- `map.containsKey(k)` followed by `map.put(k, v)` on a non-concurrent map without a lock covering both operations.

### Pattern 4 — Inconsistent Synchronization

- A field is accessed under lock in most methods, but one method accesses it without a lock (even if that method appears "read-only").
- Cross-thread access via `executor.submit()` to a field also accessed directly on the calling thread without synchronization.
- A field annotated `@GuardedBy("lockName")` accessed without holding the named lock at any call site.

### Pattern 5 — Iterator / Collection Modification Races

- A collection is iterated in one thread while another thread calls `add()`, `remove()`, or `clear()` on it without a lock covering both.
- A `for-each` loop over a `List` or `Set` on one thread while another thread mutates the same collection → `ConcurrentModificationException` at runtime.

### Pattern 6 — Unsafe Object Publication / Constructor Escape

A reference to `this` escapes from a constructor before the constructor finishes. Another thread reading the published reference may see an incompletely initialized object.

```java
public class Unsafe {
    private int value;
    public static Unsafe instance;

    public Unsafe() {
        instance = this;   // 'this' escapes — value may still be 0 for other threads
        value = 42;
    }
}
```

**Detection:** Any assignment of `this` to a static or shared field inside a constructor body, or passing `this` to an external method/thread before the constructor returns. Severity: CRITICAL.

### Pattern 7 — Double-Checked Locking Without `volatile`

```java
private static MyClass instance;   // NOT volatile — BROKEN

public static MyClass getInstance() {
    if (instance == null) {           // unsynchronized read
        synchronized (MyClass.class) {
            if (instance == null) {
                instance = new MyClass();   // partially constructed object may be visible
            }
        }
    }
    return instance;
}
```

Without `volatile`, the JVM may publish the reference to a partially-constructed object. Fix: declare the field `volatile`.

**Detection:** A non-`volatile`, non-`final` static field that is read outside a `synchronized` block and written inside one (classic DCL shape). Severity: CRITICAL.

### Pattern 8 — `ConcurrentHashMap` Compound Operation Races

`ConcurrentHashMap` guarantees atomicity of individual operations, but NOT compound operations:

```java
// RACE — two separate atomic operations, not one:
if (!map.containsKey(key)) {
    map.put(key, expensiveCompute(key));  // Another thread may put the same key between check and put
}
```

**Correct alternative:** `map.computeIfAbsent(key, k -> expensiveCompute(k))` — single atomic operation.

**Detection:** `containsKey` + `put` / `get` + `put` / `remove` + `put` on the same `ConcurrentHashMap` variable without a surrounding lock. Also: `putIfAbsent()` followed by a separate `get()` — use `computeIfAbsent()` instead. Severity: HIGH.

### Pattern 9 — Shared Mutable State in Singleton Components

Web framework components (`@RestController`, `@Service`, `@Component` in Spring; `Servlet` subclasses; `@Singleton` EJBs) are instantiated once and shared across all request threads. Instance fields on these classes are accessed concurrently.

```java
@RestController
public class OrderController {
    private List<Order> recentOrders = new ArrayList<>();  // SHARED — all request threads race here

    @PostMapping("/order")
    public void placeOrder(@RequestBody Order o) {
        recentOrders.add(o);   // ConcurrentModificationException or data loss
    }
}
```

**Detection:** Any mutable instance field (non-`final`, non-`volatile`, non-thread-safe collection) declared in a class annotated with `@RestController`, `@Controller`, `@Service`, `@Component`, `@Singleton`, or subclassing `HttpServlet` / `GenericServlet`. Severity: CRITICAL.

## Output Format (`concurrency_analysis/findings/<PARTITION_ID>-races.json`)

```json
{
  "partition_id": "p01-order-service",
  "detector": "jca-race-detector",
  "findings": [
    {
      "id": "RACE-p01-001",
      "type": "unsynchronized_shared_state",
      "severity": "HIGH",
      "title": "Unsynchronized access to orderCount in OrderService",
      "description": "Field orderCount is incremented inside synchronized(mLock) in placeOrder() but read without any lock in getOrderCount() which is called from a background reporting thread.",
      "file": "src/main/java/com/example/service/OrderService.java",
      "line": 95,
      "related_locations": [
        {
          "file": "src/main/java/com/example/service/OrderService.java",
          "line": 130,
          "note": "Synchronized write site in placeOrder()"
        }
      ],
      "recommendation": "Guard all accesses to orderCount with synchronized(mLock) or replace with AtomicInteger."
    }
  ]
}
```

### Severity

- `CRITICAL`: Race on security-sensitive state (authentication, authorization) or state that controls core system stability.
- `HIGH`: Race on frequently-mutated state in a primary service class; unsynchronized access on a hot path.
- `MEDIUM`: Race in helper or non-core classes; low-frequency code paths.
- `LOW`: `volatile` compound-operation misuse with demonstrably low contention.

## Mandatory Rules

- Read every file in the partition completely. No early stopping.
- Never modify any source file.
- Every finding must include exact `file` and `line` for both the finding site and all related locations.
- Write only to `concurrency_analysis/findings/<PARTITION_ID>-races.json`.
