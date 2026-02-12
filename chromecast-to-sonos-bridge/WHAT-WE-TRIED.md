# Everything We Tried (And Why It Failed)

This documents the complete journey to the working solution. Useful if you're troubleshooting a similar setup or considering alternative approaches.

## Approach 1: Snapcast + Music Assistant

**Idea:** Snapcast server on Pi, Music Assistant, Sonos native protocol

**What happened:**
- Installed Snapcast 0.31.0 from Debian repos
- Music Assistant VBAN Receiver showed connection errors
- Home Assistant Snapcast integration showed protocol errors
- Error: `unknown message type received: Unknown, size: 1919243042`

**Root cause:** Version incompatibility between Snapcast 0.31.0 and Music Assistant integrations

**Why we couldn't fix it:** Older Snapcast versions (0.27.0) unavailable as ARM64 packages on any repository. GitHub releases had been moved/removed.

---

## Approach 2: VBAN + Music Assistant

**Idea:** Pi sends VBAN (audio-over-UDP), Music Assistant VBAN Receiver, Sonos

**What happened:**
- Built VBAN from source (github.com/quiniouben/vban)
- `vban_emitter` successfully sent audio to Home Assistant IP
- Music Assistant VBAN Receiver received the stream
- BUT: VBAN Receiver creates an audio *source*, not a *player*
- Sonos speakers couldn't be set as output for the VBAN source

**Root cause:** Music Assistant's VBAN Receiver is designed backwards for our use case. It expects to be a source for Chromecast/AirPlay devices, not a bridge to Sonos.

---

## Approach 3: Icecast/Darkice HTTP Stream (Works, high latency)

**Idea:** Darkice encodes USB audio, Icecast HTTP server, Sonos plays HTTP stream

**What happened:**
- Installed Darkice + Icecast2
- Connected Darkice to Icecast (127.0.0.1 fixed a connection refused error)
- Sonos successfully played `http://<PI_IP>:8000/chromecast.mp3`
- **Result: Works! But 6-10 second latency**

**Why the latency:** Sonos treats HTTP streams as "internet radio" and buffers heavily for stability. This buffering is hardcoded in Sonos firmware and cannot be changed.

**Optimization attempts:**
- Reduced `burst-size` from 65535 to 12288 to 6144 bytes (saved ~2-3 seconds)
- Reduced bitrate from 320 to 128 kbps (minimal improvement)
- Reduced `bufferSecs` in Darkice (caused audio breakup below 1 second)
- **Final result: ~6 second latency** (down from 9-10 seconds)

**Still not good enough:** 6 seconds is too long when you want to pause for a phone call.

---

## Approach 4: AirPlay via PulseAudio

**Idea:** PulseAudio loopback: USB SPDIF input, AirPlay output, Sonos

**What happened:**
- Installed PulseAudio with `module-raop-sink`
- All Sonos speakers discovered via `pactl list sinks short`
- Created loopback: `pactl load-module module-loopback source=... sink=raop_output.Sonos...`
- Test with `paplay`: "Failed to drain stream: Timeout"
- Logs showed: `RTSP/1.0 403 Forbidden`

**Root cause:** Sonos requires AirPlay 2 with authentication. PulseAudio's RAOP module only supports legacy AirPlay (RAOP) without authentication. The 403 Forbidden response is Sonos rejecting the unauthenticated connection.

---

## Approach 5: Soundsync

**Idea:** Use Soundsync (github.com/geekuillaume/soundsync) for low-latency multi-room audio

**What happened:**
- npm package `soundsync` was a completely different package (Replit audio tool)
- Downloaded actual binary from GitHub releases -- no ARM64 build available
- Downloaded ARMv7l .deb package
- Failed with unresolvable dependencies: `gconf2`, `libappindicator1`, `libxtst6` etc.
- These are old GTK2 dependencies that don't exist in Debian Trixie

**Root cause:** Soundsync v0.4.16 (last release 2021) has old GUI dependencies incompatible with modern Debian. Project appears unmaintained.

---

## Approach 6: OwnTone with Icecast Stream (Works, same latency)

**Idea:** OwnTone plays Icecast stream, sends to Sonos via AirPlay 2

**What happened:**
- OwnTone successfully discovered all Sonos speakers
- Added Icecast stream URL as radio station in OwnTone
- OwnTone played the stream to Sonos via AirPlay 2
- **But: Same 6 second latency** (Icecast buffering still present)

**Lesson learned:** Adding OwnTone on top of Icecast doesn't reduce latency. The bottleneck is Icecast buffering, not the AirPlay protocol.

---

## Approach 7: OwnTone Pipe Input (WORKING SOLUTION)

**Idea:** FFmpeg writes to named pipe, OwnTone reads pipe, AirPlay 2, Sonos

**Initial problems:**
1. Metadata file was a regular file, not a FIFO
   - Error: `Source type is pipe, but path is not a fifo: /srv/music/livestream.wav.metadata`
   - **Fix:** Create `.metadata` file as a named pipe too, with continuous writing loop
2. Audio dropouts/skips
   - Cause: ALSA buffer underruns on Pi 3 under CPU load
   - **Fix:** `chrt -f 10` real-time scheduling + `thread_queue_size 16384`
3. "Device busy" errors from accumulated stopped FFmpeg processes
   - **Fix:** `killall ffmpeg` at script start

**Final result:** Near-instant latency, no skips, stable operation

**Key insight:** OwnTone's AirPlay 2 implementation handles Sonos authentication correctly, unlike PulseAudio's RAOP module. This is why OwnTone succeeds where PulseAudio fails.
