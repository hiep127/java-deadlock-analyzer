---
description: "Resolves cross-file lock-ordering edges by joining locks_held annotations from all fullscan outputs against lock acquisition sites in the registry. Appends inferred edges to lock-registry.json and reports newly confirmed deadlock cycles."
tools: [read, write]
user-invocable: false
---
Read and execute the full instructions from: `.github/skills/jca-analyze/agents/jca-cross-file-edge-resolver-agent.md`

All inter-agent data flows through files in `concurrency_analysis/`. Do not read any source files directly — consume only `lock-registry.json`, `partitions.json`, and the fullscan outputs.
