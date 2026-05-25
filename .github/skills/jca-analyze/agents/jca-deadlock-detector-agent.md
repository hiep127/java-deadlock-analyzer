# JCA Deadlock Detector — Detailed Agent Instructions

Read the protocol in `INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Identify potential deadlock conditions in your assigned partition. Your primary targets are lock-order inversion in `AudioService` and related Audio Framework classes, Binder calls made under locks, and nested monitor cycles spanning multiple classes.

## Input

- `PARTITION_ID`
- `concurrency_analysis/partitions.json`
- `concurrency_analysis/lock-registry.json` — use `lock_order_edges` and `binder_calls_under_lock` as primary hints
- `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`

## Patterns to Detect

### Pattern 1 — Lock-Order Inversion

The most common deadlock source. Two code paths acquire the same two locks in opposite orders.

**Detection steps:**
1. From `lock-registry.json`, extract all `lock_order_edges` — directed pairs `(outer → inner)`.
2. Check for any pair where both `A→B` and `B→A` edges exist (even across different classes/files). This is a confirmed inversion.
3. While reading source files, look for code paths that establish a new `(outer → inner)` order not yet in the registry; record it.

**Known AOSP Audio lock-order hazards to actively search for:**
- `AudioService.mLock` → `MediaFocusControl.mFocusLock` (must always be in this order; inversion = deadlock).
- `AudioService.mLock` → `DeviceBroker` internal lock (AudioService holds `mLock` and calls `mDeviceBroker.*()` which acquires its own lock).
- `AudioService.mLock` → `AppOpsManager` internal lock (calls to `AppOpsManager` under `mLock` can block on AppOps lock).
- Any `AudioService` lock → `ActivityManagerService.mLock` (via `broadcastStickyIntent` or `isUidActive` calls).

### Pattern 2 — Synchronous Binder Call Under Lock (Binder Deadlock)

Holding an intra-process lock while making a synchronous IPC call. The remote process may call back into the originating service and block on the held lock.

**Look for (while `locks_held` is non-empty in fullscan data):**
- `IAudioService.*` proxy calls from within AudioManager (client-side) while client holds its own lock.
- `IActivityManager.Stub.Proxy.*` calls from within `AudioService.synchronized(mLock)`:
  - `broadcastStickyIntent()`
  - `isUidActive()` / `checkPermission()`
  - `getRunningAppProcesses()`
- `IPackageManager.Stub.Proxy.*` calls from within `AudioService.synchronized(mLock)`.
- `IAudioPolicyService.*` calls made under any `AudioService` lock (crosses into native AudioPolicyService through JNI then Binder).
- `ContentResolver.query()` / `insert()` under any lock (ContentProvider = Binder).

### Pattern 3 — Nested Monitor Cycles

- Direct nesting: `synchronized(mLock) { synchronized(mFocusLock) { ... } }` in one class AND `synchronized(mFocusLock) { synchronized(mLock) { ... } }` in another class or method.
- Indirect: Method A (holding Lock1) calls method B which acquires Lock2; method C (holding Lock2) calls method D which acquires Lock1.

**Audio-specific:**
- `MediaFocusControl` methods called inside `AudioService.synchronized(mLock)` that themselves use `synchronized(mFocusLock)`.
- `AudioService.mLock` held during calls into `AudioSystem` JNI (`setParameters`, `getParameters`, `setDeviceConnectionState`) — these may block on the native `AudioPolicyManager` mutex.

### Pattern 4 — `Handler.runWithScissors()` Deadlock

`runWithScissors()` blocks the calling thread until the Runnable completes on the Handler's Looper thread. Deadlock occurs if the Looper thread needs the lock held by the calling thread.

- Detect: `mAudioHandler.runWithScissors(...)` or any `Handler.runWithScissors(...)` call inside `synchronized(mLock)` or any other lock.
- Also detect: `runWithScissors()` where the target Handler's thread also posts back to the caller's thread.

### Pattern 5 — `wait()` / `notify()` Hazards

- `wait()` called on an object outside a `synchronized(object)` block — will throw `IllegalMonitorStateException` at runtime.
- `notify()` without a corresponding guaranteed `wait()` — permanent thread stall.
- `wait()` without a while-loop guard — spurious wakeup can silently bypass the wait condition.

## Output Format (`concurrency_analysis/findings/<PARTITION_ID>-deadlocks.json`)

```json
{
  "partition_id": "p01-audio-service",
  "detector": "jca-deadlock-detector",
  "findings": [
    {
      "id": "DEAD-p01-001",
      "type": "binder_call_under_lock",
      "severity": "CRITICAL",
      "title": "broadcastStickyIntent() called while holding AudioService.mLock",
      "description": "AudioService.setStreamVolumeLocked() holds mLock and calls IActivityManager.Stub.Proxy.broadcastStickyIntent(). If ActivityManagerService attempts to call back into AudioService (e.g., via IAudioService.setStreamVolume), it will deadlock waiting for mLock.",
      "file": "frameworks/base/services/core/java/com/android/server/audio/AudioService.java",
      "line": 7890,
      "locks_held_at_call_site": ["mLock"],
      "callee": "IActivityManager.Stub.Proxy.broadcastStickyIntent",
      "related_locations": [
        {
          "file": "frameworks/base/services/core/java/com/android/server/audio/AudioService.java",
          "line": 7860,
          "note": "mLock acquired here"
        }
      ],
      "recommendation": "Defer the broadcast to a Handler message sent after releasing mLock."
    }
  ]
}
```

### Severity

- `CRITICAL`: Binder call under lock in `AudioService`, confirmed lock-order inversion between `mLock` and `mFocusLock`.
- `HIGH`: Binder call under lock in helper classes; `runWithScissors()` under lock.
- `MEDIUM`: Nested monitor cycle spanning two classes; `wait()` on wrong monitor.
- `LOW`: Potential inversion not confirmed due to private method visibility.

## Mandatory Rules

- Read every file in the partition completely. No early stopping.
- Never modify any source file.
- Every finding must include exact `file` and `line`.
- Write only to `concurrency_analysis/findings/<PARTITION_ID>-deadlocks.json`.
