# Play198x Amiga ANIM — bounded contract

**Status:** Approved 2026-08-31 by
[play198x/play198x#57](https://github.com/play198x/play198x/issues/57).

**Continues:**
[`2026-08-25-data-driven-core-design.md`](2026-08-25-data-driven-core-design.md).

**Goal:** add one historically useful time-based visual format without turning
Play198x into a general video decoder or an Amiga emulator.

## Why this slice is justified

The decision was tested against media rather than inferred from the existing
ILBM decoder.

- The local TOSEC catalogue contains 1,124 Amiga "Animations and Videos" disk
  images. This is a collection count, not a claim that every disk contains an
  ANIM file.
- One deliberately ordinary disk, `'copter Animation`, contains three
  `FORM ANIM` files. FFmpeg decodes them as 64, 36, and 21 frames at 320×256.
- Every delta frame in those files uses operation 5, default interleave
  (`0`, meaning two buffers), and store rather than XOR. That gives the first
  implementation a real 118-delta cohort instead of a format-shaped guess.
- FFmpeg's IFF demuxer and `iff_ilbm` decoder independently decode ANIM and can
  emit a checksum per frame. Differential validation is therefore practical.

This is stronger evidence than the alternatives presently offer. AY and the
bounded callable-PSID slice already exist; another code-driven audio format
would begin a new host and corpus investigation. A CLI has broad potential,
but its consumers and stable output contracts are not yet evidenced. ANIM adds
the missing time-based visual class and lets the later CLI design observe that
class rather than anticipate it.

## Accepted cohort

The first slice accepts only files with this shape:

- outer `FORM ANIM`;
- a complete initial `FORM ILBM` frame;
- subsequent `FORM ILBM` frame records containing `ANHD` and `DLTA`;
- `ANHD.operation == 5` (byte vertical delta);
- `ANHD.interleave == 0`, the format's default two-frame interleave;
- `ANHD.bits == 0`, store operation;
- 1–8 planes, fixed dimensions and display mode;
- frame timing from `ANHD.reltime`, in 1/60-second jiffies;
- optional per-frame `CMAP` changes.

Unknown ancillary chunks are skipped with ordinary IFF bounds checking.
Palette changes apply to the displayed frame without changing the indexed
bitmap buffers.

The following are identified and refused, not decoded approximately:

- operations 1–4, 6–8, and 74;
- interleave 1 ANIM brushes and any interleave other than the default;
- XOR deltas or other non-zero `bits` combinations;
- geometry or display-mode changes after the initial frame;
- embedded audio such as `SBDY`;
- malformed, truncated, or allocation-exceeding files.

These refusals are typed and include the operation, interleave, or feature that
put the file outside the cohort. FLI/FLC does not enter scope.

## Playback contract

ANIM is a sequential frame source. Opening it yields metadata and a player at
frame zero. Advancing yields the next indexed frame, palette, duration, and
display information; the Play198x boundary converts that frame to RGBA for a
shell.

The first slice supports restart, pause/resume, and sequential advance. It does
not promise arbitrary seeking. Restart reconstructs from the initial frame;
future seeking may replay from the start or add bounded checkpoints after real
consumer needs establish which trade-off is useful. The public contract must
not imply that every frame is materialised.

End of stream stops on the final frame. Looping is a shell choice and restarts
the decoder; the file is not presumed to loop merely because old players often
did.

## State and memory

Operation 5 with default interleave applies each delta to the frame two places
back. The decoder therefore holds two indexed/planar reconstruction buffers
and swaps between them. It produces one RGBA presentation buffer on demand.
It never retains all decoded frames.

At 320×256 with eight planes, two planar buffers plus one RGBA frame are about
480 KiB. The core adds an animation-specific pixel cap of 1,048,576 pixels:
at eight planes the same working set is about 6 MiB, excluding the caller-owned
input. Declared dimensions, row sizes, chunk sizes, frame counts, and all
products and offsets use checked arithmetic before allocation or slicing.

The browser currently receives the complete input as bytes, so its shell must
also reject an animation above a documented input-size limit before copying it
into wasm. The implementation plan measures the existing transfer path and
sets that number; this contract does not invent it in advance.

## Reuse boundary

The current ILBM crate returns chunky indexed pixels. ANIM5 deltas operate on
planar byte-columns, so its returned `Ilbm` is useful for metadata and rendered
semantics but is not a reconstruction buffer. Implementation must extract a
small, tested planar BODY seam from the ILBM crate or add an equivalently
bounded internal representation; converting every delta through RGBA would be
both incorrect and wasteful.

ADF, ZIP, and PowerPacker reuse is genuine: they already deliver entry bytes
to probing and decoding. Existing ILBM palette, aspect, and RGBA conventions
remain the presentation authority. No machine, chipset, or CPU emulation is
involved.

## Correctness gates

- constructed fixtures cover operation-5 literal, repeat, skip, unchanged
  plane, padding, palette change, timing, and every typed refusal;
- all decoded frames of the three measured files match FFmpeg frame hashes
  after normalising pixel format;
- an ignored local-corpus sweep reports accepted/refused/corrupt counts and a
  histogram of operation, interleave, bits, dimensions, and frame counts;
- truncation and hostile-size tests prove bounded failure without panic;
- native and `wasm32-unknown-unknown` builds pass before a web API is added.

No third-party media is committed.

## Reopening the boundary

Another operation or interleave enters scope only with a measured local
cohort, primary documentation, an independent validation path, and a separate
decision about its extra state. Popularity by recollection or decoder
completeness on its own is not enough.
