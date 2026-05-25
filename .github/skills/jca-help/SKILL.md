# Skill: /jca-help

## Trigger

```
/jca-help
```

## Description

Displays JCA usage instructions, available slash commands, the agent pipeline overview, and links to reference documentation.

## Arguments

None.

## Example Usage

```
/jca-help
```

## Output

---

## JCA — Java Concurrency Analyzer

Detects concurrency defects in AOSP Android Framework Java modules, with a primary focus on the **Audio Framework** (`AudioService`, `MediaFocusControl`, `AudioDeviceBroker`, `AudioManager`, `AudioTrack`, `AudioRecord`).

### Commands

| Command | Description |
|---|---|
| `/jca-analyze <SOURCE_PATH>` | Run the full analysis pipeline on an AOSP Java path |
| `/jca-publish` | Publish the latest report to Confluence (or configured destination) |
| `/jca-help` | Show this help message |

### /jca-analyze Examples

```
/jca-analyze frameworks/base/services/core/java/com/android/server/audio/
/jca-analyze frameworks/base/media/java/android/media/
/jca-analyze frameworks/base/services/core/java/com/android/server/media/
```

### Agent Pipeline

```
jca-orchestrator
  ├── Phase 1+2 (parallel):
  │   ├── jca-partitioner
  │   └── jca-diagram-generator
  ├── Phase 3 (per-partition, parallel):
  │   └── jca-source-scanner
  ├── Phase 4 (per-partition, parallel):
  │   └── jca-fullscan-worker
  ├── Phase 5 (per-partition, all three in parallel):
  │   ├── jca-race-detector
  │   ├── jca-deadlock-detector
  │   └── jca-edge-case-analyzer
  ├── Phase 6: jca-merger
  └── Phase 7: jca-consolidator
```

### Output Files

All results are written to `concurrency_analysis/`:

- `concurrency_analysis/report.md` — full Markdown report with severity badges
- `concurrency_analysis/report.json` — machine-readable report
- `concurrency_analysis/lock-dependency.dot` — lock dependency graph (Graphviz)
- `concurrency_analysis/merged-findings.json` — deduplicated raw findings

### Reference Documents

| Document | Purpose |
|---|---|
| `.github/skills/jca-analyze/references/analysis-phases.md` | Pipeline phase definitions |
| `.github/skills/jca-analyze/references/java-aosp-assumptions.md` | AOSP thread model and JMM rules |
| `.github/skills/jca-analyze/references/aosp-audio-edge-cases.md` | Audio Framework hazard catalogue |
| `.github/skills/jca-analyze/references/issue-output-format.md` | Finding schemas and report format |

### Configuration

Edit `jca-config.json` to adjust `pipeline.parallelWorkers`, `pipeline.maxPartitionSizeKB`, `publish.spaceKey`, and `analysis.minSeverityToReport`.

---

### Execution Principles

- Agents are **read-only** — source code is never modified.
- Every finding includes an exact `File: <path>  Line: <N>` reference.
- Scanning is **exhaustive** — every file is scanned top-to-bottom.
- Each agent runs in its own **isolated context window**.
- All inter-agent data flows through **files on disk** in `concurrency_analysis/`.
