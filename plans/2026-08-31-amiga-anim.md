# Amiga ANIM implementation plan

**Specification:**
[`../specs/2026-08-31-amiga-anim.md`](../specs/2026-08-31-amiga-anim.md)

**Issues:** [#58](https://github.com/play198x/play198x/issues/58), then
[#59](https://github.com/play198x/play198x/issues/59), then
[#60](https://github.com/play198x/play198x/issues/60).

Each phase is independently reviewable. Do not widen the accepted cohort when
a corpus file is refused; record its measured variant first.

## Phase 1 — corpus probe and fixtures

- Add a read-only ignored probe for files supplied through the existing
  container path.
- Report FORM/chunk validity, operation, interleave, bits, dimensions, frame
  count, timing range, palette changes, and accepted/refused reason.
- Run it over a representative sample from the local 1,124-disk TOSEC
  collection, not merely the first known-good disk.
- Preserve aggregate results in #58; commit no media or personal paths.
- Construct minimal byte fixtures for the measured ANIM5 shape and refusals.

Gate: the useful cohort remains operation 5/default interleave/store. If the
sample contradicts that, return to the specification rather than silently
adding variants.

## Phase 2 — bounded core decoder

- Add `FORM ANIM` probing ahead of generic IFF/ILBM handling.
- Parse the outer FORM and nested frame FORMs with checked lengths and IFF pad
  bytes.
- Establish a planar initial-frame seam that shares ILBM's BODY rules without
  duplicating its public rendering semantics.
- Implement operation-5 byte vertical store deltas over two alternating
  planar buffers.
- Apply timing and per-frame palettes; convert the current indexed frame using
  the existing Amiga image presentation conventions.
- Expose metadata plus sequential advance, restart, and playback state. Do not
  expose arbitrary seek or retained frame vectors.
- Enforce the 1,048,576-pixel working-set cap and typed refusals.

Gate: formatting, Clippy with warnings denied, full tests, documentation tests,
and the wasm target all pass.

## Phase 3 — differential and corpus validation

- Decode every frame of the three `'copter Animation` files and compare
  normalised frame hashes with FFmpeg.
- Add at least two more independently authored accepted files from different
  disks before calling the cohort representative.
- Run the corpus probe and record accepted/refused/corrupt totals and variant
  histograms in #58.
- Fuzz chunk boundaries, ANHD fields, delta offsets/op-counts, dimensions, and
  frame counts with deterministic mutations.
- Measure peak working memory at the ordinary 320×256 case and at the pixel
  cap.

Gate: no unexplained frame mismatch and no panic or unbounded allocation.

## Phase 4 — wasm boundary

- Add a narrow animation wrapper alongside image and audio surfaces.
- Return metadata and current RGBA frame; advance by elapsed time without
  depending on browser globals in the core.
- Measure wasm size with and without animation from otherwise identical
  release builds.
- Measure copies in the current file-to-wasm path and set a documented browser
  input-size limit before allocation.
- Test restart, end-of-stream, malformed input, and repeated open/drop cycles.

Gate: `@play198x/web` tests and wasm build pass, and bundle/memory measurements
are recorded in #59.

## Phase 5 — site playback

- Drive frames from `requestAnimationFrame` while deriving due frames from a
  monotonic clock; do not assume one animation frame per browser callback.
- Reuse the existing canvas sizing, pixel-aspect, drop, error, and teardown
  paths.
- Add play/pause and restart only. Do not introduce a seek bar for a decoder
  that does not promise random access.
- Honour reduced motion by opening paused and requiring an explicit play
  action.
- Verify stale callbacks and wasm objects are released when another file is
  dropped.

Gate: production build, automated browser checks, keyboard operation, reduced
motion, and manual playback of the measured files pass before deployment.

## After the slice

Use the resulting time-based visual contract as evidence when designing the
CLI in #61. Do not begin FLI/FLC or broader ANIM methods automatically.
