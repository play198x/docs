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

This applies the family's graduation rule — a format moves to Format198x once
it has a standalone consumer, the same trigger that moved ADF out of build198x.
MOD and PowerPacker are authored there directly rather than in Play198x, which
is a deliberate departure from the rule's letter: it keeps all format work in
one place and leaves Play198x to be a player. Recorded as a choice, not an
oversight.

### Play198x — the player

One repo, `play198x/play198x`, two crates. The GUI is a standalone crate
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

**`play198x-gui`** — an `eframe`/`egui` application. A thin client that holds no
logic of its own: anything it can do is an operation the core exposes, which is
what makes a scriptable surface a later addition rather than a retrofit
(binding decision, clause 7 — reachable from Forge198x with agent-native
parity).

### Why egui rather than Tauri

`cat198x/decisions/agent-native-surface-and-ui.md` chose Tauri for a
data-management UI and stated the choice does not transfer to real-time
rendering. Play198x sits between the two, and three things settle it: `eframe`
compiles to **both** native and WASM from one source, where Tauri has no
web-embed story — and the binding decision requires embeddable previews;
animation in a later slice needs a field-rate framebuffer, which that same
Cat198x decision says Tauri fights; and Emu198x already set the precedent for
media-shaped UIs.

The cost is accepted and named: egui is immediate-mode, so the metadata and
browsing surface is more work than it would be in HTML.

## Data flow

```
path ──► container::open ──► [Entry] ──► probe::identify ──► decode
                (plain | zip | adf)         (from bytes)         │
                 PP20 transparent                                ├─► Image ─► egui texture
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
dependency on this one: because the GUI is `eframe`, the same application
compiles to WASM, so the site can host the working player rather than describe
it. That is a reason to keep `play198x-core` free of any assumption that an OS
audio device exists — already required by the engine rule above, and noted here
so the constraint is not weakened later by someone who does not know the site
depends on it.

**The Emu198x packaging question** — whether the chip crates are published and
independently versioned — gates the code-driven slice and should be raised as
an Emu198x issue rather than solved here.

## Open questions

None blocking. Two recorded for the implementation plan to settle in passing:

- **SCR identification is weak.** A 6912-byte file is *probably* a SCR, but the
  length is the only signal. `probe` should report identification confidence
  rather than assert, and the GUI should show a low-confidence result as such.
- **Sample-rate conversion quality** is unspecified. Nearest-neighbour is
  closest to Paula and is the starting point; anything better is a later,
  measured change rather than a default.
