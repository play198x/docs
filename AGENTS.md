# Play198x docs

> Read [`PRINCIPLES.md`](PRINCIPLES.md) first. [`MANIFESTO.md`](MANIFESTO.md) is why the project exists.

Documentation repo for Play198x. Part of the `play198x` org container; see
[`../AGENTS.md`](../AGENTS.md) for the org layout and [`../../AGENTS.md`](../../AGENTS.md)
for the 198x umbrella.

Play198x is **building**. The data-driven core is designed in
[`specs/`](specs/), planned in [`plans/`](plans/), and implemented — it probes a
file, decodes the four image formats to RGBA, plays a ProTracker module, and
plays ZX Spectrum `.ay` and ROM-free callable PSID tunes behind separate
optional features. Their own Z80 or 6502 code runs against Emu198x's published
CPU and sound-chip crates on deliberately small hosts; neither ships a ROM.
RSID and zero-play-address PSID belong to Emu198x because they require a
continuously scheduled C64 machine. ROM-dependent and multi-SID tunes are
identified and explicitly declined.
`play198x-core` is published on crates.io. Read the specs before proposing
changes to scope or architecture.

## Not here

- The binding decision and boundaries — umbrella
  [`../../decisions/play198x-media-player.md`](../../decisions/play198x-media-player.md).
- Whole-machine emulation and the chip cores Play198x reuses — Emu198x.
- Hardware facts — the umbrella [`../../reference/`](../../reference/).
