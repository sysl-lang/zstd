# zstd

**Zstandard for sysl — the compressor that made the speed/ratio question stop being one.** At its
default level zstd is several times faster than zlib and compresses better; at level 1 it is faster
than almost anything that compresses at all; at 19 it is in brotli's range. One library, one format,
and a level dial running from -131072 to 22 across the whole of that.

*An independent sysl package for Zstandard. It is not affiliated with or endorsed by Zstandard's authors or trademark holders.*

```
dependencies {
  zstd { git = "github.com/sysl-lang/zstd", version = "0.1.0" }
}
```

`sh.sysl.miniz` is the package for deflate and gzip and `sh.sysl.brotli` the one for
`Content-Encoding: br`. This is the one for storage, for logs, for a build cache, for a wire protocol
— for anything where both ends are yours.

## Compressing and decompressing

```sysl
import sh.sysl.zstd.*

val small = compress(page)?                        // a Buf[u8] — nothing to size
val back  = decompress(small.view(), 1 << 20)?     // a limit, always
```

**`compress` needs no output buffer.** It sizes one at `compress_bound(src.len)` and hands back a
buffer holding exactly the result. `compress_into` is for a destination that already exists.

**`decompress` takes a limit, and it is not optional.** A zstd frame is entitled to say nothing about
how large it becomes — one written by a streaming encoder that was not told the length says exactly
that — and even where the header *does* carry a size, that number is the frame's claim rather than a
fact. A server decompressing a request body it did not create has to say how much it is prepared to
hold, and a bound that is easy to leave out is one that gets left out. This is `sh.sysl.brotli`'s
rule and it is here for the same reason.

**`frame_content_size` is how a caller finds out what a frame claims** before deciding what limit to
pass, and it answers three things rather than a number:

```sysl
frame_content_size(bytes) match
    Known(n)  -> …          // the header says n bytes. A claim; zstd checks it while decoding
    Unknown   -> …          // a valid frame written without the length known. Ordinary
    NotAFrame -> …          // not zstd, or fewer bytes than a header
```

The level defaults to 3 and the scale is not linear. Negative levels are a regime of their own — they
skip most of the match search and go down to -131072, where zstd stops compressing at all — 1 to 3
are for anything on a hot path, 9 is a good "compress once" setting, and 19 to 22 need `--ultra`-class
memory at the writing end.

## The API

| | |
|---|---|
| `compress(src, level, checksum, window, workers)` | a `Buf[u8]` holding exactly the frame |
| `compress_into(src, dst, …)` | into a slice you own; answers bytes written |
| `decompress(src, limit)` | a `Buf[u8]`, never larger than `limit` |
| `decompress_into(src, dst)` | into a slice you own; `dst.len` **is** the limit |
| `compress_bound(len)` | the largest a frame can be for that much input |
| `is_frame(bytes)` | the four-byte magic, and nothing else |
| `frame_content_size(bytes)` | `Known(n)` / `Unknown` / `NotAFrame` |
| `frame_compressed_size(bytes)` | how long the first frame of a concatenation is |
| `min_level()` `max_level()` `default_level()` | `-131072`, `22`, `3` on zstd 1.5 |
| `max_workers()` `has_multithreading()` | what this *build* of the library allows |
| `version()` `version_number()` | `"1.5.7"`, `10507` |
| `encoder(level, checksum, content_size, window, workers)` | a `&Encoder` |
| `Encoder.push(src, dst, op)` / `.drained()` / `.written` / `.footprint()` | a chunk at a time |
| `pledge(encoder, len)` | put a length in a streamed frame's header |
| `decoder(window)` | a `&Decoder`; `window` bounds what a frame may make it hold |
| `Decoder.push(src, dst, more)` / `.drained()` / `.written` / `.done` / `.footprint()` | |
| `compress_with_dict(src, dict, level)` / `decompress_with_dict(src, dict, limit)` | one-shot |
| `cdict(dict, level)` / `ddict(dict)` and their `.compress` / `.decompress` | digested once |
| `dict_id(dict)` / `frame_dict_id(frame)` | zero where there is no ID to give |

