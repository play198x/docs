# Play198x data-driven core — design

**Status:** Approved 2026-08-25. First sub-project of Play198x; the project's
"not yet started" status ends here.

**Binding decision:** [`play198x-media-player.md`](../../../decisions/play198x-media-player.md)
(umbrella). Nothing below overrides it; where this spec is narrower, it is a
first slice and says so.

**Goal:** a desktop application that opens retro media from the library as it
actually sits on disk, shows images and plays tracker music, and is correct
against the ProTracker replayer rather than against a guess.

---

## Why this slice, and why this shape

The binding decision splits Play198x by media type — audio, image, animation —
and names two triggers for starting: curriculum media previews, or a Cat198x
preview surface. **Neither fired.** What fired was a direct want: to look at
Spectrum loading screens without converting them, and to play Amiga tracker
files.

Investigation on 2026-08-25 found the decision's media-type split is the wrong
seam for planning. The real fault line is **data-driven versus code-driven**:

| | needs | status |
|---|---|---|
| **Data-driven** — SCR, koala, Art Studio, ILBM, MOD, AY trackers | decode, then render or mix | nothing blocks it |
| **Code-driven** — SID, `.ay`, NSF, SAP | run a CPU *and* emulate a chip | blocked |

A `.mod` is sample data plus pattern data that a *player* interprets. A `.sid`
is 6502 machine code plus a register-write stream — the tune **is** a program.
The decision's clause 4 (thin consumer of Emu198x's cores) therefore binds only
the code-driven half, and that half is blocked: **all 220 Emu198x crates are
`publish = false` with `version.workspace = true`**, so a separate repo has no
supported way to depend on `mos-sid-6581`. That is an Emu198x packaging
decision nobody has taken, and it is not this slice's to take.

**So this slice is the data-driven half in full, and the code-driven half is
out of scope entirely.** A proof exists: a working ProTracker renderer was
built and verified the same day, with no emulator dependency of any kind.

## Scope

**In:** ZX Spectrum SCR; C64 Koala and Art Studio; Amiga ILBM; ProTracker MOD.
Container handling for plain files, ZIP, and ADF, with PowerPacker (PP20)
decrunching. A desktop GUI with real-time audio.

**Out, and where each goes instead:** SID, AY, NSF and SAP (a later slice,
gated on the Emu198x packaging decision). Animation — ANIM, FLI/FLC (a later
slice; it needs a field-rate framebuffer this one does not). Cataloguing and
collection indexing (Cat198x, by the binding decision — *"Not a cataloguer"*).
Media conversion and authoring (Build198x — *"Not an authoring/conversion
tool"*). CLI and MCP surfaces (deferred, but the architecture below must not
foreclose them). The website (its own spec; see **Related work**).

## Architecture

Three repos, already the family's shape.

### Format198x — the decoders

Six new crates beside the existing `format-commodore-amiga-adf` (0.2.1),
each independently versioned, dependency-free, `format-{manufacturer}-{system}-{format}`:

| Crate | Origin |
|---|---|
| `format-sinclair-zx-spectrum-scr` | graduated from `build198x/src/format/scr.rs` |
| `format-commodore-c64-koala` | graduated from build198x |
| `format-commodore-c64-art-studio` | graduated from build198x |
| `format-commodore-amiga-ilbm` | graduated from build198x |
| `format-commodore-amiga-mod` | new |
| `format-commodore-amiga-powerpacker` | new — PP20 decrunch |

The four graduated crates keep their existing `decode()` signature and their
malformed-input rejection tests. **build198x switches to consuming them**, so
the extraction is proved by build198x's own suite continuing to pass rather
than by inspection.

#### Uniform shape, not a shared type

Family members must be able to share these crates and convert between formats
without N×M code. That is achieved by **convention rather than by a dependency**:

- every image crate exposes `decode(&[u8]) -> Result<T, E>` and
  `T::to_rgba8(&self) -> (u32, u32, Vec<u8>)`
- every audio crate yields interleaved frames through the same shape the engine
  uses
- errors are typed per crate; none panic

