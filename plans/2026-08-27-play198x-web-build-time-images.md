# `play198x-web` Build-Time Images Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a Code198x page show a retro image file decoded by `play198x-core` at build time, emitting a plain `<img>` so no wasm, JavaScript or decoding work reaches the reader.

**Architecture:** A new standalone crate `play198x-web` wraps `play198x-core` behind two `wasm-bindgen` entry points and builds to two `wasm-pack` targets from one Rust surface. The website's `deploy.yml` builds the `nodejs` target and an Astro component calls it during the build, encoding the RGBA to a PNG with a dependency-free writer over `node:zlib` and emitting a `data:` URI.

**Tech Stack:** Rust 1.98, `wasm-bindgen`, `wasm-pack` 0.14.0, Astro 7, Node 24 (`node:zlib`, `node --test`), Playwright.

**Spec:** [`../specs/2026-08-27-web-shell-build-time-images.md`](../specs/2026-08-27-web-shell-build-time-images.md)

## Global Constraints

- **Rust 1.98.0.** `RUSTUP_TOOLCHAIN=1.95.0` is exported in these sessions and **silently overrides `rust-toolchain.toml`**. Run every cargo/rustup command as `env -u RUSTUP_TOOLCHAIN <cmd>` and print `rustc --version` alongside.
- **No new npm dependencies.** The PNG writer uses `node:zlib`; tests use `node --test`. Both ship with Node 24.
- **New Rust dependencies limited to `wasm-bindgen` and `wasm-bindgen-test`.** Both are intrinsic to the approved approach. Anything else needs asking.
- **`play198x-web` is `publish = false`** — a `#[wasm_bindgen]` surface has no Rust caller.
- **One Rust surface, two targets.** Neither `nodejs` nor `web` may gain logic the other would reimplement.
- **`pixel_aspect` is never ignored.** A shell that drops it draws a C64 multicolour picture at half its real width.
- **No panics across the boundary.** `play198x-core` guarantees this; the shell must not introduce one (no `unwrap`, no indexing).
- **No media committed.** Test fixtures are constructed in code, not checked in as files.
- **Every assertion states a specific expected value.** Expected colours come from `mediaspec198x`, never from our own output.
- **Conventional commits** (`feat:`, `fix:`, `docs:`, `chore:`) — release-plz reads them.

## Reference values, verified before this plan was written

**ZX Spectrum palette** (`mediaspec198x` 0.1.0, `emu198x-v1`; normal `0xC2`, bright `0xFF`):

| Index | Colour | RGB |
|---|---|---|
| 0 | black | `(0x00, 0x00, 0x00)` |
| 5 | cyan | `(0x00, 0xC2, 0xC2)` |
| 13 | bright cyan | `(0x00, 0xFF, 0xFF)` |

**SCREEN$**: 6912 bytes — 6144 bitmap + 768 attributes. Attribute byte is `FBPPPIII`: bit 7 FLASH (ignored — a still picture has no frames to count), bit 6 BRIGHT, bits 5–3 PAPER, bits 2–0 INK. A set bitmap bit takes INK, a clear bit takes PAPER. BRIGHT selects the palette's upper half.

**Mode geometry** (`mediaspec198x`):

| Mode | Mode pixels | `pixel_aspect` | Display size |
|---|---|---|---|
| Spectrum standard | 256 × 192 | 1:1 | 256 × 192 |
| C64 multicolour bitmap | 160 × 200 | 2:1 | **320 × 200** |

**Core API** (`play198x-core` 0.1.0):

```rust
probe::identify(bytes: &[u8]) -> Option<(Format, Confidence)>
decode::image(bytes: &[u8], format: Format) -> Result<Image, Error>

enum Format { Scr, Koala, ArtStudio, Ilbm, ProTracker }   // #[non_exhaustive]
enum Confidence { Certain, Probable }
struct Image { width: u32, height: u32, rgba: Vec<u8>,
               pixel_aspect: (u32, u32), palette: Vec<(u8,u8,u8)>, format: Format }
```

---

## File Structure

**`play198x/play198x`**

| File | Responsibility |
|---|---|
| `Cargo.toml` | gains `exclude = ["crates/play198x-web"]` |
| `crates/play198x-web/Cargo.toml` | the standalone shell manifest |
| `crates/play198x-web/src/lib.rs` | the whole boundary — `probe`, `decode_image`, and the two returned types |
| `crates/play198x-web/tests/boundary.rs` | `wasm-bindgen-test` suite |
| `.github/workflows/ci.yml` | gains a job building both targets |

**`code198x/website`**

| File | Responsibility |
|---|---|
| `src/lib/png.ts` | RGBA → PNG bytes. Nothing else. |
| `src/lib/png.test.ts` | `node --test` suite for the writer |
| `src/lib/native-image.ts` | resolve, decode, size — the component's logic, testable without Astro |
| `src/lib/native-image.test.ts` | `node --test` suite for resolution, sizing and every failure |
| `src/components/NativeImage.astro` | the component: props in, `<img>` out |
| `tests/native-image.spec.ts` | Playwright: an independent browser decodes our PNG |
| `.github/workflows/deploy.yml` | checkout play198x, install Rust, build the wasm |
| `scripts/build-wasm.mjs` | one command both CI and local dev use |

---

### Task 1: The crate, and `probe` across the boundary

**Files:**
- Create: `crates/play198x-web/Cargo.toml`, `crates/play198x-web/src/lib.rs`, `crates/play198x-web/tests/boundary.rs`
- Modify: `Cargo.toml` (workspace `exclude`), `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: `play198x_core::probe::{identify, Format, Confidence}`
- Produces: `probe(bytes: &[u8]) -> Option<Probed>` where `Probed` has string getters `format()` and `confidence()`; format strings are `"scr" | "koala" | "art-studio" | "ilbm" | "protracker"`, confidence strings are `"certain" | "probable"`.

- [ ] **Step 1: Exclude the new crate from the workspace**

`play198x-web` is standalone so its wasm dependency tree and lints stay out of the library — the arrangement `cat198x` uses for its Tauri UI (`exclude = ["ui"]`).

In `Cargo.toml`, after `members`:

```toml
# `crates/play198x-web` is a standalone wasm-bindgen shell. It is excluded so its
# wasm toolchain and its `unsafe_code` allowance (see that crate's manifest) stay
# out of the library, which forbids unsafe outright.
exclude = ["crates/play198x-web"]
```

- [ ] **Step 2: Write the manifest**

`crates/play198x-web/Cargo.toml`:

```toml
[package]
name = "play198x-web"
version = "0.1.0"
edition = "2024"
rust-version = "1.98"
license = "GPL-2.0-or-later"
repository = "https://github.com/play198x/play198x"
description = "wasm-bindgen shell over play198x-core, for build-time and browser decoding."
# Not published: a #[wasm_bindgen] surface returning JavaScript values has no
# Rust caller. Anything native uses play198x-core, which is published.
publish = false

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
play198x-core = { path = "../play198x-core" }
wasm-bindgen = "0.2"

