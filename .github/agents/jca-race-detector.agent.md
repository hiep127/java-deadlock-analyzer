---
description: "Identifies race conditions in an assigned partition: unsynchronized shared mutable state, volatile misuse, non-atomic check-then-act sequences, unsafe object publication, and singleton shared state. Writes concurrency_analysis/findings/<PARTITION_ID>-races.json."
tools: [read, write]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-race-detector-agent.md`

Use the `PARTITION_ID` provided by the orchestrator. Read `concurrency_analysis/lock-registry.json` and `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`. Write output only to `concurrency_analysis/findings/<PARTITION_ID>-races.json`.