**No `format198x-types` crate, deliberately.** A shared type crate would let the
compiler enforce the convention, and it was rejected because *dependency-free*
is the property that makes these crates usable outside the family — the stated
reason Format198x exists at all. Someone pulling in the ADF crate for their own
Amiga tool should not inherit a 198x-shaped type system.

The cost is accepted and named: the convention is held by review and by these
docs, not by the compiler. A crate that breaks the shape will compile.

This applies the family's graduation rule — a format moves to Format198x once
it has a standalone consumer, the same trigger that moved ADF out of build198x.
MOD and PowerPacker are authored there directly rather than in Play198x, which
is a deliberate departure from the rule's letter: it keeps all format work in
one place and leaves Play198x to be a player. Recorded as a choice, not an
oversight.

### Play198x — the player

One repo, `play198x/play198x`, three crates. The shells are standalone crates
excluded from the workspace so the library keeps its own lints and profiles —
the arrangement `cat198x` already uses for its Tauri UI.

**`play198x-core`**

| Module | Responsibility |
|---|---|
| `container` | Open a path — plain file, ZIP, or ADF — and enumerate entries. Decrunches PP20 transparently. |
| `probe` | Identify a format **from bytes, never from the extension**. |
| `decode` | Dispatch to the Format198x crates; yield an `Image` or a `Module`. |
| `engine` | The ProTracker mixer. |
| `metadata` | What the interface shows about a work. |

**`play198x-gui`** — a GPUI desktop application, and `play198x-web` — a
WASM shell for the site. Both are thin clients holding no logic of their own: anything a shell can do is
an operation the core exposes, which is what makes a scriptable surface a later
addition rather than a retrofit (binding decision, clause 7 — reachable from
Forge198x with agent-native parity).

#### `play198x-core` is published, not internal

**`play198x-core` is `publish = true` and independently versioned from the first
release**, on Format198x's model rather than Emu198x's.

This is not tidiness. Siblings are told to consume it — Studio198x by
[`studio198x-authoring.md`](../../../decisions/studio198x-authoring.md) clause
4, and the same substrate serves a curriculum preview or a Cat198x preview
surface. A library marked `publish = false` cannot be consumed from another org,
which is precisely the trap recorded as `emu198x/emu198x#1214`: the chip cores
Play198x is instructed to reuse are unconsumable, and the instruction has been
unsatisfiable since it was written. That failure is visible in advance here, so
it is closed rather than repeated.

The GUI and web shells are **not** published — they are applications, and
nothing consumes them.

#### Constraints the core carries for surfaces it does not yet have

Two later surfaces are planned (see **Roadmap**), and both are painful to
retrofit. `play198x-core` therefore holds these from the first commit, even
though nothing in this slice exercises them:

- **No global mutable state**, and no assumption of being on a main thread. A
  thumbnailer runs inside somebody else's process, possibly several at once.
- **No `panic!` reachable from a decode path.** Unwinding across an FFI boundary
  is undefined behaviour; the typed-error rule below is therefore not merely
  good manners.
- **Cold start is a budget, not an afterthought.** A thumbnailer is spawned per
  file and must produce a picture quickly; nothing may do lazy global init that
  assumes a long-lived process.
- **No assumption that an OS audio device exists** — already required by the
  engine rule, restated here because four separate consumers now depend on it:
  the desktop app, the web shell, the differential harness, and the thumbnailers.

### Why GPUI, and why two shells rather than one

**Desktop is GPUI** — Zed's framework, Apache-2.0, on crates.io (v0.2.2),
hybrid immediate/retained and GPU-accelerated, with native support for macOS
(Metal), Linux/FreeBSD (Wayland/X11) and Windows (Win32/DirectWrite) — the same
three platforms the thumbnailer work targets.

It is chosen on the axis that motivated the project. The complaint about
existing players was **interface and metadata handling**, not fidelity, and a
rich stateful browsing surface is precisely what pure immediate-mode makes
hardest. GPUI's retained side is built for that, and Zed is the evidence it
carries.

Tauri was rejected for the reason `cat198x/decisions/agent-native-surface-and-ui.md`
gives: it suits data-management UIs, and explicitly does not transfer to
real-time rendering — which animation, in a later slice, will need.

