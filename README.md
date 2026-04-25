# ffmosh

`ffmosh` is the codec library that (will) power **Mosh**, the real-time
datamoshing system inside [Nuvotion 4](https://nuvotion.live). It is a
hard fork of [FFmpeg](https://ffmpeg.org/) and
[FFglitch](https://ffglitch.org/), stripped to a focused subset and
shipped as a set of LGPL shared libraries.

`ffmosh` is not a standalone application. It contains no CLI tools and
no scripting runtime. It exposes a C API that Nuvotion's host
application links against dynamically and drives from its own
scripting and node graph.

## Goals

- **Frame reordering and bitstream editing.** Both classic
  frame-reorder moshing and FFglitch-style codec-level bitstream
  manipulation are first-class.
- **Tight codec scope.** MPEG-2, MPEG-4 part 2, MJPEG, and PNG,
  mirroring FFglitch's supported set. All four are patent-clear today,
  which keeps the distribution story simple.
- **No bundled tooling.** No `ffmpeg`, `ffplay`, `ffprobe`, `ffedit`,
  `fflive`, or `ffgac` binaries. No QuickJS, no Python, no
  SDL/RtMidi/ZeroMQ bindings. All of those concerns belong to the host
  application.
- **Library only.** Builds as `libavcodec`, `libavformat`, `libavutil`,
  and `libswscale` shared objects. Static builds and CLI tools are
  disabled by default.
- **LGPL-clean.** No GPL or non-free dependencies enabled. The shared
  libraries can be linked from proprietary host applications under the
  terms of the LGPL.

## Non-goals

- General-purpose media transcoding. Use upstream FFmpeg.
- Network streaming, capture devices, audio routing, filter graphs.
  All disabled at configure time.
- Patent-encumbered codecs (H.264, H.265, AAC, etc.). Out of scope for
  bitstream editing.
- CLI compatibility with FFmpeg or FFglitch. The binaries are not
  built.

## Relationship to upstream

- **FFmpeg** — the foundation. Tracked as a remote so security and
  decoder fixes can be merged.
- **FFglitch** — provides the `ffedit_*` bitstream-editing primitives
  in `libavcodec/` and `libavformat/`. Tracked as a remote so its
  improvements can be merged.
- **ffmosh** — narrows the scope, removes the scripting runtime and
  CLI tools, and exposes the bitstream-editing API for direct C
  linkage from the host application.

## Build

`ffmosh` builds as a set of shared libraries only. Tested on macOS 14+
on Apple Silicon with Apple Clang from the Command Line Tools. Linux
builds should work with GCC or Clang; Windows builds are untested.

```
./configure \
    --install-name-dir=@rpath \
    --enable-shared \
    --disable-static \
    --disable-programs \
    --disable-doc \
    --disable-avdevice \
    --disable-avfilter \
    --disable-postproc \
    --disable-swresample \
    --disable-network \
    --disable-everything \
    --enable-decoder=mpeg2video,mpeg4,mjpeg,png \
    --enable-encoder=mpeg1video,mpeg2video,mpeg4,h263,mjpeg,png \
    --enable-parser=mpegvideo,mpeg4video,mjpeg,png \
    --enable-demuxer=avi,mov,matroska,mpegps,mpegts,mjpeg,image2,rawvideo \
    --enable-protocol=file,pipe
make -j
```

`--install-name-dir=@rpath` is macOS-specific: it bakes
`@rpath/libavcodec.61.dylib` (and friends) into each shared library's
`LC_ID_DYLIB`, so the consuming application finds them via its own
`@rpath` rather than at a hard-coded absolute path. Required for
bundling into Nuvotion's `.app`. Harmless on Linux (ignored).

The encoders are pulled in deliberately: FFglitch's transplication
path calls into encoder helpers (motion-vector encoding, Huffman
emission, bitstream stuffing) so the decoder can re-emit modified
bitstream chunks. They aren't surfaced as a host-facing encode API —
they're internal to the moshing pipeline.

The build is consumed in-place: Nuvotion's build system pulls the
generated shared libraries and headers directly out of this tree (or
out of a build directory) and bundles them inside the Nuvotion
application. There is no `make install` step and no system install.
ffmosh is not intended to be installed system-wide or distributed
separately — it is a private dependency of Nuvotion.

## Generated libraries

A successful build produces four shared libraries, totalling roughly
3.8 MB on Apple Silicon (file extensions vary per platform: `.dylib`
on macOS, `.so` on Linux, `.dll` on Windows).

### `libavcodec` (~1.5 MB)
The codec library. Contains the four decoders (MPEG-2, MPEG-4 part 2,
MJPEG, PNG), their matching encoders (used internally for
transplication), parsers, and bitstream filters — plus FFglitch's
`ffedit_*` bitstream-editing hooks. Exposes the public moshing API
via the `ffe_*` symbol prefix.

### `libavformat` (~620 KB)
The container library. Demuxes AVI, MP4/MOV, Matroska, MPEG-PS,
MPEG-TS, raw, MJPEG, and image2 streams; muxes the same set when
writing transplicated output. File and pipe protocols only — no
network. Includes FFglitch's container-level transplication writeback.

### `libavutil` (~920 KB)
The shared primitives library. Math, logging, frame and packet
allocators, hardware-context glue — plus FFglitch's `json_*` helpers
used by `ffedit_*` to represent per-frame feature data.

### `libswscale` (~770 KB)
Software colorspace and scale conversion. Kept as a fallback path;
production code paths in the host application are expected to do
YUV→RGB and resampling on the GPU.

## Linking from Nuvotion

Nuvotion's build picks up the four shared libraries and the public
headers directly from this submodule's source tree, links against
them, and bundles them inside the Nuvotion application package.
ffmosh is not a discoverable system library — Nuvotion finds it by
relative path.

Headers used:

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavcodec/ffedit.h>       // FFEditFeature enum + helpers
#include <libavcodec/ffedit_json.h>  // per-frame feature data API
```

Link line:

```
-lavformat -lavcodec -lavutil -lswscale
```

The moshing-relevant API surface beyond standard FFmpeg is:

- Three `uint64_t` bitmask fields on `AVCodecContext`:
  `ffedit_export`, `ffedit_import`, `ffedit_apply`.
- The `enum FFEditFeature` values (`FFEDIT_FEAT_MV`,
  `FFEDIT_FEAT_MV_DELTA`, `FFEDIT_FEAT_MB`, `FFEDIT_FEAT_Q_DCT`, etc.)
  passed via the bitmasks.
- The `ffe_jblock_*` and `ffe_jmb_*` accessors for reading and writing
  per-frame feature trees.

A typical per-frame loop is: set feature bitmasks before
`avcodec_open2()`, then for each packet, `avcodec_send_packet()`
(which populates the feature tree), read and mutate via the `ffe_*`
accessors, then `avcodec_receive_frame()` to obtain the decoded
result of the modified bitstream.

## License

`ffmosh` is distributed under the LGPL, matching the relevant subset
of FFmpeg and FFglitch. The original FFmpeg project's `README.md` has
been preserved as [`ffmpeg-README.md`](ffmpeg-README.md). License
texts are in [`LICENSE.md`](LICENSE.md) and the `COPYING.*` files.

## Status

Early-stage fork. The strip-down pass is in progress; the
Nuvotion-side C API is not yet stable. This library is not intended
for use outside Nuvotion at this time, but is published in the open
to satisfy LGPL distribution terms.
