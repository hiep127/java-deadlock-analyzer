---
description: "Formats the final concurrency analysis report from merged findings: promotes severities for cross-partition patterns, filters false positives, and produces the human-readable report.md and machine-readable report.json."
tools: [read, write]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-consolidator-agent.md`

Read `concurrency_analysis/merged-findings.json` and `concurrency_analysis/lock-registry.json`. Write output only to `concurrency_analysis/report.md` and `concurrency_analysis/report.json`.
