---
name: kmp-streaming-audio
description: Only for apps that record or play raw PCM (calls, voice notes, dictation, voice AI). Platform traps of raw PCM capture and streaming playback in Kotlin Multiplatform, on Android (AudioTrack, AudioRecord, AudioManager focus, mode and routing) and iOS (AVAudioEngine, AVAudioPlayerNode, AVAudioConverter, AVAudioSession) through Kotlin/Native. Use when you play audio that is still arriving (streaming TTS, live PCM), record the mic as PCM, own the audio session or audio focus for a call, or when streamed audio loses its tail, the app dies on play, the first recording fails, the mic stops after something plays, audio stutters, or playback hangs after a headset change or a phone call. Triggers - AudioTrack, playbackHeadPosition, AudioRecord, audio focus, MODE_IN_COMMUNICATION, setCommunicationDevice, AVAudioEngine, AVAudioPlayerNode, AVAudioConverter, AVAudioSession, PlayAndRecord, route change, PCM16.
---

# Streaming audio in KMP

Only for apps that record or play raw PCM. Each rule is a behaviour of the Android or iOS audio
stack. Rules with a narrower condition name it first. The most expensive trap is first.

Seen on: Kotlin/Native 2.4.10, Compose Multiplatform 1.11.1, kotlinx-coroutines 1.10.1, Android
API 26 to 36, iOS 18 SDK.

## 1. AudioTrack.stop() resets the playhead, so wait for playout before you stop

- **Symptom:** The end of a streamed clip is cut. A "wait until played" loop after `stop()` never ends.
- **Cause:** After the last `write`, the device buffer still holds hundreds of ms of audio. If you
  report "finished" then, the caller releases the track and cuts the tail. `stop()` also resets
  `playbackHeadPosition` to 0.
- **Fix:** Count the frames you wrote. Poll the head until it reaches that count, with a deadline,
  then `stop()`. The head is an `Int` that wraps, so read it as unsigned.

```kotlin
val deadline = SystemClock.elapsedRealtime() + 3_000
while (SystemClock.elapsedRealtime() < deadline &&
    (track.playbackHeadPosition.toLong() and 0xFFFFFFFFL) < framesWritten) Thread.sleep(20)
track.stop()   // only now
```

- **Seen on:** Android API 26 to 36, `MODE_STREAM`.

## 2. play() on a stopped AVAudioEngine raises an ObjC exception that Kotlin cannot catch

- **Symptom:** The app dies on play. `runCatching` around the call does not help.
- **Cause:** `startAndReturnError` reports failure by returning `false` (for example, when another
  app holds a non-mixable session). `AVAudioPlayerNode.play()` on an engine that is not running raises
  an Objective-C exception, and Kotlin/Native ends the process unless the binary is built with
  `-Xforeign-exception-mode=objc-wrap`. The system can also stop a running engine later.
- **Fix:** Call `node.play()` only if `startAndReturnError` returned `true`. Before every later
  `play()`, check `engine.running`; if it is false, mark the player failed instead.
- **Seen on:** Kotlin/Native 2.4.10, iOS 18 SDK.

## 3. Get the record permission before you touch the input

- **Symptom:** On a fresh install the first recording fails, or the app crashes when it builds the
  converter. On Android an error shows behind the permission dialog.
- **Cause:** iOS: if the session activates while the prompt is open or refused, the input node
  reports 0 Hz. `AVAudioConverter` from that format returns nil, which reaches Kotlin as a crash.
  Android: the activity-result answer arrives later. Code that reads the permission in the same frame
  reads "denied".
- **Fix:** Suspend until the user answers, then open the input. On iOS, `check(format.sampleRate > 0.0)`
  before you build the converter. On Android, register the launcher in the Activity, wrap it in
  `suspendCancellableCoroutine`, and resume a pending waiter in `onDestroy`, because a configuration
  change drops the result. The same wrapper works for any runtime permission.
- **Seen on:** iOS 18 SDK (`AVAudioApplication`), AndroidX Activity Result API.

## 4. A system-stopped AVAudioEngine drops its buffers and still looks like it is playing

- **Symptom:** After a phone call or a headset change, nothing plays, the position freezes, and your
  "is playing" state stays true. Anything that waits for the end waits forever.
- **Cause:** On an interruption or a route change, iOS can stop the engine and drop every buffer
  scheduled on the node. Nothing tells you about the lost audio.
- **Fix:** On `AVAudioSessionInterruptionNotification` or a device route change during playback, stop
  your player on purpose and mark it failed. Do not revive it: the buffers are gone. Make `enqueue()`
  a no-op after `stop()`, so chunks still in flight cannot restart it.
- **Seen on:** iOS 18 SDK.

## 5. AVAudioPlayerNode: use DataPlayedBack, and expect completions from stop()

