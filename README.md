# Play198x documentation

Documentation for [Play198x](https://github.com/play198x), the 198x family's retro media player/viewer.

Play198x is **building**. The data-driven core and its code-driven audio hosts
are implemented: `play198x-core` is on crates.io, `@play198x/web` is on npm,
and [play198x.github.io](https://play198x.github.io) plays what you drop on it.
Today that means four still-image formats, ProTracker MOD, ZX Spectrum `.ay`,
and ROM-free callable PSID. RSID, self-driven PSID, multi-SID and tunes that
read mapped C64 ROMs are identified and explicitly declined. See
[`specs/`](specs/) for the design and measured boundaries.

## Scope (eventually)

- The media formats it plays/views (audio, image, animation) and how each is rendered.
- How it consumes Emu198x's chip/CPU cores for player-driven formats, and decodes pure-data formats directly — without reimplementing emulation.
- The embeddable surface (e.g. WASM previews) for the curriculum and Cat198x.

## Not here

- The decision to pursue Play198x and its boundaries — the umbrella decision record `198x/decisions/play198x-media-player.md`.
- Whole-machine emulation and the chip cores it reuses — Emu198x.
- Hardware facts — the umbrella `reference/` library.
