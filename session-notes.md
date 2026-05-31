# Session Notes — 2026-05-31

## Context
Resuming work on JCA (Java Concurrency Analyzer) — a GitHub Copilot Skill for detecting concurrency defects in Java codebases.

Last session ended at commit `472c9db` (rewrote all agent stubs to be imperative and self-contained).

---

## JCA Detailed Flow (explained this session)

### Trigger
User runs `/jca-analyze <SOURCE_PATH>` in Copilot Chat → `jca-orchestrator` takes control.

### Pipeline phases

| Phase | Agent(s) | What it does | Output |
|---|---|---|---|
| 0 | orchestrator (self) | Creates `concurrency_analysis/` dirs, writes pipeline.log | `pipeline.log` |
| 1+2 (parallel) | jca-partitioner + jca-diagram-generator | Partition source tree; build lock dependency graph | `partitions.json`, `lock-registry.json`, `lock-dependency.dot` |
| 3 (per partition, parallel) | jca-source-scanner | Structural pass: classes, lock fields, sync blocks, thread entry points | `scans/<id>-structure.json` |
| 4 (per partition, parallel) | jca-fullscan-worker | Line-by-line deep scan with lock stack tracking; annotates every sync event | `scans/<id>-fullscan.json` |
| 4.5 (sequential) | jca-cross-file-edge-resolver | Joins `cross_method_lock_entry` annotations against lock registry to infer indirect multi-file lock chains | `cross-file-edges.json` + enriches `lock-registry.json` |
| 5 (per partition, all 3 parallel) | jca-race-detector + jca-deadlock-detector + jca-edge-case-analyzer | Detection using the complete lock graph | `findings/<id>-races.json`, `-deadlocks.json`, `-edge-cases.json` |
| 6 (sequential) | jca-merger | Deduplicates all findings, ranks by severity | `merged-findings.json` |
| 7 (sequential) | jca-consolidator | Formats final report, filters false positives | `report.md`, `report.json` |

### Key design rules
- Orchestrator never reads source files or subagent output content — existence checks only.
- All inter-agent data flows through files in `concurrency_analysis/`.
- Every finding must include `File: <path>  Line: <N>`.
- Fail-fast: missing output file → abort, leave partials on disk for debugging.

---

## Bugs Found and Fixed This Session

### Bug 1: Diagram generator context exhaustion (commit `20846f2`)

**Symptom:** Diagram generator printed JSON to chat instead of writing files. Partitioner worked fine.

**Root cause:** Both stubs have identical `tools: [read, write, glob]` — not a permissions issue.
The diagram generator reads **every `.java` file completely** across the entire `SOURCE_PATH` (not one partition). On a non-trivial codebase this fills the agent's context window before it reaches the write step. Copilot exits the agent and dumps output to chat.
The partitioner avoids this because it only reads `package` declaration lines (~5 lines per file).

**Fix:** Changed checkpoint threshold from "after 200+ files" to **every 25 files**. Agent now overwrites `lock-registry.json` incrementally so partial data is preserved even if context exhausts mid-scan.

Files changed:
- `.github/agents/jca-diagram-generator.agent.md`
- `.github/skills/jca-analyze/agents/jca-diagram-generator-agent.md`

---

### Bug 2: Diagram generator narrates writes instead of calling the write tool (commit `2c78706`)

**Symptom:** Even after the checkpoint fix, agent said "Let me write both files to disk:" in chat but never called the write tool.

**Root causes (two):**
1. Stub said `"Full instructions are in ... — read that file first, then execute."` → agent loaded 144 lines of complex prose before doing anything, entering narration/planning mode for the rest of its run.
2. First write was deferred to step 5 (after all files were scanned) → agent spent its entire execution generating chat text, then continued that pattern when it reached the write step.

**Fix:** Rewrote the stub to be fully self-contained:
- **Step 1 is now two immediate `write` tool calls** (skeleton files) before reading any source. Forces write-mode from the very start.
- Removed reference to detailed instructions file — all extraction rules inlined.
- Checkpoint write every 25 files + two explicit final writes (steps 6 and 7).

File changed:
- `.github/agents/jca-diagram-generator.agent.md`

---

## Commits This Session

| Hash | Message |
|---|---|
| `20846f2` | Fix diagram generator context exhaustion: write checkpoint every 25 files |
| `2c78706` | Fix diagram generator: write skeletons first, inline all steps, drop detail-file reference |

---

---

### Bug 3: Source scanner narrates writes instead of calling the write tool

**Symptom:** jca-source-scanner agents (Phase 3, 10 parallel) said "The subagents can read but not write files" and attempted to work around it by running a Python script in the terminal instead of calling the write tool.

**Root causes (same as Bug 2):**
1. Stub said `"Full instructions are in ... — read that file first, then execute."` → agent entered narration/planning mode before doing any work.
2. Write step was deferred to Step 3 (after scanning all files) → agent spent its entire execution generating chat text, then continued that pattern when it reached the write step.

**Fix (same pattern as Bug 2):**
- **Step 1 is now an immediate `write` tool call** (skeleton file) before reading partitions.json or any source. Forces write-mode from the start.
- Removed reference to detail instructions file — all schema rules inlined into the stub.
- Checkpoint write every 10 files + explicit final write in Step 5.

File changed:
- `.github/agents/jca-source-scanner.agent.md`

---

## Open / Next Steps

- Re-test all fixed agents — verify they each call the write tool in Step 1 without being prompted.

## Systemic Fix Applied (all phases)

After Bug 3 was reported, audited every agent stub. All 7 remaining agents with the broken pattern were fixed in the same session:

| Agent | Fixed? | Notes |
|---|---|---|
| jca-diagram-generator | Yes (commit `2c78706`) | Original fix |
| jca-source-scanner | Yes (Bug 3 session) | Writes skeleton, inline schema |
| jca-fullscan-worker | Yes (Bug 3 session) | Writes skeleton, checkpoint every 10 files |
| jca-race-detector | Yes (Bug 3 session) | Writes skeleton, inline schema |
| jca-deadlock-detector | Yes (Bug 3 session) | Writes skeleton, inline schema |
| jca-edge-case-analyzer | Yes (Bug 3 session) | Writes skeleton, inline schema |
| jca-merger | Yes (Bug 3 session) | Writes skeleton, checkpoint every 5 partitions |
| jca-consolidator | Yes (Bug 3 session) | Writes two skeletons, inline schema |
| jca-cross-file-edge-resolver | Yes (Bug 3 session) | Writes skeleton cross-file-edges.json |
| jca-partitioner | Left as-is | Was working fine; only reads ~5 lines per file |
| jca-orchestrator | Left as-is | No large JSON output; orchestrates via agent calls |
