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
  "file": "string (project-relative path, e.g. src/main/java/com/example/service/OrderService.java)",
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
| `unsafe_publication` | jca-race-detector | `this` reference escapes from constructor before construction completes |
| `double_checked_locking` | jca-race-detector | DCL pattern on a non-`volatile` field — partially-constructed object may be visible |
| `concurrent_map_compound_race` | jca-race-detector | Non-atomic compound operation on `ConcurrentHashMap` (e.g., containsKey + put) |
| `singleton_shared_mutable_state` | jca-race-detector | Mutable instance field in a singleton component (`@RestController`, `@Service`, `HttpServlet`) accessed from multiple request threads |
| `lock_order_inversion` | jca-deadlock-detector | Two locks acquired in opposite orders on different code paths |
| `blocking_call_under_lock` | jca-deadlock-detector | Blocking operation (IPC, I/O, future.get()) made while holding a Java lock |
| `nested_monitor_cycle` | jca-deadlock-detector | Nested `synchronized` blocks creating a potential cycle across classes |
| `future_get_under_lock` | jca-deadlock-detector | `Future.get()` or `CompletableFuture.join()` called while holding a lock |
| `wait_notify_hazard` | jca-deadlock-detector | `wait()`/`notify()` misuse (wrong monitor, no while-loop guard, unmatched notify) |
| `readwritelock_upgrade_deadlock` | jca-deadlock-detector | Attempt to acquire write lock while holding read lock on the same `ReentrantReadWriteLock` |
| `trylock_unchecked` | jca-deadlock-detector | `tryLock()` return value ignored — critical section entered without the lock |
| `condition_await_no_loop` | jca-deadlock-detector | `condition.await()` inside `if` rather than `while` — spurious wakeup bypasses guard |
| `signal_instead_of_signal_all` | jca-deadlock-detector | `signal()` used when multiple threads may be waiting — other waiters stall permanently |
| `reentrant_callback_deadlock` | jca-edge-case-analyzer | Callback or listener dispatched synchronously while caller holds a lock the callback may need |
| `thread_pool_starvation` | jca-edge-case-analyzer | Tasks submitted to a pool block waiting for tasks from the same pool — deadlock |
| `blocking_call_under_lock_jni` | jca-edge-case-analyzer | Native JNI call made inside a `synchronized` block |
| `async_sync_ordering_confusion` | jca-edge-case-analyzer | Incorrect assumptions about ordering between async and synchronous calls |
| `listener_cleanup_race` | jca-edge-case-analyzer | Listener invoked on background thread while component is being torn down |
| `cross_component_lock_cycle` | jca-edge-case-analyzer | Two components hold their own locks while calling into each other |
| `forkjoinpool_blocking_starvation` | jca-edge-case-analyzer | Blocking work in `ForkJoinPool` (including commonPool) exhausts worker threads — MEDIUM (symptom; code fix exists) |
| `completable_future_chain_deadlock` | jca-edge-case-analyzer | `CompletableFuture` stage calls `.join()` on a future that itself depends on the current stage |
| `completable_future_exception_masking` | jca-edge-case-analyzer | Exception in a `CompletableFuture` chain is swallowed — chain stalls silently |
| `connection_pool_exhaustion` | jca-edge-case-analyzer | Nested transactions or concurrent tasks exhaust JDBC connection pool — LOW (symptom finding; investigate root cause) |
| `transactional_synchronized_deadlock` | jca-edge-case-analyzer | Method is both `@Transactional` and `synchronized` — transaction and lock boundaries conflict |
| `finalization_deadlock` | jca-edge-case-analyzer | `finalize()` acquires a lock that may be held by an application thread |
| `livelock` | jca-edge-case-analyzer | Competing threads retry indefinitely without backoff — no progress but no blocking |

---

## 3. Severity Definitions

| Severity | Meaning | Example |
|---|---|---|
| `CRITICAL` | Near-certain deadlock or data corruption in frequently-exercised production paths. Must be fixed before release. | Blocking IPC call under a primary service lock; confirmed two-lock inversion in a core service |
| `HIGH` | Likely deadlock or race on common code paths. Fix in the next sprint. | Blocking call under lock in a non-core component; unsynchronized write to shared mutable state on a hot path |
| `MEDIUM` | Possible hazard on less-frequent paths or in helper classes. Fix within the release. | Nested monitor spanning two classes; `wait()` without while-loop guard |
| `LOW` | Unlikely hazard, design smell, or **symptom finding** — the finding signals that something may be wrong but the root cause must be investigated before any code fix is meaningful. Consider fixing. | `volatile` compound operation on a low-contention field; thread/connection pool exhaustion (symptom of excessive concurrent blocking, not fixable by raising the pool cap) |
| `INFO` | Informational note; filtered finding; pattern is safe under current usage but worth noting. | `final` field assumed thread-safe (verified safe) |

> **Symptom findings and LOW severity:** Pool exhaustion findings (Binder thread pool, JDBC connection pool, `ForkJoinPool` / `ExecutorService` starvation) are classified `LOW` because they are downstream symptoms. The finding should guide investigation into *why* so many concurrent blocking calls exist, not prompt raising a pool size limit. A `LOW` pool-exhaustion finding often co-occurs with `CRITICAL`/`HIGH` findings (e.g., blocking calls under lock) that are the actual root cause.

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
| `lockA (ServiceX)` | `lockB (ServiceY)` | ServiceX.java:NNNN |
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
| `concurrency_analysis/scans/<id>-structure.json` | `<id>` = partition ID, e.g. `p01-order-service` |
| `concurrency_analysis/scans/<id>-fullscan.json` | Same |
| `concurrency_analysis/findings/<id>-races.json` | Same |
| `concurrency_analysis/findings/<id>-deadlocks.json` | Same |
| `concurrency_analysis/findings/<id>-edge-cases.json` | Same |
| `concurrency_analysis/merged-findings.json` | Fixed name |
| `concurrency_analysis/report.md` | Fixed name |
| `concurrency_analysis/report.json` | Fixed name |
| `concurrency_analysis/pipeline.log` | Fixed name |
| `concurrency_analysis/consolidation.log` | Fixed name |