## A chunk at a time

`Encoder` and `Decoder` take a chunk of input and a slice to write into, and are deliberately the
same shape `sh.sysl.brotli`'s and `sh.sysl.miniz`'s are — so a program choosing an encoding at run
time writes the branches the same way.

```sysl
var e = encoder(5)?

var window: []u8 = [0; 4096]

while more()
    e.push(next_chunk(), window, if last() then End.Finish else End.Continue)?

    send(window[0..<e.written])
    e.drained()
```

**`drained` is what makes the caller-owned destination workable for a socket** and is this package's
one addition to the shape: without it the single slice would have to be as large as the whole
compressed stream. It moves nothing inside zstd — the frame carries on across the call — it only puts
this encoder's own cursor back to the start.

`End` is `Continue`, `Flush` or `Finish`. **`Flush` is the difference between a stream and a file**:
it closes a block so that everything offered so far can be decoded by a reader that has it, at a
little cost in ratio. `Finish` writes the frame epilogue, and a frame is not a frame without one.

**`Progress.done` means the frame is complete and nothing else.** A `Continue` that happened to leave
nothing buffered also reports zero bytes outstanding, and a caller reading *that* as a finished frame
would stop before the epilogue was written and produce something no decoder accepts. What a `Flush`
caller wants is `Progress.remaining`.

**`pledge` is what puts a length in a streamed frame's header**, so that a reader can ask
`frame_content_size` and get an answer. zstd checks the pledge at the end of the frame, so a stream
that does not honour it is refused rather than written wrong.

## Two things zstd does that a caller has to know about

**A `Finish` on the first call is delegated to the blocking single-shot path, so it starts no
workers.** zstd's own note says so, and the consequence is measurable: a context asked for two
workers costs 1.57 MB when the frame is closed in one call, and 51.7 MB against 3.7 MB when the input
is offered first. `compress` therefore drives a worker-carrying compression in two steps rather than
one, and a program using `Encoder` directly has to do the same or the parameter means nothing.

**zstd's one-shot needs headroom past the answer.** 8,000 bytes of repeating text compress to 66
bytes, and a destination of 66, 70 or 73 is refused with `dstSize_tooSmall` — 74 is the first that
works. So `Short` from `compress_into` carries `compress_bound(src.len)` rather than the size of the
answer: a figure a caller retries at and fails at again, silently, in a loop, is worse than a ceiling.
`sh.sysl.brotli` can name the exact figure and this deliberately does not; the reason is the
library's, and the suite pins it so that the day zstd stops needing the headroom is the day a test
goes red.

Decompression has no such quirk, so `Short` from `decompress_into` **is** the exact figure. Where the
frame declares no length, finding it out costs a whole decompression — which is why that call is not
the one for input you do not trust.

## Errors say which thing happened

| | |
|---|---|
| `Short(needed)` | the destination is too small. A ceiling when compressing, the exact figure when decompressing — see above |
| `TooLarge(limit)` | the answer passed the limit. `limit` is what was passed, not how large the answer would have been: finding *that* out is what the limit exists to refuse to do |
| `Truncated` | the input ended mid-frame. **Half a download, not a corrupt one** — the difference matters to whatever is retrying. Also what a broken `pledge` reports |
| `Corrupt(code, why)` | not a valid frame, or a checksum disagreed. `code` and `why` are zstd's own |
| `NoMemory` | zstd refused to allocate. Reachable without anything being wrong: see the numbers below |
| `OutOfRange(what, low, high)` | a parameter outside what **this build** accepts. `workers` on a single-threaded library lands here, with a range of 0 to 0 |

**A frame without a checksum can be corrupted into a different answer and report nothing.** That is
the format working as designed rather than a defect in this package: zstd's literals are not
protected unless you ask. `compress(src, level, checksum = true)` costs four bytes and makes
corruption an error, and the suite has both halves of that.

## Requirements

