# Play198x docs

> Read [`PRINCIPLES.md`](PRINCIPLES.md) first. [`MANIFESTO.md`](MANIFESTO.md) is why the project exists.

Documentation repo for Play198x. Part of the `play198x` org container; see
[`../AGENTS.md`](../AGENTS.md) for the org layout and [`../../AGENTS.md`](../../AGENTS.md)
for the 198x umbrella.

Play198x is **building**. The data-driven core is designed in
[`specs/`](specs/), planned in [`plans/`](plans/), and implemented — it probes a
file, decodes the four image formats to RGBA, plays a ProTracker module, and
plays ZX Spectrum `.ay` tunes behind an optional `ay` feature by running the
tune's own Z80 against Emu198x's published chip crates, with no ROM.
`play198x-core` is published on crates.io. Read the specs before proposing
changes to scope or architecture.

## Not here

- The binding decision and boundaries — umbrella
  [`../../decisions/play198x-media-player.md`](../../decisions/play198x-media-player.md).
- Whole-machine emulation and the chip cores Play198x reuses — Emu198x.
- Hardware facts — the umbrella [`../../reference/`](../../reference/).
