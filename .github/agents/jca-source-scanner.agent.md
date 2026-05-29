---
description: "Performs a structural pass over every Java file in an assigned partition; catalogs classes, lock fields, synchronized blocks, and thread entry-points. Writes concurrency_analysis/scans/<PARTITION_ID>-structure.json."
tools: [read, write]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-source-scanner-agent.md`

Use the `PARTITION_ID` provided by the orchestrator. Read `concurrency_analysis/partitions.json` to find your assigned files. Write output only to `concurrency_analysis/scans/<PARTITION_ID>-structure.json`.
