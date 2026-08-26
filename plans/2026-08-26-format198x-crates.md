# Format198x Crates Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the four image codecs out of Build198x into independent
Format198x crates, and add the two new crates Play198x needs, so every 198x
project reads and writes these formats through one shared, published
implementation.

**Architecture:** Six crates in the existing `format198x/format198x` workspace,
each independently versioned, dependency-free (`core`/`std` only), and
bidirectional. Build198x keeps its `crate::format::{scr, koala, art_studio,
ilbm}` paths by turning `src/format/mod.rs` into a re-export facade, so no call
site changes and its existing test suite is what proves the extraction was
faithful.

**Tech Stack:** Rust 2024 edition, no dependencies beyond `core`/`std`.
release-plz for per-crate versioning. `libxmp` 4.7.2 and `openmpt123` 0.8.9 are
installed for the MOD differential check.

**Spec:** [`../specs/2026-08-25-data-driven-core-design.md`](../specs/2026-08-25-data-driven-core-design.md)

## Global Constraints

- **Dependency-free.** `core`/`std` only. No crate here may depend on another
  Format198x crate, and there is deliberately no shared types crate.
- **Uniform shape by convention.** Every image crate exposes
  `decode(&[u8]) -> Result<T, DecodeError>` and `encode(&T) -> Result<Vec<u8>, EncodeError>`.
- **Bidirectional.** Both directions ship. Build198x encodes, Play198x decodes,
  Studio198x will need both — no direction may be dropped as unused.
- **Never panic on input bytes.** Every malformed input is a typed error.
  Unwinding across an FFI boundary is undefined behaviour and these crates will
  sit behind one.
- **Naming:** `format-{manufacturer}-{system}-{format}`, matching the names
  `build198x/src/format/mod.rs` already predicts in its module doc.
- **Licence:** `GPL-2.0-or-later`, inherited via `license.workspace = true`.
- **Edition:** 2024, via `edition.workspace = true`.
- **Independent versions.** Each crate carries its own `version = "0.1.0"`;
  **never** `version.workspace = true`. The workspace has no shared version by
  design (`format198x/Cargo.toml` comment).
- **Commit style:** Conventional Commits — release-plz derives versions and
  CHANGELOG entries from the prefixes.

---

## File Structure

**Created — `Format198x/format198x/crates/`:**

| Path | Responsibility |
|---|---|
| `format-sinclair-zx-spectrum-scr/` | Spectrum 6912-byte screen dump. Moved from build198x. |
| `format-commodore-c64-koala/` | Koala Painter multicolour bitmap. Moved. |
| `format-commodore-c64-art-studio/` | OCP Art Studio hires bitmap. Moved. |
| `format-commodore-amiga-ilbm/` | EA-IFF-85 ILBM. Moved; the largest at 582 lines. |
| `format-commodore-amiga-powerpacker/` | PP20 decrunch. New, decode-only. |
| `format-commodore-amiga-mod/` | ProTracker MOD parse and write. New. |

**Modified — `Build198x/build198x/crates/build198x/`:**

| Path | Change |
|---|---|
| `src/format/mod.rs` | Becomes a re-export facade; the shared error enums and `pub mod` declarations go. |
| `src/format/{scr,koala,art_studio,ilbm}.rs` | Deleted — they moved. |
| `Cargo.toml` | Gains four path dependencies. |
| `tests/{scr,koala,art_studio,ilbm,netpbm}.rs` | Deleted — they moved with their crates. |
| `tests/fixtures/golden/*` | Copied to the owning crates, then deleted here. |

**Why the facade rather than editing call sites.** Build198x reaches these
codecs from exactly two places: `src/convert/pipeline.rs:15` imports the four
modules and uses their types and constants (`scr::Screen`, `scr::COLUMNS`,
`art_studio::CELL_COLUMNS`), and `src/main.rs:772-779` calls `encode()` and maps
errors with `.map_err(|e| e.to_string())`. A facade preserves both. `to_string()`
works on any `Display` type, so four separate error types need no mapping layer.

**Why per-crate error types.** `DecodeError` and `EncodeError` currently live in
`build198x/src/format/mod.rs` and are shared by all four codecs. Independent
dependency-free crates cannot share them. Each crate therefore carries its own
copy, trimmed to the variants it actually constructs.

---

### Task 1: Graduate the SCR codec

Establishes the pattern the next two tasks repeat.

**Files:**
- Create: `Format198x/format198x/crates/format-sinclair-zx-spectrum-scr/Cargo.toml`
- Create: `Format198x/format198x/crates/format-sinclair-zx-spectrum-scr/src/lib.rs`
- Create: `Format198x/format198x/crates/format-sinclair-zx-spectrum-scr/src/error.rs`
- Create: `Format198x/format198x/crates/format-sinclair-zx-spectrum-scr/README.md`
- Create: `Format198x/format198x/crates/format-sinclair-zx-spectrum-scr/tests/scr.rs`
- Create: `Format198x/format198x/crates/format-sinclair-zx-spectrum-scr/tests/fixtures/pattern.scr`
- Source: `Build198x/build198x/crates/build198x/src/format/scr.rs` (173 lines)
- Source: `Build198x/build198x/crates/build198x/tests/scr.rs` (144 lines)

