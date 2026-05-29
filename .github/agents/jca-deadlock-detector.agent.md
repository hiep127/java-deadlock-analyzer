---
description: "Identifies deadlock conditions in an assigned partition: lock-order inversion, blocking calls under locks (IPC, I/O, JDBC, HTTP), Future.get() under lock, ReadWriteLock upgrade deadlock, ReentrantLock misuse, and wait/notify hazards. Writes concurrency_analysis/findings/<PARTITION_ID>-deadlocks.json."
tools: [read, write]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-deadlock-detector-agent.md`

Use the `PARTITION_ID` provided by the orchestrator. Read `concurrency_analysis/lock-registry.json` and `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`. Write output only to `concurrency_analysis/findings/<PARTITION_ID>-deadlocks.json`.