**GPUI has no web or WASM support**, and no roadmap for it. That is a real cost
and it is paid deliberately: it means the browser player is a **second, much
smaller shell**, not the same binary. Accepted because the two shells are not
comparable in size — the web shell needs a drop target, an image canvas and
play/pause, where the desktop app needs the whole browsing and metadata surface.
The core is shared; only the shell differs.

**GPUI is pre-1.0 and warns of frequent breaking changes.** Accepted as a
maintenance tax, and the reason the GUI crate stays thin: churn should land in
one small crate, never in `play198x-core`.

### The two shells

| Crate | Target | Scope |
|---|---|---|
| `play198x-gui` | desktop, GPUI | the full application |
| `play198x-web` | browser, `wasm-bindgen` | drop target, image canvas, play/pause — nothing more |

`play198x-web` uses **no Rust UI framework**. The site is already Astro, so HTML
and CSS provide the chrome, `<canvas>` displays decoded images, and WebAudio
takes engine frames. `play198x-core` compiles to `wasm32-unknown-unknown` and
does decode and mixing only. This keeps the WASM bundle small — a Rust UI
toolkit compiled to WASM would be most of the download for a UI this thin.

## Data flow

```
path ──► container::open ──► [Entry] ──► probe::identify ──► decode
                (plain | zip | adf)         (from bytes)         │
                 PP20 transparent                                ├─► Image ─► shell texture
                                                                 └─► Module ─► engine ─► frames
```

### The engine's one rule

```rust
impl Engine {
    fn render(&mut self, out: &mut [f32]) -> usize;   // interleaved stereo; allocates nothing
}
```

**The engine never owns an audio device.** It is a pull-based frame source, so
the same engine serves the audio callback, the offline differential harness,
and a future WebAudio build with no divergence between them. A harness that had
to capture a live device could not compare numerically against a reference
player at all — this constraint is what makes the fidelity target testable.

Transport is a lock-free command channel in; position is atomics out. The
engine lives in the audio callback: a four-channel mixer is cheap enough that a
producer thread and ring buffer would be machinery without a purpose.

### Playback correctness the engine must carry

ProTracker's replayer has **three effect dispatch tables**, not one:

| Table | When | Handles |
|---|---|---|
| `prefx_tab` | note tick, before the period is set | `3` `5` tone portamento, `9` sample offset |
| `morefx_tab` | note tick after setup, or a row with no note | `9` `B` `C` `D` `E` `F` |
| `fx_tab` | every tick that does **not** fetch a row | `0` `1` `2` `3` `4` `5` `6` `7` `A` `E` |

Modelling this is not gold-plating. A per-tick effect runs **`speed − 1` times
per row, not `speed`**, so a single effect switch produces a silent 20% error
in every per-tick effect at the default speed — and structurally cannot express
effect `9` appearing in both note-tick tables. The details, with line-numbered
citations into the replayer, are in
[`protracker-playback-reference.md`](../../../reference/by-topic/music-formats/protracker-playback-reference.md).

### Metadata

Module: title, format tag, channel count, pattern and order counts, and the
**full sample-name list** — authors hid messages there, so it is content rather
than diagnostics. Image: format, dimensions, palette, and the container path it
came from.

Plus **computed duration and loop detection**, by running the engine in a
silent fast mode that walks the sequence without mixing. This falls out of the
engine for free and answers a named complaint about existing players: no song
lengths.

## Testing

**Differential harness against libxmp 4.7.2.** Sample-exact comparison is *not*
achievable — different interpolation and mixing make waveforms legitimately
differ — so the harness compares **derived measures**: pitch track per waveform
cycle, onset timing, per-effect rate, envelope correlation.

**Fixtures are synthetic single-effect modules generated in code** — one held
note plus one effect — because real music confounds the measurement. This is
not hypothetical: a first attempt on 2026-08-25 measured a real module's
vibrato and got the row rate instead, because at speed 5 the 0.100 s row period
dominated the envelope.

