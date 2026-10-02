---
type: llm
weight: 1
focus: last_message
---
Context: streamed PCM on an Android `AudioTrack` loses its tail because the code stops and releases right after the last `write`. `stop()` resets the playhead.

PASS if the fix, after the last write and BEFORE `stop()`, waits until `playbackHeadPosition` reaches the number of frames written, with a deadline (a timeout of a few seconds) so a stalled track cannot hang forever, and only then calls `stop()`, `release()` and `onFinished()`. Reading the head as unsigned (`toLong() and 0xFFFFFFFFL`) is expected but not required. An equally correct alternative: `setNotificationMarkerPosition(framesWritten)` with a playback position listener, bounded by a timeout, then stop, release and report finished.

FAIL if any of these is true:
- It waits after `stop()`.
- It uses a fixed sleep or delay that is not tied to the playhead or the frames written.
- It calls `flush()` before `stop()`.
- The wait has no deadline or timeout at all.
- It pads the stream with silence as the fix, or switches to `MODE_STATIC`.
- It adds harmful advice, such as catching `CancellationException` broadly.
