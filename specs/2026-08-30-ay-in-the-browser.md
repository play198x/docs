# AY in the browser — design

**Status:** Implemented 2026-08-30 in `play198x-core` 0.4.0,
`@play198x/web` 0.2.0 and the public player.

**Binding decisions:** [`play198x-media-player.md`](../../../decisions/play198x-media-player.md)
(umbrella), and the thin-consumer rule in particular.

**Continues:** [`2026-08-28-code-driven-audio.md`](2026-08-28-code-driven-audio.md),
which built the `.ay` host and named two things it deliberately left for
later: the `Player` trait, and exposing `.ay` through `play198x-web`. Both are
in this slice. Also [`2026-08-26-website-design.md`](2026-08-26-website-design.md),
whose player panel this extends.

**Goal:** drop an `.ay` file on play198x.github.io and hear it — every song in
it, not just the first.

---

## Why now

**The bundle cost is settled, and it is small.** The previous spec listed bundle
size as an open question and said to measure before committing. Measured, by
wiring the player into a throwaway build so the linker could not strip it:

| | raw | gzipped |
|---|---|---|
| today | 231.7 KiB | 109.9 KiB |
| with `ay` | 264.6 KiB | 122.1 KiB |
| added | +32.9 KiB | **+12.2 KiB** (+11.1%) |

Four more crates enter the `.wasm` and cost twelve kilobytes over the wire.
Naive measurement says 320 bytes, because nothing calls the code and the
linker removes it; that number is an artefact and is recorded here so it is
not repeated.

Those figures were the pre-implementation decision measurement. The shipped
build, measured after the complete boundary and metadata surface landed, adds
**37.7 KiB raw and 13.6 KiB gzipped**. The difference is why both numbers are
kept: the first answered whether to proceed; the second is what the released
feature actually costs.

**The trait's trigger condition has fired.** The previous spec proposed a
`Player` trait and the plan deferred it "when SID gives it a second
implementation, not before". `.ay` is that second implementation. Deferring
again means the worklet grows a second playback path, and SID makes three.

## The problem this design exists to solve

The two players do not have the same contract, and the difference is not
cosmetic.

- `Engine::render(&mut [f32]) -> usize` fills whatever it is asked for and
  keeps its own time. A worklet asks for 128 frames and gets 128.
- `AyPlayer` runs `frame()`, which executes one 50 Hz interrupt, then
  `render()`, which fills **exactly one frame's worth** — 960 stereo frames at
  48 kHz — and expects to be called once per `frame()`.

A worklet asks for 128. The tune produces 960 at a time. Something has to
reconcile those, and where it goes decides how much of this design leaks.

## Architecture

### The `Player` trait

```rust
pub trait Player {
    /// Fill `out` with interleaved stereo, returning frames written.
    fn render(&mut self, out: &mut [f32]) -> usize;
    fn set_playing(&mut self, playing: bool);
    fn position(&self) -> Position;
}
```

`render` is the whole contract: **ask for n, get n**. `Engine` already
satisfies it. `AyPlayer` does not, so the reconciliation lives inside the AY
implementation — a ring buffer that `render` drains, refilled by calling
`frame()` and the inner render whenever it runs dry.

That placement is the design decision. The alternative — teaching the worklet
that AY comes in 960-sample lumps — would put a format's quantisation into the
one piece of code that should not know which format is playing, and SID (which
is also frame-driven) would then have to agree with it.

`Position` is an enum rather than four integers, because a `.ay` has no order,
pattern, row or tick and a module has no song index:

```rust
pub enum Position {
    Module { order: usize, pattern: usize, row: usize, tick: u8 },
    Ay { song: usize, frame: u32 },
}
```

### One type crosses the boundary

`play198x-web` exports **one** `Player` class wrapping an enum, and
`ModulePlayer` is removed rather than left beside it. It takes bytes, a song
index and a sample rate, probes the bytes, and constructs whichever player the
format calls for.

Removing a published export is a breaking change, and it is the right one to
take now. `@play198x/web` is a 0.x package, and `play198x.github.io` is its
only consumer anywhere — checked across the family, not assumed — and this
slice edits that site regardless. Keeping `ModulePlayer` alongside a class
that subsumes it would buy compatibility nobody needs and leave every future
format deciding which of two players to extend. `@play198x/web` goes to 0.2.0.

### Where standardising stops

Two players become one because the *operations* are the same: fill `n` frames
of stereo, play or pause, report a position. That is a real shared contract
and a single type states it honestly.

The metadata does not follow, and should not. A module has 31 sample names,
patterns and orders; an `.ay` has an author, a misc string and a song table.
Merging them produces a type whose fields are mostly empty depending on what
was loaded — the same defect this design rejects for the panel, moved into the
API where it is harder to see. `ModuleMeta` and `AyMeta` stay separate, and
the core's existing `Metadata` enum stays the thing that says which is which.

