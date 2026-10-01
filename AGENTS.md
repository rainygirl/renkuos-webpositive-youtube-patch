# YouTube in WebPositive -- Development Notes

What the patch does and why, what was measured, and the traps. Build and use
are in [`README.md`](README.md). Measurements are from a Sony VAIO P
(Atom Z520, 1.33 GHz, one core with two hyperthreads, Intel GMA500, 2 GB)
on 2026-10-01, playing `https://www.youtube.com/watch?v=vOTiJkg1voo`.
Three patches, applied in this order: the x86 build's own
`haikuwebkit-1.9.19-x86.patch` (memory leaks), `haikuwebkit-1.9.19-youtube.patch`
(media, filters) and `haikuwebkit-1.9.19-x86-llint.patch` (JavaScriptCore).

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
for five-second segments. The first frame now arrives roughly three minutes
after the page starts loading (it was about 40 seconds in the first
measurements; YouTube's page has grown since, or the machine is slower --
the two builds below were measured side by side, so the comparison holds).

The page keeps WebPositive's main thread at 75-85% of a logical CPU for the
whole time. A profile of 125 s of it (Haiku's `profile`, 2 ms ticks):
JavaScriptCore 53%, WebCore 33%, of which 39% was one function (next
section). The JavaScript part is spread thin: property-access slow paths,
the parser and lexer (YouTube ships megabytes of script), garbage
collection. R Chromium on the same machine and stream has V8's JIT and
keeps up.

## An assembly interpreter for JavaScriptCore on 32-bit x86

`haikuwebkit-1.9.19-x86-llint.patch`. Upstream builds JavaScriptCore on
32-bit x86 with `ENABLE_C_LOOP`: the portable C++ interpreter, because the
x86-32 backend of offlineasm (the assembler the LLInt is written for) was
removed in 2024 (WebKit 8e3653b4aa). Reverting that removal does not help:
it had not assembled the LLInt since the 2019 bytecode format change, whose
32-bit `PB`/`metadataTable` code has `else error` for anything but ARMv7 and
the C loop.

So the backend is new, modelled on ARMv7, the other 32-bit LLInt:

- Registers: t0=eax, t1=edx, t2=ecx, t3=ebx, t4=esi (the bytecode PC, as
  `LLIntPCRanges` expects), t5=edi, cfr=ebp. a0=ecx and a1=edx, which is
  exactly fastcall, the convention JSC's host functions use on x86.
- x86 has no registers left for `PB` and `metadataTable`, which the LLInt
  keeps for a whole function. They live in the frame's callee-save slot
  (-4 and -8 from cfr; ARMv7 saves its csr0/csr1 there). A pre-pass in
  `x86.rb` rewrites every instruction that names one: a load through one
  borrows its own destination register, a compare or two-operand ALU
  instruction takes the slot as a memory operand, anything else gets a
  scratch register pushed and popped around it.
- Slow paths are cdecl: arguments pushed, 16-byte stack alignment kept at
  every call. `vmEntryToJavaScript` and friends take theirs off the stack.
- No JIT, no WebAssembly, no `MacroAssembler`: `ENABLE(ASSEMBLER)` stays off,
  and the few C++ places that assumed "not C loop" means "has an assembler"
  now ask for `ENABLE(ASSEMBLER)`.
- Global addresses (the opcode map, on every dispatch) are absolute
  immediates. Through the GOT, 32-bit x86 needs a call/pop pair to find its
  own address first; that cost 25% of a tight loop. The library gets
  `DT_TEXTREL`, which Haiku's runtime_loader applies.

Measured on the VAIO, `jsc` shell, ms (two runs each):

| | fib(25) | int loop | objects | strings | properties |
| --- | --- | --- | --- | --- | --- |
| C loop | 178, 173 | 986, 1026 | 1615, 1575 | 242, 251 | 835, 864 |
| LLInt, GOT dispatch | 165, 170 | 948, 944 | 1388, 1457 | 208, 239 | 758, 875 |
| LLInt | 157, 139 | 739, 657 | 1336, 1290 | 209, 178 | 749, 745 |

and inside WebPositive, one page: fib 274 -> 180, loop 994 -> 747, objects
1557 -> 1363, strings 277 -> 198, properties 950 -> 900. A quarter to a
third on interpreter-bound code, much less on code that lives in the
runtime.

On the YouTube stream, alternating builds, first frame after 240 and 190 s
(C loop) against 170 and 170 s (LLInt); segments received in four minutes
2 and 8 against 8 and 6. Better, inside the noise of a live page, and not
enough: an interpreter cannot close the distance to a JIT. Bringing back
JavaScriptCore's baseline JIT for 32-bit x86 is what that would take.

## A blurred background repainted for every video frame

The single most expensive function on that page's main thread was
`FEGaussianBlurSoftwareApplier::boxBlurUnaccelerated`: 12.7% of it. The
layer is YouTube's `player-container-background-image`, an 885x407 blurred
copy of the thumbnail behind the player. WebKitLegacy on Haiku paints
without compositing, so every repaint that touches the layer -- each video
frame, the progress bar, the clock -- painted the layer again and blurred it
again, 7-8 times a second, although it never changed.

`RenderLayerFilters` already learns about every change inside a layer whose
filter moves pixels (`setFilterBackendNeedsRepaintingInRect`), so the patch
keeps the filter's `FilterResults` between renders and draws the previous
output while that dirty rect stays empty. One more change was needed for it
to ever hit: the filter's box was the layer clipped to the paint's dirty
rect, so it changed with every different repaint nearby and the filter was
rebuilt each time. For a pixel-moving filter on a layer up to a million
pixels the box is now the whole layer. After: 0.46% of the main thread.

## Quitting during Media Source playback

WebPositive crashed on quit after a Media Source video had played, in the
sound player's node thread or in the decoding thread. Nothing tears a page
down before the application exits, and an `atexit` handler is too late:
Media Kit add-ons (the decoders) are loaded after it is registered, so their
static destructors run before it. `BWebPage::ShutdownOnce()`, which
WebPositive calls on quit, now stops every Media Source player's decoding
thread and deletes the sound players first; the `atexit` handler remains for
applications that do not call it. Two quits of the old build: two crashes;
of this one: none.

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
  the new window simply never appears. Those Debuggers also stay around and
  hold the crashed team's memory: seven of them had taken half a gigabyte
  and slowed every later measurement.
- **Measure two builds side by side, not against yesterday.** The same
  stream started in 40 s one day and in three minutes another. Loading the
  other build's libraries through `LIBRARY_PATH` avoids a package swap per
  run; on the x86 side of a hybrid, list the `x86` subdirectories
  explicitly (`/boot/system/lib/x86`, ...): a `LIBRARY_PATH` that names only
  the primary directories makes the loader fail without a message.
- **`-z text` is not set for libJavaScriptCore**, and must not be: the
  LLInt's absolute addresses are text relocations.
