---
type: llm
weight: 2
focus: last_message
---
Context: an Android `AudioTrack` in `MODE_STREAM` writes streamed PCM chunks, then calls `stop()`, `release()` and `onFinished()` right after the last `write`. The last word of each clip is cut. A wait loop placed after `stop()` that compared `playbackHeadPosition` with `framesWritten` never ended. A bigger buffer made the cut longer.

PASS if the answer states both:
- When the last `write` returns, that audio is only queued. The track and device buffer still hold hundreds of ms that have not played. Stopping, releasing and reporting "finished" at that moment throws the tail away (a bigger buffer holds more, so more is cut).
- `stop()` resets `playbackHeadPosition` to 0, so a wait loop placed after `stop()` compares against a head that restarted from 0 and never reaches `framesWritten`.

FAIL if any of these is true:
- It blames the server, the socket, a lost last chunk, or an odd byte count or partial frame.
- It says the buffer is too small, or recommends `flush()` (which discards queued audio).
- It says to use `MODE_STATIC` or `WRITE_NON_BLOCKING` as the explanation.
- It explains the loop only by integer overflow, wrong units or a missing `play()`, without the playhead reset by `stop()`.
- It names only one of the two facts above.