**No media in the repository, ever.** Real modules and images are referenced by
local path and never committed, per
[`capturing-published-software.md`](../../../decisions/capturing-published-software.md)
— the screenshot travels, the tape image does not.

**Arbitration when sources disagree:** the replayer source first, libxmp second,
the community specs last. This ordering is load-bearing. The widely-cited
community spec states vibrato completes `(x × ticks)/64` cycles per division; it
is `(x × (ticks − 1))/64`, a 20% error at the default speed. libxmp and
libopenmpt also disagree with each other on vibrato rate, so a consensus of
implementations is not available as an authority.

Each Format198x crate additionally keeps or gains malformed-input rejection
tests; the four graduated ones already have them.

## Failure

Decoders return typed errors and never panic — the graduated crates and the ADF
crate already hold that line.

Container errors state what is true rather than what is convenient. **A
bootblock Amiga disk is not a corrupt ADF**; it is a disk with no DOS
filesystem, and `Disk::open` returning `Corrupt { what: "root block" }` for one
is the reader being right. Conflating the two cost real time on 2026-08-25 and
the GUI must not repeat it.

Unknown entries are **listed with the reason they cannot be played**, never
silently omitted. Errors surface inline against the entry; nothing is modal.

## Related work

**The website** (`play198x/play198x.github.io`) gets its own spec. It has a real
dependency on this one: `play198x-core` compiles to `wasm32-unknown-unknown`, so
the site can host a working player rather than describe it — via the small
`play198x-web` shell, since GPUI has no web target. That is the reason
`play198x-core` must stay free of any assumption that an OS audio device exists,
and free of global state: noted here so the constraint is not weakened later by
someone who does not know the site depends on it.

**The Emu198x packaging question** — whether the chip crates are published and
independently versioned — gates the code-driven slice and should be raised as
an Emu198x issue rather than solved here.

## Roadmap

Confirmed direction, each its own spec. Recorded so this slice's architecture is
judged against where it is going, not only against what it does.

**Code-driven formats — SID, AY, NSF, SAP.** Wanted, and out of scope here only
because they are blocked. Unblocking is an Emu198x decision about publishing and
independently versioning the chip crates; until it is taken, no amount of work
in this repo reaches a `.sid`.

**System preview integration.** A macOS **Quick Look** extension, with
equivalents on the other platforms. The three want different wrappers over one
core:

| Platform | Mechanism | Needs |
|---|---|---|
| macOS | Quick Look app extension (Swift) | a C-ABI static library to link |
| Linux | `.thumbnailer` file invoking `cmd %i %o %s` | **a CLI, and nothing else** |
| Windows | COM in-process DLL (`IThumbnailProvider`) | a C-ABI shared library |

So the work is one `play198x-ffi` crate plus a CLI, not three implementations.
Linux comes almost free once a CLI exists, which is a reason to expect the CLI
to be the next surface after the GUI rather than a distant one. The constraints
this places on the core are listed under **Architecture** and hold from the
first commit.

**Sharing across the family.** The uniform-shape convention above is what lets
build198x, Cat198x and Play198x use the same decoders and convert between
formats. Build198x already encodes these four image formats; the graduated
crates therefore carry **both** directions, and neither side may drop one.

**Authoring — Studio198x.** Creating media, as opposed to converting or
rendering it, was found to be unowned across the family on 2026-08-26 and is now
allocated to a new sibling
([`studio198x-authoring.md`](../../../decisions/studio198x-authoring.md)). It is
not started, but it binds two things here: the format crates stay bidirectional,
and `play198x-core` is published. A tracker is a module player that can also
write — most of it is this slice's engine, which is the reason those two
requirements are cheap now and expensive later.

## Open questions

None blocking. Two recorded for the implementation plan to settle in passing:

- **SCR identification is weak.** A 6912-byte file is *probably* a SCR, but the
  length is the only signal. `probe` should report identification confidence
  rather than assert, and the GUI should show a low-confidence result as such.
- **Sample-rate conversion quality** is unspecified. Nearest-neighbour is
  closest to Paula and is the starting point; anything better is a later,
  measured change rather than a default.
