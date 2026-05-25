# JCA Source Scanner — Detailed Agent Instructions

Read the protocol in `INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Perform an initial structural pass over every Java file in your assigned partition. Build a structural index (class hierarchy, lock fields, synchronized blocks, thread entry-points) that seeds the deeper jca-fullscan-worker scan.

## Input

- `PARTITION_ID`: The partition ID you are assigned (e.g., `p01-audio-service`).
- `concurrency_analysis/partitions.json`: Contains the file list for your partition.

## Step-by-Step Instructions

### Step 1 — Load your file list

Read `concurrency_analysis/partitions.json`. Find the entry matching your `PARTITION_ID` and collect its `files` array.

### Step 2 — Scan each file

For every file in the list, read it **completely from line 1 to the last line**. Extract:

**Class declarations:**
- Class name, superclass, all implemented interfaces.
- Flag: is this class a subclass of `Binder`, a `Service`, an AIDL `*.Stub`?
- Flag: is it an AOSP Audio class (`AudioService`, `AudioManager`, `AudioFlinger`, `AudioTrack`, `AudioRecord`, `MediaFocusControl`, etc.)?

**Lock fields:**
- Every field whose type is `Object`, `ReentrantLock`, `ReentrantReadWriteLock`, or any type used as a monitor (infer from naming: `*Lock`, `*Monitor`, `*Guard`).
- Record field name, type, modifier (`private`/`static`/`final`), and declared line.

**Synchronized methods and blocks:**
- Synchronized method declarations: record class, method signature, line number.
- `synchronized(expr)` blocks: record the lock expression, start line, end line, and enclosing method.

**Thread entry-points:**
- `run()` overrides (Runnable, Thread subclasses).
- `Handler.Callback.handleMessage()` overrides.
- AIDL `onTransact()` overrides (called on Binder thread pool).
- `AsyncTask.doInBackground()` overrides.
- Lambda/anonymous Runnable passed to `Handler.post()`, `Executor.execute()`, etc.

**Cross-thread call sites:**
- `handler.post(...)`, `handler.sendMessage(...)`, `handler.sendMessageAtFrontOfQueue(...)`
- `handler.runWithScissors(...)`
- `executor.execute(...)`, `new Thread(...).start()`
- Audio-specific: `mAudioHandler.sendMessage(...)`, `mBrokerHandler.sendMessage(...)`

### Step 3 — Write output

Write `concurrency_analysis/scans/<PARTITION_ID>-structure.json`.

## Output Format

```json
{
  "partition_id": "p01-audio-service",
  "files_scanned": 1,
  "classes": [
    {
      "name": "AudioService",
      "file": "frameworks/base/services/core/java/com/android/server/audio/AudioService.java",
      "superclass": "IAudioService.Stub",
      "interfaces": ["AudioFocusStateListener"],
      "is_binder_service": true,
      "is_audio_framework_class": true,
      "lock_fields": [
        {
          "name": "mLock",
          "type": "Object",
          "modifiers": ["private", "final"],
          "declared_line": 312
        }
      ],
      "synchronized_methods": [
        { "signature": "void setStreamVolume(int, int, int, String)", "line": 1100 }
      ],
      "synchronized_blocks": [
        {
          "lock_expr": "mLock",
          "start_line": 2300,
          "end_line": 2340,
          "method": "requestAudioFocusForClient"
        }
      ],
      "thread_entry_points": [
        { "type": "onTransact", "line": 890, "thread": "binder_pool" },
        { "type": "handleMessage", "line": 4200, "thread": "mAudioHandler" }
      ],
      "cross_thread_calls": [
        {
          "type": "handler.sendMessage",
          "target_handler": "mAudioHandler",
          "line": 4500,
          "method": "setStreamVolume"
        }
      ]
    }
  ]
}
```

## Mandatory Rules

- Read every file in the partition completely. No partial scans. `files_scanned` must equal the partition file count.
- Never modify any source file.
- Every entry must include the exact `file` path and `line` number.
- Write only to `concurrency_analysis/scans/<PARTITION_ID>-structure.json`.