[dev-dependencies]
wasm-bindgen-test = "0.3"

[lints.rust]
unused_must_use = "deny"

[lints.clippy]
dbg_macro = "deny"
unwrap_used = "deny"
expect_used = "deny"
```

- [ ] **Step 3: Find out whether `unsafe_code = "deny"` survives the macro**

`#[wasm_bindgen]` expands to glue containing `unsafe`. Whether the lint fires on expanded code decides the manifest, so measure rather than assume.

Add `unsafe_code = "deny"` to `[lints.rust]`, write a trivial `#[wasm_bindgen] pub fn ping() -> u32 { 1 }` in `src/lib.rs`, and run:

```bash
env -u RUSTUP_TOOLCHAIN cargo build -p play198x-web --manifest-path crates/play198x-web/Cargo.toml
```

- **If it builds:** keep `unsafe_code = "deny"`. Done.
- **If it fails on the expansion:** change it to `unsafe_code = "allow"` with this comment, and add the grep in Step 8:

```toml
# wasm-bindgen's expansion contains unsafe glue, so this crate cannot deny the
# lint the way the library does. CI greps our own source instead — see ci.yml.
unsafe_code = "allow"
```

Record which branch you took in the task report.

- [ ] **Step 4: Write the failing boundary test**

`crates/play198x-web/tests/boundary.rs`:

```rust
#![allow(clippy::unwrap_used, clippy::expect_used)]

use wasm_bindgen_test::wasm_bindgen_test;

/// A 6912-byte SCREEN$: every bitmap bit clear, every attribute the same.
/// `attribute` is `FBPPPIII` — see the plan's reference values.
fn screen(attribute: u8) -> Vec<u8> {
    let mut bytes = vec![0u8; 6912];
    bytes[6144..].fill(attribute);
    bytes
}

#[wasm_bindgen_test]
fn a_screen_dollar_is_identified_certainly() {
    let probed = play198x_web::probe(&screen(0x28)).expect("6912 bytes is a SCREEN$");
    assert_eq!(probed.format(), "scr");
    assert_eq!(probed.confidence(), "certain");
}

#[wasm_bindgen_test]
fn nothing_at_all_is_not_a_format() {
    assert!(play198x_web::probe(&[]).is_none());
    assert!(play198x_web::probe(&[0u8; 3]).is_none());
}
```

- [ ] **Step 5: Run it and watch it fail**

```bash
env -u RUSTUP_TOOLCHAIN cargo test --manifest-path crates/play198x-web/Cargo.toml
```

Expected: FAIL — `probe` not found in `play198x_web`.

- [ ] **Step 6: Write `probe`**

`crates/play198x-web/src/lib.rs`:

```rust
//! A `wasm-bindgen` shell over `play198x-core`.
//!
//! The surface is deliberately dumb: bytes in, data out. No PNG encoding and no
//! canvas work happens here, because the build-time consumer wants PNG bytes and
//! a browser consumer wants `putImageData` — and neither may grow logic the
//! other would have to reimplement.

use play198x_core::probe::{Confidence, Format};
use wasm_bindgen::prelude::*;

/// What `probe` found.
#[wasm_bindgen]
pub struct Probed {
    format: String,
    confidence: String,
}

#[wasm_bindgen]
impl Probed {
    /// One of `scr`, `koala`, `art-studio`, `ilbm`, `protracker`.
    #[wasm_bindgen(getter)]
    #[must_use]
    pub fn format(&self) -> String {
        self.format.clone()
    }

    /// `certain` or `probable`.
    #[wasm_bindgen(getter)]
    #[must_use]
    pub fn confidence(&self) -> String {
        self.confidence.clone()
    }
}

/// The format's stable name across the boundary.
///
/// A string, not a discriminant: `Format` is `#[non_exhaustive]`, so a number
/// would shift silently when the core gains a format — and a build-time decode
/// that picks the wrong decoder produces a plausible wrong picture rather than
/// an error.
fn format_name(format: Format) -> &'static str {
    match format {
        Format::Scr => "scr",
        Format::Koala => "koala",
        Format::ArtStudio => "art-studio",
        Format::Ilbm => "ilbm",
        Format::ProTracker => "protracker",
        // `Format` is #[non_exhaustive]: a new variant must be named here
        // before it can cross, rather than crossing as something wrong.
        _ => "unknown",
    }
}

/// Identify `bytes`. Returns `null` in JavaScript when nothing matches.
#[wasm_bindgen]
#[must_use]
pub fn probe(bytes: &[u8]) -> Option<Probed> {
    let (format, confidence) = play198x_core::probe::identify(bytes)?;
    Some(Probed {
        format: format_name(format).to_owned(),
        confidence: match confidence {
            Confidence::Certain => "certain",
            Confidence::Probable => "probable",
        }
        .to_owned(),
    })
}
```

- [ ] **Step 7: Run the test and watch it pass**

```bash
env -u RUSTUP_TOOLCHAIN rustc --version
env -u RUSTUP_TOOLCHAIN cargo test --manifest-path crates/play198x-web/Cargo.toml
```

Expected: 2 passed.

- [ ] **Step 8: Add CI coverage for the excluded crate**

`cargo test --workspace` does **not** reach an excluded crate. Without this the shell has no CI at all.

Add to `.github/workflows/ci.yml` a `shell` job mirroring the existing `clippy` job's toolchain and cache steps, then:

```yaml
      - name: Install wasm-pack
        uses: taiki-e/install-action@v2
        with:
          tool: wasm-pack

      - name: Test the shell
        run: cargo test --manifest-path crates/play198x-web/Cargo.toml

      - name: Clippy the shell
        run: cargo clippy --manifest-path crates/play198x-web/Cargo.toml --all-targets -- -D warnings

      # Both targets are built so the browser target cannot rot: it has no
      # consumer yet, and nothing else would notice it breaking.
      - name: Build the nodejs target
        run: wasm-pack build crates/play198x-web --target nodejs --out-dir pkg-node

      - name: Build the web target
        run: wasm-pack build crates/play198x-web --target web --out-dir pkg-web
