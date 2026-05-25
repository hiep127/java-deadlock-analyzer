# JCA — Java Concurrency Analyzer

A GitHub Copilot Skill for detecting concurrency defects in **Google AOSP Android Framework Java modules**, with a primary focus on the **Audio Framework**.

## What It Detects

- **Deadlocks** — lock-order inversion, nested monitor cycles, synchronous Binder calls under a lock
- **Binder hazards** — synchronous IPC while holding a Java lock, re-entrant AIDL callbacks (audio focus dispatch)
- **Race conditions** — unsynchronized access to `AudioService` state (`mStreamStates`, `mAudioMode`, `mFocusStack`)
- **`AudioSystem` JNI blocking under lock** — JNI calls to native AudioFlinger/AudioPolicyManager inside `synchronized` blocks
- **Handler/Looper misuse** — `runWithScissors()` under lock, message queue priority inversion
- **`DeathRecipient` races** — `linkToDeath` registration gaps; `binderDied()` race on focus-owner state

## Quick Start

### Run an Analysis

```
/jca-analyze frameworks/base/services/core/java/com/android/server/audio/
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
| `/jca-analyze <path>` | Run full analysis pipeline |
| `/jca-publish` | Publish latest report |
| `/jca-help` | Show usage |

## Architecture

```
.github/
  agents/                    ← Lightweight stub files (tool permissions + pointer to real instructions)
  skills/
    jca-analyze/
      agents/                ← Full agent instruction files (the real system prompts)
      references/            ← AOSP rules, edge cases, output format definitions
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
- `publish.spaceKey` — Confluence space
- `analysis.minSeverityToReport` — minimum severity threshold

## Reference Documents

| Document | Purpose |
|---|---|
| `skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` | File reading and output writing protocol |
| `skills/jca-analyze/references/analysis-phases.md` | Pipeline phase definitions |
| `skills/jca-analyze/references/java-aosp-assumptions.md` | AOSP thread model and JMM rules |
| `skills/jca-analyze/references/aosp-audio-edge-cases.md` | Audio Framework hazard catalogue |
| `skills/jca-analyze/references/issue-output-format.md` | Finding schemas and report format |
