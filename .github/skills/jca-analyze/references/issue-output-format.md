# JCA Issue Output Format

This document defines the canonical format for all JCA output files: finding JSON schemas, the Markdown report structure, and severity classification rules.

---

## 1. Finding JSON Schema

Every finding produced by any detector agent must conform to this schema:

```json
{
  "id": "string (detector-local ID, e.g. RACE-p01-001)",
  "type": "string (see type taxonomy below)",
  "severity": "CRITICAL | HIGH | MEDIUM | LOW | INFO",
  "title": "string (short, ≤ 120 chars, no period at end)",
  "description": "string (full explanation; multi-sentence; include thread context and why it is a hazard)",
  "file": "string (project-relative path, e.g. frameworks/base/services/core/java/.../AudioService.java)",
  "line": "integer (1-indexed; opening line of the affected construct)",
  "end_line": "integer | null (closing line if the construct spans multiple lines)",
  "related_locations": [
    {
      "file": "string",
      "line": "integer",
      "note": "string (why this location is related, e.g. 'Lock acquired here')"
    }
  ],
  "recommendation": "string (specific, actionable; ≤ 3 sentences)"
}
```

**Required fields:** `id`, `type`, `severity`, `title`, `description`, `file`, `line`, `recommendation`.  
**Optional fields:** `end_line`, `related_locations` (default: empty array).

---

## 2. Finding Type Taxonomy

| Type | Detector | Description |
|---|---|---|
| `unsynchronized_shared_state` | jca-race-detector | Field read/written by multiple threads without consistent lock coverage |
| `volatile_misuse` | jca-race-detector | Compound operation on a `volatile` field |
| `check_then_act_race` | jca-race-detector | Non-atomic read-check-then-write on shared state |
| `inconsistent_synchronization` | jca-race-detector | Field guarded by a lock in most places but accessed unlocked in at least one place |
| `lock_order_inversion` | jca-deadlock-detector | Two locks acquired in opposite orders on different code paths |
| `binder_call_under_lock` | jca-deadlock-detector | Synchronous IPC call while holding a Java lock |
| `nested_monitor_cycle` | jca-deadlock-detector | Nested `synchronized` blocks creating a potential cycle across classes |
| `run_with_scissors_under_lock` | jca-deadlock-detector | `Handler.runWithScissors()` called while holding a lock |
| `wait_notify_hazard` | jca-deadlock-detector | `wait()`/`notify()` misuse (wrong monitor, no while-loop guard, unmatched notify) |
| `reentrant_aidl_callback_deadlock` | jca-edge-case-analyzer | AIDL callback dispatched synchronously while caller holds a lock that the callback may need |
| `handler_queue_priority_inversion` | jca-edge-case-analyzer | Handler message ordering hazard leading to stale state or starvation |
| `jni_blocking_under_lock` | jca-edge-case-analyzer | JNI call (e.g., `AudioSystem.*`) made inside a `synchronized` block |
| `aidl_oneway_confusion` | jca-edge-case-analyzer | Incorrect assumptions about `oneway` vs. synchronous AIDL call ordering |
| `death_recipient_race` | jca-edge-case-analyzer | `linkToDeath` registration gap or `binderDied` race on shared state |
| `cross_service_lock_cycle` | jca-edge-case-analyzer | Two system services hold their own locks while calling into each other |

---

## 3. Severity Definitions

| Severity | Meaning | Example |
|---|---|---|
| `CRITICAL` | Near-certain deadlock or data corruption in production paths of a core system service. Must be fixed before release. | Binder call under `AudioService.mLock`; confirmed `mLock` ↔ `mFocusLock` inversion |
| `HIGH` | Likely deadlock or race on frequently-exercised paths. Fix in the next sprint. | Binder call under lock in a non-core service; unsynchronized access to `mStreamStates` |
| `MEDIUM` | Possible hazard on less-frequent paths or in helper classes. Fix within the release. | Nested monitor spanning two classes; `wait()` without while-loop guard |
| `LOW` | Unlikely hazard; may be a design smell rather than an active bug. Consider fixing. | `volatile` compound operation on a low-contention field |
| `INFO` | Informational note; filtered finding; pattern is safe under current usage but worth noting. | `final` field assumed thread-safe (verified safe) |

---

## 4. Markdown Report Structure

The final `concurrency_analysis/report.md` must follow this exact structure:

```markdown
# JCA Concurrency Analysis Report

| | |
|---|---|
| **Target Path** | `<SOURCE_PATH>` |
| **Analysis Date** | <ISO 8601 date and time> |
| **Total Files Scanned** | <N> |
| **Total Findings** | <N> (after filtering) |
| **False Positives Filtered** | <N> |

---

## Executive Summary

<2–3 sentences summarizing the most critical findings and the primary risk areas.>

---

## Findings by Severity

| Severity | Count |
|---|---|
| 🔴 CRITICAL | N |
| 🟠 HIGH | N |
| 🟡 MEDIUM | N |
| 🔵 LOW | N |
| ⚪ INFO | N |

---

## Detailed Findings

### 🔴 [CRITICAL] JCA-0001 — <Title>

| | |
|---|---|
| **File** | `<file path>` |
| **Line** | `<N>` |
| **Type** | `<type>` |
| **Detected by** | jca-deadlock-detector, jca-edge-case-analyzer |

**Description:**
<Full description paragraph(s).>

**Related Locations:**
- `File: <path>  Line: <N>` — <note>

**Recommendation:**
<Actionable recommendation.>

---

### 🟠 [HIGH] JCA-0002 — <Title>

...

---

## Appendix: Lock Dependency Graph

See `concurrency_analysis/lock-dependency.dot` for the full Graphviz DOT graph.
Render with: `dot -Tsvg concurrency_analysis/lock-dependency.dot -o lock-graph.svg`

Key lock pairs identified:

| Outer Lock | Inner Lock | Location |
|---|---|---|
| `mLock (AudioService)` | `mFocusLock (MediaFocusControl)` | AudioService.java:NNNN |
```

---

## 5. Severity Icon Reference

Use these icons consistently in Markdown output:

| Severity | Icon |
|---|---|
| CRITICAL | 🔴 |
| HIGH | 🟠 |
| MEDIUM | 🟡 |
| LOW | 🔵 |
| INFO | ⚪ |

---

## 6. Naming Conventions for Output Files

| File | Convention |
|---|---|
| `concurrency_analysis/partitions.json` | Fixed name |
| `concurrency_analysis/lock-registry.json` | Fixed name |
| `concurrency_analysis/lock-dependency.dot` | Fixed name |
| `concurrency_analysis/scans/<id>-structure.json` | `<id>` = partition ID, e.g. `p01-audio-service` |
| `concurrency_analysis/scans/<id>-fullscan.json` | Same |
| `concurrency_analysis/findings/<id>-races.json` | Same |
| `concurrency_analysis/findings/<id>-deadlocks.json` | Same |
| `concurrency_analysis/findings/<id>-edge-cases.json` | Same |
| `concurrency_analysis/merged-findings.json` | Fixed name |
| `concurrency_analysis/report.md` | Fixed name |
| `concurrency_analysis/report.json` | Fixed name |
| `concurrency_analysis/pipeline.log` | Fixed name |
| `concurrency_analysis/consolidation.log` | Fixed name |
