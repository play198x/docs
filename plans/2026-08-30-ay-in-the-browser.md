# AY in the browser — implementation plan

> **For agentic workers:** Use superpowers:subagent-driven-development or
> superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax.

**Goal:** drop an `.ay` on play198x.github.io and hear any song in it.

**Spec:** [`../specs/2026-08-30-ay-in-the-browser.md`](../specs/2026-08-30-ay-in-the-browser.md)

**Architecture:** a `Player` trait in `play198x-core` with two implementations
— `Engine` (self-timing) and `FramePump<AyPlayer>` (frame-driven). One
`Player` class crosses the wasm boundary; `ModulePlayer` is removed. The site
gains a subtune selector.

## Global Constraints

- **No media file is ever committed.** Every fixture is built in code.
- **Comments state the constraint and why, never the development narrative.**
  Git holds the story. This rule has been broken four times on this codebase
  and each time produced a confidently false sentence.
- **A metric must be able to observe what it is named for.** Three have failed
  this here: a clipping count taken after the clamp, an audibility threshold
  that counted DC as sound, an overrun flag that could not see magnitude.
- **`RUSTUP_TOOLCHAIN` is set in this environment and silently overrides the
  repo pin.** Prefix cargo with `env -u RUSTUP_TOOLCHAIN`.
- Verify with: tests both feature ways, `cargo clippy --all-targets` both ways
  with `-D warnings`, `cargo fmt --check`, and
  `cargo tree -p play198x-core | grep -c zilog` = 0 on default features.
- `play198x-core` is breaking in this slice (`AyPlayer`'s render contract
  changes, `AyMeta` gains per-song data) — it goes to **0.4.0**.

---

### Task 1: The `Player` trait and `Position`

**Files:** create `crates/play198x-core/src/player/mod.rs` additions;
modify `crates/play198x-core/src/engine/mod.rs`; test
`crates/play198x-core/tests/player_trait.rs`.

**Interfaces — produces:**

```rust
pub trait Player {
    fn render(&mut self, out: &mut [f32]) -> usize;
    fn set_playing(&mut self, playing: bool);
    fn position(&self) -> Position;
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Position {
    Module { order: usize, pattern: usize, row: usize, tick: u8 },
    Frame { song: usize, frame: u32 },
}
```

`Position` is **not** `#[non_exhaustive]`: a third *kind* of position would be
a decision every consumer should be forced to take, and the spec's argument
for `Metadata` applies unchanged. The trait lives outside the `ay` feature —
`Engine` implements it on default features.

- [ ] **Step 1:** Write a failing test asserting `Engine` satisfies `Player`
      through a `&mut dyn Player`, that `render` returns what was asked for,
      and that `position()` reports the engine's order/pattern/row/tick.
- [ ] **Step 2:** Run it; expect a compile failure (no trait).
- [ ] **Step 3:** Define the trait and `Position`; implement for `Engine` by
      delegating to its existing `render`, `set_playing` and position getters.
      Do not change `Engine`'s inherent methods — the site still calls them.
- [ ] **Step 4:** Run; expect pass. Verify per Global Constraints.
- [ ] **Step 5:** Commit: `feat: give the core one Player contract`.

---

### Task 2: `FrameSource` and `FramePump`

**Files:** create `crates/play198x-core/src/player/pump.rs`; test
`crates/play198x-core/tests/frame_pump.rs`.

**This is the task the slice turns on.** It is the seam where a 50 Hz frame
meets a 128-sample worklet quantum, and a seam that drops or repeats samples
is audible as a click at 50 Hz.

**Interfaces — produces:**

```rust
pub trait FrameSource {
    fn frame(&mut self);
    fn render_frame(&mut self, out: &mut [f32]) -> usize;
    fn samples_per_frame(&self) -> usize;
}

pub struct FramePump<S: FrameSource> { /* source, ring buffer, read cursor */ }
impl<S: FrameSource> FramePump<S> {
    pub fn new(source: S) -> Self;
    pub fn source(&self) -> &S;
}
impl<S: FrameSource> Player for FramePump<S> { /* … */ }
```

`samples_per_frame` is asked **after each `frame()`**, never cached across
frames: a CIA-driven SID does not run at a constant rate, and caching is what
would make that format a special case.

- [ ] **Step 1:** Write the failing tests. The load-bearing one: a fake
      `FrameSource` emitting a known ramp (frame *k* yields samples
      `k*N .. k*N+N`), pumped through `render(128)` repeatedly until several
      frame boundaries have passed, asserting the concatenated output is the
      **exact** sequence one long render would have produced — no gap, no
      repeat, no reorder. Also: a request smaller than a frame, larger than a
      frame, exactly one frame, and a source whose `samples_per_frame`
      *changes between frames*.
- [ ] **Step 2:** Run; expect failure.
- [ ] **Step 3:** Implement. The buffer is sized from the source's largest
      seen frame and grows only when a frame exceeds it; `render` fills from
      the buffer, calling `frame()` + `render_frame` whenever it empties.
      A paused pump renders exact zeroes for the full request rather than
      fewer frames — a worklet callback given fewer samples than it asked for
      is a worklet callback that clicks.
- [ ] **Step 4:** Run; expect pass. Verify per Global Constraints.
- [ ] **Step 5:** Commit: `feat: pump frame-driven players into a worklet quantum`.

