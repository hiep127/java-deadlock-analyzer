# JCA — Java Concurrency Analyzer

A GitHub Copilot Skill for detecting concurrency defects in **any Java codebase**.

## What It Detects

- **Deadlocks** — lock-order inversion, nested monitor cycles, blocking calls under a lock
- **Race conditions** — unsynchronized access to shared mutable state, `volatile` misuse, check-then-act races
- **Executor hazards** — `Future.get()` under lock, thread pool starvation, `runWithScissors()` under lock
- **Callback re-entrancy** — listener/callback dispatch while holding a lock
- **Native boundary blocking** — JNI calls made inside `synchronized` blocks
- **Lifecycle races** — listener cleanup races, `ThreadLocal` leaks in pooled threads, static initializer cycles
- **Cross-component lock cycles** — two components hold their own locks while calling into each other

## Quick Start

### Run an Analysis

```
/jca-analyze src/main/java/com/example/service/
```

### Publish the Report

```
/jca-publish
```

### Get Help

```
/jca-help
```

## Commands

| Command | Description |
|---|---|
| `/jca-analyze <path>` | Run full analysis pipeline on any Java source path |
| `/jca-publish` | Publish latest report |
| `/jca-help` | Show usage |

## Architecture

```
.github/
  agents/                    ← Lightweight stub files (tool permissions + pointer to real instructions)
  skills/
    jca-analyze/
      agents/                ← Full agent instruction files (the real system prompts)
      references/            ← Java concurrency rules, edge cases, output format definitions
      workflows/             ← Orchestration workflow
      SKILL.md
    jca-help/SKILL.md
    jca-publish/SKILL.md
  tools/
  copilot-instructions.md    ← Master instruction set
  jca-config.json            ← Runtime configuration
  README.md                  ← This file
```

## Pipeline

```
jca-orchestrator
  ├── Phase 1+2 (parallel): jca-partitioner + jca-diagram-generator
  ├── Phase 3 (per-partition): jca-source-scanner
  ├── Phase 4 (per-partition): jca-fullscan-worker
  ├── Phase 5 (per-partition, parallel): jca-race-detector + jca-deadlock-detector + jca-edge-case-analyzer
  ├── Phase 6: jca-merger
  └── Phase 7: jca-consolidator
```

## Output

All results are written to `concurrency_analysis/` in your workspace:

| File | Description |
|---|---|
| `concurrency_analysis/report.md` | Full Markdown report with severity badges |
| `concurrency_analysis/report.json` | Machine-readable report |
| `concurrency_analysis/lock-dependency.dot` | Graphviz lock dependency graph |
| `concurrency_analysis/merged-findings.json` | Deduplicated raw findings |
| `concurrency_analysis/pipeline.log` | Execution log |

## Configuration

Edit `jca-config.json` to adjust:
- `pipeline.parallelWorkers` — concurrent agent count
- `pipeline.maxPartitionSizeKB` — partition size limit
- `publish.spaceKey` — publish destination space
- `analysis.minSeverityToReport` — minimum severity threshold

## Reference Documents

| Document | Purpose |
|---|---|
| `skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` | File reading and output writing protocol |
| `skills/jca-analyze/references/analysis-phases.md` | Pipeline phase definitions |
| `skills/jca-analyze/references/java-aosp-assumptions.md` | Java thread model and JMM rules |
| `skills/jca-analyze/references/aosp-audio-edge-cases.md` | Known Java concurrency hazard patterns |
| `skills/jca-analyze/references/issue-output-format.md` | Finding schemas and report format |
