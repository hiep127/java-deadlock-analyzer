# Workflow: analyze

This document defines the complete step-by-step orchestration process for the `/jca-analyze` command. The `jca-orchestrator` agent follows these instructions exactly.

---

## Trigger

```
/jca-analyze <SOURCE_PATH>
```

`SOURCE_PATH` is the repository-relative path to the AOSP Java directory to analyze (e.g., `frameworks/base/services/core/java/com/android/server/audio/`).

---

## Pre-flight Checks

Before starting the pipeline, verify:

1. `SOURCE_PATH` is a non-empty string.
2. The directory at `SOURCE_PATH` exists and contains at least one `.java` file (recursively).
3. `.github/jca-config.json` is present and valid JSON.

If any check fails, print an error message and **abort**. Do not create any output files.

---

## Phase 0 — Initialize

1. Create the following directories (create parent directories as needed):
   - `concurrency_analysis/`
   - `concurrency_analysis/scans/`
   - `concurrency_analysis/findings/`

2. Write `concurrency_analysis/pipeline.log` with:
   ```
   [<ISO 8601 timestamp>] JCA pipeline started
   [<timestamp>] SOURCE_PATH: <SOURCE_PATH>
   [<timestamp>] Phase 0: Initialization complete
   ```

---

## Phase 1 — Partition  *(and Phase 2 in parallel)*

**Invoke** `jca-partitioner` agent with `SOURCE_PATH`.

**In parallel**, invoke `jca-diagram-generator` agent with `SOURCE_PATH`.

**Wait for both** to complete. Verify:
- `concurrency_analysis/partitions.json` exists and contains at least one partition.
- `concurrency_analysis/lock-registry.json` exists.
- `concurrency_analysis/lock-dependency.dot` exists.

If either file is missing, log the failure and **abort**.

Log:
```
[<timestamp>] Phase 1: Partitioner complete — <N> partitions
[<timestamp>] Phase 2: Diagram generator complete — <M> locks, <K> Binder interfaces
```

---

## Phase 3 — Source Scan (parallel across partitions)

For each partition `P` in `concurrency_analysis/partitions.json`:

**Invoke** `jca-source-scanner` agent with `PARTITION_ID = P.id`.

Run all partition scanners **in parallel** (up to `pipeline.parallelWorkers` concurrent agents, from `jca-config.json`).

**Wait for all** to complete. Verify that `concurrency_analysis/scans/<P.id>-structure.json` exists for every `P`.

If any structure file is missing, log the failure and **abort**.

Log:
```
[<timestamp>] Phase 3: Source scan complete — <N> partitions scanned
```

---

## Phase 4 — Full Scan (parallel across partitions)

For each partition `P`:

**Invoke** `jca-fullscan-worker` agent with `PARTITION_ID = P.id`.

Run all fullscan workers **in parallel** (up to `pipeline.parallelWorkers` concurrent agents).

**Wait for all** to complete. Verify that `concurrency_analysis/scans/<P.id>-fullscan.json` exists for every `P`.

If any fullscan file is missing, log the failure and **abort**.

Log:
```
[<timestamp>] Phase 4: Full scan complete — <N> partitions, <M> total annotations
```

---

## Phase 5 — Detection (parallel: three detectors per partition, partitions also in parallel)

For each partition `P`, **concurrently invoke all three detector agents**:

1. `jca-race-detector` with `PARTITION_ID = P.id`
2. `jca-deadlock-detector` with `PARTITION_ID = P.id`
3. `jca-edge-case-analyzer` with `PARTITION_ID = P.id`

Multiple partitions' detector sets may also run in parallel (up to `pipeline.parallelWorkers` total active agents).

**Wait for all** to complete. Verify that all three finding files exist for every partition:
- `concurrency_analysis/findings/<P.id>-races.json`
- `concurrency_analysis/findings/<P.id>-deadlocks.json`
- `concurrency_analysis/findings/<P.id>-edge-cases.json`

If any finding file is missing, log the failure and **abort**.

Log:
```
[<timestamp>] Phase 5: Detection complete — <N> raw findings across <M> partitions
```

---

## Phase 6 — Merge (reduce)

**Invoke** `jca-merger` agent.

**Wait for completion.** Verify that `concurrency_analysis/merged-findings.json` exists.

If missing, log the failure and **abort**.

Log:
```
[<timestamp>] Phase 6: Merge complete — <N> unique findings after deduplication
```

---

## Phase 7 — Consolidate

**Invoke** `jca-consolidator` agent.

**Wait for completion.** Verify that both `concurrency_analysis/report.md` and `concurrency_analysis/report.json` exist.

If either is missing, log the failure and **abort**.

Log:
```
[<timestamp>] Phase 7: Consolidation complete — <N> final findings (<M> filtered)
[<timestamp>] JCA pipeline finished successfully
```

---

## Completion

Print the following message to the Copilot chat panel:

```
JCA analysis complete.

Report:     concurrency_analysis/report.md
JSON:       concurrency_analysis/report.json
Lock graph: concurrency_analysis/lock-dependency.dot

Summary:
  CRITICAL: N
  HIGH:     N
  MEDIUM:   N
  LOW:      N
  INFO:     N
  Total:    N findings across <files_scanned> files
```

---

## Abort Procedure

If any phase fails:

1. Log the failure to `concurrency_analysis/pipeline.log`:
   ```
   [<timestamp>] ABORT: Phase <N> failed — <reason>
   ```
2. Do **not** invoke any subsequent phase.
3. Print to the Copilot chat panel:
   ```
   JCA pipeline aborted at Phase <N>: <reason>
   Check concurrency_analysis/pipeline.log for details.
   ```
4. Do not delete any partial output files (they may be useful for debugging).

---

## Parallelism Constraints Summary

| Phases | Parallel? |
|---|---|
| Phase 1 (Partition) + Phase 2 (Diagram) | Yes — run together |
| Phase 3 workers across partitions | Yes |
| Phase 4 workers across partitions | Yes |
| Phase 5 — three detectors within a partition | Yes |
| Phase 5 — across different partitions | Yes |
| Phase 6 | No — must wait for all Phase 5 |
| Phase 7 | No — must wait for Phase 6 |
| Maximum concurrent agents | `pipeline.parallelWorkers` from `jca-config.json` |
