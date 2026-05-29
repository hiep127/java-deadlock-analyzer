# JCA Cross-File Edge Resolver — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Close the cross-file lock-ordering gap left by the diagram generator. The diagram generator can only record a `(outer → inner)` lock edge when both acquisitions are visible within the same file. This agent reads the `locks_held` annotations emitted by all fullscan workers and joins them against the lock-registry to discover indirect, multi-file lock chains. It appends the inferred edges to `lock-registry.json` and reports any newly confirmed cycles.

## The Gap Being Closed

```
// FileA.java
synchronized (lockA) {
    serviceB.process();   // calls FileB — lockB acquired inside, not visible here
}

// FileB.java
synchronized (lockB) {
    serviceA.callback();  // calls FileA — lockA acquired inside, not visible here
}
```

Neither file shows both locks nesting, so the diagram generator emits no edge for either chain. This agent infers the edges by joining:
- `cross_method_lock_entry` annotations (which file called which callee, while holding which locks)
- Lock acquisition sites in the registry (which locks does the callee acquire)

## Input

- `concurrency_analysis/lock-registry.json`
- `concurrency_analysis/partitions.json`
- `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json` for every partition (read one at a time)

## Step-by-Step Instructions

### Step 1 — Build indexes from the lock registry

Read `concurrency_analysis/lock-registry.json` once. Build the following in-memory indexes:

**`lock_map`** — resolve a lock name to its lock ID:
```
key:   "<declaring_class>.<expression>"  (e.g., "OrderService.mLock")
value: lock_id  (e.g., "lock_001")
```
Also index by expression alone for cases where the class is ambiguous:
```
key:   "<expression>"  (e.g., "mLock")
value: [lock_id, ...]  (list, may be ambiguous)
```

**`method_acquires`** — what locks does a method acquire:
```
key:   "<class>.<method>"  (e.g., "InventoryService.reserveItem")
value: [lock_id, ...]
```
Populate from `locks[*].acquisition_sites`: for each lock, for each acquisition site, add the lock's ID to `method_acquires[class.method]`.

**`existing_edges`** — set of already-known edges (to avoid duplicates):
```
entry: "<outer_lock_id>-><inner_lock_id>"
```

### Step 2 — Process fullscan files one at a time

For each `PARTITION_ID` in `concurrency_analysis/partitions.json`:

1. Read `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`.
2. Extract every annotation where:
   - `type == "cross_method_lock_entry"`
   - `locks_held` is non-empty
   - `callee_class` and `callee_method` are both present and non-empty
3. For each such annotation:
   a. **Resolve held locks to IDs:** For each name in `locks_held`, look up `lock_map` using the annotation's `file` to narrow the declaring class. If ambiguous, try all candidates.
   b. **Look up callee lock acquisitions:** Look up `method_acquires["<callee_class>.<callee_method>"]`. If the callee is not in the index, skip (callee may be in another partition or an external library).
   c. **Emit new edges:** For each pair `(held_lock_id, acquired_lock_id)`:
      - If `"<held_lock_id>-><acquired_lock_id>"` is already in `existing_edges`, skip.
      - Otherwise, record a new edge:
        ```json
        {
          "outer_lock_id": "<held_lock_id>",
          "inner_lock_id": "<acquired_lock_id>",
          "file": "<annotation.file>",
          "line": <annotation.line>,
          "method": "<annotation.method>",
          "callee": "<callee_class>.<callee_method>",
          "source": "cross_file_inferred"
        }
        ```
      - Add to `existing_edges` to prevent re-emitting within this run.
4. Release the fullscan file content from working context before loading the next partition.

### Step 3 — Cycle detection

After processing all partitions, combine the original `lock_order_edges` from Step 1 with all newly inferred edges. Run a depth-first search on the combined directed graph to find cycles.

For each cycle found:
- Record the cycle as an ordered list of lock IDs (e.g., `["lock_001", "lock_003", "lock_001"]`).
- Annotate each edge in the cycle with whether it was `"original"` (from diagram generator) or `"cross_file_inferred"` (from this agent).

### Step 4 — Write output

**Update `concurrency_analysis/lock-registry.json`:** Append the new edges to the existing `lock_order_edges` array. Do not remove or modify any existing entries.

**Write `concurrency_analysis/cross-file-edges.json`:**

```json
{
  "agent": "jca-cross-file-edge-resolver",
  "new_edges_count": 4,
  "cycles_found": [
    {
      "cycle": ["lock_001", "lock_003", "lock_001"],
      "edges": [
        {
          "outer_lock_id": "lock_001",
          "inner_lock_id": "lock_003",
          "file": "src/main/java/com/example/service/OrderService.java",
          "line": 198,
          "method": "placeOrder",
          "callee": "InventoryService.reserveItem",
          "source": "cross_file_inferred"
        },
        {
          "outer_lock_id": "lock_003",
          "inner_lock_id": "lock_001",
          "file": "src/main/java/com/example/service/InventoryService.java",
          "line": 87,
          "method": "reserveItem",
          "callee": "OrderService.callback",
          "source": "original"
        }
      ],
      "severity": "CRITICAL",
      "note": "Lock-order inversion confirmed via cross-file call chain. Not detectable by single-file analysis."
    }
  ],
  "new_edges": [
    {
      "outer_lock_id": "lock_001",
      "inner_lock_id": "lock_003",
      "file": "src/main/java/com/example/service/OrderService.java",
      "line": 198,
      "method": "placeOrder",
      "callee": "InventoryService.reserveItem",
      "source": "cross_file_inferred"
    }
  ]
}
```

**Append to `concurrency_analysis/pipeline.log`:**
```
[<timestamp>] Phase 4.5: Cross-file edge resolver complete — <N> new edges, <C> cycles confirmed
```

## Severity of Cycles Found

- `CRITICAL`: Any cycle that includes at least one lock held on a hot path (a lock with ≥ 3 acquisition sites, or a lock used in any `synchronized` method rather than a `synchronized` block).
- `HIGH`: Any other confirmed cycle.

## Mandatory Rules

- Never modify any source file.
- Read each fullscan file completely before extracting annotations, then release it before loading the next.
- Only write to `concurrency_analysis/lock-registry.json` (append only — never remove existing entries) and `concurrency_analysis/cross-file-edges.json`.
- If `callee_class` or `callee_method` is missing from a `cross_method_lock_entry` annotation, skip that annotation and log a warning to `pipeline.log` — do not abort.
- If a lock name in `locks_held` cannot be resolved to a lock ID, log a warning and skip — do not abort.