**Interfaces:**
- Consumes: nothing.
- Produces: `format_sinclair_zx_spectrum_scr::{Screen, decode, encode, DecodeError,
  EncodeError, BITMAP_LEN, ATTRIBUTES_LEN, FILE_LEN, WIDTH, HEIGHT, COLUMNS,
  ATTRIBUTE_ROWS, bitmap_file_offset, attribute_file_offset}`.
  `decode(&[u8]) -> Result<Screen, DecodeError>`,
  `encode(&Screen) -> Result<Vec<u8>, EncodeError>`.

- [ ] **Step 1: Create the crate manifest**

```toml
# crates/format-sinclair-zx-spectrum-scr/Cargo.toml
# The ZX Spectrum screen format — 6,144 bytes of bitmap in the ULA's
# thirds/pixel-row/character-row order, followed by 768 attribute bytes.
# Graduated from build198x's format::scr once Play198x became a second
# consumer (../../decisions/module-and-crate-naming.md).
[package]
name = "format-sinclair-zx-spectrum-scr"
description = "ZX Spectrum SCR screen dumps — decode and encode the 6912-byte display file, including the ULA's interleaved bitmap order."
version = "0.1.0"
edition.workspace = true
license.workspace = true
repository.workspace = true
readme = "README.md"
keywords = ["zx-spectrum", "sinclair", "scr", "screen", "retro"]
categories = ["encoding", "multimedia::images"]

[lints]
workspace = true
```

- [ ] **Step 2: Copy the source and give it local errors**

```bash
cd /Users/stevehill/Projects/198x
SRC=Build198x/build198x/crates/build198x/src/format
DST=Format198x/format198x/crates/format-sinclair-zx-spectrum-scr
mkdir -p $DST/src $DST/tests/fixtures
cp $SRC/scr.rs $DST/src/lib.rs
cp Build198x/build198x/crates/build198x/tests/scr.rs $DST/tests/scr.rs
cp Build198x/build198x/crates/build198x/tests/fixtures/golden/pattern.scr $DST/tests/fixtures/
```

Then in `$DST/src/lib.rs`, replace the single line

```rust
use super::{DecodeError, EncodeError};
```

with

```rust
mod error;
pub use error::{DecodeError, EncodeError};
```

- [ ] **Step 3: Write the local error module**

Copy the two enums verbatim from
`Build198x/build198x/crates/build198x/src/format/mod.rs:36-162` into
`$DST/src/error.rs` — the enum bodies, their `Display` impls, and their
`impl std::error::Error` lines. Then delete any variant this crate never
constructs; `cargo build` names them as unused.

- [ ] **Step 4: Point the tests at the new crate**

In `$DST/tests/scr.rs`, replace the `build198x::format::scr` import path with
`format_sinclair_zx_spectrum_scr`, and repoint the golden fixture:

```rust
// was: include_bytes!("fixtures/golden/pattern.scr")
const GOLDEN: &[u8] = include_bytes!("fixtures/pattern.scr");
```

- [ ] **Step 5: Add the crate to the workspace and build**

The workspace uses `members = ["crates/*"]`, so no edit is needed.

Run: `cd Format198x/format198x && cargo build -p format-sinclair-zx-spectrum-scr`
Expected: compiles, possibly with `dead_code` warnings naming unused error
variants. Remove those variants and rebuild until clean.

- [ ] **Step 6: Run the moved tests**

Run: `cargo test -p format-sinclair-zx-spectrum-scr`
Expected: every test that passed in build198x passes here, same count.

- [ ] **Step 7: Prove the round-trip against the golden fixture**

Add to `$DST/tests/scr.rs`:

```rust
#[test]
fn golden_round_trips_byte_for_byte() {
    let decoded = format_sinclair_zx_spectrum_scr::decode(GOLDEN)
        .expect("golden fixture decodes");
    let reencoded = format_sinclair_zx_spectrum_scr::encode(&decoded)
        .expect("decoded screen re-encodes");
    assert_eq!(reencoded, GOLDEN, "re-encoding the golden fixture changed bytes");
}

#[test]
fn malformed_input_never_panics() {
    for len in [0usize, 1, 6143, 6911, 6913, 8192] {
        let bytes = vec![0u8; len];
        assert!(
            format_sinclair_zx_spectrum_scr::decode(&bytes).is_err(),
            "length {len} should be rejected, not accepted"
        );
    }
}
```

- [ ] **Step 8: Run them**

Run: `cargo test -p format-sinclair-zx-spectrum-scr`
Expected: PASS, including the two new tests.

- [ ] **Step 9: Write the README**

Mirror `crates/format-commodore-amiga-adf/README.md` in shape: what the format
is, a `decode` example, a `encode` example, a Notes section stating that the
bitmap is stored in thirds/pixel-row/character-row order rather than linearly,
the "Part of the 198x family" section, and the GPL-2.0-or-later licence
section.

- [ ] **Step 10: Commit**

```bash
cd /Users/stevehill/Projects/198x/Format198x/format198x
git add crates/format-sinclair-zx-spectrum-scr
git commit -m "feat: add format-sinclair-zx-spectrum-scr

Graduated from build198x's format::scr, which its own module doc predicted
would become this crate once a second consumer appeared. Play198x is that
consumer.

Carries its own DecodeError/EncodeError rather than the shared enums it used
inside build198x: these crates are dependency-free by design, so they cannot
share a type without the shared crate the rule exists to prevent."
```

