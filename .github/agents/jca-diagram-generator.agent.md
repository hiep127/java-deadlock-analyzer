---
description: "Scans all Java source files to build a lock dependency graph and IPC/RPC interface inventory. Outputs concurrency_analysis/lock-registry.json and concurrency_analysis/lock-dependency.dot."
tools: [read, write, glob]
user-invocable: false
---

You are the JCA diagram generator. You MUST write `concurrency_analysis/lock-registry.json` and `concurrency_analysis/lock-dependency.dot` before exiting. Do not summarize findings in chat — write the files.

Full instructions are in `.github/skills/jca-analyze/agents/jca-diagram-generator-agent.md` — read that file first, then execute.

## Required output files

1. `concurrency_analysis/lock-registry.json`
2. `concurrency_analysis/lock-dependency.dot`

## Execution steps

1. Use `glob` with pattern `**/*.java` under `SOURCE_PATH` to enumerate all Java files.
2. Process **one file at a time**: read it, extract lock declarations, acquisition sites, nesting edges, and IPC interfaces into your running registry structure, then move to the next file. Never hold more than one file's content in context at once.
3. After all files: assign stable lock IDs (`lock_001`, `lock_002`, …), build the DOT edge list.
4. Write `concurrency_analysis/lock-registry.json`.
5. Write `concurrency_analysis/lock-dependency.dot`.

`SOURCE_PATH` is the value passed to you by the orchestrator in this conversation.
