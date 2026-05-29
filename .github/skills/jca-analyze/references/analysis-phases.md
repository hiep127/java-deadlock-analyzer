# JCA Analysis Phases

This document defines the phases of the JCA map-reduce pipeline, their sequencing, dependencies, and expected I/O at each stage.

---

## Pipeline Overview

```
Phase 0: Initialize
Phase 1: Partition        ← Map setup
Phase 2: Diagram          ← Global graph (parallel with Phase 1)
Phase 3: Source Scan      ← Map (one worker per partition)
Phase 4: Full Scan        ← Map (one worker per partition, after Phase 3)
Phase 5: Detection        ← Map (three detectors per partition, parallel, after Phase 4)
Phase 6: Merge            ← Reduce (after all Phase 5 workers finish)
Phase 7: Consolidate      ← Final output (after Phase 6)
```

---

## Phase 0 — Initialize

**Agent:** jca-orchestrator
**Trigger:** User invokes `/jca-analyze <SOURCE_PATH>`

**Actions:**
1. Create `concurrency_analysis/` directory.
2. Create `concurrency_analysis/scans/` subdirectory.
3. Create `concurrency_analysis/findings/` subdirectory.
4. Write `concurrency_analysis/pipeline.log` with start timestamp and `SOURCE_PATH`.

**Completion condition:** All directories exist; `pipeline.log` is written.

---

## Phase 1 — Partition

**Agent:** jca-partitioner
**Depends on:** Phase 0

**Actions:**
1. Walk `SOURCE_PATH` recursively; enumerate all `.java` files.
2. Group files by top-level package or module boundary.
3. Split oversized groups by sub-package.
4. Assign stable partition IDs.

**Output:** `concurrency_analysis/partitions.json`
**Completion condition:** `partitions.json` exists and contains at least one partition.

---

## Phase 2 — Diagram (global, parallel with Phase 1)

**Agent:** jca-diagram-generator
**Depends on:** Phase 0 (not Phase 1 — runs in parallel with Partition)

**Actions:**
1. Scan every `.java` file under `SOURCE_PATH`.
2. Extract all lock objects, acquisition sites, lock-order edges, and IPC interfaces.
3. Build lock dependency graph.

**Output:**
- `concurrency_analysis/lock-registry.json`
- `concurrency_analysis/lock-dependency.dot`

**Completion condition:** Both output files exist and are valid JSON/DOT.

---

## Phase 3 — Source Scan (per partition, parallel)

**Agent:** jca-source-scanner (one instance per partition)
**Depends on:** Phase 1

**Actions:**
1. Read the partition's file list from `partitions.json`.
2. Perform structural pass over each file.
3. Extract class hierarchies, lock fields, synchronized blocks, thread entry-points.

**Output:** `concurrency_analysis/scans/<partition_id>-structure.json`
**Completion condition:** One structure file per partition exists.

---

## Phase 4 — Full Scan (per partition, parallel)

**Agent:** jca-fullscan-worker (one instance per partition)
**Depends on:** Phase 3 (corresponding partition's structure file)

**Actions:**
1. Re-read every source file in the partition.
2. Maintain per-method lock stack; annotate every synchronization event.
3. Flag `blocking_call_under_lock`, `nested_synchronized`, `future_get_under_lock`, etc.

**Output:** `concurrency_analysis/scans/<partition_id>-fullscan.json`
**Completion condition:** One fullscan file per partition exists.

---

## Phase 5 — Detection (per partition, three detectors run in parallel)

**Agents:** jca-race-detector, jca-deadlock-detector, jca-edge-case-analyzer
**Depends on:** Phase 4 (corresponding fullscan file) and Phase 2 (lock-registry.json)

**Actions (each detector independently):**
1. Re-read source files.
2. Apply detector-specific pattern matching.
3. Write findings to separate output files.

**Output:**
- `concurrency_analysis/findings/<partition_id>-races.json`
- `concurrency_analysis/findings/<partition_id>-deadlocks.json`
- `concurrency_analysis/findings/<partition_id>-edge-cases.json`

**Completion condition:** All three finding files exist for every partition.

---

## Phase 6 — Merge (reduce)

**Agent:** jca-merger
**Depends on:** All Phase 5 finding files for all partitions

**Actions:**
1. Verify all finding files are present.
2. Validate and load all findings.
3. Deduplicate by `(file, line)`.
4. Cross-reference with `lock-registry.json`.
5. Assign merged IDs and sort.

**Output:** `concurrency_analysis/merged-findings.json`
**Completion condition:** `merged-findings.json` exists.

---

## Phase 7 — Consolidate

**Agent:** jca-consolidator
**Depends on:** Phase 6

**Actions:**
1. Apply false-positive filters.
2. Finalize severity levels.
3. Write Markdown and JSON reports.

**Output:**
- `concurrency_analysis/report.md`
- `concurrency_analysis/report.json`
- `concurrency_analysis/consolidation.log`

**Completion condition:** `report.md` and `report.json` exist. Pipeline is complete.

---

## Parallelism Summary

| Can run in parallel | Phases |
|---|---|
| Yes | Phase 1 and Phase 2 |
| Yes | All Phase 3 workers across partitions |
| Yes | All Phase 4 workers across partitions |
| Yes | All three Phase 5 detectors within a partition |
| Yes | Phase 5 workers across different partitions |
| No | Phase 6 must wait for all Phase 5 |
| No | Phase 7 must wait for Phase 6 |

---

## Output Directory Layout

```
concurrency_analysis/
  pipeline.log
  partitions.json
  lock-registry.json
  lock-dependency.dot
  scans/
    <partition_id>-structure.json
    <partition_id>-fullscan.json
  findings/
    <partition_id>-races.json
    <partition_id>-deadlocks.json
    <partition_id>-edge-cases.json
  merged-findings.json
  consolidation.log
  report.md
  report.json
```
