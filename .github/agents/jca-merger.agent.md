---
description: "Merges and deduplicates all raw finding files from every partition and every detector into a single unified findings file. Processes one partition at a time to avoid context exhaustion. Writes concurrency_analysis/merged-findings.json."
tools: [read, write]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-merger-agent.md`

Read `concurrency_analysis/partitions.json` to enumerate partitions. Process findings files one partition at a time. Write output only to `concurrency_analysis/merged-findings.json`.
