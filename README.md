# Play198x documentation

Documentation for [Play198x](https://github.com/play198x), the 198x family's retro media player/viewer.

Play198x is **started**. The first sub-project — the data-driven core (images
and tracker music) — is designed and approved; see
[`specs/`](specs/). Implementation has not begun.

## Scope (eventually)

- The media formats it plays/views (audio, image, animation) and how each is rendered.
- How it consumes Emu198x's chip/CPU cores for player-driven formats, and decodes pure-data formats directly — without reimplementing emulation.
- The embeddable surface (e.g. WASM previews) for the curriculum and Cat198x.

## Not here

- The decision to pursue Play198x and its boundaries — the umbrella decision record [`../../decisions/play198x-media-player.md`](../../decisions/play198x-media-player.md).
- Whole-machine emulation and the chip cores it reuses — Emu198x.
- Hardware facts — the umbrella `reference/` library.
