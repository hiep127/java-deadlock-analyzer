# AOSP Audio Framework — Known Edge Cases & Concurrency Hazards

This document catalogues known or historically observed concurrency hazard patterns specific to the Android Audio Framework (`frameworks/base/services/core/java/com/android/server/audio/` and related paths). JCA detector agents must actively search for these patterns.

---

## 1. AudioService Lock Architecture

### 1.1 Primary Lock: `mLock`

`AudioService` uses a single primary lock (`mLock`, type `Object`) to protect the majority of its state. Key state protected by `mLock`:

- `mStreamStates[]` — per-stream volume state array
- `mRingerMode`, `mRingerModeExternal` — ringer/silent mode
- `mAudioMode` — audio mode (normal, in-call, in-communication, ringtone)
- `mForcedUseForComm`, `mForcedUseForCommExt`
- `mMuteAffectedStreams`, `mPersistSafeVolumeState`

**Hazard:** Any code that reads these fields outside `synchronized(mLock)` is a race.

### 1.2 Audio Focus Lock: `mFocusLock` (in `MediaFocusControl`)

`MediaFocusControl` maintains `mFocusStack` (the audio focus request stack) protected by its own lock `mFocusLock`. `AudioService` calls into `MediaFocusControl` from within `synchronized(mLock)`.

**Known inversion pattern:** If any code path acquires `mFocusLock` first and then attempts to call into `AudioService` (acquiring `mLock`), a deadlock cycle is formed.

### 1.3 DeviceBroker Lock

`AudioDeviceBroker` has its own `mDeviceBrokerLock`. `AudioService` holds `mLock` and calls `mDeviceBroker.setBluetoothA2dpOnInt()` and similar methods that acquire `mDeviceBrokerLock`.

**Rule:** `mLock` must always be acquired before `mDeviceBrokerLock`. Inversion = deadlock.

---

## 2. Audio Focus Callback Re-Entrancy

### 2.1 Focus Change Dispatch Under Lock

`MediaFocusControl.notifyTopOfAudioFocusStack()` and `dispatchAudioFocusChange()` iterate over `mFocusStack` and call `IAudioFocusDispatcher.dispatchAudioFocusChange()` on each registered client. These callbacks are synchronous Binder calls.

**Hazard:** These dispatches are sometimes made while `mLock` is held in `AudioService`. The client receiving the callback may call `AudioManager.requestAudioFocus()` or `abandonAudioFocus()`, which goes through Binder back into `AudioService`, where it waits for `mLock` → deadlock.

**Signature to detect:**
```java
// Inside synchronized(mLock) in AudioService:
mMediaFocusControl.requestAudioFocus(...);  // triggers focus change notifications
// OR
synchronized (mLock) {
    // direct focus dispatch
    focusDispatcher.dispatchAudioFocusChange(...);  // DANGEROUS
}
```

### 2.2 Focus Stack Iteration During Concurrent Modification

`mFocusStack` (a `LinkedList` or `ArrayDeque`) is iterated in `dispatchAudioFocusChange()` while a Binder thread may concurrently call `abandonAudioFocus()` which removes from `mFocusStack`. Without consistent locking, this causes `ConcurrentModificationException` at runtime (itself a symptom of a race).

---

## 3. AudioSystem JNI Blocking Calls

### 3.1 Calls That Cross the JNI Boundary

The following static `AudioSystem` methods make JNI calls into the native `AudioFlinger` or `AudioPolicyManager`, which may block on native mutexes (C++ `std::mutex` or `android::Mutex`):

```java
AudioSystem.setParameters(String keyValuePairs)
AudioSystem.getParameters(String keys)
AudioSystem.setStreamVolumeIndex(int stream, int index, int device)
AudioSystem.getStreamVolumeIndex(int stream, int device)
AudioSystem.setDeviceConnectionState(int device, int state, String deviceAddress, String deviceName, int codecFormat)
AudioSystem.getDeviceConnectionState(int device, String deviceAddress)
AudioSystem.setPhoneState(int state)
AudioSystem.setForceUse(int usage, int config)
```

**Hazard:** If any of these are called inside `synchronized(mLock)` in `AudioService`, the thread holds the Java lock while a native mutex is acquired. This creates a "hidden" lock acquisition invisible to Java static analysis. A native-side deadlock cycle can involve the Java `mLock` even though the Java code never directly acquires the native mutex.