---

### Task 2: Graduate the Koala and Art Studio codecs

Two C64 bitmap formats, same shape as Task 1, small enough to review together.

**Files:**
- Create: `Format198x/format198x/crates/format-commodore-c64-koala/{Cargo.toml,README.md,src/lib.rs,src/error.rs,tests/koala.rs,tests/fixtures/pattern.koa}`
- Create: `Format198x/format198x/crates/format-commodore-c64-art-studio/{Cargo.toml,README.md,src/lib.rs,src/error.rs,tests/art_studio.rs,tests/fixtures/pattern.art}`
- Source: `Build198x/.../src/format/koala.rs` (196 lines), `art_studio.rs` (168 lines)
- Source: `Build198x/.../tests/koala.rs` (148 lines), `art_studio.rs` (133 lines)

**Interfaces:**
- Consumes: nothing.
- Produces: `format_commodore_c64_koala::{Koala, decode, encode, DecodeError,
  EncodeError, LOAD_ADDRESS, BITMAP_LEN, SCREEN_RAM_LEN, COLOR_RAM_LEN,
  FILE_LEN, WIDTH, HEIGHT, CELL_COLUMNS, bitmap_offset}` and
  `format_commodore_c64_art_studio::{ArtStudio, decode, encode, DecodeError,
  EncodeError, LOAD_ADDRESS, BITMAP_LEN, SCREEN_RAM_LEN, TRAILING_PAD_LEN,
  MIN_FILE_LEN, FILE_LEN, WIDTH, HEIGHT, CELL_COLUMNS, bitmap_offset}`.

- [ ] **Step 1: Create both crates by the Task 1 pattern**

```bash
cd /Users/stevehill/Projects/198x
SRC=Build198x/build198x/crates/build198x
for pair in "koala:format-commodore-c64-koala:pattern.koa" \
            "art_studio:format-commodore-c64-art-studio:pattern.art"; do
  MOD=${pair%%:*}; REST=${pair#*:}; CRATE=${REST%%:*}; FIX=${REST##*:}
  DST=Format198x/format198x/crates/$CRATE
  mkdir -p $DST/src $DST/tests/fixtures
  cp $SRC/src/format/$MOD.rs $DST/src/lib.rs
  cp $SRC/tests/$MOD.rs $DST/tests/$MOD.rs
  cp $SRC/tests/fixtures/golden/$FIX $DST/tests/fixtures/
done
```

- [ ] **Step 2: Write both manifests**

`format-commodore-c64-koala/Cargo.toml`:

```toml
# Koala Painter multicolour bitmaps — 160x200 at 4 colours per 4x8 cell,
# with the $6000 load address the format carries. Graduated from build198x.
[package]
name = "format-commodore-c64-koala"
description = "Commodore 64 Koala Painter multicolour bitmaps — decode and encode the 10003-byte format including its load address."
version = "0.1.0"
edition.workspace = true
license.workspace = true
repository.workspace = true
readme = "README.md"
keywords = ["commodore", "c64", "koala", "bitmap", "retro"]
categories = ["encoding", "multimedia::images"]

[lints]
workspace = true
```

`format-commodore-c64-art-studio/Cargo.toml` is identical except:

```toml
name = "format-commodore-c64-art-studio"
description = "Commodore 64 OCP Art Studio hires bitmaps — decode and encode the 320x200 monochrome-per-cell format, with or without its trailing pad."
keywords = ["commodore", "c64", "art-studio", "bitmap", "retro"]
```

- [ ] **Step 3: Give each crate local errors**

In both `src/lib.rs`, replace `use super::{DecodeError, EncodeError};` with:

```rust
mod error;
pub use error::{DecodeError, EncodeError};
```

Copy the enums into each crate's `src/error.rs` from
`build198x/src/format/mod.rs:36-162` as in Task 1, then delete variants the
crate does not construct.

- [ ] **Step 4: Repoint the tests**

In each `tests/*.rs`, change the import path to the new crate name and the
fixture path to `include_bytes!("fixtures/<name>")`.

- [ ] **Step 5: Build and test both**

Run: `cd Format198x/format198x && cargo test -p format-commodore-c64-koala -p format-commodore-c64-art-studio`
Expected: PASS, same test counts as in build198x.

- [ ] **Step 6: Add round-trip and fuzz-length tests to each**

For Koala, in `tests/koala.rs`:

```rust
const GOLDEN: &[u8] = include_bytes!("fixtures/pattern.koa");

#[test]
fn golden_round_trips_byte_for_byte() {
    let decoded = format_commodore_c64_koala::decode(GOLDEN).expect("decodes");
    let reencoded = format_commodore_c64_koala::encode(&decoded).expect("re-encodes");
    assert_eq!(reencoded, GOLDEN);
}

#[test]
fn wrong_load_address_is_rejected() {
    let mut bytes = GOLDEN.to_vec();
    bytes[0] = 0x00;
    bytes[1] = 0x10;
    assert!(format_commodore_c64_koala::decode(&bytes).is_err());
}
```

For Art Studio, in `tests/art_studio.rs`, the same two tests with
`format_commodore_c64_art_studio` and `include_bytes!("fixtures/pattern.art")`.

- [ ] **Step 7: Run them**