zstd has to be installed, and so do its headers. **Nothing is vendored**: zstd is over a megabyte of
C with its own build system, it is not a library anybody would put on a microcontroller, and every
platform ships it.

```
brew install zstd
sudo apt install libzstd-dev
sudo pacman -S zstd
```

The build needs no flags — `pkg-config` is asked where the library is.

**`requires { heap = true }`, and there is no way around it.** Both contexts are allocated by the
library, and the stable API has no caller-supplied-slice shape: `ZSTD_initStaticCCtx`, the call that
would place one in storage a program owns, sits behind `ZSTD_STATIC_LINKING_ONLY` and is documented as
unstable, so a binding using it would be pinned to one build of the library. **No `posix`** — nothing
here opens a file, reads a clock or touches a socket. (The *suite* reaches `sysl.process` to drive the
`zstd` command; a `@tests` file's needs are not the package's, and a consumer pays nothing for it.)

## What the memory actually does

Measured with a counting allocator through `ZSTD_createCCtx_advanced`, not estimated.

**Every compression is 2 allocations and 2 frees; every decompression is 1 and 1.** The sizes are
what vary, and they vary enormously:

| context | working set |
|---|---|
| compressing 1 MB, level 1 | 583 KB |
| compressing 1 MB, level 3 | 1.30 MB |
| compressing 1 MB, level 19 or 22 | 18.1 MB |
| compressing 100 bytes, level 3 | 27.6 KB |
| **streaming, size not pledged**, level 1 | 1.37 MB |
| **streaming, size not pledged**, level 3 | 3.66 MB |
| **streaming, size not pledged**, level 19 | 93.8 MB |
| **streaming, size not pledged**, level 22 | **874 MB** |
| decompressing, any level, any size | 96 KB |

Two things fall out of that table. **A pledged size is worth a great deal at the writing end** — zstd
sizes the window to the content when it knows it — and **the two ends are wildly asymmetric**, which
is the same arrangement `sh.sysl.miniz` describes for embedded work: compress on a host, decompress
wherever.

It is also why the allocator is **not** redirected, though zstd offers to. Several bindings in this
org put a library's state in storage the program owns; there is no figure a caller could size an arena
with here before choosing a level, a window and whether to pledge. What the library does supply is the
answer afterwards, so `Encoder.footprint()` and `Decoder.footprint()` report the real working set
rather than an estimate.

## Checking the contexts are released

A `ZSTD_CCtx` is over a megabyte and nothing in a suite notices one being leaked — the compression is
still correct. What notices is the resident set over a loop:

```sysl
for i in 0..<50000
    val out = compress("hello, this is a payload worth compressing a little".bytes)?
    val back = decompress(out.view(), 4096)?
```

```
/usr/bin/time -l ./probe
```

**2.44 MB at fifty thousand iterations, against 2.38 MB at a thousand** — flat, so the destructors
run. With the constructors returning plain values instead of `&Encoder` and `&Decoder`, the same
program reaches **6.6 GB**. That number is the proof the measurement can fail; without it a flat line
says nothing.

That is why every constructor hands back a box. A destructor runs when a box's strong count reaches
zero and for nothing else, so a `Drop` on a type nobody boxes is never called — and a type annotation
at the call site does not fix it, because reading a value into a `&Encoder` binding boxes a *copy*.
The error paths need more than that: `encoder` releases the context where it stands when a parameter
is refused, since on that path no box was ever made.

## Testing

```
sysl test .
```

**51 tests, no network.** Six of them are the raw layer asserting that sysl and clang agree about
zstd's memory — every field of `ZSTD_inBuffer`, `ZSTD_outBuffer` and `ZSTD_bounds` against the
header's own `offsetof`, so an upstream that reorders one fails a test rather than moving a cursor to
the wrong place for ever.

**Two of them drive the `zstd` command line**, which is the corroboration that could actually fail:
frames this package writes at level 9 are decompressed by the command and compared, frames the command
writes at -1, -9 and -19 are decompressed by this package and compared, and `zstd -t` is asked to
verify a checksummed frame we wrote — then asked again after a byte is flipped, so that the check is
seen to be capable of refusing. They are **skipped rather than failed** where the command is not
installed.

