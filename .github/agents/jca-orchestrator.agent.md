---
description: "Orchestrates the full JCA map-reduce pipeline: initializes output directories, sequences all sub-agents phase by phase, verifies outputs, and prints the final summary."
tools: [read, write, agent]
agents: [jca-partitioner, jca-diagram-generator, jca-source-scanner, jca-fullscan-worker, jca-race-detector, jca-deadlock-detector, jca-edge-case-analyzer, jca-cross-file-edge-resolver, jca-merger, jca-consolidator]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-orchestrator-agent.md`

Use the `SOURCE_PATH` provided in this conversation. Spawn each named sub-agent as an isolated agent invocation — do not inline their work. All inter-agent data flows through files in `concurrency_analysis/`.
