# YouTube in WebPositive -- Development Notes

What the patch does and why, what was measured, and the traps. Build and use
are in [`README.md`](README.md). Measurements are from a Sony VAIO P
(Atom Z520, 1.33 GHz, one core with two hyperthreads, Intel GMA500, 2 GB)
on 2026-10-01, playing `https://www.youtube.com/watch?v=vOTiJkg1voo`.

## Why Media Source, and why not a different user agent

YouTube's desktop site plays everything through Media Source Extensions.
HaikuWebKit 1.9.19 is built with `ENABLE_MEDIA_SOURCE` off, so for an
ordinary video YouTube falls back to one progressive 360p MP4 (itag 18), and
a live stream never starts: the desktop page asks
`canPlayType("application/x-mpegURL")` a dozen times but its player data
carries no `hlsManifestUrl`, so there is nothing to give native HLS.

YouTube's iPhone site does use HLS, and with an iPhone user agent the live
stream played through this patch's HLS reader. That approach was dropped on
purpose: WebPositive should say what it is. (For the record: with an iOS 16
user agent YouTube sends ordinary videos as HLS with DRM parameters and
refuses to play them; iOS 12 gets HLS for live and itag 18 otherwise. Its HLS
segments answer 403 to a plain fetch -- they need the PO token the page's own
script adds.)

## Design

All new code is in `WebCore/platform/graphics/haiku/`.

- `SourceBufferPrivateHaiku` parses appended bytes as fragmented MP4: `moov`
  (tkhd, mdhd, hdlr, `avc1`/`avc3` + avcC, `mp4a` + esds -> AudioSpecificConfig,
  trex defaults) into an initialization segment, and each `moof` with the
  `mdat` after it (tfhd, tfdt, trun with every optional field) into samples.
  A moof is only consumed once its mdat is complete. WebCore's
  `SourceBufferPrivate` does the rest: buffered ranges, timestamp offsets,
  eviction, and the seek arithmetic.
- Track IDs are `(source buffer index << 32) | mp4 track_ID`: YouTube's audio
  and video source buffers both use MP4 track 1.
- `MediaPlayerPrivateHaikuMSE` keeps a queue per track. WebCore hands samples
  over in decode order through `enqueueSample()` while `isReadyForMoreSamples()`
  says yes (60 video, 120 audio); when the decoder drains a queue below half,
  the player calls `provideMediaData()` on the main thread.
- A decoding thread runs each track through a `BMediaDecoder` subclass that
  takes chunks from the queue. The format is the one the Media Kit's ffmpeg
  reader registers (`B_MISC_FORMAT_FAMILY`, `'ffmp'`, AVCodecID 27 / 86018),
  so decoder selection is the Media Kit's own: `msvdx_h264` (GMA500) when
  installed, the ffmpeg plugin otherwise. Measured: "H.264 on the GMA500
  video decoder" and "AAC (Advanced Audio Coding)".
- Audio is decoded about half a second ahead into a float ring that a
  `BSoundPlayer` drains; the clock is the presentation time of the next audio
  frame the sound player takes. Video frames are decoded up to three ahead
  into BBitmaps and shown when the clock reaches them; frames decoded too late
  count as dropped in `getVideoPlaybackQuality()`, which YouTube's quality
  selection reads.
- `BMediaDecoder` has no flush, so a flush (seek, quality switch) deletes the
  decoder and makes a new one. A video decoder is not called with fewer than
  six queued samples: an H.264 decoder that runs out of input in the middle of
  its reorder window drains itself and has to be restarted.
- A flush outside a seek keeps the clock and moves the "not before" floor to
  the current time, so a quality switch neither jumps the clock back to the
  last seek nor counts the re-decoded frames before now as dropped.

## Measured

While data is buffered, playback is clean: 30 frames every second, 0 dropped,
half a second of decoded audio ahead, video decoded on the GMA500.

```
status: clock 46790.233 playing, queues v50 a88, ring 0.50 s, frames 3, shown 6 dropped 0
status: clock 46792.253 playing, queues v35 a85, ring 0.49 s, frames 3, shown 67 dropped 0
status: clock 46794.263 playing, queues v40 a110, ring 0.42 s, frames 0, shown 121 dropped 0
status: clock 46796.263 playing, queues v26 a89, ring 0.50 s, frames 3, shown 187 dropped 0
status: clock 46798.263 playing, queues v21 a53, ring 0.51 s, frames 3, shown 247 dropped 0
status: clock 46799.979 playing, queues v5 a1, ring 0.00 s, frames 0, shown 294 dropped 0
```

and then it stops at the end of what YouTube has appended.

## What is still wrong: YouTube's script is too slow to feed it

YouTube's desktop player streams with SABR: POST requests to
`googlevideo.com/videoplayback` answered with `application/vnd.yt-ump`. A
loader log (`[net]` lines, since removed from the patch) shows each request
completing in 0.3-2 s, but the requests going out only every 8-30 seconds,
for five-second segments. In one five-minute run YouTube appended two
segment pairs. The first `play()` reached the player about 40 seconds after
the page loaded, and the first seek took 30 more, with its target already
buffered.

The reason is the JavaScript engine. JavaScriptCore on 32-bit x86 is built
`ENABLE_JIT=0`, `ENABLE_C_LOOP=1`: upstream removed both the x86-32 JIT and
the x86-32 backend of the assembly interpreter (`offlineasm/backends.rb`
lists X86_64, ARMv7, ARM64, ARM64E, RISCV64 and C_LOOP only). So every script
runs in the portable C interpreter, the slowest tier there is, and YouTube's
player is a very large script on a 1.33 GHz in-order Atom. WebPositive's main
thread sat at 40-44% of the machine (most of a core) throughout. R Chromium
on the same machine and stream has V8's JIT and keeps up.

So the media side is done and the remaining limit is JavaScriptCore. The
options, roughly in order of cost: build JSC's C loop with better flags
(computed gotos, `-O3`, `-march=atom`); or bring back an x86-32 LLInt backend,
which upstream removed and which is a large piece of work.

## Traps

- **CMake caches `ENABLE_MEDIA_SOURCE`.** Changing the port default in
  `OptionsHaiku.cmake` does nothing to an existing build directory, which
  keeps `ENABLE_MEDIA_SOURCE:BOOL=OFF`; every MSE file then compiles to
  nothing, without an error. The symptom is a page where `typeof
  MediaSource` is `"undefined"`. Set it in the cache, or pass
  `-DENABLE_MEDIA_SOURCE=ON`.
- **The old engine's MSE `load()` had a stale signature** (`load(const
  String&, MediaSourcePrivateClient*)`); with MSE on it no longer overrides
  anything. It is now the current `load(const URL&, const ContentType&,
  MediaSourcePrivateClient&)`, refusing, and the old engine says no to
  `isMediaSource` queries so WebCore picks the MSE engine.
- `makeUnique<T>` needs a fast-allocated `T`; `makeString` needs `_s`
  literals, not `const char*`.
- **YouTube probes `canPlayType` with codecs that do not exist** (`av99`) and
  with AV1; answering "probably" by container alone made it refuse to play.
- **WebPositive restores its last session**, so a test that launches it
  repeatedly accumulates tabs, and every tab with a video holds a player --
  only one of which gets the GMA500. Clear the session between test runs.
- **WebPositive is single-launch.** A crashed instance held by a
  `Debugger --save-report` that never finishes swallows every later launch;
  the new window simply never appears.
