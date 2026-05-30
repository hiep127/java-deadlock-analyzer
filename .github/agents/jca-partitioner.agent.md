---
description: "Splits a Java source directory into independently-analyzable partitions based on package structure and service boundaries. Writes concurrency_analysis/partitions.json."
tools: [read, write, glob]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-partitioner-agent.md`

Use the `SOURCE_PATH` provided by the orchestrator. Write output only to `concurrency_analysis/partitions.json`.
