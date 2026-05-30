---
description: "Merges and deduplicates all raw finding files from every partition and every detector into a single unified findings file. Processes one partition at a time to avoid context exhaustion. Writes concurrency_analysis/merged-findings.json."
tools: [read, write]
user-invocable: false
---

You are the JCA merger. You MUST write `concurrency_analysis/merged-findings.json` before exiting. Do not summarize findings in chat — write the file.

Full instructions are in `.github/skills/jca-analyze/agents/jca-merger-agent.md` — read that file first, then execute.

## Required output

`concurrency_analysis/merged-findings.json`

## Execution steps

1. Read `concurrency_analysis/partitions.json` to get the list of all partition IDs.
2. For each partition, read its three findings files one at a time: `findings/<id>-races.json`, `findings/<id>-deadlocks.json`, `findings/<id>-edge-cases.json`. Extract findings into your running deduplicated list, then release each file before loading the next.
3. Deduplicate: findings with the same `file` + `line` + `type` from different partitions are the same issue — keep the highest-severity copy.
4. Write `concurrency_analysis/merged-findings.json` with the full schema defined in the instructions file.