- **Symptom:** "Finished" fires a render cycle early, or for audio you stopped.
- **Cause:** The default `scheduleBuffer` completion fires when the node takes the buffer, not when it
  is heard. `stop()` calls the completion of every buffer it discards.
- **Fix:** Pass `completionCallbackType = AVAudioPlayerNodeCompletionDataPlayedBack`. Keep an atomic
  count of pending buffers. Report the end only when input has ended AND the count is 0 AND you did
  not stop the node. Run that check at end of input and in the completion, because either can be last.
  The completion runs on an engine queue, so shared flags need `@Volatile`.
- **Seen on:** iOS 18 SDK.

## 6. The input node taps only its hardware format; resample with the block form of AVAudioConverter

- **Symptom:** A tap asked for 16 kHz fails, or `convertToBuffer(_:from:)` fails for a rate change.
- **Cause:** The input node gives its hardware format only (48 kHz float on the phone, another format
  on a Bluetooth headset). The simple `convertToBuffer(_:from:)` cannot change the sample rate.
- **Fix:** Tap with `inputFormatForBus(0u)`. Build a new converter on each start from the current
  format. Size the output as frames times the rate ratio, plus slack. Give the buffer once, then
  answer `NoDataNow`. For playback, schedule the engine's standard float format, so the mixer needs
  no converter node.

```kotlin
var supplied = false
converter.convertToBuffer(out, null) { _, status ->
    if (supplied) { status?.pointed?.value = AVAudioConverterInputStatus_NoDataNow; null }
    else { supplied = true; status?.pointed?.value = AVAudioConverterInputStatus_HaveData; tapBuffer }
}
```

- **Seen on:** iOS 18 SDK, Kotlin/Native 2.4.10 cinterop.

## 7. If one feature records and plays, take the AVAudioSession once, for the whole feature

- **Symptom:** Audio jumps between earpiece and speaker. Other apps duck again and again. The mic
  stops after a sound plays. A recording is cancelled by a route change you caused.
- **Cause:** Each activation can move the route and duck other audio. `PlayAndRecord` without
  `DefaultToSpeaker` plays on the earpiece. A player that sets `Playback` closes the input. Your own
  category change or activation also posts route change notifications.
- **Fix:** Set `PlayAndRecord` with `DefaultToSpeaker | AllowBluetooth` once, and keep it until the
  feature ends. The iOS 26 SDK deprecates `AllowBluetooth` in favour of `AllowBluetoothHFP`
  (`AVAudioSessionCategoryOptionAllowBluetoothHFP`, same value); keep the old name on older SDKs.
  Players must not change the category while you own the session. Treat only
  `OldDeviceUnavailable` and `NewDeviceAvailable` as a lost device. On interruption `began`, mark the
  session inactive and activate it again before next use. Deactivate with `NotifyOthersOnDeactivation`.
- **Seen on:** iOS 18 SDK; the deprecation in the iOS 26.5 SDK headers.

## 8. If you run call-style audio on Android, focus, mode and route are global and routing is async

- **Symptom:** Other apps play over your call. A Bluetooth headset mic is not used. The first
  recording is cancelled as "device changed" right after start.
- **Cause:** Focus, `AudioManager.mode` and the communication device are shared.
  `setCommunicationDevice` (API 31+) is asynchronous and can report a null route before it settles.
  `registerAudioDeviceCallback` first reports the devices already connected, like a headset event.
- **Fix:** Request `AUDIOFOCUS_GAIN_TRANSIENT` with `USAGE_VOICE_COMMUNICATION`; any focus change
  except GAIN is a loss. Save the old mode, then set `MODE_IN_COMMUNICATION` and
  `volumeControlStream = STREAM_VOICE_CALL`. Snapshot the devices before you register and treat only a
  difference as a change. Ignore route callbacks until the requested route arrives, with a deadline.
  Record from `VOICE_COMMUNICATION`, play with `USAGE_VOICE_COMMUNICATION`. Below API 31, use
  `isSpeakerphoneOn` when no wired device is connected. Restore everything on release (rule 11).
- **Seen on:** Android API 26 to 36; `setCommunicationDevice` and its listener need API 31.

## 9. If you stream into AudioTrack on API 31+, own the start threshold and rebuffer from frames left

- **Symptom:** Streamed audio stutters. After a pause for data, the short last piece never plays.
- **Cause:** On API 31+, a paused AudioTrack does not restart until its start threshold is filled, so a
  short tail stays in the buffer. A count of bytes received is not the audio left to play.