```

**Only if Step 3 took the `allow` branch**, also add:

```yaml
      - name: No unsafe in our own source
        run: |
          if grep -rn '\bunsafe\b' crates/play198x-web/src/; then
            echo "::error::unsafe found in play198x-web source"
            exit 1
          fi
```

- [ ] **Step 9: Commit**

```bash
git add Cargo.toml crates/play198x-web .github/workflows/ci.yml
git commit -m "feat: identify a file's format across the wasm boundary"
```

---

### Task 2: `decode_image` across the boundary

**Files:**
- Modify: `crates/play198x-web/src/lib.rs`, `crates/play198x-web/tests/boundary.rs`

**Interfaces:**
- Consumes: `play198x_core::decode::{image, Image}`, `Probed` from Task 1
- Produces: `decode_image(bytes: &[u8], format: &str) -> Result<DecodedImage, JsError>` with getters `width() -> u32`, `height() -> u32`, `rgba() -> Vec<u8>`, `pixel_aspect_w() -> u32`, `pixel_aspect_h() -> u32`, `palette() -> Vec<u8>` (flat RGB triples).

- [ ] **Step 1: Write the failing tests**

Append to `crates/play198x-web/tests/boundary.rs`:

```rust
/// Attribute `0x28` is `0b0_0_101_000`: FLASH off, BRIGHT off, PAPER 5 (cyan),
/// INK 0 (black). Every bitmap bit is clear, so every pixel takes PAPER.
const PAPER_CYAN_INK_BLACK: u8 = 0x28;

/// `0x68` is the same with BRIGHT set, which moves both INK and PAPER into the
/// palette's upper half together.
const BRIGHT_PAPER_CYAN: u8 = 0x68;

fn pixel(rgba: &[u8], x: usize, y: usize, width: usize) -> (u8, u8, u8, u8) {
    let i = (y * width + x) * 4;
    (rgba[i], rgba[i + 1], rgba[i + 2], rgba[i + 3])
}

#[wasm_bindgen_test]
fn a_clear_screen_is_its_paper_colour_everywhere() {
    let decoded = play198x_web::decode_image(&screen(PAPER_CYAN_INK_BLACK), "scr").unwrap();

    assert_eq!(decoded.width(), 256);
    assert_eq!(decoded.height(), 192);

    let rgba = decoded.rgba();
    assert_eq!(rgba.len(), 256 * 192 * 4);

    // mediaspec198x emu198x-v1, index 5: cyan at the normal 0xC2 level.
    assert_eq!(pixel(&rgba, 0, 0, 256), (0x00, 0xC2, 0xC2, 0xFF));
    assert_eq!(pixel(&rgba, 255, 191, 256), (0x00, 0xC2, 0xC2, 0xFF));
}

#[wasm_bindgen_test]
fn a_set_bit_takes_ink_and_its_neighbour_does_not() {
    let mut bytes = screen(PAPER_CYAN_INK_BLACK);
    // Bitmap byte 0 is the leftmost eight pixels of row 0; bit 7 is x = 0.
    bytes[0] = 0x80;

    let decoded = play198x_web::decode_image(&bytes, "scr").unwrap();
    let rgba = decoded.rgba();

    assert_eq!(pixel(&rgba, 0, 0, 256), (0x00, 0x00, 0x00, 0xFF), "INK 0 is black");
    assert_eq!(pixel(&rgba, 1, 0, 256), (0x00, 0xC2, 0xC2, 0xFF), "PAPER 5 is cyan");
}

#[wasm_bindgen_test]
fn bright_selects_the_upper_half_of_the_palette() {
    let decoded = play198x_web::decode_image(&screen(BRIGHT_PAPER_CYAN), "scr").unwrap();
    // Index 13: bright cyan at 0xFF, not the 0xC2 of index 5.
    assert_eq!(pixel(&decoded.rgba(), 0, 0, 256), (0x00, 0xFF, 0xFF, 0xFF));
}

#[wasm_bindgen_test]
fn the_spectrum_pixel_is_square_and_says_so() {
    let decoded = play198x_web::decode_image(&screen(PAPER_CYAN_INK_BLACK), "scr").unwrap();
    assert_eq!(decoded.pixel_aspect_w(), 1);
    assert_eq!(decoded.pixel_aspect_h(), 1);
}

#[wasm_bindgen_test]
fn the_palette_crosses_whole_and_in_hardware_order() {
    let decoded = play198x_web::decode_image(&screen(PAPER_CYAN_INK_BLACK), "scr").unwrap();
    let palette = decoded.palette();

    assert_eq!(palette.len(), 16 * 3, "sixteen RGB triples");
    assert_eq!(&palette[0..3], &[0x00, 0x00, 0x00], "index 0 black");
    assert_eq!(&palette[15..18], &[0x00, 0xC2, 0xC2], "index 5 cyan");
    assert_eq!(&palette[39..42], &[0x00, 0xFF, 0xFF], "index 13 bright cyan");
}

#[wasm_bindgen_test]
fn a_wrong_format_is_an_error_carrying_the_decoders_words() {
    let err = play198x_web::decode_image(&screen(PAPER_CYAN_INK_BLACK), "koala").unwrap_err();
    let message = format!("{err:?}");
    assert!(!message.is_empty(), "the error must say something: {message}");
}

#[wasm_bindgen_test]
fn an_unknown_format_name_is_an_error_not_a_guess() {
    assert!(play198x_web::decode_image(&screen(PAPER_CYAN_INK_BLACK), "jpeg").is_err());
}
```

Note the palette indices: index 5's triple starts at byte 15, index 13's at byte 39.

- [ ] **Step 2: Run and watch them fail**

```bash
env -u RUSTUP_TOOLCHAIN cargo test --manifest-path crates/play198x-web/Cargo.toml
```

Expected: FAIL — `decode_image` not found.

- [ ] **Step 3: Write `decode_image`**

Append to `crates/play198x-web/src/lib.rs`:

```rust
/// A decoded picture, flattened for JavaScript.
#[wasm_bindgen]
pub struct DecodedImage {
    inner: play198x_core::decode::Image,
}

#[wasm_bindgen]
impl DecodedImage {
    /// Width in mode pixels — not display pixels. See `pixel_aspect_w`.
    #[wasm_bindgen(getter)]
    #[must_use]
    pub fn width(&self) -> u32 {
        self.inner.width
    }

