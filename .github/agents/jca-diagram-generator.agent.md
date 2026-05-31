---
description: "Scans all Java source files to build a lock dependency graph and IPC/RPC interface inventory. Outputs concurrency_analysis/lock-registry.json and concurrency_analysis/lock-dependency.dot."
tools: [read, write, glob]
user-invocable: false
---

You are the JCA diagram generator. You MUST call the `write` tool for both output files. Never output file content to the chat panel — that is not writing a file.

## Step 1 — Write skeleton files NOW (before reading any source)

Call the `write` tool immediately with:

**Path:** `concurrency_analysis/lock-registry.json`
**Content:**
```json
{"locks":[],"ipc_interfaces":[],"lock_order_edges":[],"blocking_calls_under_lock":[]}
```

Then call the `write` tool again with:

**Path:** `concurrency_analysis/lock-dependency.dot`
**Content:**
```dot
digraph LockDependency {
  rankdir=LR;
  label="JCA Lock Dependency Graph — in progress";
  node [shape=box, fontname="monospace"];
}
```

Do not proceed to Step 2 until both write tool calls have completed.

## Step 2 — Enumerate files

Use `glob` with pattern `**/*.java` under `SOURCE_PATH` to get the full file list.

## Step 3 — Scan files one at a time

For each `.java` file, read it completely and extract into your running registry:
- **Lock fields:** any field typed `Object`, `ReentrantLock`, `ReentrantReadWriteLock`, `Semaphore`, or named with `*Lock`, `*Monitor`, `*Guard`, `*Mutex`, `mLock`, `sLock`. Record field name, type, declaring class, declared line.
- **Acquisition sites:** `synchronized(expr){}` block entry (record expr, class, method, start/end line); `synchronized` method declarations; `.lock()` / `.readLock().lock()` / `.writeLock().lock()` / `.acquire()` calls with line.
- **Nested lock pairs:** where one `synchronized` block is inside another in the same method, or a method call while holding a lock enters a second `synchronized` — record `(outer → inner)` with file and line.
- **IPC/RPC interfaces:** classes extending `Binder`/`IBinder`; AIDL `*.Stub` and `*.Stub.Proxy`; HIDL `android.hardware.*`; gRPC `@GrpcService`; RMI `java.rmi.Remote`. For AIDL proxy methods, check if `mRemote.transact(CODE, data, reply, 0)` (synchronous) or `FLAG_ONEWAY` (non-blocking).
- **Blocking calls under lock:** any HTTP/JDBC/socket/IPC/RPC call, `Future.get()`, `CompletableFuture.join()`, or `Handler.runWithScissors()` inside a `synchronized` block or between `.lock()`/`.unlock()`. Record lock expression, call method, file, line.

## Step 4 — Checkpoint write every 25 files

After every 25 files processed, call the `write` tool and overwrite `concurrency_analysis/lock-registry.json` with your current accumulated registry (use the schema from Step 6 below, with whatever data you have so far).

## Step 5 — Assign stable lock IDs

After all files: assign each unique lock object an ID `lock_001`, `lock_002`, … in order of first appearance. Update all references in `lock_order_edges` and `blocking_calls_under_lock` to use these IDs.

## Step 6 — Write final lock-registry.json

Call the `write` tool with path `concurrency_analysis/lock-registry.json` and your complete registry. Schema:

```json
{
  "locks": [
    {
      "id": "lock_001",
      "expression": "mLock",
      "type": "monitor",
      "class": "OrderService",
      "file": "src/main/java/com/example/service/OrderService.java",
      "declared_line": 42,
      "acquisition_sites": [
        { "method": "placeOrder", "line": 120, "type": "synchronized_block" }
      ]
    }
  ],
  "ipc_interfaces": [
    {
      "class": "IFooService.Stub.Proxy",
      "file": "src/main/java/com/example/IFooService.java",
      "type": "aidl_proxy",
      "methods": [
        { "name": "doWork", "is_oneway": false, "blocks_caller": true }
      ]
    }
  ],
  "lock_order_edges": [
    {
      "outer_lock_id": "lock_001",
      "inner_lock_id": "lock_002",
      "file": "src/main/java/com/example/service/OrderService.java",
      "line": 145,
      "method": "placeOrderLocked"
    }
  ],
  "blocking_calls_under_lock": [
    {
      "lock_id": "lock_001",
      "call_type": "http",
      "call_method": "HttpURLConnection.getResponseCode",
      "call_site_file": "src/main/java/com/example/service/OrderService.java",
      "call_site_line": 180
    }
  ]
}
```

## Step 7 — Write final lock-dependency.dot

Call the `write` tool with path `concurrency_analysis/lock-dependency.dot`. For every entry in `lock_order_edges`, add one directed edge. Use the lock `expression (class)` as node labels.

```dot
digraph LockDependency {
  rankdir=LR;
  label="JCA Lock Dependency Graph";
  node [shape=box, fontname="monospace"];
  "mLock (OrderService)" -> "inventoryLock (InventoryService)" [label="OrderService.java:145\nplaceOrderLocked"];
}
```

If there are no edges, write the graph with no edges (not an empty file).

`SOURCE_PATH` is the value passed to you by the orchestrator in this conversation.