The rule the rest of this slice follows: **standardise where the operations
are the same; keep separate where the data is different.**

### How SID and the rest fit

`.ay` is the first of several, so the seam is built for the family rather
than for this format. Sorting what is coming by *how it produces audio*, not
by machine, gives three groups and only three:

| Family | Examples | How it renders |
|---|---|---|
| Self-timing | ProTracker, and other tracker engines | `render(n)` directly; the engine keeps its own clock |
| Frame-driven code | **`.ay`**, SID, NSF, SAP | a CPU runs a routine per frame, then the chip is drained |
| Frame-driven data | YM, VGM | a register dump is applied per frame; no CPU at all |

The second and third rows differ only in what happens inside a frame. Both
produce a fixed lump of samples and both meet a worklet asking for 128. So
the reconciliation is not an `.ay` problem, and it does not belong inside
`AyPlayer`:

```rust
/// Something that produces audio exactly one frame at a time.
pub trait FrameSource {
    /// Advance one frame — run the tune's routine, or apply the next dump.
    fn frame(&mut self);
    /// Fill one frame's worth of interleaved stereo.
    fn render_frame(&mut self, out: &mut [f32]) -> usize;
    /// Samples this frame will produce. Asked after each `frame()`, not
    /// once: a SID driven by a CIA timer does not run at a fixed rate, and
    /// a fixed answer here is what would force that format to be special.
    fn samples_per_frame(&self) -> usize;
}
```

`FramePump<S: FrameSource>` holds the ring buffer and implements `Player`.
`AyPlayer` implements `FrameSource` and gets the quantum reconciliation for
free — and, more to the point, gets the *tested* seam for free, which is
where 50 Hz clicks come from. SID, NSF and SAP implement the same trait and
inherit both.

This slice builds `FrameSource`, `FramePump` and one implementation. It does
not build a second: the generalisation is justified by SID being the next
scheduled slice and by the register-stream formats sharing the same shape,
not by a hypothetical.

### One position for every frame-driven format

`Position` therefore does not grow a variant per format:

```rust
pub enum Position {
    Module { order: usize, pattern: usize, row: usize, tick: u8 },
    Frame  { song: usize, frame: u32 },
}
```

Every frame-driven format reports `Frame`. Which format it *is* comes from
`probe` and the metadata, which already say so — repeating it in the position
would be a second place for the same fact to be stated and to drift. Adding
SID changes this enum not at all.

### Subtunes

A `.ay` carries a table of songs: **278 of the 696 local archive files are
multi-song, 1,915 songs in total**. Playing only song 0 leaves 64% of them
unreachable, which is why the selector is in this slice rather than after it.

The boundary exposes the song count and the song names; selecting a song
constructs a new player at that index. It does not seek — `.ay` has no seek,
and a song is a separate entry point with its own register state, so
"switching song" and "starting a tune" are the same operation.

### Metadata

`AyMeta` crosses as its own boundary type: author, misc, per-song names, and
the selected song's declared length. The panel's existing duration display
works unchanged — a song declares `length_frames` at 50 Hz, so a real
duration exists without playing anything.

The panel shows order/pattern/row for a module. For an `.ay` it shows the song
selector and the frame counter. Those are different enough that the panel
branches on `Position` rather than showing empty module fields.

## Failure

Named, as everywhere else here.

- **A tune whose init never returns** is reported as a tune that would not
  start, naming that, rather than played as silence. 143 of the archive's
  1,915 songs are in this state by design — their init routine *is* the
  player, waiting on an interrupt the format's stub never raises.
- **A song index past the end** is refused before anything runs.
- **Bytes that are not a `.ay`** keep the core's own message.

## Testing

- The ring buffer is the new logic and gets the most attention: a test that
  asks for 128 frames repeatedly across a frame boundary and asserts the
  output is the same samples, in the same order, that one 960-frame render
  would have produced. A seam that drops or repeats samples at the boundary
  is the defect to design against, and it is audible as a click at 50 Hz.
- Trait conformance for both implementations: ask for fewer frames than a
  quantum, more, and exactly one.
- Boundary tests in `play198x-web` for song count, song selection, and
  refusing an out-of-range index.
- The site's a11y sweep grows the selector, per the existing sweep's rule
  that every interactive state is audited.
- No media committed; fixtures are built in code, as everywhere in this repo.

## Out of scope

- SID. It remains the next slice and this trait is what it will implement;
  [play198x/play198x#38](https://github.com/play198x/play198x/issues/38)
  tracks the bounded ROM-free PSID work.
- A compatibility alias for `ModulePlayer`. The implementation removed it in
  favour of the one `Player` boundary described above when the package moved
  to 0.2.0; keeping both would have preserved two extension points.
- Seeking within an `.ay`. The format has no seek.
- The desktop shell.

## Open questions

- **What the panel shows while a tune is fading.** `.ay` songs declare a fade
  length as well as a play length; nothing uses it yet.