    #[wasm_bindgen(getter)]
    #[must_use]
    pub fn height(&self) -> u32 {
        self.inner.height
    }

    /// Row-major RGBA8, `width * height * 4` bytes, alpha always opaque.
    #[wasm_bindgen(getter)]
    #[must_use]
    pub fn rgba(&self) -> Vec<u8> {
        self.inner.rgba.clone()
    }

    /// Horizontal component of one mode pixel's shape, against the machine's
    /// own single-width pixel. A consumer that ignores this draws a C64
    /// multicolour picture at half its real width.
    #[wasm_bindgen(getter)]
    #[must_use]
    pub fn pixel_aspect_w(&self) -> u32 {
        self.inner.pixel_aspect.0
    }

    #[wasm_bindgen(getter)]
    #[must_use]
    pub fn pixel_aspect_h(&self) -> u32 {
        self.inner.pixel_aspect.1
    }

    /// The picture's colours in hardware index order, flattened to RGB triples.
    ///
    /// Crosses the boundary even though the build-time consumer draws none of
    /// it: it cannot be recovered from the pixels afterwards — a picture that
    /// never uses colour 5 has lost it — and the palette view is the first
    /// interactive figure anyone will ask for.
    #[wasm_bindgen(getter)]
    #[must_use]
    pub fn palette(&self) -> Vec<u8> {
        self.inner
            .palette
            .iter()
            .flat_map(|&(r, g, b)| [r, g, b])
            .collect()
    }
}

/// Parse a format name from the boundary back into the core's enum.
fn format_from_name(name: &str) -> Option<Format> {
    match name {
        "scr" => Some(Format::Scr),
        "koala" => Some(Format::Koala),
        "art-studio" => Some(Format::ArtStudio),
        "ilbm" => Some(Format::Ilbm),
        "protracker" => Some(Format::ProTracker),
        _ => None,
    }
}

/// Decode `bytes` as `format`, which is one of the names [`probe`] returns.
///
/// # Errors
///
/// When `format` is not a name this shell knows, or when the core's decoder
/// rejects the bytes — carrying the core's own message unchanged.
#[wasm_bindgen]
pub fn decode_image(bytes: &[u8], format: &str) -> Result<DecodedImage, JsError> {
    let Some(format) = format_from_name(format) else {
        return Err(JsError::new(&format!("`{format}` is not a format this build knows")));
    };

    play198x_core::decode::image(bytes, format)
        .map(|inner| DecodedImage { inner })
        .map_err(|err| JsError::new(&err.to_string()))
}
```

- [ ] **Step 4: Run the tests and watch them pass**

```bash
env -u RUSTUP_TOOLCHAIN cargo test --manifest-path crates/play198x-web/Cargo.toml
env -u RUSTUP_TOOLCHAIN cargo clippy --manifest-path crates/play198x-web/Cargo.toml --all-targets -- -D warnings
env -u RUSTUP_TOOLCHAIN cargo fmt --manifest-path crates/play198x-web/Cargo.toml --check
```

Expected: 9 passed, clippy clean, fmt clean.

If `the_palette_crosses_whole_and_in_hardware_order` fails on length, report the actual palette length rather than editing the expectation to match — the count is a fact about the machine, not about our code.

- [ ] **Step 5: Build both targets and read what `nodejs` emits**

```bash
wasm-pack build crates/play198x-web --target nodejs --out-dir pkg-node
wasm-pack build crates/play198x-web --target web --out-dir pkg-web
ls crates/play198x-web/pkg-node
head -20 crates/play198x-web/pkg-node/play198x_web.js
```

```bash
cat crates/play198x-web/pkg-node/play198x_web.d.ts
```

**Record three things in the task report**, because Tasks 4 and 5 are written
against them and guessing costs an hour there:

1. Whether the emitted glue is CommonJS or ESM, and the exact filenames.
2. **The exact property names on `Probed` and `DecodedImage` in the `.d.ts`.**
   `wasm-bindgen` may or may not rename `pixel_aspect_w` to `pixelAspectW`
   depending on version and attributes. Task 4's `Wasm` interface uses the
   snake_case form; if the `.d.ts` disagrees, **add explicit
   `#[wasm_bindgen(js_name = "…")]` attributes pinning the snake_case names**
   rather than editing Task 4 to match, so the boundary reads the same from
   both sides.
3. Whether the getters surface as JavaScript properties or methods.

- [ ] **Step 6: Commit**

```bash
git add crates/play198x-web
git commit -m "feat: decode an image to RGBA across the wasm boundary"
```

---

### Task 3: A PNG writer with no dependency

> **Deviation from the spec, declared.** The spec asks for "one byte-for-byte
> comparison against a known-good PNG, checked in". A golden file produced by
> our own encoder only detects change, not correctness — it is circular. This
> task substitutes two checks that are not: **chunk CRCs computed by `node:zlib`
> rather than by us**, and **a real browser decoding our output** in Task 6.
> Chromium is an independent PNG decoder and Playwright is already installed.
> The spec's intent — that the header half is verified by something other than
> the round trip — is met more strongly. Raise it if you disagree.


**Files:**
- Create: `src/lib/png.ts`, `src/lib/png.test.ts` (in `code198x/website`)

**Interfaces:**
- Produces: `encodePng(rgba: Uint8Array, width: number, height: number): Uint8Array`

- [ ] **Step 1: Find out whether `node --test` runs TypeScript here**

Node 24 strips types natively, but confirm rather than assume:

```bash
cd <website>
printf 'import { test } from "node:test";\nimport assert from "node:assert";\ntest("ok", () => { const n: number = 1; assert.equal(n, 1); });\n' > /tmp/probe.test.ts
node --test /tmp/probe.test.ts
```

- **If it passes:** write `.ts` files as this task specifies.
- **If it fails:** write `src/lib/png.mjs` and `src/lib/png.test.mjs` in plain JavaScript instead, dropping the type annotations. Everything else is unchanged.

Record which branch you took. Add `"test:unit": "node --test src/lib/*.test.*"` to `package.json` scripts either way.

- [ ] **Step 2: Write the failing tests**

`src/lib/png.test.ts`:

