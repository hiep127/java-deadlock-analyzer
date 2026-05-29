# JCA Source Scanner — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Perform an initial structural pass over every Java file in your assigned partition. Build a structural index (class hierarchy, lock fields, synchronized blocks, thread entry-points) that seeds the deeper jca-fullscan-worker scan.

## Input

- `PARTITION_ID`: The partition ID you are assigned (e.g., `p01-order-service`).
- `concurrency_analysis/partitions.json`: Contains the file list for your partition.

## Step-by-Step Instructions

### Step 1 — Load your file list

Read `concurrency_analysis/partitions.json`. Find the entry matching your `PARTITION_ID` and collect its `files` array.

### Step 2 — Scan each file

For every file in the list, read it **completely from line 1 to the last line**. Extract:

**Class declarations:**
- Class name, superclass, all implemented interfaces.
- Flag: is this class a subclass of `Thread`, `Runnable`, an RPC stub/service, or a `TimerTask`?
- Flag: does this class implement any listener/callback/observer interface?

**Lock fields:**
- Every field whose type is `Object`, `ReentrantLock`, `ReentrantReadWriteLock`, `Semaphore`, or any type used as a monitor (infer from naming: `*Lock`, `*Monitor`, `*Guard`, `*Mutex`, `*Latch`).
- Record field name, type, modifier (`private`/`static`/`final`), and declared line.

**Synchronized methods and blocks:**
- Synchronized method declarations: record class, method signature, line number.
- `synchronized(expr)` blocks: record the lock expression, start line, end line, and enclosing method.

**Thread entry-points:**
- `run()` overrides (Runnable, Thread subclasses).
- `call()` overrides (Callable).
- `handleMessage()` overrides (Android Handler).
- RPC/stub `onTransact()` overrides or generated service method overrides (called on a framework thread pool).
- `doInBackground()` / `doCall()` overrides in async task classes.
- Lambda/anonymous Runnable passed to `Executor.execute()`, `Handler.post()`, `CompletableFuture.runAsync()`, etc.

**Cross-thread call sites:**
- `executor.execute(...)`, `executor.submit(...)`
- `handler.post(...)`, `handler.sendMessage(...)`
- `CompletableFuture.supplyAsync(...)`, `CompletableFuture.runAsync(...)`
- `new Thread(...).start()`
- `scheduledExecutor.schedule(...)`, `timer.schedule(...)`

### Step 3 — Write output

Write `concurrency_analysis/scans/<PARTITION_ID>-structure.json`.

## Output Format

```json
{
  "partition_id": "p01-order-service",
  "files_scanned": 1,
  "classes": [
    {
      "name": "OrderService",
      "file": "src/main/java/com/example/service/OrderService.java",
      "superclass": "AbstractService",
      "interfaces": ["OrderOperations", "ShutdownHook"],
      "is_runnable_or_thread": false,
      "is_rpc_service": false,
      "is_listener": false,
      "lock_fields": [
        {
          "name": "mLock",
          "type": "Object",
          "modifiers": ["private", "final"],
          "declared_line": 42
        }
      ],
      "synchronized_methods": [
        { "signature": "void placeOrder(Order)", "line": 120 }
      ],
      "synchronized_blocks": [
        {
          "lock_expr": "mLock",
          "start_line": 210,
          "end_line": 240,
          "method": "cancelOrder"
        }
      ],
      "thread_entry_points": [
        { "type": "executor_lambda", "line": 310, "thread": "orderExecutor" }
      ],
      "cross_thread_calls": [
        {
          "type": "executor.submit",
          "target_executor": "orderExecutor",
          "line": 305,
          "method": "processOrderAsync"
        }
      ]
    }
  ]
}
```

## Mandatory Rules

- Read every file in the partition completely. No partial scans. `files_scanned` must equal the partition file count.
- Never modify any source file.
- Every entry must include the exact `file` path and `line` number.
- Write only to `concurrency_analysis/scans/<PARTITION_ID>-structure.json`.
