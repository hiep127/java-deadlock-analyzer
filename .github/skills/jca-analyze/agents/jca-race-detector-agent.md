# JCA Race Detector — Detailed Agent Instructions

Read the protocol in `INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Identify all race conditions in your assigned partition's Java source code. Focus on AOSP Audio Framework patterns: unsynchronized shared mutable state, `volatile` misuse, and non-atomic check-then-act sequences in `AudioService`, `AudioManager`, `MediaFocusControl`, and related classes.

## Input

- `PARTITION_ID`
- `concurrency_analysis/partitions.json`
- `concurrency_analysis/lock-registry.json` (identifies which fields are lock-protected)
- `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json` (pre-annotated events; use as a hint but re-read source)

## Patterns to Detect

### Pattern 1 — Unsynchronized Shared Mutable State

A field is written in one thread context without a lock **and** read in another thread context without a lock (or with inconsistent locks).

**How to detect:**
1. From the structural index, identify all instance fields of Audio Framework service classes.
2. For each field, collect all write sites (assignment `=`, `+=`, `++`, `--`, collection mutations).
3. For each write site, check whether it is inside a `synchronized(X)` block or method.
4. For each read site, do the same check.
5. If the same field is written/read without consistent lock coverage, it's a race.

**AOSP Audio-specific fields to watch:**
- `mStreamStates[]` in `AudioService` — must always be accessed under `mLock`.
- `mAudioMode` in `AudioService` — accessed from Binder threads and the audio handler.
- `mFocusStack` in `MediaFocusControl` — must be accessed under `mAudioService.mLock` or its own focus lock.
- `mConnectedDevices` / `mAudioDeviceInventory` — accessed by multiple threads in AudioService and DeviceBroker.
- `mPlaybackMonitor` / `mRecordMonitor` state fields.

### Pattern 2 — `volatile` Misuse

Detect compound operations on `volatile` fields that are not atomic:

- Read-check-then-write: `if (mVolatileFlag) { mVolatileFlag = false; }` (two separate memory accesses).
- Increment: `mVolatileCounter++` (read-modify-write is not atomic).
- Object reference swap without CAS: `mVolatileRef = newValue; if (mVolatileRef == expected) { ... }`.

### Pattern 3 — Non-Atomic Check-Then-Act

- Condition on a shared field checked outside a lock, followed by acting on it outside a lock:
  `if (mAudioMode != AudioManager.MODE_IN_CALL) { setMode(...); }` — if `setMode()` is not synchronized with the check.
- `AtomicReference.get()` followed by a non-CAS modification: `ref.set(ref.get().withField(x))`.

### Pattern 4 — Inconsistent Synchronization

- A field is accessed under lock in most methods, but one method accesses it without a lock (even if that method appears "read-only").
- Cross-thread access via `Handler.post()` to a field also accessed directly on the calling thread without synchronization.

### Pattern 5 — AOSP Audio-Specific Races

- `onTransact()` (Binder thread) accessing `mStreamStates` or `mAudioMode` that the audio `Handler` thread also modifies without synchronization.
- `AudioRecord`/`AudioTrack` state fields accessed from the Java API thread and the JNI callback thread simultaneously.
- `IAudioFocusDispatcher` callbacks delivered on a Binder thread while `mFocusStack` is being mutated on the audio handler thread.

## Output Format (`concurrency_analysis/findings/<PARTITION_ID>-races.json`)

```json
{
  "partition_id": "p01-audio-service",
  "detector": "jca-race-detector",
  "findings": [
    {
      "id": "RACE-p01-001",
      "type": "unsynchronized_shared_state",
      "severity": "HIGH",
      "title": "Unsynchronized access to mAudioMode in AudioService",
      "description": "Field mAudioMode is written inside synchronized(mLock) in setMode() but read without any lock in getMode() via the Binder onTransact path.",
      "file": "frameworks/base/services/core/java/com/android/server/audio/AudioService.java",
      "line": 3812,
      "related_locations": [
        {
          "file": "frameworks/base/services/core/java/com/android/server/audio/AudioService.java",
          "line": 1230,
          "note": "Synchronized write site in setMode()"
        }
      ],
      "recommendation": "Guard all accesses to mAudioMode with synchronized(mLock) or use an AtomicInteger."
    }
  ]
}
```

### Severity

- `CRITICAL`: Race on security-sensitive state (UID, permission flags) or on state that controls audio routing affecting system stability.
- `HIGH`: Race on frequently-mutated state in `AudioService` (stream states, audio mode, focus stack).
- `MEDIUM`: Race in non-core audio manager/helper classes.
- `LOW`: `volatile` compound-operation misuse with low contention.

## Mandatory Rules

- Read every file in the partition completely. No early stopping.
- Never modify any source file.
- Every finding must include exact `file` and `line` for both the finding site and all related locations.
- Write only to `concurrency_analysis/findings/<PARTITION_ID>-races.json`.