```ts
import { test } from 'node:test';
import assert from 'node:assert';
import { inflateSync, crc32 } from 'node:zlib';
import { encodePng } from './png.ts';

/** A 2×2 image: red, green / blue, white — all fully opaque. */
function swatch(): Uint8Array {
  return new Uint8Array([
    255, 0, 0, 255,   0, 255, 0, 255,
    0, 0, 255, 255,   255, 255, 255, 255,
  ]);
}

test('the signature is the eight bytes every PNG starts with', () => {
  const png = encodePng(swatch(), 2, 2);
  assert.deepEqual([...png.subarray(0, 8)], [0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a]);
});

test('IHDR declares 8-bit truecolour with alpha, uncompressed and uninterlaced', () => {
  const png = encodePng(swatch(), 2, 2);
  // 8 signature + 4 length + 4 type = byte 16 is the start of IHDR's data.
  const ihdr = png.subarray(16, 29);
  const view = new DataView(ihdr.buffer, ihdr.byteOffset, ihdr.byteLength);

  assert.equal(view.getUint32(0), 2, 'width');
  assert.equal(view.getUint32(4), 2, 'height');
  assert.equal(ihdr[8], 8, 'bit depth');
  assert.equal(ihdr[9], 6, 'colour type 6 = RGBA');
  assert.equal(ihdr[10], 0, 'compression method');
  assert.equal(ihdr[11], 0, 'filter method');
  assert.equal(ihdr[12], 0, 'interlace method');
});

test('every chunk carries the CRC32 zlib computes for it', () => {
  const png = encodePng(swatch(), 2, 2);
  let at = 8;
  let chunks = 0;

  while (at < png.length) {
    const view = new DataView(png.buffer, png.byteOffset + at, 8);
    const length = view.getUint32(0);
    const body = png.subarray(at + 4, at + 8 + length);      // type + data
    const stated = new DataView(png.buffer, png.byteOffset + at + 8 + length, 4).getUint32(0);

    assert.equal(stated, crc32(body) >>> 0, `chunk ${chunks} CRC`);
    at += 12 + length;
    chunks += 1;
  }

  assert.equal(chunks, 3, 'IHDR, IDAT, IEND');
});

test('the IDAT inflates back to the exact scanlines it was given', () => {
  const rgba = swatch();
  const png = encodePng(rgba, 2, 2);

  // Locate IDAT rather than assuming its offset.
  let at = 8;
  let idat: Uint8Array | null = null;
  while (at < png.length) {
    const length = new DataView(png.buffer, png.byteOffset + at, 4).getUint32(0);
    const type = String.fromCharCode(...png.subarray(at + 4, at + 8));
    if (type === 'IDAT') idat = png.subarray(at + 8, at + 8 + length);
    at += 12 + length;
  }
  assert.ok(idat, 'there is an IDAT');

  const raw = new Uint8Array(inflateSync(idat));
  // Each row is one filter byte (0 = None) then its pixels.
  assert.equal(raw.length, 2 * (1 + 2 * 4));
  assert.equal(raw[0], 0, 'row 0 filter is None');
  assert.deepEqual([...raw.subarray(1, 9)], [...rgba.subarray(0, 8)]);
  assert.equal(raw[9], 0, 'row 1 filter is None');
  assert.deepEqual([...raw.subarray(10, 18)], [...rgba.subarray(8, 16)]);
});

test('a byte count that disagrees with the dimensions is refused', () => {
  assert.throws(() => encodePng(new Uint8Array(15), 2, 2), /15/);
});
```

- [ ] **Step 3: Run and watch them fail**

```bash
node --test src/lib/png.test.ts
```

Expected: FAIL — cannot find `./png.ts`.

- [ ] **Step 4: Write the encoder**

`src/lib/png.ts`:

```ts
/**
 * RGBA to PNG, using only `node:zlib`.
 *
 * A PNG encoder is not worth a dependency: this is IHDR + IDAT + IEND with
 * filter type 0, and Node ships the deflate. Hand-written binary formats are
 * where quiet corruption lives, so `png.test.ts` verifies the output three
 * ways — structure, chunk CRCs, and an inflate that must reproduce the exact
 * scanlines — and `tests/native-image.spec.ts` has a real browser decode one.
 */
import { deflateSync, crc32 } from 'node:zlib';

const SIGNATURE = Uint8Array.from([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a]);

function chunk(type: string, data: Uint8Array): Uint8Array {
  const body = new Uint8Array(4 + data.length);
  for (let i = 0; i < 4; i += 1) body[i] = type.charCodeAt(i);
  body.set(data, 4);

  const out = new Uint8Array(12 + data.length);
  const view = new DataView(out.buffer);
  view.setUint32(0, data.length);
  out.set(body, 4);
  view.setUint32(8 + data.length, crc32(body) >>> 0);
  return out;
}

/** Encode row-major RGBA8 as a PNG. */
export function encodePng(rgba: Uint8Array, width: number, height: number): Uint8Array {
  const expected = width * height * 4;
  if (rgba.length !== expected) {
    throw new Error(
      `${width}×${height} RGBA needs ${expected} bytes, got ${rgba.length}`,
    );
  }

  const header = new Uint8Array(13);
  const headerView = new DataView(header.buffer);
  headerView.setUint32(0, width);
  headerView.setUint32(4, height);
  header[8] = 8;   // bit depth
  header[9] = 6;   // colour type: truecolour with alpha
  header[10] = 0;  // compression: deflate
  header[11] = 0;  // filter: adaptive
  header[12] = 0;  // interlace: none

  // One filter byte per row, always 0 (None). Filtering would shrink the file;
  // these are small, and an unfiltered stream is one less thing to get wrong.
  const stride = width * 4;
  const raw = new Uint8Array(height * (1 + stride));
  for (let y = 0; y < height; y += 1) {
    raw[y * (1 + stride)] = 0;
    raw.set(rgba.subarray(y * stride, (y + 1) * stride), y * (1 + stride) + 1);
  }

  const ihdr = chunk('IHDR', header);
  const idat = chunk('IDAT', new Uint8Array(deflateSync(raw, { level: 9 })));
  const iend = chunk('IEND', new Uint8Array(0));

  const png = new Uint8Array(
    SIGNATURE.length + ihdr.length + idat.length + iend.length,
  );
  let at = 0;
  for (const part of [SIGNATURE, ihdr, idat, iend]) {
    png.set(part, at);
    at += part.length;
  }
  return png;
}
```

- [ ] **Step 5: Run the tests and watch them pass**

```bash
node --test src/lib/png.test.ts
```

Expected: 5 passed.

- [ ] **Step 6: Commit**

```bash
git add src/lib/png.ts src/lib/png.test.ts package.json
git commit -m "feat: encode RGBA to PNG using only node:zlib"
```

---

### Task 4: The component's logic

**Files:**
- Create: `src/lib/native-image.ts`, `src/lib/native-image.test.ts`

