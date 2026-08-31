# Play198x code-driven audio — design

**Status:** AY slice implemented 2026-08-30; ROM-free callable PSID implemented
2026-08-31 in [play198x/play198x#38](https://github.com/play198x/play198x/issues/38).
Open-ROM and self-driven SID work were split into
[#37](https://github.com/play198x/play198x/issues/37) and
[#39](https://github.com/play198x/play198x/issues/39) respectively.

**Binding decision:** [`play198x-media-player.md`](../../../decisions/play198x-media-player.md)
(umbrella), and its thin-consumer rule in particular: Play198x consumes
Emu198x's chip and CPU cores and never reimplements them.

**Continues:** [`2026-08-25-data-driven-core-design.md`](2026-08-25-data-driven-core-design.md),
which drew the fault line this spec crosses. That spec put SID and `.ay` in a
"code-driven — blocked" row. They are no longer blocked.

**Goal:** establish how `.ay` and `.sid` play by running each tune's own code
against Emu198x's published CPU and chip crates, on a host Play198x supplies —
and **without shipping ROMs**.

The AY half of that goal now ships in the core and browser. The SID half is a
plan, not a statement of current support; its research history and corpus
measurements remain in
[play198x/play198x#22](https://github.com/play198x/play198x/issues/22).

---

## Why now

Three things changed, and all three were measured rather than assumed.

**The crates are published and independently versioned.** Six leaf crates now
publish from Emu198x at 0.6.0, each carrying a literal version rather than the
workspace's — so they are genuinely consumable, not merely public:
`emu198x-mos-6502`, `emu198x-zilog-z80`, `emu198x-mos-sid-6581`,
`emu198x-gi-ay-3-8910`, `emu198x-commodore-paula-8364`,
`emu198x-ricoh-apu-2a03`. Each depends on `serde` and nothing else, so they
build for `wasm32` without argument. See emu198x/emu198x#1214.

**Their shape is the one this needs.** Both CPUs are pin-level — `addr`,
`data`, `rw`/`mreq`/`iorq`, `data_in`, `irq`, `nmi`, and a `tick()` — with no
bus trait and no machine to instantiate. The Z80 documents `bus_request` as
the path for "ordinary host dispatchers". The host supplies memory, which is
exactly the arrangement a tune player wants.

**The ROM question has a number.** A scratch probe ran 5,000 HVSC tunes'
6502 code against RAM with no ROM mapped, recording reads that would have
resolved to BASIC, KERNAL or CHARGEN under the bank register each tune set:

| | |
|---|---|
| never touched ROM | **89.6%** |
| touched BASIC | 225 |
| touched KERNAL | 299 |
| touched CHARGEN | 1 |
| `init` never returned | 208 (4.2%) |

Re-running a 2,000-tune sample for six times as long moved four files (89.0% →
88.8%), so the figure is stable rather than an artefact of a short run: ROM
access happens during `init` or the first seconds, or not at all.

A header census over all 61,157 files separately found 93.6% PSID, 6.4% RSID,
and only 589 files (1.0%) declaring a ROM dependency in the header — RSID's
C64 BASIC flag, the one case rejectable without running anything.

## Scope

**Slice 1 — `.ay`, including the beeper (implemented).** ROM-free by construction: the
format's player is a stub, not the Spectrum ROM. Smallest host, no licence
decision, and 696 `.ay.zip` archives already on the Time Capsule to test
against.

**The beeper is in this slice and was nearly left out.** `.ay` wraps Z80 code
that plays *either* the AY chip *or* the Spectrum's 1-bit speaker — player
version 3 of the format exists to add beeper support — so 1-bit tunes ride in
the same container. A host that decodes only the AY ports would run a beeper
tune perfectly and render silence: a tune that appears to work and makes no
sound, which is the failure mode this project keeps designing out. The beeper
is bit 4 of any write to port `$FE`, and rendering it is sampling that bit over
time.

**Slice 2 — `.sid`, PSID with a play address (implemented in #38).** ~93% of HVSC
by header, of which ~89% needs no ROM. The host drives: call `init` once, call
`play` per frame.

**Not in this spec.** RSID and the 111 self-driven PSIDs, which install their
own interrupt handlers and expect a power-on machine; open ROMs; NSF, SAP and
other code-driven formats; any desktop shell work.

**Never.** Shipping Commodore or Sinclair ROMs, or requiring a visitor to
supply one. Booting a machine — that is Emu198x's job, and the boundary in the
README.

## Architecture

### A host is not a machine

The unit this spec introduces is a **host**: RAM, a CPU, a clock, and a sound
chip, assembled to run someone else's code. It has no keyboard, no display, no
disk, and no ROM. It is not chip emulation — the chips are Emu198x's — so the
thin-consumer rule holds.

The distinction is load-bearing rather than rhetorical. `emu198x-c64` and
`common-commodore-c64` are both `publish = false`, so the assembled-machine
layer is not consumable even if we wanted it; only the parts are. That is the
right constraint for a media player.

### Where it lives

`play198x-core` uses optional features for code-driven audio, off by default.
The `ay` and `sid` features exist today; each acquires only its own CPU/chip pair.

This is the boring answer and it is chosen deliberately. `play198x-core` is
published and today has no emulation dependencies; a consumer decoding SCREEN$
files should not acquire a 6502 to do it. Features keep the default dependency
surface exactly as it is, and make the code-driven half opt-in for the two
consumers that want it — the desktop shell and `play198x-web`.

```
play198x-core
├── container, decode, engine, metadata, probe   (unchanged, always built)
├── host                                          feature: ay | sid
│   ├── memory      address decode + ROM detector
│   └── clock       frame pacing, cycle budgets
├── player::ay                                    feature: ay
└── player::sid                                   feature: sid
```

### The address decoder is the detector

One function resolves an access, and it is the same function in both roles:

```rust
enum Resolved {
    Ram,
    Io(Chip),
    Rom(RomKind),   // Basic, Kernal, Chargen
}
```

A read resolving to `Rom` returns zero and records the dependency. Nothing is
instrumented and nothing is bolted on: the decoder has to exist for the player
to work at all, and the detector is a branch inside it.

For the C64 host the decode models the three `$01` lines — LORAM, HIRAM,
CHAREN — because a tune that banks RAM in over the ROM is running its own code
underneath it and must count as clean. A detector keyed on address range alone
would report 3,060 false positives on HVSC. Writes always reach RAM and are
never a dependency.

### Reaching the existing surface

`probe()` gained `Format::Ay`. `Format::Sid` belongs to #38.

**The shared player seam is implemented.** The core now has a `Player` trait
with `render`, `set_playing` and a format-shaped `Position`; seeking remains a
module-only operation rather than a trait method every frame-driven player
would have to refuse. `FrameSource` and `FramePump` reconcile fixed hardware
frames with arbitrary render requests. The wasm package exports one `Player`
class wrapping modules and AY, ready for SID to add another inner variant.

**Metadata took the deliberate breaking change.** `Metadata::Ay(AyMeta)` now
sits beside `Metadata::Module(ModuleMeta)`, and the wasm boundary exposes
`AyMeta` separately because author, misc text and a song table are not tracker
patterns, orders and samples. SID still needs its own metadata decision in
[#38](https://github.com/play198x/play198x/issues/38). The released core is
0.4.0; the browser package moved to 0.2.0 when its one `Player` boundary
replaced `ModulePlayer`.

## Data flow

### `.ay`

1. Parse: header, song table, block list, player version.
2. Load each block at its stated address into 64KB of RAM.
3. Install the stub: fill `$0000-$00FF` with `RET`, and set SP and the
   register pairs from the song's `points` and `LoReg`/`HiReg`.
4. Run the Z80 via `bus_request`, **calling** the tune's interrupt routine at
   its rate rather than raising a real `INT`. The format's `points` structure
   hands the player an interrupt address to call, so this needs no IM1/IM2
   vector modelling on a host with no interrupting hardware. A tune that
   installs its own handler and expects a real interrupt is out of slice 1;
   the corpus sweep will say how many those are.
5. Decode port writes: `$FFFD`/`$BFFD` to `emu198x-gi-ay-3-8910`, `$FE` bit 4
   to the speaker. Clock the chip, sample the speaker, mix both into the
   engine's frame buffer.

### `.sid`

The header fields and PSID environment below follow HVSC's canonical
[`SID_file_format.txt`](https://www.hvsc.c64.org/download/C64Music/DOCUMENTS/SID_file_format.txt),
the copy distributed with HVSC #85 as well as published by the project.

1. Parse: PSID/RSID header, v2NG fields, data blocks.
2. Load at `loadAddress`, or the first two bytes of the body when it is zero.
3. Establish the spec's default PSID environment: `$02A6` from the PAL/NTSC
   flag, CIA 1 timer A at 60Hz, VIC raster IRQ gated on the speed flag, and the
   bank register written per call from the address being called.
4. Call `init(song)` once with a cycle budget; then `play()` per frame.
5. Clock `emu198x-mos-sid-6581` between calls; drain via `take_buffer()`.

Both players expose the same `render(frames)` contract the worklet already
drives for ProTracker, so the web shell's audio path does not change shape.

## Testing

**The correctness bar is differential.** The ProTracker engine is trustworthy because it was verified against
the replayer source with a differential harness; a SID player asserted correct
by its author is worth nothing. Slice 2 is not done until its output is
compared against a reference player over a corpus.

- **Differential**: `.sid` is compared against sidplayfp over a two-second
  real HVSC tune. Both render 96,000 audible samples; their 20 ms energy
  envelopes correlate at 0.877. The envelope is the gate because different
  SID filter models do not promise sample identity. AY has extensive constructed-fixture and full-corpus coverage, but no
  retained reference-player differential harness; do not rewrite that as a
  completed claim.
- **Corpora, both already local**: 696 `.ay.zip` archives under
  `/Volumes/Data/WOS-Archive/music/ay`; HVSC #85 at
  `/Volumes/Data/Library/Music/HVSC_85-all-of-them.7z`.
- **Detector regression**: the ~89% measured clean must stay clean. A change
  that starts reporting ROM access on a known-clean tune is a bug in the
  decoder, and the probe's numbers are the baseline.
- **No committed media**, as everywhere else here. Fixtures are built in code;
  corpus runs read from the archive volume.

## Failure

Every failure is named rather than silent — the discipline the rest of this
family applies to gates.

- **Needs ROM**: "this tune needs the C64 ROMs, which Play198x does not ship",
  naming which. Not silence, and not a wrong-sounding rendition.
- **Header-declared ROM dependency** (RSID C64 BASIC flag): refused before
  anything runs.
- **Out of slice** (RSID, `playAddress` 0): identified and declined with the
  reason, not attempted and fumbled.
- **`init` never returns** inside its budget: reported as a failure to
  initialise, not played as silence.

## Open questions

- **Bundle size (answered).** AY adds 37.7 KiB raw and 13.6 KiB gzipped. SID,
  measured 2026-08-31 from otherwise identical release wasm builds, adds
  44,598 bytes raw and 18,548 bytes gzipped (320,397/145,025 with SID versus
  275,799/126,477 without).
- **Where the parsers live.** `.ay` parsing started here and has no second
  consumer, so it has not graduated. #38 must make the same deliberate choice
  for `.sid`; either location remains subject to the Format198x graduation
  rule when a standalone consumer appears.
- **Whether the host graduates.** If a second project ever needs to run tune
  code, `host` is the part worth extracting. Not now.
- **Open ROMs**, deferred by decision: the seam above supports loading a ROM
  image, and slice 3 can add it. The licence needs reading first — GitHub's
  detector classifies MEGA65/open-roms as `NOASSERTION`, and its CHARGEN is
  separately LGPL-3.0, against this project's `GPL-2.0-or-later`.
