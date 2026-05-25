# JCA Edge Case Analyzer — Detailed Agent Instructions

Read the protocol in `.github/skills/jca-analyze/agents/INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Surface non-obvious, multi-class, multi-process concurrency hazards in the AOSP Audio Framework that simpler detectors miss. You focus on AIDL callback re-entrancy, Handler queue ordering issues, `runWithScissors` timing hazards, audio focus race sequences, and `DeathRecipient` registration gaps.

## Input

- `PARTITION_ID`
- `concurrency_analysis/partitions.json`
- `concurrency_analysis/lock-registry.json`
- `concurrency_analysis/scans/<PARTITION_ID>-fullscan.json`

## Patterns to Detect

### Pattern 1 — Re-entrant AIDL Callback Deadlock (Audio Focus)

Audio focus changes involve a complex callback chain:
1. Client calls `AudioManager.requestAudioFocus()` → Binder → `AudioService.requestAudioFocusForClient()` (holds `mLock`).
2. Inside, `AudioService` dispatches `IAudioFocusDispatcher.dispatchAudioFocusChange()` back to all current focus owners (synchronous Binder call back to clients).
3. A client receiving the focus-change callback may immediately call `AudioManager.abandonAudioFocus()` → Binder → `AudioService` → deadlock on `mLock`.

**Detect:**
- Any `IAudioFocusDispatcher` or `IPlaybackConfigDispatcher` proxy call inside `synchronized(mLock)` in `AudioService` or `MediaFocusControl`.
- Calls to `mFocusOwnersCallbacks.callback()` or similar listener iteration inside a lock.

### Pattern 2 — Handler Queue Priority Inversion in Audio Handler

`AudioService` uses `mAudioHandler` (a single-threaded `Handler`) for serialized operations. Hazards:

- `sendMessageAtFrontOfQueue(MSG_SET_DEVICE_VOLUME)` sent inside `synchronized(mLock)`, bypassing a queued `MSG_PERSIST_VOLUME` that is waiting for the same lock when it runs — this can cause persistent state to disagree with in-memory state.
- `removeMessages(MSG_PERSIST_VOLUME)` called concurrently while the message is being processed by the Handler thread — the processing is partially complete but the removal is a no-op, creating silent data loss.
- Circular posting: Handler message A posts message B, and message B posts message A under certain conditions — unbounded queue growth and starvation.

**Audio-specific message types to watch for circular patterns:**
- `MSG_SET_FORCE_USE` ↔ `MSG_AUDIO_SERVER_DIED`
- `MSG_SET_DEVICE_VOLUME` → `MSG_PERSIST_VOLUME` → (potential back-posting)

### Pattern 3 — `AudioSystem` JNI Blocking Under Lock

`AudioSystem` native methods (`setParameters`, `getParameters`, `setStreamVolumeIndex`, `setDeviceConnectionState`) cross the JNI boundary into native AudioFlinger/AudioPolicyManager. These native calls may block on a native mutex (C++ `std::mutex`), creating an untracked lock acquisition from the Java perspective.

**Detect:**
- Any `AudioSystem.*()` static call inside a `synchronized` block.
- Any `AudioRecord.native_*()` or `AudioTrack.native_*()` called while holding a Java lock.
- These constitute a "hidden lock" acquisition that can invert with the Java lock order.

### Pattern 4 — AIDL One-Way vs. Two-Way Confusion

- `IAudioService` methods declared without `oneway` are synchronous; a client holding its own lock while calling them can deadlock if the service calls back.
- `oneway` methods guarantee delivery ordering only per-interface, not across interfaces — code that assumes ordering between a `oneway` call and a subsequent synchronous call is racy.
- A `oneway` AIDL callback dispatched from `AudioService` while holding `mLock` (even though `oneway` doesn't block, the Binder thread pool thread executing the callback may try to acquire `mLock` via a re-entrant call).

### Pattern 5 — `DeathRecipient` / `linkToDeath` Race

- `IBinder.linkToDeath(recipient, 0)` called outside a `synchronized` block, where the binder reference was last checked for null/validity inside the block — TOCTOU gap.
- `binderDied()` callback executes on a Binder thread while the service thread holds the lock protecting the data structure `binderDied()` needs to clean up.

**Audio-specific:**
- `AudioService` registers `DeathRecipient` for each focus owner's binder; `binderDied()` removes from `mFocusStack`. If `mFocusStack` is also being iterated on the audio handler thread without the `binderDied()` path holding `mFocusLock`, this is a race.

### Pattern 6 — `MediaSession` ↔ `AudioService` Callback Cycle

`MediaSessionService` and `AudioService` interact through callbacks. If:
- `AudioService` holds `mLock` and calls a `MediaSessionService` API,
- `MediaSessionService` holds its own lock and calls an `AudioService` API,

a cross-service lock inversion can deadlock both services.

## Output Format (`concurrency_analysis/findings/<PARTITION_ID>-edge-cases.json`)

```json
{
  "partition_id": "p01-audio-service",
  "detector": "jca-edge-case-analyzer",
  "findings": [
    {
      "id": "EDGE-p01-001",
      "type": "reentrant_aidl_callback_deadlock",
      "severity": "CRITICAL",
      "title": "IAudioFocusDispatcher callback dispatched while holding mLock",
      "description": "MediaFocusControl.dispatchAudioFocusChange() is called inside synchronized(mLock) in AudioService.requestAudioFocusForClient(). The callback is a synchronous Binder call to the client. If the client's dispatchAudioFocusChange() implementation calls back into AudioService (e.g., abandonAudioFocus), it will deadlock on mLock.",
      "file": "frameworks/base/services/core/java/com/android/server/audio/MediaFocusControl.java",
      "line": 512,
      "related_locations": [
        {
          "file": "frameworks/base/services/core/java/com/android/server/audio/AudioService.java",
          "line": 3100,
          "note": "mLock held here when calling into MediaFocusControl"
        }
      ],
      "recommendation": "Copy the list of focus owners to dispatch under the lock, then release the lock before issuing the callbacks."
    }
  ]
}
```

### Severity

- `CRITICAL`: Re-entrant AIDL callback deadlock; `AudioSystem` JNI call under lock in `AudioService`.
- `HIGH`: `runWithScissors()` under lock; `DeathRecipient` race on focus-owner binder; `MediaSession`↔`AudioService` lock cycle.
- `MEDIUM`: Handler queue priority inversion; AIDL one-way/two-way confusion causing contention.
- `LOW`: Potential `oneway` ordering assumption in non-critical paths.

## Mandatory Rules

- Read every file in the partition completely. No early stopping.
- Never modify any source file.
- Every finding must include exact `file` and `line`.
- Write only to `concurrency_analysis/findings/<PARTITION_ID>-edge-cases.json`.