**Interfaces:**
- Consumes: `encodePng` from Task 3; the wasm module's `probe` and `decode_image` from Tasks 1–2
- Produces:
  - `displaySize(width, height, aspectW, aspectH): { width: number; height: number }`
  - `resolveSource(src: string, codeSamplesPath: string): string`
  - `renderNativeImage(options): Promise<{ dataUri: string; width: number; height: number }>`

Keeping this out of the `.astro` file is what makes every failure below testable without rendering a page.

- [ ] **Step 1: Write the failing tests**

`src/lib/native-image.test.ts`:

```ts
import { test } from 'node:test';
import assert from 'node:assert';
import { displaySize, resolveSource, MAX_PNG_BYTES } from './native-image.ts';

test('a square pixel displays at its mode size', () => {
  // ZX Spectrum standard: 256×192 mode pixels, 1:1.
  assert.deepEqual(displaySize(256, 192, 1, 1), { width: 256, height: 192 });
});

test('a double-wide pixel displays twice as wide', () => {
  // C64 multicolour bitmap: 160×200 mode pixels at 2:1 is 320×200 on screen.
  // Getting this wrong draws the picture at half its real width.
  assert.deepEqual(displaySize(160, 200, 2, 1), { width: 320, height: 200 });
});

test('a source resolves under the code-samples checkout', () => {
  assert.equal(
    resolveSource('sinclair-zx-spectrum/assembly/gloaming/loading-screen/gloaming.scr', '/tmp/cs'),
    '/tmp/cs/sinclair-zx-spectrum/assembly/gloaming/loading-screen/gloaming.scr',
  );
});

test('a source cannot escape the code-samples checkout', () => {
  assert.throws(
    () => resolveSource('../../../etc/passwd', '/tmp/cs'),
    /outside/,
    'a traversing path must be refused, not resolved',
  );
});

test('the PNG budget is stated, not implied', () => {
  assert.equal(MAX_PNG_BYTES, 96 * 1024);
});
```

- [ ] **Step 2: Run and watch them fail**

```bash
node --test src/lib/native-image.test.ts
```

Expected: FAIL — cannot find `./native-image.ts`.

- [ ] **Step 3: Write the sizing and resolution**

`src/lib/native-image.ts`:

