---
name: audiotrack-tail
tags: [kmp-streaming-audio]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "Streamed PCM loses its last words on Android because the code stops, releases and reports finished right after the last write; a wait loop after stop() never ends because stop() resets the playhead."
expected_outcome: "Explains that audio is still buffered when the last write returns and that stop() resets playbackHeadPosition to 0, and waits for the playhead (unsigned) to reach the written frames with a deadline before stop(), release() and onFinished."
---
Our city guide app plays narration that our server synthesizes and streams as raw PCM (16-bit mono, 24 kHz). On iOS every clip plays to the end. On Android the last word or two of almost every clip is missing: "Turn left at the old..." and the clip ends. The server logs show the full audio was sent, and the bytes we write equal the bytes we receive.

```kotlin
class NarrationPlayer(private val onFinished: () -> Unit) {
    private val track = AudioTrack.Builder()
        .setAudioAttributes(AudioAttributes.Builder().setUsage(AudioAttributes.USAGE_MEDIA).build())
        .setAudioFormat(AudioFormat.Builder().setSampleRate(24_000)
            .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
            .setChannelMask(AudioFormat.CHANNEL_OUT_MONO).build())
        .setTransferMode(AudioTrack.MODE_STREAM)
        .build()

    suspend fun play(chunks: ReceiveChannel<ByteArray>) = withContext(Dispatchers.IO) {
        track.play()
        var framesWritten = 0L
        for (chunk in chunks) framesWritten += track.write(chunk, 0, chunk.size) / 2
        track.stop()
        track.release()
        onFinished()   // the UI moves on, and the next clip gets a new player
    }
}
```

I tried waiting after `stop()` until `track.playbackHeadPosition` reaches `framesWritten`, with a short delay loop. That loop never ends, so I removed it. I also tried a bigger buffer size: the cut got longer, not shorter. Android 14, Pixel 8, Kotlin 2.4.10, CMP 1.11.1. What is going on, and what is the correct way to finish a streamed clip?
