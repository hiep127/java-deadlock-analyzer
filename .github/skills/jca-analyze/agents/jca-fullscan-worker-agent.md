# JCA Fullscan Worker — Detailed Agent Instructions

Read the protocol in `INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Execute a deep, line-by-line context scan on every file in your assigned partition. Annotate every synchronization primitive, lock acquisition/release event, Binder call, Handler interaction, and `volatile` access with full contextual information — including which locks are held at the moment of each event.

## Input

- `PARTITION_ID`
- `concurrency_analysis/partitions.json`
- `concurrency_analysis/scans/<PARTITION_ID>-structure.json` (use as a guide; do not trust it fully — re-read source files directly)

## Step-by-Step Instructions

### Step 1 — Load context

Read the structural index for your partition. Use it to pre-populate the list of known lock fields and their expressing variables.

### Step 2 — Per-file deep scan

For every file, read from line 1 to the last line without stopping. Maintain a **lock stack** per method as you read:

- **Push** when entering `synchronized(expr) { }` or calling `.lock()` / `.readLock().lock()` / `.writeLock().lock()`.
- **Pop** when exiting a `synchronized` block (matching `}`) or calling `.unlock()`.
- At every event below, record the current lock stack contents as `locks_held`.

**Events to annotate:**

| Event Type | What to Look For |
|---|---|
| `lock_acquisition` | `synchronized(expr)` block entry or `.lock()` call |
| `lock_release` | `synchronized` block exit or `.unlock()` call |
| `binder_call_under_lock` | AIDL proxy call, `IBinder.transact()`, `ContentResolver.*()` while `locks_held` is non-empty |
| `handler_post_under_lock` | `Handler.post()`, `sendMessage()`, `sendMessageAtFrontOfQueue()` while `locks_held` is non-empty |
| `run_with_scissors` | `Handler.runWithScissors()` anywhere (always annotate; flag as HIGH if `locks_held` non-empty) |
| `wait_notify` | `object.wait()`, `object.notify()`, `object.notifyAll()` |
| `volatile_access` | Read or write to a `volatile` field |
| `nested_synchronized` | `synchronized` block inside another `synchronized` block (same method) |
| `cross_method_lock_entry` | Method call made while locks are held that itself contains a `synchronized` block |

### Step 3 — Audio Framework specific patterns

Beyond the generic events above, additionally annotate:

- `mAudioHandler.sendMessage*()` or `mBrokerHandler.sendMessage*()` calls made inside `synchronized(mLock)`.
- Any call to `IAudioService` proxy methods inside a `synchronized` block (cross-service Binder call).
- `AudioSystem.setParameters()` or `AudioSystem.getParameters()` called under a lock (JNI call that may block on native mutex).
- `mDeviceBroker.*()` calls made under `mLock` in `AudioService` (potential second-lock acquisition via DeviceBroker's own lock).

### Step 4 — Write output

Write `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`.

## Output Format

```json
{
  "partition_id": "p01-audio-service",
  "files_scanned": 1,
  "annotations": [
    {
      "file": "frameworks/base/services/core/java/com/android/server/audio/AudioService.java",
      "line": 7890,
      "end_line": 7890,
      "type": "binder_call_under_lock",
      "detail": "Call to IActivityManager.Stub.Proxy.broadcastStickyIntent() while holding mLock",
      "locks_held": ["mLock"],
      "method": "setStreamVolumeLocked",
      "severity_hint": "HIGH"
    },
    {
      "file": "frameworks/base/services/core/java/com/android/server/audio/AudioService.java",
      "line": 3300,
      "end_line": 3340,
      "type": "nested_synchronized",
      "detail": "synchronized(mFocusLock) acquired inside synchronized(mLock)",
      "locks_held": ["mLock"],
      "method": "requestAudioFocusForClient",
      "severity_hint": "MEDIUM"
    }
  ]
}
```

### Severity Hints

- `CRITICAL`: Binder/JNI call while holding a lock in AudioService, AMS, or WMS.
- `HIGH`: Any Binder/JNI call while holding any lock; `runWithScissors()` under lock.
- `MEDIUM`: Nested `synchronized`; `wait()`/`notify()` on a non-canonical monitor; cross-method lock entry.
- `LOW`: `volatile` compound read-modify-write; `handler.post()` under lock with no circular dependency visible.

## Mandatory Rules

- Read every file in the partition completely from line 1 to the last line. **No early stopping.**
- Never modify any source file.
- `files_scanned` must equal the partition file count.
- Every annotation must include the exact `file` and `line`.
- Write only to `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`.
