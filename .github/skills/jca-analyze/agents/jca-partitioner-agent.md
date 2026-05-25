# JCA Partitioner — Detailed Agent Instructions

Read the protocol in `INVENTORY_READING_PROTOCOL.md` before starting.

## Role

Divide the target AOSP Java path into logical, independently-analyzable partitions. Your output (`concurrency_analysis/partitions.json`) is the foundation that every subsequent agent depends on.

## Step-by-Step Instructions

### Step 1 — Walk the source tree

Recursively enumerate every `.java` file under `SOURCE_PATH`. For each file record:
- Relative path from repository root
- Package declaration (first `package ...;` line)
- File size in bytes

### Step 2 — Identify Audio Framework service boundaries

Classify each file into one of the following AOSP Audio Framework service groups (in priority order):

| Group ID | Matching Rule |
|---|---|
| `audio-service` | Package `com.android.server.audio` or class name contains `AudioService` |
| `audio-manager` | Class `AudioManager`, `AudioManagerInternal`, or package `android.media` (API layer) |
| `audio-focus` | Class name contains `AudioFocus`, `FocusRequester`, `MediaFocusControl` |
| `audio-policy` | Class name contains `AudioPolicy`, `AudioEffect`, `AudioProductStrategy` |
| `media-session` | Package `com.android.server.media` or class name contains `MediaSession` |
| `audio-track-record` | Class `AudioTrack`, `AudioRecord`, `AudioTimestamp` |
| `audio-routing` | Class name contains `AudioRouting`, `AudioDeviceInfo`, `AudioDevicePort` |
| `aidl-stubs` | File is an AIDL-generated stub (class ends in `.Stub` or `.Stub.Proxy`, or file ends in `AIDL.java`) |
| `misc` | Everything else |

### Step 3 — Size-limit splitting

After grouping by service boundary, split any group whose **total file size** exceeds `maxPartitionSizeKB` (from `jca-config.json`, default 512 KB) by further subdividing on sub-package boundaries.

### Step 4 — Assign partition IDs

Assign each partition a stable ID: `p<two-digit-index>-<group-id>` (e.g., `p01-audio-service`, `p02-audio-focus`).

### Step 5 — Write output

Write `concurrency_analysis/partitions.json`.

## Output: `concurrency_analysis/partitions.json`

```json
{
  "source_path": "<SOURCE_PATH>",
  "total_files": 0,
  "total_size_kb": 0,
  "partition_count": 0,
  "partitions": [
    {
      "id": "p01-audio-service",
      "group": "audio-service",
      "files": [
        "frameworks/base/services/core/java/com/android/server/audio/AudioService.java"
      ],
      "file_count": 0,
      "size_kb": 0
    }
  ]
}
```

## Mandatory Rules

- Every `.java` file under `SOURCE_PATH` must appear in exactly one partition.
- No file may be omitted.
- Never modify any source file.
- Write only to `concurrency_analysis/partitions.json`.