Run: `cargo test -p format-commodore-c64-koala -p format-commodore-c64-art-studio`
Expected: PASS.

- [ ] **Step 8: Write both READMEs**

Mirror the ADF crate's shape. For Art Studio, state in Notes that the format is
accepted both with and without its 7-byte trailing pad — that behaviour is
already covered by the moved test
`decode_accepts_padless_and_canonical_lengths_and_ignores_the_pad`.

- [ ] **Step 9: Commit**

```bash
cd /Users/stevehill/Projects/198x/Format198x/format198x
git add crates/format-commodore-c64-koala crates/format-commodore-c64-art-studio
git commit -m "feat: add format-commodore-c64-koala and format-commodore-c64-art-studio

Graduated from build198x, both carrying their own error enums for the same
dependency-free reason as the SCR crate."
```

---

### Task 3: Graduate the ILBM codec

The largest of the four, and the only one with cross-checks against an outside
tool.

**Files:**
- Create: `Format198x/format198x/crates/format-commodore-amiga-ilbm/{Cargo.toml,README.md,src/lib.rs,src/error.rs}`
- Create: `.../tests/ilbm.rs`, `.../tests/netpbm.rs`
- Create: `.../tests/fixtures/{pattern-uncompressed.iff,pattern-byterun1.iff}`
- Source: `Build198x/.../src/format/ilbm.rs` (582 lines)
- Source: `Build198x/.../tests/ilbm.rs` (398 lines), `tests/netpbm.rs` (191 lines)

**Interfaces:**
- Consumes: nothing.
- Produces: `format_commodore_amiga_ilbm::{Ilbm, Compression, decode, encode,
  DecodeError, EncodeError, MAX_DIMENSION, CAMG_LACE, CAMG_HIRES, row_bytes}`.
  Note `encode` takes two arguments here, unlike the other three crates:
  `encode(&Ilbm, Compression) -> Result<Vec<u8>, EncodeError>`.

- [ ] **Step 1: Copy source, tests and both fixtures**

```bash
cd /Users/stevehill/Projects/198x
SRC=Build198x/build198x/crates/build198x
DST=Format198x/format198x/crates/format-commodore-amiga-ilbm
mkdir -p $DST/src $DST/tests/fixtures
cp $SRC/src/format/ilbm.rs $DST/src/lib.rs
cp $SRC/tests/ilbm.rs $SRC/tests/netpbm.rs $DST/tests/
cp $SRC/tests/fixtures/golden/pattern-uncompressed.iff \
   $SRC/tests/fixtures/golden/pattern-byterun1.iff $DST/tests/fixtures/
```

- [ ] **Step 2: Write the manifest**

```toml
# EA-IFF-85 ILBM — the Amiga's interleaved bitplane image format, both
# uncompressed and ByteRun1. Graduated from build198x.
[package]
name = "format-commodore-amiga-ilbm"
description = "Amiga IFF/ILBM images — decode and encode interleaved bitplanes, uncompressed or ByteRun1, with CAMG screen-mode flags."
version = "0.1.0"
edition.workspace = true
license.workspace = true
repository.workspace = true
readme = "README.md"
keywords = ["amiga", "ilbm", "iff", "bitplane", "retro"]
categories = ["encoding", "multimedia::images"]

[lints]
workspace = true
```

- [ ] **Step 3: Local errors, repointed tests**

Replace `use super::{DecodeError, EncodeError};` in `src/lib.rs` with the
`mod error; pub use error::{DecodeError, EncodeError};` pair, copy the enums
into `src/error.rs`, and change the import path and fixture paths in both test
files.

- [ ] **Step 4: Build and run the default suite**

Run: `cd Format198x/format198x && cargo test -p format-commodore-amiga-ilbm`
Expected: PASS. The netpbm cross-checks are `#[ignore]`d and do not run here.

- [ ] **Step 5: Run the ignored netpbm cross-checks**

Run: `cargo test -p format-commodore-amiga-ilbm -- --ignored`
Expected: PASS if netpbm is installed; skipped or failing-to-spawn if not.
If netpbm is absent, record that in the commit message rather than deleting
the tests — they are validation-tier and run when the tool is present.

- [ ] **Step 6: Add the round-trip test for both compressions**

```rust
const UNCOMPRESSED: &[u8] = include_bytes!("fixtures/pattern-uncompressed.iff");
const BYTERUN1: &[u8] = include_bytes!("fixtures/pattern-byterun1.iff");

#[test]
fn both_compressions_round_trip_byte_for_byte() {
    use format_commodore_amiga_ilbm::{decode, encode, Compression};
    for (golden, compression) in
        [(UNCOMPRESSED, Compression::None), (BYTERUN1, Compression::ByteRun1)]
    {
        let img = decode(golden).expect("golden decodes");
        let out = encode(&img, compression).expect("re-encodes");
        assert_eq!(out, golden, "round-trip changed bytes for {compression:?}");
    }
}
```

If `Compression::None` is spelled differently in the moved source, use the
spelling in `src/lib.rs:76`.

- [ ] **Step 7: Run it**

Run: `cargo test -p format-commodore-amiga-ilbm`
Expected: PASS.

- [ ] **Step 8: README, then commit**