```ts
/**
 * The logic behind `NativeImage.astro`, kept out of the component so every
 * failure it can produce is testable without rendering a page.
 */
import { readFile } from 'node:fs/promises';
import path from 'node:path';
import { encodePng } from './png.ts';

/**
 * The largest PNG that may be inlined as a data URI.
 *
 * A Spectrum SCREEN$ and a C64 bitmap encode to a few kilobytes, and a 320×256
 * Amiga ILBM stays well inside this. A hi-res interlaced HAM ILBM will exceed
 * it — and that is this budget working: such a picture should not be inlined
 * into every reader's HTML, and the build should say so by name rather than
 * quietly shipping a third of a megabyte of base64.
 */
export const MAX_PNG_BYTES = 96 * 1024;

/** Mode pixels to display pixels, honouring the mode's pixel shape. */
export function displaySize(
  width: number,
  height: number,
  aspectW: number,
  aspectH: number,
): { width: number; height: number } {
  return { width: width * aspectW, height: height * aspectH };
}

/** Resolve `src` inside the code-samples checkout, refusing anything outside it. */
export function resolveSource(src: string, codeSamplesPath: string): string {
  const root = path.resolve(codeSamplesPath);
  const resolved = path.resolve(root, src);
  if (resolved !== root && !resolved.startsWith(root + path.sep)) {
    throw new Error(`\`${src}\` resolves outside the code-samples checkout`);
  }
  return resolved;
}
```

- [ ] **Step 4: Run and watch them pass**

```bash
node --test src/lib/native-image.test.ts
```

Expected: 5 passed.

- [ ] **Step 5: Write the failing tests for rendering and its failures**

Append to `src/lib/native-image.test.ts`:

```ts
import { renderNativeImage } from './native-image.ts';
import { mkdtemp, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';

/** A 6912-byte SCREEN$: clear bitmap, attribute 0x28 = PAPER cyan, INK black. */
function screen(attribute = 0x28): Uint8Array {
  const bytes = new Uint8Array(6912);
  bytes.fill(attribute, 6144);
  return bytes;
}

async function fixture(name: string, bytes: Uint8Array): Promise<string> {
  const dir = await mkdtemp(path.join(tmpdir(), 'native-image-'));
  await writeFile(path.join(dir, name), bytes);
  return dir;
}

test('a SCREEN$ renders to an inline PNG at its mode size', async () => {
  const dir = await fixture('a.scr', screen());
  const result = await renderNativeImage({ src: 'a.scr', codeSamplesPath: dir });

  assert.equal(result.width, 256);
  assert.equal(result.height, 192);
  assert.match(result.dataUri, /^data:image\/png;base64,/);
});

test('a missing file names the file', async () => {
  const dir = await fixture('a.scr', screen());
  await assert.rejects(
    renderNativeImage({ src: 'absent.scr', codeSamplesPath: dir }),
    /absent\.scr/,
  );
});

test('bytes nothing recognises are refused by name', async () => {
  const dir = await fixture('a.scr', new Uint8Array([1, 2, 3]));
  await assert.rejects(
    renderNativeImage({ src: 'a.scr', codeSamplesPath: dir }),
    /a\.scr/,
  );
});

test('a module is not an image, and says so', async () => {
  // 1084 bytes of zeros then "M.K." is a ProTracker module's signature position.
  const mod = new Uint8Array(1084 + 4);
  mod.set([0x4d, 0x2e, 0x4b, 0x2e], 1080);
  const dir = await fixture('a.mod', mod);
  await assert.rejects(
    renderNativeImage({ src: 'a.mod', codeSamplesPath: dir }),
    /not an image|protracker/i,
  );
});

test('a weak identification must be declared by the author', async () => {
  // Art Studio has no magic number: probing it returns Probable, and a wrong
  // call shows a wrong-looking picture rather than raising an error. The author
  // has to say so, so a misprobe cannot ship quietly.
  const dir = await fixture('a.art', new Uint8Array(9009));
  await assert.rejects(
    renderNativeImage({ src: 'a.art', codeSamplesPath: dir }),
    /probable|declare/i,
  );
});

test('a declared format that contradicts a certain probe is a failure', async () => {
  const dir = await fixture('a.scr', screen());
  await assert.rejects(
    renderNativeImage({ src: 'a.scr', codeSamplesPath: dir, format: 'koala' }),
    /scr.*koala|koala.*scr/is,
  );
});
```

- [ ] **Step 6: Run and watch them fail**

```bash
node --test src/lib/native-image.test.ts
```

Expected: FAIL — `renderNativeImage` is not exported.

- [ ] **Step 7: Write `renderNativeImage`**

Append to `src/lib/native-image.ts`. Use the loader shape Task 2 Step 5 recorded — this is the CommonJS form; if the glue is ESM, use a plain `import` instead:

```ts
import { createRequire } from 'node:module';

const require_ = createRequire(import.meta.url);

interface Wasm {
  probe(bytes: Uint8Array): { format: string; confidence: string } | undefined;
  decode_image(bytes: Uint8Array, format: string): {
    width: number; height: number; rgba: Uint8Array;
    pixel_aspect_w: number; pixel_aspect_h: number;
  };
}

let wasm: Wasm | null = null;

function load(): Wasm {
  if (wasm) return wasm;
  const dir = process.env.PLAY198X_WASM_PATH;
  if (!dir) {
    throw new Error(
      'PLAY198X_WASM_PATH is unset: run scripts/build-wasm.mjs before building the site',
    );
  }
  wasm = require_(path.join(dir, 'play198x_web.js')) as Wasm;
  return wasm;
}

/** Formats this component will render. Anything else is not an image. */
const IMAGE_FORMATS = new Set(['scr', 'koala', 'art-studio', 'ilbm']);

export interface NativeImageOptions {
  src: string;
  codeSamplesPath: string;
  /** Required when probing is not `certain`. */
  format?: string;
}

export async function renderNativeImage(
  options: NativeImageOptions,
): Promise<{ dataUri: string; width: number; height: number }> {
  const { src, codeSamplesPath, format: declared } = options;
  const file = resolveSource(src, codeSamplesPath);

  let bytes: Uint8Array;
  try {
    bytes = new Uint8Array(await readFile(file));
  } catch {
    throw new Error(`\`${src}\` is not in the code-samples checkout at ${codeSamplesPath}`);
  }

  const probed = load().probe(bytes);
  if (!probed) {
    throw new Error(`\`${src}\` is not a format this build recognises`);
  }
  if (!IMAGE_FORMATS.has(probed.format)) {
    throw new Error(`\`${src}\` is a ${probed.format}, which is not an image`);
  }

  if (probed.confidence !== 'certain' && !declared) {
    throw new Error(
      `\`${src}\` probes as ${probed.format} only probably — nothing downstream ` +
        `can catch a miss, so declare it: format="${probed.format}"`,
    );
  }
  if (declared && probed.confidence === 'certain' && declared !== probed.format) {
    throw new Error(
      `\`${src}\` is certainly a ${probed.format}, but format="${declared}" was declared`,
    );
  }

  const image = load().decode_image(bytes, declared ?? probed.format);
  const png = encodePng(image.rgba, image.width, image.height);

  if (png.length > MAX_PNG_BYTES) {
    throw new Error(
      `\`${src}\` encodes to ${png.length} bytes, past the ${MAX_PNG_BYTES}-byte ` +
        `inline budget — it wants an emitted asset file, which this build does not do yet`,
    );
  }

  const size = displaySize(image.width, image.height, image.pixel_aspect_w, image.pixel_aspect_h);
  return {
    dataUri: `data:image/png;base64,${Buffer.from(png).toString('base64')}`,
    ...size,
  };
}
```

- [ ] **Step 8: Run and watch them pass**

```bash
PLAY198X_WASM_PATH=<play198x>/crates/play198x-web/pkg-node node --test src/lib/native-image.test.ts
```

Expected: 11 passed.

If `a weak identification must be declared` fails because 9009 zero bytes do not probe as Art Studio, find the length the core's Art Studio probe accepts by reading `play198x-core`'s `probe.rs`, and use that — do not weaken the assertion to make it pass.

- [ ] **Step 9: Commit**

```bash
git add src/lib/native-image.ts src/lib/native-image.test.ts
git commit -m "feat: decode a code-samples image to an inline PNG at build time"
```

---

### Task 5: The component, and the build that feeds it

**Files:**
- Create: `src/components/NativeImage.astro`, `scripts/build-wasm.mjs`
- Modify: `.github/workflows/deploy.yml`, `.github/workflows/ci.yml`, `package.json`

**Interfaces:**
- Consumes: `renderNativeImage` from Task 4
- Produces: `<NativeImage src alt title? source? format? />`

- [ ] **Step 1: Write the wasm build script**

`scripts/build-wasm.mjs`:

```js
/**
 * Build play198x-web's nodejs target, which NativeImage decodes with.
 *
 * One script so CI and a local `npm run dev` build it the same way. PLAY198X_PATH
 * points at a play198x checkout; deploy.yml provides one, and a developer sets it
 * to their own.
 */
import { execFileSync } from 'node:child_process';
import path from 'node:path';

const root = process.env.PLAY198X_PATH;
if (!root) {
  console.error('PLAY198X_PATH is unset — point it at a play198x checkout.');
  process.exit(1);
}

const crate = path.join(root, 'crates', 'play198x-web');
execFileSync('wasm-pack', ['build', crate, '--target', 'nodejs', '--out-dir', 'pkg-node'], {
  stdio: 'inherit',
});
console.log(path.join(crate, 'pkg-node'));
```

Add to `package.json` scripts:

```json
"build:wasm": "node scripts/build-wasm.mjs"
```

- [ ] **Step 2: Write the component**

`src/components/NativeImage.astro`:

```astro
---
/**
 * NativeImage — a retro image file decoded at build time by play198x-core.
 *
 * The reader receives a plain <img>: no wasm, no script, no decoding. A decoded
 * SCREEN$ never changes, so there is nothing for the browser to do that the
 * build cannot do once.
 *
 *   <NativeImage src="sinclair-zx-spectrum/assembly/gloaming/loading-screen/gloaming.scr"
 *                alt="The Gloaming loading screen" />
 *
 * `src` is relative to the code-samples checkout. Every failure stops the build
 * and names the file: a red build is cheap, a wrong picture across the site is not.
 */
import { renderNativeImage } from '../lib/native-image.ts';

interface Props {
  src: string;
  alt: string;
  title?: string;
  source?: string;
  format?: string;
}

const { src, alt, title, source, format } = Astro.props;

const codeSamplesPath = process.env.CODE_SAMPLES_PATH;
if (!codeSamplesPath) {
  throw new Error('CODE_SAMPLES_PATH is unset — NativeImage needs the code-samples checkout');
}

const image = await renderNativeImage({ src, codeSamplesPath, format });
---

<figure class="native-image">
  <img
    src={image.dataUri}
    width={image.width}
    height={image.height}
    alt={alt}
    loading="lazy"
    decoding="async"
  />
  {(title || source) && (
    <figcaption>
      {title}
      {source && <span class="source">{source}</span>}
    </figcaption>
  )}
</figure>

<style>
  /* Native pixels, scaled without smoothing: these are pixel-art artefacts and
     interpolation invents detail the machine never drew. */
  .native-image img {
    image-rendering: pixelated;
    max-width: 100%;
    height: auto;
  }
</style>
```

- [ ] **Step 3: Wire the wasm into the site build**

In `.github/workflows/deploy.yml`, after the code-samples checkout:

```yaml
      - name: Checkout play198x
        uses: actions/checkout@v7
        with:
          repository: play198x/play198x
          path: play198x

      - name: Read pinned toolchain
        id: toolchain
        run: echo "channel=$(grep '^channel' play198x/rust-toolchain.toml | cut -d'"' -f2)" >> "$GITHUB_OUTPUT"

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ steps.toolchain.outputs.channel }}
          targets: wasm32-unknown-unknown

      - name: Cache Rust build artifacts
        uses: Swatinem/rust-cache@v2
        with:
          workspaces: play198x/crates/play198x-web

      - name: Install wasm-pack
        uses: taiki-e/install-action@v2
        with:
          tool: wasm-pack

      # Before npm run build, so a Rust failure is attributed to Rust rather
      # than surfacing as a mystery inside the Astro build.
      - name: Build the decoder
        run: node scripts/build-wasm.mjs
        env:
          PLAY198X_PATH: ${{ github.workspace }}/play198x
