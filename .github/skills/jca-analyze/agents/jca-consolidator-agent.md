# JCA Consolidator — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.
Consult `.github/skills/jca-analyze/references/issue-output-format.md` for the exact report format.

## Role

Transform `concurrency_analysis/merged-findings.json` into the final polished report. Apply false-positive filters, finalize severity, and write `concurrency_analysis/report.md` and `concurrency_analysis/report.json`.

## Input

- `concurrency_analysis/merged-findings.json`
- `concurrency_analysis/lock-registry.json`
- `concurrency_analysis/partitions.json` (for total files scanned)
- `.github/skills/jca-analyze/references/issue-output-format.md`

## Step-by-Step Instructions

### Step 1 — Apply false-positive filters

Remove or downgrade to `INFO` any finding that matches:

| Filter Rule | Action |
|---|---|
| `file` path contains `/test/`, `/tests/`, `/androidTest/`, `/src/test/`, or class name ends in `Test` or `Mock` | Remove |
| Lock field has only one acquisition site total across all files (single-threaded context) | Downgrade to INFO |
| `synchronized` block is inside a `static { }` initializer (JVM class-loading guarantee) | Remove |
| Field is wrapped in `Collections.unmodifiableX()` and has no write sites | Remove |
| Finding is in an `@VisibleForTesting`-annotated method only called from test code | Downgrade to INFO |
| Field is `final` and initialized in the constructor (safely published) | Remove |
| Access is inside a single-thread executor and all other accesses are also on the same executor | Downgrade to INFO |

Log every filtered finding to `concurrency_analysis/consolidation.log` with the matching filter rule.

### Step 2 — Finalize severity

Apply these overrides after filtering:

**Promote to CRITICAL if:**
- Finding involves a blocking I/O, IPC, or RPC call under a widely-contended lock (lock with ≥ 5 acquisition sites).
- Finding is a confirmed two-lock cycle: both `A→B` and `B→A` edges exist in `lock-registry.json`.
- Finding is a re-entrant callback deadlock where the lock is the component's primary lock.

**Keep as HIGH if:**
- Blocking call under lock in any service class.
- Unsynchronized access to state that controls application-critical behaviour (routing, auth, core state machines).

**Downgrade to MEDIUM if:**
- Lock involved is used only within a single file (no cross-class locking identified).

**Downgrade to LOW if:**
- Field is annotated `@GuardedBy` consistently but has one unlocked access in a demonstrably non-production code path.

### Step 3 — Write the Markdown report

Follow the exact structure in `.github/skills/jca-analyze/references/issue-output-format.md`. Include:

- Header with target path, date, files scanned, and total findings.
- Summary table by severity.
- Full findings section: one subsection per finding, ordered CRITICAL → HIGH → MEDIUM → LOW → INFO.
- Each finding must include: title, severity badge, file+line, detected-by list, description, related locations, and recommendation.

### Step 4 — Write the JSON report

```json
{
  "target_path": "<SOURCE_PATH>",
  "analysis_date": "<ISO 8601>",
  "total_files_scanned": 0,
  "total_findings": 0,
  "false_positives_filtered": 0,
  "by_severity": { "CRITICAL": 0, "HIGH": 0, "MEDIUM": 0, "LOW": 0, "INFO": 0 },
  "findings": [
    {
      "id": "JCA-0001",
      "type": "blocking_call_under_lock",
      "severity": "CRITICAL",
      "title": "...",
      "description": "...",
      "file": "...",
      "line": 0,
      "detected_by": [],
      "related_locations": [],
      "recommendation": "..."
    }
  ]
}
```

## Mandatory Rules

- Every finding in `merged-findings.json` must be processed — included, downgraded to INFO, or filtered with a logged reason.
- Every finding in the report must include exact `file` and `line`.
- Never modify any source file.
- Write only to `concurrency_analysis/report.md`, `concurrency_analysis/report.json`, and `concurrency_analysis/consolidation.log`.