---

### Task 3: `AyPlayer` becomes a `FrameSource`, and metadata grows per-song

**Files:** modify `crates/play198x-core/src/player/ay/mod.rs`,
`crates/play198x-core/src/metadata.rs`; tests `tests/ay_player.rs`,
`tests/metadata.rs`; also `tests/ay_corpus.rs` where it calls `render`.

`AyPlayer::render` becomes `FrameSource::render_frame` and `frame()` becomes
`FrameSource::frame` — the corpus sweep drives it directly and must keep
working, so update it in the same commit and confirm every sweep figure is
unchanged. **Any figure that moves is a regression, not a new baseline.**

`AyMeta.length_frames` is the *first* song's length, which a subtune selector
makes wrong. Replace it:

```rust
pub struct AySong { pub name: String, pub length_frames: u16, pub fade_frames: u16 }
// AyMeta.songs: Vec<String> -> Vec<AySong>;  AyMeta.length_frames removed
```

`title` stays as song 0's name, documented as standing in for a file-level
title the format does not have.

- [ ] **Step 1:** Failing tests — `FramePump<AyPlayer>` renders a fixture
      tune through a 128-sample quantum; `ay_meta` reports each song's own
      length and fade.
- [ ] **Step 2:** Run; expect failure.
- [ ] **Step 3:** Implement; update the corpus sweep's call sites.
- [ ] **Step 4:** Run, **including the `#[ignore]`d corpus sweep** (both
      passes; the all-songs pass takes 15-20 minutes). Every figure must match
      the previous run.
- [ ] **Step 5:** Commit: `feat(ay)!: render through the frame pump`.

---

### Task 4: One player at the wasm boundary

**Files:** modify `crates/play198x-web/src/lib.rs`, `Cargo.toml`;
test `crates/play198x-web/tests/boundary.rs`.

Enable `play198x-core`'s `ay` feature. Export **one** `Player` class and
**remove `ModulePlayer`**. Measured cost of the feature: +32.9 KiB raw,
+12.2 KiB gzipped.

**Interfaces — produces (JS names):**

```
Player.renderQuantum() -> number
new Player(bytes, song, sampleRate)   // probes, dispatches on format
  render(frames) -> number
  leftPtr() / rightPtr() -> pointer
  setPlaying(bool)
  positionKind() -> "module" | "frame"
  order()/pattern()/row()/tick()      // module only
  song()/frame()                      // frame-driven only
ayMeta(bytes) -> AyMeta               // author, misc, songs[], per-song lengths
AyMeta: author(), misc(), songCount(), songName(i), songLengthMs(i)
```

Getters for the wrong position kind return `undefined` rather than throwing —
nothing on this side of the FFI boundary is allowed to fail loudly.

- [ ] **Step 1:** Failing boundary tests: an `.ay` fixture constructs a
      `Player`, reports `positionKind() == "frame"`, renders a full quantum,
      and refuses an out-of-range song index. Fixtures built in code and at
      least `AY_MIN_LEN` bytes — an eight-byte magic no longer probes.
- [ ] **Step 2:** Run `wasm-pack test --node`; expect failure.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** `wasm-pack test --node` passes; `cargo clippy` and
      `cargo fmt --check` clean; rebuild and **record the real bundle size**.
- [ ] **Step 5:** Commit: `feat!: one player at the boundary, and .ay with it`.

---

### Task 5: The site plays it, and offers the songs

**Files:** modify `play198x.github.io/src/scripts/audio.ts`,
`src/scripts/player.ts`, the player component, `scripts/a11y-sweep.mjs`;
`package.json` to `@play198x/web` 0.2.0.

**This is a different repo.** Commit there, not in `play198x`.

- [ ] **Step 1:** Move the worklet from `ModulePlayer` to `Player`. The
      worklet must not learn which format is playing — it constructs one
      thing and calls `render`. If format knowledge leaks into `audio.ts`,
      Task 2 was built wrong; say so rather than working around it.
- [ ] **Step 2:** Panel branches on `positionKind()`: order/pattern/row for a
      module, song and frame for a tune. No empty module fields for an `.ay`.
- [ ] **Step 3:** Subtune selector, shown only when `songCount() > 1`.
      Selecting a song **constructs a new player** — `.ay` has no seek, and a
      song is a separate entry point with its own register state.
- [ ] **Step 4:** Extend `scripts/a11y-sweep.mjs` to drive the selector, per
      its existing rule that every interaction state is audited — including
      the focus pass and the contrast-only pass.
- [ ] **Step 5:** Verify by **listening and looking**, not by "serves 200":
      drop a real multi-song `.ay`, hear song 0, switch song, hear it change.
      Screenshot the panel in both states.
- [ ] **Step 6:** Commit; open a PR.

---

## Verification for the whole slice

- The frame-pump seam test is the one that matters most; a click at 50 Hz is
  the failure it exists to prevent.
- The corpus sweep's figures must be **identical** to the pre-slice run.
- Bundle size recorded against the +12.2 KiB gzipped measurement.
- The site's a11y sweep covers the selector in every state.

## Open questions this plan does not settle

- What the panel shows while a song is fading. `AySong.fade_frames` now
  crosses the boundary; nothing displays it.
- Whether `Engine`'s inherent methods eventually go, leaving only the trait.
