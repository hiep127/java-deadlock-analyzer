---
description: "Scans all Java source files to build a lock dependency graph and IPC/RPC interface inventory. Outputs concurrency_analysis/lock-registry.json and concurrency_analysis/lock-dependency.dot."
tools: [read, write]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-diagram-generator-agent.md`

Use the `SOURCE_PATH` provided by the orchestrator. Process one file at a time. Write output only to `concurrency_analysis/lock-registry.json` and `concurrency_analysis/lock-dependency.dot`.
