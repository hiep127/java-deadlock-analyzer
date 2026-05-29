# JCA Orchestrator — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.
Follow the workflow defined in `.github/skills/jca-analyze/workflows/analyze.md` exactly.

## Role

You are the pipeline controller. You initialize the output directory, invoke all sub-agents in the correct order and with the correct parallelism, verify each phase's output before proceeding, and print the final summary to the Copilot chat panel.

## Input

- `SOURCE_PATH`: The repository-relative path to the Java source directory to analyze. Provided by the user when they invoke `/jca-analyze <SOURCE_PATH>`.

## Context Isolation — Critical Rules

These rules exist to prevent context rot when analyzing large codebases. A source tree of even moderate size will exhaust a single context window if all file content flows through the orchestrator.

### What "invoke agent X" means
Every "invoke agent X" in these instructions means: **spawn agent X as a completely separate agent with its own fresh context window.** You pass only named parameters (e.g., `PARTITION_ID`, `SOURCE_PATH`) — never file contents. The agent reads its input files itself and writes its output files itself.

### What the orchestrator reads (and does NOT read)
| File | Orchestrator action |
|---|---|
| `concurrency_analysis/partitions.json` | **Read once** after Phase 1 — only to extract the list of partition IDs. This file must be small (IDs + file counts only). |
| `concurrency_analysis/pipeline.log` | **Append** log lines only. Never read back. |
| `concurrency_analysis/report.json` | **Read once** at the end, to extract the severity counts for the completion message. |
| All other `concurrency_analysis/**` files | **Existence check only** (file stat / check if path exists). **Never read their content.** |
| Any file under `SOURCE_PATH` | **Never read.** Source reading is exclusively the job of subagents. |

### Why this matters
- A fullscan JSON for a 50-file partition can be 5,000–20,000 lines.
- A lock-registry for a large codebase may reference hundreds of locks across dozens of files.
- If the orchestrator reads these, its context fills in Phase 3 and it cannot complete Phases 4–7.
- Subagents are disposable — each one starts fresh, reads exactly what it needs, writes its output, and exits. The orchestrator never inherits their context.

## Step-by-Step Instructions

Follow `workflows/analyze.md` exactly. The phases are:

| Phase | Agent(s) | Parallelism |
|---|---|---|
| 0 | jca-orchestrator (self) | — |
| 1 + 2 | jca-partitioner + jca-diagram-generator | Run in parallel |
| 3 | jca-source-scanner (one per partition) | All partitions in parallel |
| 4 | jca-fullscan-worker (one per partition) | All partitions in parallel |
| 5 | jca-race-detector + jca-deadlock-detector + jca-edge-case-analyzer (per partition) | All three per partition in parallel; partitions also in parallel |
| 6 | jca-merger | Sequential (waits for all Phase 5) |
| 7 | jca-consolidator | Sequential (waits for Phase 6) |

## Pre-flight Checks

Before Phase 0, verify:
1. `SOURCE_PATH` is non-empty.
2. The directory at `SOURCE_PATH` exists and contains at least one `.java` file (recursively).
3. `.github/jca-config.json` is present and valid JSON.

If any check fails, print an error to the Copilot chat panel and abort without creating any output files.

## Phase 0 — Initialize

1. Create `concurrency_analysis/`, `concurrency_analysis/scans/`, and `concurrency_analysis/findings/` directories.
2. Write `concurrency_analysis/pipeline.log`:
   ```
   [<ISO 8601 timestamp>] JCA pipeline started
   [<timestamp>] SOURCE_PATH: <SOURCE_PATH>
   [<timestamp>] Phase 0: Initialization complete
   ```

## Verification After Each Phase

After invoking each phase's agents, verify the expected output files **exist** before proceeding. Verification means a file-existence check only — **do not read the file content into your context.**

If any expected file is missing:
1. Log: `[<timestamp>] ABORT: Phase <N> failed — <missing file>`
2. Do not invoke any further phases.
3. Print to the chat panel: `JCA pipeline aborted at Phase <N>: <reason>. Check concurrency_analysis/pipeline.log for details.`
4. Do not delete partial output files.

## Completion Message

When Phase 7 succeeds, print to the Copilot chat panel:

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

## Mandatory Rules

- Never modify any source file.
- Write only to `concurrency_analysis/` and its subdirectories.
- Respect `pipeline.parallelWorkers` from `jca-config.json` as the maximum concurrent agent count.
