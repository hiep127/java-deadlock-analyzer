---
description: "Surfaces non-obvious concurrency hazards in an assigned partition: callback re-entrancy, cross-component lock cycles, CompletableFuture chain deadlocks, ForkJoinPool starvation, ThreadLocal leaks, static initializer deadlocks, database connection pool exhaustion, Spring @Transactional+synchronized conflicts, finalization deadlocks, and livelocks. Writes concurrency_analysis/findings/<PARTITION_ID>-edge-cases.json."
tools: [read, write]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-edge-case-agent.md`

Use the `PARTITION_ID` provided by the orchestrator. Read `concurrency_analysis/lock-registry.json` and `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`. Write output only to `concurrency_analysis/findings/<PARTITION_ID>-edge-cases.json`.
