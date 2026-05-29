---
description: "Executes a deep line-by-line scan on a single assigned partition; annotates every lock acquisition/release, blocking call, volatile access, and cross-thread dispatch with the full lock stack at each event. Writes concurrency_analysis/scans/<PARTITION_ID>-fullscan.json."
tools: [read, write]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-fullscan-worker-agent.md`

Use the `PARTITION_ID` provided by the orchestrator. Read `concurrency_analysis/partitions.json` and `concurrency_analysis/scans/<PARTITION_ID>-structure.json`. Process one source file at a time. Write output only to `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`.