```bash
cd /Users/stevehill/Projects/198x/Format198x/format198x
git add crates/format-commodore-amiga-ilbm
git commit -m "feat: add format-commodore-amiga-ilbm

The largest of the four graduated codecs, and the only one carrying
cross-checks against netpbm. Those stay #[ignore]d validation-tier tests."
```

---

### Task 4: Build198x consumes the graduated crates

This task is what proves Tasks 1-3 were faithful. Nothing in build198x's
behaviour may change.

**Files:**
- Modify: `Build198x/build198x/crates/build198x/Cargo.toml` (and `Cargo.lock`)
- **Prerequisite:** the four crates must be published to crates.io first.
- Rewrite: `Build198x/build198x/crates/build198x/src/format/mod.rs` (162 lines → ~30)
- Delete: `src/format/{scr,koala,art_studio,ilbm}.rs`
- Delete: `tests/{scr,koala,art_studio,ilbm,netpbm}.rs`
- Delete: `tests/fixtures/golden/{pattern.scr,pattern.koa,pattern.art,pattern-uncompressed.iff,pattern-byterun1.iff}`
- Unchanged, and this is the point: `src/convert/pipeline.rs`, `src/main.rs`

**Interfaces:**
- Consumes: all four crates from Tasks 1-3.
- Produces: `build198x::format::{scr, koala, art_studio, ilbm}` resolving to the
  new crates, so existing call sites compile untouched.

- [ ] **Step 1: Record the baseline before changing anything**

```bash
cd /Users/stevehill/Projects/198x/Build198x/build198x
cargo test 2>&1 | tail -30 > /tmp/build198x-baseline.txt
grep -cE "^test .* \.\.\. ok" /tmp/build198x-baseline.txt
```

Note the passing count. That number is the acceptance criterion for Step 7.

- [ ] **Step 2: Add the path dependencies**

In `crates/build198x/Cargo.toml` under `[dependencies]`:

```toml
# Graduated to the format198x org and consumed from crates.io like any external
# user — the same shape as the ADF crate two lines above. NOT a path dependency:
# build198x's CI checks out only its own repo, so a path into a sibling checkout
# resolves locally and fails in CI.
format-sinclair-zx-spectrum-scr = "0.1.0"
format-commodore-c64-koala = "0.1.0"
format-commodore-c64-art-studio = "0.1.0"
format-commodore-amiga-ilbm = "0.1.0"
```