**Detection instruction:** Flag any `AudioSystem.*()` call that appears inside a `synchronized` block, regardless of what other detectors report.

---

## 4. Audio Handler (`mAudioHandler`) Patterns

### 4.1 Handler Thread vs. Binder Thread State Races

`AudioService` uses `mAudioHandler` (running on its own `HandlerThread`) for deferred state persistence and some state mutations. Patterns where both the Binder thread and the handler thread access the same state:

- `mStreamStates[stream].setIndex(index, device)` can be called from both `mAudioHandler.handleMessage(MSG_SET_DEVICE_VOLUME)` and directly from `setStreamVolume()` on the Binder thread. If the Binder-thread path is not consistently guarded by `mLock` and the handler path is also not guarded, there is a race.

### 4.2 Volume Persistence Race

`MSG_PERSIST_VOLUME` is posted to `mAudioHandler` to write volume settings to `Settings.System`. If `mStreamStates[stream]` is mutated between the time the message is posted and the time the handler processes it (without the handler re-reading from `mStreamStates` under `mLock`), the persisted value may be stale.

### 4.3 `MSG_AUDIO_SERVER_DIED` Reinitialisation

When `AudioFlinger`/`AudioPolicyService` crashes and is restarted, `AudioService` receives `MSG_AUDIO_SERVER_DIED` and re-sends all state. During the window between crash detection and re-initialization, Binder calls to `AudioSystem` will fail or return stale data. Code that does not check return values from `AudioSystem.*()` during this window may corrupt state.

---

## 5. AudioRecord / AudioTrack State Machine

### 5.1 Native Callback Thread vs. Java API Thread

`AudioTrack` and `AudioRecord` use a native callback thread (`AudioTrackThread` / `AudioRecordThread`) to deliver `OnPlaybackPositionUpdateListener`, `OnRecordPositionUpdateListener`, and periodic callbacks. These callbacks run on the native thread, not the Java thread that created the `AudioTrack`/`AudioRecord`.

**Hazard:** Callback listener code that accesses `AudioTrack`/`AudioRecord` state fields (e.g., `mPlaybackHeadPosition`, `mState`) without synchronization while the Java API thread also modifies them is a race.

### 5.2 State Transitions

`AudioTrack` uses `mState` (an int field) to track its lifecycle (STATE_UNINITIALIZED → STATE_INITIALIZED → ...). If `stop()` and `flush()` are called from different threads without synchronization, the state machine can enter an invalid state.

---

## 6. MediaSession ↔ AudioService Interaction

### 6.1 Callback Cycle

`MediaSessionService` calls into `AudioService` for volume key handling, and `AudioService` calls into `MediaSessionService` for active media session queries. If either service holds its primary lock while calling into the other, a cross-service lock inversion exists.

**Known path:**
- `AudioService.dispatchMediaKeyEvent()` may call `mMediaSessionService.dispatchMediaKeyEvent()` while holding `mLock`.
- `MediaSessionService.notifyActiveSessionsChanged()` may call `AudioService.setStreamVolume()` indirectly via a volume callback.

### 6.2 `IMediaSessionService.Stub` Callbacks Under Lock

Any call to an `IMediaSessionService` AIDL proxy from within `AudioService.synchronized(mLock)` is a potential deadlock entry point.

---

## 7. AppOps / Permission Check Patterns

`AppOpsManager.noteOp()` and `PermissionManager.checkPermission()` are Binder calls. They must **never** be called while holding `mLock` in `AudioService`, as the AppOps and Permission services may call back into `AudioService`.

**Detection signature:**
```java
synchronized (mLock) {
    mAppOps.noteOp(...);           // DANGEROUS — Binder call under lock
    checkCallingPermission(...);   // May be a Binder call under lock
}
```

---

## 8. `linkToDeath` / `DeathRecipient` Race Windows

### 8.1 Focus Owner Binder Death

When a client that holds audio focus dies, `AudioService.AudioFocusDeathHandler.binderDied()` is called on a Binder thread. This method modifies `mFocusStack` in `MediaFocusControl`. If the main AudioService thread is also modifying `mFocusStack` without acquiring `mFocusLock`, a race occurs.

### 8.2 Registration Gap

Pattern to detect:
```java
IBinder binder = client.asBinder();
if (binder != null) {                    // Check outside lock
    // ... (gap where binder could die)
    binder.linkToDeath(handler, 0);      // Registration outside lock
}
```

The correct pattern acquires the lock around both the null check and `linkToDeath`.