- **Fix:** On API 31+, set the threshold to 1 frame and own the cushions. Left = frames written minus
  the playhead. Under about 40 ms left, with input still coming: `pause()`, wait for a cushion that
  grows each time (400 ms, 800 ms, cap 1.2 s), then `play()`. After input ends, play every sample.
  Below API 31, do not pause; keep writing. Wait for a pre-buffer before the first `play()`: a bursty
  producer at about real time needs about 2 s.

```kotlin
if (Build.VERSION.SDK_INT >= 31) track.setStartThresholdInFrames(1)
val leftMs = (framesWritten - (track.playbackHeadPosition.toLong() and 0xFFFFFFFFL))
    .coerceAtLeast(0) * 1000 / sampleRate
```

- **Seen on:** Android API 31 to 36.

## 10. AudioTrack.write and AudioRecord.read block: design the threads around it

- **Symptom:** A writer stays blocked after stop, the UI freezes on stop, frames get overwritten, or
  captured audio has gaps while the UI animates.
- **Cause:** In `MODE_STREAM`, `write` blocks while the device buffer is full. `flush()` wakes it, and
  it tries to write the rest of its chunk and blocks again. `AudioRecord.read` blocks until the buffer
  is full, so the frame length is also your stop latency. The `getMinBufferSize` size "doesn't
  guarantee a smooth recording under load" (its docs): a reader stalled behind the main thread
  overruns it, and the samples are lost with no error.
- **Fix:** One writer thread that polls a queue with a timeout. Set a `stopped` flag before `pause()`
  and `flush()`, and check it in the inner write loop. For capture, open `AudioRecord` on the calling
  thread so a refusal throws where you can report it, with a buffer of about 10 frames (400 ms,
  floored at the minimum); `read` still returns per frame, so it adds no latency. Call
  `Process.setThreadPriority(THREAD_PRIORITY_URGENT_AUDIO)` first in the reader thread, read about
  40 ms frames, use a new `ByteArray` per frame if the consumer queues it, stop with a flag that
  belongs to that run, and never `join()` the reader on the main thread.
- **Seen on:** Android API 26 to 36.

## 11. If audio sessions can overlap, give shared state an owner token and a teardown that finishes

- **Symptom:** Recording stops or the audio mode reverts during a new session, or the input stays busy.
- **Cause:** The iOS session and the Android mode, focus and route belong to the whole process. A
  teardown in a launched coroutine can arrive after the next session started. A network close on a
  dead link can suspend forever, and the calling job may already be cancelled.
- **Fix:** Store the owner of each shared resource and ignore a release from any other owner. A player
  gives back only a session that it took, on `stop()` as well as `release()`. Run cleanup in
  `NonCancellable`, put a time limit on each network step, and free native audio in `finally`.

```kotlin
fun deactivate(owner: Any) { if (holder !== owner) return; holder = null /* setActive(false) */ }
suspend fun close() = withContext(NonCancellable) {
    try { withTimeoutOrNull(750) { runCatching { socket.close() } } }
    finally { socket.cancel(); recorder.release(); player.release() }
}
```

- **Seen on:** iOS 18 SDK, Android API 26 to 36, kotlinx-coroutines 1.10.1, Ktor 3.1.0. See
  kmp-ktor-networking: WebSocket timeouts and auth on the upgrade.

## 12. If you send speech to speech-to-text, detect empty input from the audio, not the confidence

- **Symptom:** A very short or silent recording comes back as a real word with high confidence.
- **Cause:** On Android the first samples arrive some milliseconds after `startRecording()`, so a
  very short capture can be all zeros (a 100 to 250 ms recording can yield 0 to 40 ms of zeros).
  Some speech-to-text services return a common word for silent or near-silent audio, and the
  confidence score does not show it.
- **Fix:** Measure the capture. Under about 100 ms, or a loudest 20 ms window at digital silence,
  means no input: do not send it for recognition. Do not use a minimum recording length instead,
  because it drops one-word utterances.
- **Seen on:** Android `AudioRecord`, `VOICE_COMMUNICATION` source, targetSdk 36.

## Checklist

- [ ] AudioTrack: wait for the playhead to reach the written frames, then `stop()`.
- [ ] AVAudioEngine: `startAndReturnError` and `engine.running` before every `play()`. Permission first.
- [ ] iOS interruption or route change: stop the player on purpose. No `enqueue()` after `stop()`.
- [ ] `DataPlayedBack` and a buffer count. Hardware-format tap, block-form converter per start.
- [ ] One AVAudioSession owner per feature. Android: focus, mode and route per session.
- [ ] API 31+: `setStartThresholdInFrames(1)`, rebuffer from written minus played.
- [ ] `AudioRecord` buffer well above the minimum; reader thread at `THREAD_PRIORITY_URGENT_AUDIO`.
- [ ] Owner token on shared audio state. `NonCancellable`, time-limited teardown.
- [ ] Empty input is judged by captured length and energy.