⚠ **This task cannot run until the four crates are published.** Publish them
first (Task 1-3's release), then do this. Two earlier versions of this snippet
were wrong: a bare path dep, which `release-plz`'s `cargo-package` step refuses,
and then path-plus-version, which still fails in CI because the sibling repo is
not checked out. `crates/build198x/Cargo.toml` carried the answer all along in
its `format-commodore-amiga-adf = "0.2.0"` line.

Verify the relative depth resolves before trusting it:

```bash
cd /Users/stevehill/Projects/198x/Build198x/build198x/crates/build198x
ls ../../../../Format198x/format198x/crates/ | head
```

Expected: the crate directories are listed. If not, count the levels again
rather than guessing — this is the error class that cost a day elsewhere in
this family.

- [ ] **Step 3: Replace the module with a facade**

Rewrite `src/format/mod.rs` entirely:

```rust
//! Retro screen-format codecs — re-exported from the Format198x crates.
//!
//! These codecs lived here until Play198x became a second consumer and made
//! the split real, exactly as the previous version of this doc predicted.
//! They now live in `format198x/format198x/crates/`, are independently
//! versioned, and are published for use outside the family.
//!
//! The module paths are kept as aliases so call sites read unchanged:
//! `crate::format::scr::encode(..)` still resolves.
//!
//! **There is no longer a shared `DecodeError`/`EncodeError`.** Each crate
//! carries its own, because Format198x crates are dependency-free and cannot
//! share a type. Call sites convert with `.to_string()`, which works on any
//! of them via `Display`.

pub use format_commodore_amiga_ilbm as ilbm;
pub use format_commodore_c64_art_studio as art_studio;
pub use format_commodore_c64_koala as koala;
pub use format_sinclair_zx_spectrum_scr as scr;
```

- [ ] **Step 4: Delete the moved files**

```bash
cd /Users/stevehill/Projects/198x/Build198x/build198x/crates/build198x
rm src/format/scr.rs src/format/koala.rs src/format/art_studio.rs src/format/ilbm.rs
rm tests/scr.rs tests/koala.rs tests/art_studio.rs tests/ilbm.rs tests/netpbm.rs
rm tests/fixtures/golden/pattern.scr tests/fixtures/golden/pattern.koa \
   tests/fixtures/golden/pattern.art tests/fixtures/golden/pattern-uncompressed.iff \
   tests/fixtures/golden/pattern-byterun1.iff
```

- [ ] **Step 5: Build**

Run: `cd /Users/stevehill/Projects/198x/Build198x/build198x && cargo build`
Expected: compiles. If `src/main.rs` fails on error types, the cause is a
call site that matched on `DecodeError` variants rather than calling
`to_string()`; convert it to `.map_err(|e| e.to_string())` to match its
neighbours at `src/main.rs:772-779`.

- [ ] **Step 6: Clippy**

Run: `cargo clippy --all-targets -- -D warnings`
Expected: clean.

- [ ] **Step 7: Run the full suite and compare to the baseline**

Run: `cargo test 2>&1 | tail -30`
Expected: passing count equals the Step 1 baseline **minus** the tests that
moved out (the five deleted test files). No test that remains may fail.

This comparison is the whole point of the task: build198x's own suite, not
inspection, is what proves the extraction did not change behaviour.

- [ ] **Step 8: Commit**

```bash
cd /Users/stevehill/Projects/198x/Build198x/build198x
git add -A
git commit -m "refactor: consume the graduated format crates

The four screen codecs moved to Format198x now that Play198x is a second
consumer — the split format::mod's own doc predicted. src/format/mod.rs
becomes a re-export facade, so pipeline.rs and main.rs are untouched and the
existing suite is what proves the move was faithful.

The shared DecodeError/EncodeError are gone: each crate carries its own,
because Format198x crates are dependency-free. Call sites already mapped
errors with to_string(), which works on all four."
```

---

### Task 5: PowerPacker decrunching

New crate, decode-only. Three of the first four Amiga modules found on a real
music disk were PP20-crunched, so this is not optional for reading real media.

**Files:**
- Create: `Format198x/format198x/crates/format-commodore-amiga-powerpacker/{Cargo.toml,README.md,src/lib.rs,src/error.rs,tests/pp20.rs}`

**Interfaces:**
- Consumes: nothing.
- Produces: `format_commodore_amiga_powerpacker::{decrunch, is_powerpacked, DecodeError}`.
  `is_powerpacked(&[u8]) -> bool`, `decrunch(&[u8]) -> Result<Vec<u8>, DecodeError>`.

- [ ] **Step 1: Write the failing test**

`tests/pp20.rs`:

```rust
use format_commodore_amiga_powerpacker::{decrunch, is_powerpacked};

#[test]
fn recognises_the_pp20_magic() {
    assert!(is_powerpacked(b"PP20\x09\x0a\x0c\x0d\x00\x00\x00\x00"));
    assert!(!is_powerpacked(b"FORM\x00\x00\x00\x00ILBM"));
    assert!(!is_powerpacked(b"PP"));
    assert!(!is_powerpacked(b""));
}

#[test]
fn rejects_truncated_input_without_panicking() {
    for len in 0..12usize {
        let bytes = vec![b'P'; len];
        assert!(decrunch(&bytes).is_err(), "length {len} must be rejected");
    }
}
```

- [ ] **Step 2: Run it to see it fail**

Run: `cd Format198x/format198x && cargo test -p format-commodore-amiga-powerpacker`
Expected: FAIL — the crate does not exist yet.

- [ ] **Step 3: Write the manifest and the header parser**

```toml
[package]
name = "format-commodore-amiga-powerpacker"
description = "Amiga PowerPacker (PP20) decrunching — read crunched files, including the packed modules common on Amiga music disks."
version = "0.1.0"
edition.workspace = true
license.workspace = true
repository.workspace = true
readme = "README.md"
keywords = ["amiga", "powerpacker", "pp20", "decompression", "retro"]
categories = ["compression", "encoding"]

[lints]
workspace = true
```

PP20 layout: `"PP20"` magic, then 4 offset-size bits (one byte each), then the
crunched data, then a final 4 bytes holding the decrunched length in the top 3
bytes and the initial bit-skip in the lowest byte. The bitstream is read
**backwards** from the end.

```rust
pub const MAGIC: [u8; 4] = *b"PP20";
pub const MIN_LEN: usize = 12;

#[must_use]
pub fn is_powerpacked(bytes: &[u8]) -> bool {
    bytes.len() >= MIN_LEN && bytes[..4] == MAGIC
}
```

- [ ] **Step 4: Run the tests again**

Run: `cargo test -p format-commodore-amiga-powerpacker`
Expected: `recognises_the_pp20_magic` PASSES; the truncation test still fails
because `decrunch` is not written.

- [ ] **Step 5: Implement `decrunch`**

Read the trailer, then walk the bitstream backwards writing the output from its
end toward its start. Every read must be bounds-checked and return
`DecodeError::Truncated` rather than indexing past the slice — this crate will
sit behind an FFI boundary where a panic is undefined behaviour.

- [ ] **Step 6: Verify against a real crunched file**

The Gathering'92 music disk holds three PP20 files:

```bash
cd /Users/stevehill/Projects/198x
ls "/Volumes/Data/Library/ROMs/TOSEC/Commodore/Amiga/Demos/Music/10 Best Tunes from the Music Competition at the Gathering 1992, The (1992-04-19)(Spaceballs)(Disk 1 of 3).zip"
```

Extract `Ash-Vixen_Soulside Journey` (180,316 bytes, magic `PP20`) with the ADF
crate, decrunch it, and assert the result begins with a recognisable module
magic at offset 1080 (`M.K.`). **Do not commit the file** — reference it by
path, per the no-media-in-the-repo rule.

- [ ] **Step 7: Run everything**

Run: `cargo test -p format-commodore-amiga-powerpacker`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
cd /Users/stevehill/Projects/198x/Format198x/format198x
git add crates/format-commodore-amiga-powerpacker
git commit -m "feat: add format-commodore-amiga-powerpacker

PP20 decrunching. Three of the first four modules found on a real Amiga music
disk were PowerPacked, so reading real media needs this rather than treating
it as an edge case."
```

---

### Task 6: ProTracker MOD

The largest new crate. Its playback semantics are already researched and
verified — read the reference before writing code.

**Required reading:** [`198x/reference/by-topic/music-formats/protracker-playback-reference.md`](../../../reference/by-topic/music-formats/protracker-playback-reference.md).
It carries line-numbered citations into the replayer source and records one
place the widely-cited community spec is **wrong**.

**Files:**
- Create: `Format198x/format198x/crates/format-commodore-amiga-mod/{Cargo.toml,README.md,src/lib.rs,src/error.rs,src/parse.rs,src/write.rs,tests/mod_format.rs,tests/synthetic.rs}`

**Interfaces:**
- Consumes: nothing.
- Produces:
  ```rust
  pub struct Note { pub sample: u8, pub period: u16, pub effect: u8, pub param: u8 }
  pub fn decode(bytes: &[u8]) -> Result<Module, DecodeError>;
  pub fn encode(module: &Module) -> Result<Vec<u8>, EncodeError>;
  pub fn is_module(bytes: &[u8]) -> bool;
  ```

  ⚠ **`Module` and `Sample` must be LOSSLESS — `encode(decode(x)) == x` for any
  well-formed module.** A first attempt specified `title: String`,
  `name: String` and `orders: Vec<u8>` truncated to song length, and **0 of 17
  real modules round-tripped**. The format carries bytes those shapes discard:
  the restart byte at offset 951, the magic variant, order-table entries beyond
  song length (verified nonzero in real files), bytes after a name's NUL, the
  one-word no-loop ambiguity, and unused finetune bits.

  This is not pedantry. `studio198x-authoring.md` has Studio198x consuming these
  crates to *author* media, and an editor that cannot re-emit what it parsed
  corrupts every file it saves. Keep a trimmed-`&str` accessor for ergonomics,
  but never as the only representation.
  **This crate parses and writes only. It does not play** — the mixer is
  `play198x-core`'s engine, per the spec's separation of format from playback.

- [ ] **Step 1: Write the failing identification test**

`tests/mod_format.rs`:

```rust
use format_commodore_amiga_mod::is_module;

#[test]
fn identifies_by_magic_at_1080_not_by_extension() {
    let mut bytes = vec![0u8; 1084];
    bytes[1080..1084].copy_from_slice(b"M.K.");
    assert!(is_module(&bytes));

    for magic in [b"M!K!", b"FLT4", b"4CHN"] {
        let mut b = vec![0u8; 1084];
        b[1080..1084].copy_from_slice(magic);
        assert!(is_module(&b), "{} should be recognised", String::from_utf8_lossy(magic));
    }

    let mut wrong = vec![0u8; 1084];
    wrong[1080..1084].copy_from_slice(b"XXXX");
    assert!(!is_module(&wrong));
    assert!(!is_module(&[0u8; 100]));
}
```

- [ ] **Step 2: Run it to see it fail**

Run: `cd Format198x/format198x && cargo test -p format-commodore-amiga-mod`
Expected: FAIL — crate does not exist.

- [ ] **Step 3: Manifest and `is_module`**

```toml
[package]
name = "format-commodore-amiga-mod"
description = "ProTracker MOD modules — parse and write the Amiga tracker format: 31 samples, pattern data, and the order table."
version = "0.1.0"
edition.workspace = true
license.workspace = true
repository.workspace = true
readme = "README.md"
keywords = ["amiga", "protracker", "mod", "tracker", "music"]
categories = ["encoding", "multimedia::audio", "parser-implementations"]

[lints]
workspace = true
```

```rust
pub const MAGIC_OFFSET: usize = 1080;
const MAGICS: [&[u8; 4]; 6] = [b"M.K.", b"M!K!", b"FLT4", b"4CHN", b"6CHN", b"8CHN"];

#[must_use]
pub fn is_module(bytes: &[u8]) -> bool {
    bytes.len() >= MAGIC_OFFSET + 4
        && MAGICS.iter().any(|m| &bytes[MAGIC_OFFSET..MAGIC_OFFSET + 4] == *m)
}
```

- [ ] **Step 4: Run it**

Run: `cargo test -p format-commodore-amiga-mod`
Expected: PASS.

- [ ] **Step 5: Write the failing parse test with a synthetic module**

`tests/synthetic.rs` builds a module in code rather than shipping one, because
no media may enter the repository:

```rust
use format_commodore_amiga_mod::{decode, encode};

/// One looped square-wave sample, one pattern, a C-2 on channel 0 at row 0.
fn synthetic_module() -> Vec<u8> {
    let sample: Vec<u8> = (0..64).map(|i| if i < 32 { 100u8 } else { 156u8 }).collect();
    let mut out = b"SYNTH".to_vec();
    out.resize(20, 0);
    for i in 0..31 {
        let mut hdr = vec![0u8; 30];
        if i == 0 {
            hdr[..6].copy_from_slice(b"square");
            hdr[22..24].copy_from_slice(&((sample.len() / 2) as u16).to_be_bytes());
            hdr[25] = 64;                                    // volume
            hdr[28..30].copy_from_slice(&((sample.len() / 2) as u16).to_be_bytes());
        }
        out.extend_from_slice(&hdr);
    }
    out.push(1);                                             // song length
    out.push(0);                                             // restart
    out.extend_from_slice(&[0u8; 128]);                      // order table
    out.extend_from_slice(b"M.K.");
    let mut pattern = vec![0u8; 64 * 4 * 4];
    let (period, smp) = (428u16, 1u8);                       // C-2, sample 1
    pattern[0] = (smp & 0xF0) | (period >> 8) as u8;
    pattern[1] = (period & 0xFF) as u8;
    pattern[2] = (smp & 0x0F) << 4;
    out.extend_from_slice(&pattern);
    out.extend_from_slice(&sample);
    out
}

#[test]
fn parses_a_synthetic_module() {
    let m = decode(&synthetic_module()).expect("decodes");
    assert_eq!(m.title, "SYNTH");
    assert_eq!(m.orders.len(), 1);
    assert_eq!(m.patterns.len(), 1);
    assert_eq!(m.patterns[0].len(), 64);
    assert_eq!(m.patterns[0][0][0].period, 428);
    assert_eq!(m.patterns[0][0][0].sample, 1);
    let used: Vec<_> = m.samples.iter().filter(|s| !s.data.is_empty()).collect();
    assert_eq!(used.len(), 1);
    assert_eq!(used[0].name, "square");
    assert_eq!(used[0].volume, 64);
    assert_eq!(used[0].data.len(), 64);
}

#[test]
fn round_trips_byte_for_byte() {
    let original = synthetic_module();
    let decoded = decode(&original).expect("decodes");
    let reencoded = encode(&decoded).expect("re-encodes");
    assert_eq!(reencoded, original, "round-trip changed bytes");
}

#[test]
fn malformed_input_never_panics() {
    for len in [0usize, 1, 20, 1079, 1083, 1084] {
        assert!(decode(&vec![0u8; len]).is_err(), "length {len} must be rejected");
    }
}
```

- [ ] **Step 6: Run to see them fail**

Run: `cargo test -p format-commodore-amiga-mod`
Expected: FAIL — `decode` and `encode` are not written.

- [ ] **Step 7: Implement `decode` in `src/parse.rs`**

Header layout: 20-byte title; 31 × 30-byte sample headers (22-byte name,
big-endian length in **words**, finetune, volume, big-endian repeat start and
repeat length in words); song length at 950; restart at 951; 128 order bytes at
952; magic at 1080; then `max(orders[..songlen]) + 1` patterns of 1024 bytes;
then sample data.

Each 4-byte note cell decodes as:

```rust
let sample = (b0 & 0xF0) | (b2 >> 4);
let period = (u16::from(b0 & 0x0F) << 8) | u16::from(b1);
let effect = b2 & 0x0F;
let param  = b3;
```

A repeat length of 1 word or less means "no loop"; store `loop_len = 0`.
Sample data is signed 8-bit.

- [ ] **Step 8: Implement `encode` in `src/write.rs`**

The exact inverse. Pad names with NULs to their field widths, write lengths in
words big-endian, and emit exactly `max(order) + 1` patterns so the round-trip
is byte-identical.

- [ ] **Step 9: Run the tests**

Run: `cargo test -p format-commodore-amiga-mod`
Expected: PASS, all four tests.

- [ ] **Step 10: Verify against real modules**

```bash
cd /Users/stevehill/Projects/198x/Format198x/format198x
cargo test -p format-commodore-amiga-mod -- --ignored
```

Write an `#[ignore]`d test that walks a directory given by the
`MOD198X_CORPUS` environment variable, decodes every `.mod` in it, and asserts
none errors and none panics. Point it at extracted modules on the local
library; **commit no modules**.

- [ ] **Step 11: README and commit**

The README must state that this crate parses and writes only, and that playback
lives in `play198x-core` — otherwise the next reader adds a mixer here.

```bash
cd /Users/stevehill/Projects/198x/Format198x/format198x
git add crates/format-commodore-amiga-mod
git commit -m "feat: add format-commodore-amiga-mod

Parses and writes ProTracker modules. Deliberately does not play them: the
mixer is play198x-core's engine, because the tick scheduling and effect
dispatch are playback semantics rather than file layout.

Identification is by the magic at offset 1080, never by extension."
```

---

## Self-Review

**Spec coverage.** Six crates: Tasks 1-3 and 5-6. Build198x consuming them:
Task 4. Bidirectional requirement: every crate ships `encode` and `decode`,
asserted by a round-trip test in each. Uniform-shape convention: the
Interfaces blocks give the same `decode`/`encode` signatures throughout, with
ILBM's two-argument `encode` called out as the one exception. Never-panic:
each task has an explicit malformed-input test. No media in the repo: Tasks 5,
6 and 10 reference local paths and say not to commit.

**Not covered here, by design:** `to_rgba8()` on the image types. The spec's
uniform-shape convention names it, but nothing consumes it until
`play198x-core` exists — it belongs to Plan 2, where its first caller is
written and can test it.

**Placeholder scan:** none. Every step names exact files, commands and expected
output.

**Type consistency:** `Screen`, `Koala`, `ArtStudio`, `Ilbm` match the names in
the existing build198x source. `Module`, `Sample`, `Note` are defined once in
Task 6's Interfaces and used consistently in its tests.

**One risk worth naming.** Task 4's path dependencies cross from
`Build198x/build198x/crates/build198x` to `Format198x/format198x/crates/`,
which is four levels up and across. Step 2 verifies the depth with `ls` before
trusting it, because miscounted relative paths are a recurring error class in
this family.
