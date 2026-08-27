# Play198x docs

> Read [`PRINCIPLES.md`](PRINCIPLES.md) first.

Documentation repo for Play198x. Part of the `play198x` org container; see
[`../AGENTS.md`](../AGENTS.md) for the org layout and [`../../AGENTS.md`](../../AGENTS.md)
for the 198x umbrella.

Play198x is **building**. The data-driven core is designed in
[`specs/`](specs/), planned in [`plans/`](plans/), and implemented — it probes a
file, decodes the four image formats to RGBA, and plays a ProTracker module.
Not yet published to crates.io. Read the specs before proposing changes to
scope or architecture.

## Not here

- The binding decision and boundaries — umbrella
  [`../../decisions/play198x-media-player.md`](../../decisions/play198x-media-player.md).
- Whole-machine emulation and the chip cores Play198x reuses — Emu198x.
- Hardware facts — the umbrella [`../../reference/`](../../reference/).