```

and add to the existing `Build site` step's `env`:

```yaml
          PLAY198X_WASM_PATH: ${{ github.workspace }}/play198x/crates/play198x-web/pkg-node
```

Make the same additions to `.github/workflows/ci.yml` wherever it builds the site.

- [ ] **Step 4: Verify the site builds**

```bash
PLAY198X_PATH=<play198x> npm run build:wasm
CODE_SAMPLES_PATH=<code-samples> PLAY198X_WASM_PATH=<play198x>/crates/play198x-web/pkg-node npm run build
```

Expected: the build completes. Nothing uses the component yet, so this proves only that adding it broke nothing.

- [ ] **Step 5: Commit**

```bash
git add src/components/NativeImage.astro scripts/build-wasm.mjs package.json .github/workflows/
git commit -m "feat: add NativeImage, decoding retro image files during the build"
```

---

### Task 6: Prove it in a browser, and use it

**Files:**
- Create: `tests/native-image.spec.ts`
- Create: `src/content/vault/…` usage, or modify an existing page (see Step 3)

**Interfaces:**
- Consumes: everything above

- [ ] **Step 1: Write the browser-decode test**

Our PNG writer is checked by our own code everywhere else. Chromium is an independent decoder, and Playwright is already installed — so this is the one test that could catch an encoder bug all the structural checks share a blind spot for.

`tests/native-image.spec.ts`:

```ts
import { test, expect } from '@playwright/test';
import { renderNativeImage } from '../src/lib/native-image.ts';

test('a browser decodes our PNG to the pixels the core produced', async ({ page }) => {
  const image = await renderNativeImage({
    src: 'sinclair-zx-spectrum/assembly/gloaming/loading-screen/gloaming.scr',
    codeSamplesPath: process.env.CODE_SAMPLES_PATH!,
  });

  expect(image.width).toBe(256);
  expect(image.height).toBe(192);

  const pixel = await page.evaluate(async (dataUri) => {
    const img = new Image();
    img.src = dataUri;
    await img.decode();

    const canvas = document.createElement('canvas');
    canvas.width = img.width;
    canvas.height = img.height;
    const context = canvas.getContext('2d')!;
    context.drawImage(img, 0, 0);

    return {
      width: img.width,
      height: img.height,
      topLeft: [...context.getImageData(0, 0, 1, 1).data],
    };
  }, image.dataUri);

  expect(pixel.width).toBe(256);
  expect(pixel.height).toBe(192);
  expect(pixel.topLeft[3]).toBe(255);   // opaque, as every one of these formats is
});
```

- [ ] **Step 2: Run it**

```bash
CODE_SAMPLES_PATH=<code-samples> PLAY198X_WASM_PATH=<play198x>/crates/play198x-web/pkg-node npx playwright test tests/native-image.spec.ts
```

Expected: PASS. A failure here means the encoder is wrong in a way our own checks agree about — investigate the encoder, not the test.

- [ ] **Step 3: Use it on a real page**

Find the page that discusses the Gloaming loading screen:

```bash
grep -rln "loading screen\|loading-screen" src/content/curriculum src/content/vault
```

Add to the most relevant one:

```mdx
<NativeImage
  src="sinclair-zx-spectrum/assembly/gloaming/loading-screen/gloaming.scr"
  alt="The Gloaming loading screen: a walled square at dusk, drawn in the Spectrum's two-colours-per-cell grid"
  title="gloaming.scr — the 6912 bytes the tape carries"
  source="Code198x"
/>
```

**Do not delete `loading-screen.png`.** It is `compose.py`'s output and the `.scr`'s input, not a duplicate — the README documents the conversion. If no suitable page exists, say so in the task report and leave the component unused rather than inventing a page for it.

- [ ] **Step 4: Build and look at the result**

```bash
CODE_SAMPLES_PATH=<code-samples> PLAY198X_WASM_PATH=<...>/pkg-node npm run build
npm run preview
```

Open the page and confirm the picture renders, is sharp rather than smoothed, and is 256×192.

- [ ] **Step 5: Commit**

```bash
git add tests/native-image.spec.ts src/content/
git commit -m "feat: show the Gloaming loading screen from its own SCREEN\$"
```

---

## What this plan does not build

- **Any interactivity** — attribute-grid toggle, palette view, load animation. The boundary carries the palette; nothing renders it.
- **Audio.** `AudioClip` works today with pre-rendered clips.
- **The drop-target playground.** The `web` target is built by CI and used by nothing.
- **Emitted asset files.** Over-budget PNGs fail the build by name; that is the trigger, and it is a change to one function.
- **npm publication.** Deferred with two named triggers — see the spec.
- **Retiring `AudioClip`, `Figure` or `Photo`.**
