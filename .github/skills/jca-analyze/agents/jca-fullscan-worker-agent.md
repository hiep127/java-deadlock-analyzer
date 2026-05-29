# JCA Fullscan Worker — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Execute a deep, line-by-line context scan on every file in your assigned partition. Annotate every synchronization primitive, lock acquisition/release event, blocking call, thread dispatch, and `volatile` access with full contextual information — including which locks are held at the moment of each event.

## Input

- `PARTITION_ID`
- `concurrency_analysis/partitions.json`
- `concurrency_analysis/scans/<PARTITION_ID>-structure.json` (use as a guide; do not trust it fully — re-read source files directly)

## Step-by-Step Instructions

### Step 1 — Load context

Read the structural index for your partition. Use it to pre-populate the list of known lock fields and their expressing variables.

### Step 2 — Per-file deep scan

For every file, read from line 1 to the last line without stopping. Maintain a **lock stack** per method as you read:

- **Push** when entering `synchronized(expr) { }` or calling `.lock()` / `.readLock().lock()` / `.writeLock().lock()` / `semaphore.acquire()`.
- **Pop** when exiting a `synchronized` block (matching `}`) or calling `.unlock()` / `semaphore.release()`.
- At every event below, record the current lock stack contents as `locks_held`.

**Events to annotate:**

| Event Type | What to Look For |
|---|---|
| `lock_acquisition` | `synchronized(expr)` block entry or `.lock()` / `.acquire()` call |
| `lock_release` | `synchronized` block exit or `.unlock()` / `.release()` call |
| `blocking_call_under_lock` | I/O, JDBC, HTTP, IPC/RPC proxy call, `Thread.sleep()` while `locks_held` is non-empty |
| `binder_sync_call_under_lock` | Call to an AIDL `.Stub.Proxy` method while `locks_held` is non-empty AND the call is synchronous (transact flags = 0, not `FLAG_ONEWAY`) |
| `hidl_call_under_lock` | Call to an `android.hardware.*` or `IHwInterface`-typed method while `locks_held` is non-empty |
| `future_get_under_lock` | `Future.get()` or `CompletableFuture.join()` while `locks_held` is non-empty |
| `executor_submit_under_lock` | `executor.submit()` or `executor.execute()` while `locks_held` is non-empty |
| `run_with_scissors` | `Handler.runWithScissors()` anywhere (always annotate; flag as HIGH if `locks_held` non-empty) |
| `wait_notify` | `object.wait()`, `object.notify()`, `object.notifyAll()` |
| `volatile_access` | Read or write to a `volatile` field |
| `nested_synchronized` | `synchronized` block inside another `synchronized` block (same method) |
| `cross_method_lock_entry` | Method call made while locks are held that itself contains a `synchronized` block or lock acquisition — **must record `callee_class` and `callee_method`** |
| `jni_call_under_lock` | `native` method call while `locks_held` is non-empty |
| `oneway_ordering_assumption` | `oneway` AIDL/HIDL call followed immediately by a synchronous call on the same interface with no synchronization barrier |

**Binder/AIDL/HIDL call classification — how to determine if a call is synchronous:**
1. Check the variable type: if it's an AIDL interface type (class ending in `.Stub.Proxy`, or a variable typed to an `I`-prefixed interface that has a sibling `.Stub` class), it is an AIDL proxy.
2. Check the generated `Stub.Proxy.methodName()` implementation: if it calls `mRemote.transact(CODE, data, reply, 0)` with flags=`0`, the call is **synchronous and blocks the caller**.
3. If it calls `mRemote.transact(CODE, data, null, IBinder.FLAG_ONEWAY)`, the call is **asynchronous**.
4. For HIDL: any method call on a type from `android.hardware.*` package should be treated as synchronous unless the HIDL `.hal` file declares it `oneway`.
5. When in doubt, treat an AIDL/HIDL call as synchronous (conservative analysis).

### Step 3 — Write output

Write `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`.

## Output Format

```json
{
  "partition_id": "p01-order-service",
  "files_scanned": 1,
  "annotations": [
    {
      "file": "src/main/java/com/example/service/OrderService.java",
      "line": 182,
      "end_line": 182,
      "type": "blocking_call_under_lock",
      "detail": "HttpURLConnection.getResponseCode() called while holding mLock",
      "locks_held": ["mLock"],
      "method": "placeOrder",
      "severity_hint": "HIGH"
    },
    {
      "file": "src/main/java/com/example/service/OrderService.java",
      "line": 215,
      "end_line": 240,
      "type": "nested_synchronized",
      "detail": "synchronized(inventoryLock) acquired inside synchronized(mLock)",
      "locks_held": ["mLock"],
      "method": "reserveInventory",
      "severity_hint": "MEDIUM"
    },
    {
      "file": "src/main/java/com/example/service/OrderService.java",
      "line": 198,
      "end_line": 198,
      "type": "cross_method_lock_entry",
      "detail": "call to InventoryService.reserveItem() made while holding mLock",
      "locks_held": ["mLock"],
      "method": "placeOrder",
      "callee_class": "InventoryService",
      "callee_method": "reserveItem",
      "severity_hint": "MEDIUM"
    }
  ]
}
```

### Severity Hints

- `CRITICAL`: Blocking IPC/RPC call or JNI call while holding a primary service lock on a hot path.
- `HIGH`: Any blocking call or `Future.get()` while holding any lock; `runWithScissors()` under lock.
- `MEDIUM`: Nested `synchronized`; `wait()`/`notify()` on a non-canonical monitor; cross-method lock entry.
- `LOW`: `volatile` compound read-modify-write; `executor.submit()` under lock with no visible circular dependency.

## Mandatory Rules

- Read every file in the partition completely from line 1 to the last line. **No early stopping.**
- Never modify any source file.
- `files_scanned` must equal the partition file count.
- Every annotation must include the exact `file` and `line`.
- Write only to `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`.
