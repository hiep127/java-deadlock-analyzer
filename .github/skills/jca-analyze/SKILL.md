---
name: jca-analyze
description: Runs the full JCA multi-agent concurrency analysis pipeline on a Java source path. Detects deadlocks, race conditions, executor hazards, callback re-entrancy, and JNI boundary blocking. Usage: /jca-analyze <SOURCE_PATH>
---

# Skill: /jca-analyze

## Trigger

```
/jca-analyze <SOURCE_PATH>
```

## Description

Runs the full JCA multi-agent concurrency analysis pipeline on the specified AOSP Java path. Targets the Android Audio Framework by default, but can analyze any AOSP Java module.

## Arguments

| Argument | Required | Description |
|---|---|---|
| `<SOURCE_PATH>` | Yes | Repository-relative path to analyze (e.g., `frameworks/base/services/core/java/com/android/server/audio/`). Must point to a directory containing `.java` files. |

## Example Usage

```
/jca-analyze frameworks/base/services/core/java/com/android/server/audio/
/jca-analyze frameworks/base/media/java/android/media/
/jca-analyze frameworks/base/services/core/java/com/android/server/media/
```

## Workflow

Execution follows the map-reduce workflow defined in `workflows/analyze.md`:

| Phase | Agent | Action |
|---|---|---|
| 0 | jca-orchestrator | Initialize `concurrency_analysis/` directory |
| 1+2 | jca-partitioner + jca-diagram-generator | Partition source; build lock graph (parallel) |
| 3 | jca-source-scanner | Structural scan per partition (parallel) |
| 4 | jca-fullscan-worker | Deep line-by-line scan per partition (parallel) |
| 5 | jca-race-detector, jca-deadlock-detector, jca-edge-case-analyzer | Detection per partition (all three parallel) |
| 6 | jca-merger | Merge and deduplicate all findings |
| 7 | jca-consolidator | Format final report; filter false positives |

## Output

| File | Description |
|---|---|
| `concurrency_analysis/report.md` | Human-readable Markdown report |
| `concurrency_analysis/report.json` | Machine-readable JSON report |
| `concurrency_analysis/lock-dependency.dot` | Lock dependency graph (Graphviz DOT) |
| `concurrency_analysis/merged-findings.json` | Deduplicated raw findings |
| `concurrency_analysis/pipeline.log` | Execution log |

## Execution Principles

- All agents operate **read-only** on the source tree.
- Every finding includes `File: <path>  Line: <N>`.
- Scanning is **exhaustive** — no file is skipped or partially read.
- Each agent runs in its own **isolated context window**.
- All inter-agent data flows through **files on disk** in `concurrency_analysis/`.

## Agent Instructions

Full agent instructions are in `.github/skills/jca-analyze/agents/`.  
Reference documents are in `.github/skills/jca-analyze/references/`.