**The honest limit on that is worth stating.** `zstd` the command links the same `libzstd` this
package calls, so agreement says nothing about the *format* — both would agree about a mistake in the
library. What it checks is the **binding**: that parameters reach the context, that the cursor structs
are advanced correctly, that a dictionary is really loaded, that the streaming loop loses no byte at a
chunk boundary. There is no from-scratch zstd on this machine to be the second oracle
`sysl-lang/qrcodegen`'s lesson asks for — Python 3.14's `compression.zstd` and Node's
`zlib.zstdCompress` are both bindings of this same library.

**Each test was checked to be capable of failing**, by breaking the implementation on purpose and
watching which went red:

| what was broken | what went red |
|---|---|
| the level never set on the context | the level test, and the footprint test |
| the checksum flag never set | four tests, including the `zstd -t` one |
| the window log never set | the decoder-window test |
| the worker count never set | the workers test |
| the content-size flag never set | the content-size test |
| the limit not checked as output arrives | the limit test |
| `drained` doing nothing | both draining tests |
| the dictionary never loaded | both dictionary tests |
| the constructors returning unboxed values | 2.44 MB became 6.6 GB |

That pass found a real weakness rather than confirming the suite: **purely repeating text compresses
to 63–66 bytes at every level from -20 to 22**, so the obvious "a higher level does better" test
passed just as well on a binding that never set the level at all. The fixture is now text with noise
threaded through it, which compresses to 20,010 bytes at the bottom and 977 at the top.

**AddressSanitizer** (`SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .`) is clean, on the suite
and on a program doing fifty thousand round trips, with the instrumentation confirmed rather than
assumed — `nm -u <binary> | grep asan` reports 16 symbols including `__asan_report_load8`. **What it
covers is the sysl half only**: this package resolves zstd through `pkg_config`, so the objects were
compiled by the platform months ago and no flag of ours instruments them.

## What is deliberately not bound

**Everything behind `ZSTD_STATIC_LINKING_ONLY`.** That is about half of `zstd.h` and every one of them
is documented as unstable, so binding one would pin this package to a single build of the library. It
takes with it `ZSTD_getFrameHeader`, `ZSTD_decompressBound`, the sequence-level API
(`ZSTD_generateSequences`, `ZSTD_compressSequences`), skippable frames
(`ZSTD_writeSkippableFrame`/`ZSTD_readSkippableFrame`/`ZSTD_isSkippableFrame`), the `estimate*`
family, the static-workspace initialisers, and the expert compression parameters. Several of those are
genuinely wanted — skippable frames are how a container puts its own metadata in a `.zst`, and
`ZSTD_decompressBound` would let `decompress` size its answer for a frame that declares no length —
and they become available if upstream promotes them.

**Dictionary training.** `zdict.h` is a second header and a second library surface (`ZDICT_trainFromBuffer`
and the COVER family), and training is a build-time job rather than a run-time one — the `zstd
--train` command is what people use. This package **consumes** dictionaries, including raw corpora,
and that is the half a program needs.

**A trained dictionary's ID is untested here**, and the reason is worth knowing: a raw corpus carries
no dictionary header, so `dict_id` answers zero and a frame built against one records no ID. The
mismatch protection still works — the content simply fails to decode — but it comes from the data
rather than from an ID comparison. A caller who wants the ID check needs a dictionary from `--train`.

**Long-distance matching, the row match finder, `rsyncable`, and the rest of the expert knobs.** Some
are experimental-only; the rest have defaults that are right until somebody has a measurement saying
otherwise.

**`ZSTD_CCtx_refPrefix` and `ZSTD_DCtx_refPrefix`** — a one-frame dictionary that is not copied. Worth
having for a protocol where each message references the previous one, and it needs an ownership story
(the prefix must outlive the frame) that the loaded-dictionary form does not.

None of these has been asked for.
