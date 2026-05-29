# JCA Partitioner — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Divide the target Java source path into logical, independently-analyzable partitions. Your output (`concurrency_analysis/partitions.json`) is the foundation that every subsequent agent depends on.

## Step-by-Step Instructions

### Step 1 — Walk the source tree

Recursively enumerate every `.java` file under `SOURCE_PATH`. For each file record:
- Relative path from repository root
- Package declaration (first `package ...;` line)
- File size in bytes

### Step 2 — Classify by package or module boundary

Group files by their top-level package segment or module name. Use the following priority order:

| Group ID | Matching Rule |
|---|---|
| `<top-package>-service` | Package ends in `.service` or `.services`, or class name ends in `Service` |
| `<top-package>-manager` | Class name ends in `Manager` or `Registry` |
| `<top-package>-handler` | Class name ends in `Handler`, `Processor`, or `Worker` |
| `<top-package>-repository` | Package ends in `.repository` or `.dao`, or class name ends in `Repository` or `Dao` |
| `<top-package>-model` | Package ends in `.model`, `.entity`, or `.domain` |
| `<top-package>-util` | Package ends in `.util` or `.helper` |
| `misc` | Everything else |

If the codebase uses Java modules (`module-info.java`), use the module name as the group ID instead.

Derive `<top-package>` from the first two segments of the package declaration (e.g., `com.example` from `com.example.service.OrderService`).

### Step 3 — Size-limit splitting

After grouping, split any group whose **total file size** exceeds `maxPartitionSizeKB` (from `.github/jca-config.json`, default 512 KB) by further subdividing on sub-package boundaries.

### Step 4 — Assign partition IDs

Assign each partition a stable ID: `p<two-digit-index>-<group-id>` (e.g., `p01-order-service`, `p02-payment-manager`).

### Step 5 — Write output

Write `concurrency_analysis/partitions.json`.

## Output: `concurrency_analysis/partitions.json`

```json
{
  "source_path": "<SOURCE_PATH>",
  "total_files": 0,
  "total_size_kb": 0,
  "partition_count": 0,
  "partitions": [
    {
      "id": "p01-order-service",
      "group": "order-service",
      "files": [
        "src/main/java/com/example/service/OrderService.java"
      ],
      "file_count": 0,
      "size_kb": 0
    }
  ]
}
```

## Mandatory Rules

- Every `.java` file under `SOURCE_PATH` must appear in exactly one partition.
- No file may be omitted.
- Never modify any source file.
- Write only to `concurrency_analysis/partitions.json`.
