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
