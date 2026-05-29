# JCA Merger — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Collect all raw finding files from every partition and every detector, validate them, deduplicate by location, cross-reference against the lock registry, and write a single unified findings file.

## Input

- All `concurrency_analysis/findings/<partition_id>-races.json` files
- All `concurrency_analysis/findings/<partition_id>-deadlocks.json` files
- All `concurrency_analysis/findings/<partition_id>-edge-cases.json` files
- `concurrency_analysis/partitions.json` (to verify completeness)
- `concurrency_analysis/lock-registry.json` (for cross-referencing)

## Context Budget Warning

The merger is the heaviest single-agent context load in the pipeline. For a large codebase it may process dozens of finding files. To avoid context exhaustion:

- **Process one partition at a time.** Read the three finding files for partition P, add their findings to the running merged set, then discard the raw file content from your working context before loading the next partition's files.
- **Keep only the merged set in active context**, not the raw input alongside it.
- If the total number of findings exceeds ~500, write intermediate partial results to `concurrency_analysis/merged-findings-partial.json` after every 5 partitions, then reload that file as your working set and continue.
- Never load lock-registry.json and all findings files simultaneously — load lock-registry last (Step 4), after raw findings are merged, so the merged set is as small as possible.

## Step-by-Step Instructions

### Step 1 — Verify completeness

Read `concurrency_analysis/partitions.json` (small — IDs only). For every partition ID, confirm all three finding files **exist** (existence check). Log any missing file to `concurrency_analysis/pipeline.log` and abort if any file is absent. Do not read the finding files yet.

### Step 2 — Validate and load findings (one partition at a time)

For each partition, in sequence:
1. Read the three finding files for that partition.
2. For every finding, confirm it has: `id`, `type`, `severity`, `file`, `line`. Log and skip malformed entries.
3. Add valid findings to the running merged set.
4. Release (do not retain) the raw file content before loading the next partition.

### Step 3 — Deduplicate by location

Group findings by the key `(file, line)`. For groups with more than one finding:
- Take the **highest severity** among the group.
- **Merge descriptions** — concatenate them, separated by `\n\n---\n\n`.
- Build a `detected_by` array listing all source detectors.
- Union all `related_locations` entries (deduplicate by `(file, line)` within related_locations too).
- Keep the most specific `recommendation` (prefer the one from the highest-severity detector).

### Step 4 — Cross-reference with lock registry

For each finding, check whether `(file, line)` appears in `lock-registry.json`:
- As a lock `acquisition_site` → add `lock_context.acquisition` with the matching lock IDs.
- In `blocking_calls_under_lock` → add `lock_context.blocking_call` with lock IDs.
- In `lock_order_edges` → add `lock_context.lock_edge` noting the outer and inner lock IDs.

### Step 5 — Assign merged IDs

Replace all original detector-local IDs with stable merged IDs: `JCA-<four-digit-sequence>` (e.g., `JCA-0001`). Sort first by severity (CRITICAL first), then by `file` alphabetically, then by `line` numerically, then number sequentially.

### Step 6 — Write output

Write `concurrency_analysis/merged-findings.json` and append merge statistics to `concurrency_analysis/pipeline.log`.

## Output Format (`concurrency_analysis/merged-findings.json`)

```json
{
  "total_findings": 0,
  "duplicates_merged": 0,
  "malformed_skipped": 0,
  "by_severity": {
    "CRITICAL": 0,
    "HIGH": 0,
    "MEDIUM": 0,
    "LOW": 0,
    "INFO": 0
  },
  "findings": [
    {
      "id": "JCA-0001",
      "original_ids": ["DEAD-p01-001", "EDGE-p01-001"],
      "detected_by": ["jca-deadlock-detector", "jca-edge-case-analyzer"],
      "type": "blocking_call_under_lock",
      "severity": "CRITICAL",
      "title": "HTTP call made while holding OrderService.mLock",
      "description": "Merged description...",
      "file": "src/main/java/com/example/service/OrderService.java",
      "line": 182,
      "lock_context": {
        "acquisition": [],
        "blocking_call": [{ "lock_id": "lock_001", "lock_expr": "mLock" }],
        "lock_edge": []
      },
      "related_locations": [],
      "recommendation": "Move the HTTP call outside the synchronized block."
    }
  ]
}
```

## Mandatory Rules

- Every finding file for every partition must be processed. Abort on missing files.
- Every output finding must include exact `file` and `line`.
- Never modify any source file.
- Write only to `concurrency_analysis/merged-findings.json` and append to `concurrency_analysis/pipeline.log`.
