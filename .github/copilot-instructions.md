# JCA — Java Concurrency Analyzer: Copilot Skill Instructions

## Overview

JCA (Java Concurrency Analyzer) is a GitHub Copilot Skill that detects concurrency defects in **Google AOSP Android Framework Java modules**, with a primary focus on the **Audio Framework** (`AudioService`, `MediaFocusControl`, `AudioDeviceBroker`, `AudioManager`, `AudioTrack`, `AudioRecord`).

JCA identifies:
- Lock-order inversion and nested monitor deadlocks
- Synchronous Binder calls made while holding a Java lock
- Race conditions on shared Audio Framework state
- Handler/Looper misuse (`runWithScissors`, message ordering hazards)
- Re-entrant AIDL callback deadlocks (audio focus dispatch patterns)
- `AudioSystem` JNI blocking calls under lock
- `DeathRecipient` / `linkToDeath` race windows

---

## Skills (Slash Commands)

| Command | Description |
|---|---|
| `/jca-analyze <path>` | Run the full multi-agent analysis pipeline on a given AOSP Java path |
| `/jca-publish` | Publish the most recent analysis report to Confluence (or configured destination) |
| `/jca-help` | Display usage instructions and available commands |

---

## Architecture: Stub + Skill Pattern

JCA uses a two-layer architecture:

### Layer 1 — Global Agent Stubs (`.github/agents/`)

Lightweight wrapper files that grant tool permissions and point to the real instructions. These are registered globally by Copilot. They contain **no analysis logic**.

### Layer 2 — Skill Agent Instructions (`.github/skills/jca-analyze/agents/`)

The full, detailed system prompts for each agent. Encapsulated within the `jca-analyze` skill for modularity. These contain all analysis rules, output schemas, and AOSP-specific patterns.

---

## Agent Pipeline

```
jca-orchestrator  (reads: .github/skills/jca-analyze/workflows/analyze.md)
  │
  ├── [Phase 1+2, parallel]
  │   ├── jca-partitioner
  │   └── jca-diagram-generator
  │
  ├── [Phase 3, per-partition parallel]
  │   └── jca-source-scanner
  │
  ├── [Phase 4, per-partition parallel]
  │   └── jca-fullscan-worker
  │
  ├── [Phase 5, three detectors per partition, all parallel]
  │   ├── jca-race-detector
  │   ├── jca-deadlock-detector
  │   └── jca-edge-case-analyzer
  │
  ├── [Phase 6]
  │   └── jca-merger
  │
  └── [Phase 7]
      └── jca-consolidator
```

---

## Strict Execution Principles

These principles are **mandatory** for every agent in the JCA pipeline.

### 1. Fresh Context Per Agent
Each sub-agent runs in its own isolated context window. No agent inherits conversational state from another. All data exchange happens through files in `concurrency_analysis/`.

### 2. Write Directly to Disk
All agent output — findings, annotations, diagrams, reports — is written directly to `concurrency_analysis/`. No in-memory or conversational transfer between agents.

### 3. Exhaustive Scanning — No Early Stopping
Every source file assigned to an agent must be scanned completely, from line 1 to the last line. Partial scans are a pipeline failure.

### 4. Always Include File Paths
Every finding must reference the exact source location: `File: <relative-path>  Line: <N>`. Findings without a precise file/line reference are invalid.

### 5. Read-Only — Never Modify Source
Agents are strictly read-only consumers of AOSP source code. Only `concurrency_analysis/` may be written to.

---

## Reference Documents

| Document | Purpose |
|---|---|
| `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` | File reading and output writing protocol for all agents |
| `.github/skills/jca-analyze/references/analysis-phases.md` | Pipeline phase definitions, dependencies, and I/O |
| `.github/skills/jca-analyze/references/java-aosp-assumptions.md` | AOSP Java thread model, JMM rules, and false-positive suppression |
| `.github/skills/jca-analyze/references/aosp-audio-edge-cases.md` | Audio Framework-specific known hazard patterns |
| `.github/skills/jca-analyze/references/issue-output-format.md` | Finding JSON schema, severity definitions, and report format |

---

## Output Directory

All output is written to `concurrency_analysis/` in the workspace root:

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
