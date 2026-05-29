# Inventory Reading Protocol

All JCA agents must follow this protocol when reading the source inventory and consuming inter-agent output files.

---

## 1. Reading Source Files

- **Read every `.java` file assigned to your partition completely**, from line 1 to the final line.
- **Never truncate, skip, or summarize early**. Early stopping is a pipeline failure.
- If a file exceeds your context window, split it into overlapping segments of ≤ 500 lines with a 50-line overlap, process each segment, then merge the per-segment findings before writing output.
- Record `files_scanned` in your output JSON; it must equal the partition file count from `concurrency_analysis/partitions.json`.

## 2. Reading Inter-Agent Files

All inter-agent data files are located in `concurrency_analysis/`. The read order for each phase is:

| Phase | Files to read before starting |
|---|---|
| Scan (jca-source-scanner) | `concurrency_analysis/partitions.json` |
| Fullscan (jca-fullscan-worker) | `concurrency_analysis/partitions.json`, `concurrency_analysis/scans/<id>-structure.json` |
| Detection (race/deadlock/edge-case) | `concurrency_analysis/lock-registry.json`, `concurrency_analysis/scans/<id>-fullscan.json` |
| Merge (jca-merger) | All `concurrency_analysis/findings/*.json` |
| Consolidate (jca-consolidator) | `concurrency_analysis/merged-findings.json`, `concurrency_analysis/lock-registry.json` |

## 3. File Path Format

All file references in output JSON **must** use the project-relative path from the repository root, in the form:

```
"file": "src/main/java/com/example/service/OrderService.java"
```

**Never** use absolute OS paths. **Never** omit the path.

## 4. Line Number References

Every finding, annotation, and lock entry **must** include an integer `line` field. Line numbers are 1-indexed. If a block spans multiple lines, use the **opening line** as the primary reference and record `end_line` separately.

## 5. Writing Output Files

- Write output only to `concurrency_analysis/` (or its subdirectories).
- **Never write to the source tree.**
- File names follow the pattern defined in `workflows/analyze.md`. Do not deviate.
- Write valid JSON only. Validate structure before writing.

## 6. Handling Missing or Malformed Input

- If an expected input file is missing, log the absence to `concurrency_analysis/pipeline.log` and **abort** — do not continue with incomplete data.
- If a specific finding entry is malformed, log it to `concurrency_analysis/pipeline.log` and skip that entry. Do not abort the entire phase for a single bad entry.
