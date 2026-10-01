# YouTube in WebPositive: Media Source Extensions for HaikuWebKit

For Korean, see [`README.ko.md`](README.ko.md).

A patch for HaikuWebKit 1.9.19 that adds **Media Source Extensions** (MSE),
the API YouTube, and most other video sites, use to stream video. Without it
WebPositive cannot play a YouTube live stream at all, and plays other videos
only through a single 360p fallback file. WebPositive's user agent is left as
it is.

It targets 32-bit Haiku (x86_gcc2 hybrid) and was developed and measured on
a Sony VAIO P (Atom Z520, Intel GMA500).

## What the patch does

- **Media Source Extensions** (`ENABLE_MEDIA_SOURCE`): a SourceBuffer for
  fragmented MP4 with H.264 video and AAC audio, and a media engine that
  decodes it through the Media Kit -- the GMA500's hardware H.264 decoder
  where there is one ([haiku-webpositive-msvdx](https://github.com/rainygirl/haiku-webpositive-msvdx)),
  the ffmpeg plugin otherwise -- with the video timed against the audio and
  dropped frames reported to the page. WebM/VP9/AV1 are refused, so sites
  send H.264.
- **HaikuWebKit's media engine registered at all** -- it never was, so no
  `<video>` or `<audio>` played -- and a playback clock that counts played
  audio instead of trusting decoder timestamps.
- **HTTP Live Streaming** for `<video src="...m3u8">`.
- Media is opened **off the main thread** (opening used to freeze the page
  for seconds).
- `canPlayType()` answers **per codec** (it said yes to codecs that do not
  exist).
- **No crash on quitting** WebPositive during or after playback.
- A CSS filter's output is **reused while its layer is unchanged**, instead of
  blurring the layer again for every repaint near it. YouTube's blurred
  backdrop behind the player cost 12.7% of the main thread; now 0.5%.
- **An assembly interpreter for JavaScriptCore** on 32-bit x86
  (`haikuwebkit-1.9.19-x86-llint.patch`), where upstream only has the
  portable C loop: a new x86-32 backend for offlineasm. Interpreted
  JavaScript runs a quarter to a third faster.

## Status

On the VAIO P, with the original user agent:

- `https://www.youtube.com/watch?v=vOTiJkg1voo` (ABC NEWS Australia live)
  **plays** through MSE: 640x360 H.264 decoded on the GMA500, AAC audio,
  30 fps with **0 frames dropped** while data is buffered.
- It does **not yet play continuously.** YouTube's player fetches the next
  segment only every 8-30 seconds, so it plays a few seconds and waits, and
  the first frame takes about three minutes. The requests themselves complete
  in 0.3-2 s; what is slow is YouTube's JavaScript deciding when to send
  them. The assembly interpreter helps (first frame 170 s against 190-240 s
  with the C loop) but an interpreter is not a JIT, and upstream has none
  for 32-bit x86. See [`AGENTS.md`](AGENTS.md).
- Ordinary YouTube videos keep playing as before.

## Building

`webkit/build-haikuwebkit-x86.sh` cross-builds `haikuwebkit_x86` 1.9.19-22
on Linux. It runs
[haiku-rwebpositive-arm64](https://github.com/rainygirl/haiku-rwebpositive-arm64)'s
x86 build (memory-leak fixes) with this repository's two patches applied on
top; set `RWP` to that checkout.

```sh
RWP=/path/to/haiku-rwebpositive-arm64 webkit/build-haikuwebkit-x86.sh
```

Or apply the patches by hand to a HaikuWebKit 1.9.19 tree that already has
`haikuwebkit-1.9.19-x86.patch` applied:

```sh
patch -p0 < webkit/haikuwebkit-1.9.19-youtube.patch
patch -p0 < webkit/haikuwebkit-1.9.19-x86-llint.patch
```

If the build directory already exists, set `ENABLE_MEDIA_SOURCE` there too
(`-DENABLE_MEDIA_SOURCE=ON`), and `-DENABLE_C_LOOP=OFF`: CMake keeps the old
cached values, and the MSE code compiles to nothing while the C loop stays.
The LLInt build needs Ruby (offlineasm), as every WebKit build does.

## Debugging

`WEBKIT_MEDIA_DEBUG=1` in WebPositive's environment prints, to stderr, every
`canPlayType`/`isTypeSupported` answer, the parsed tracks and fragments, the
decoders chosen, and a status line every two seconds (clock, queue depths,
frames shown and dropped). It also copies the page's console to stderr.

## Licence

The patch modifies HaikuWebKit and is under its licences (GNU LGPL v2 and
BSD); the new files carry the LGPL header. See [`LICENSE`](LICENSE).
