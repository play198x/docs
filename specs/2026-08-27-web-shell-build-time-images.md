# `play198x-web`: decoding retro images at build time

**Status:** Proposed 2026-08-27. Not started.

**Goal:** Let a Code198x page show a `SCREEN$`, Koala, Art Studio or ILBM file
**decoded by the same code the player uses**, rather than a screenshot of one —
with no wasm, no JavaScript and no decoding work reaching the reader.

**Depends on:** [`play198x-core`](https://crates.io/crates/play198x-core) 0.1.0,
published. This spec adds no capability to the core and changes none of it.

**Also binding:**
[`2026-08-25-data-driven-core-design.md`](2026-08-25-data-driven-core-design.md)
(amended below),
[`play198x-media-player.md`](../../../decisions/play198x-media-player.md),
[`shared-media-spec.md`](../../../decisions/shared-media-spec.md).

---

## Why build time, and not the browser

The site is static and deploys to GitHub Pages, so it cannot set COOP/COEP
headers and has no `SharedArrayBuffer`. That rules out the usual worker designs
outright — but for still images it does not matter, because the stronger
argument is simpler:

**A decoded `SCREEN$` never changes.** Decoding it in the browser ships a wasm
bundle, the source file and a script in order to produce a picture that is the
same on every load. Decoding it in Node during the build produces a PNG the
browser already knows how to draw: nothing to download, nothing to execute,
works with JavaScript off, and correct in an RSS reader.

`AudioClip` already establishes this shape — it reads a 16-bit PCM WAV at build
time and draws the real amplitude envelope, falling back to a decorative pattern
only for formats it cannot read.

What build time cannot do is interact: no attribute-grid toggle, no palette
view, no scrubbing a bitmap as it loads. Those are real curriculum ideas and
they are **out of this slice**, held reachable by the two-target rule below.

## The crate

`play198x-web`, in `play198x/play198x`, a standalone crate **excluded from the
workspace** — the arrangement the core design already sets for shells, so the
library keeps its own lints and profiles.

It builds to two `wasm-pack` targets from **one** Rust surface:

| Target | Built by | Consumer |
|---|---|---|
| `nodejs` | CI and the website build | this slice |
| `web` | CI only | none yet — built so it cannot rot |

**The two-target rule.** Whatever the build-time path needs, the crate exposes
as data; whatever a browser path would need, it exposes as the *same* data.
Neither target may gain logic the other would have to reimplement. This is what
keeps the interactive figure a `--target` flag and a component rewrite, rather
than a second decoder.

### The boundary

Deliberately dumb. No PNG in Rust, no canvas in Rust:

```rust
/// Returns null when nothing matches.
#[wasm_bindgen]
pub fn probe(bytes: &[u8]) -> Option<Probed>;   // { format: string, confidence: string }

/// `format` is one of the strings `probe` returns.
#[wasm_bindgen]
pub fn decode_image(bytes: &[u8], format: &str) -> Result<DecodedImage, JsError>;

#[wasm_bindgen]
pub struct DecodedImage {
    // width, height, rgba, pixel_aspect_w, pixel_aspect_h, palette
}
```

`rgba` is row-major RGBA8, `width * height * 4` bytes, alpha always opaque.
Errors carry `play198x_core::Error`'s own message unchanged — the shell invents
no wording of its own.

**`format` crosses as a string, not a number.** `probe::Format` is
`#[non_exhaustive]`; a numeric discriminant would silently shift when the core
adds a format, and a build-time decode that picks the wrong decoder produces a
plausible wrong picture rather than an error.

**`pixel_aspect` is not optional.** `decode::Image` carries it and its own
documentation says why: *"a shell that ignores this draws a C64 multicolour
picture at half its real width."* Ignoring it is a defect, not a simplification.

**`palette` crosses the boundary in this slice even though nothing renders it**,
because the core keeps it for a reason that applies here: it cannot be recovered
from the pixels afterwards. A picture that never uses colour 5 has lost it. The
palette view is the first interactive figure anyone will ask for, and the
boundary should not need reopening to serve it.

## PNG, with no new dependency

Node has `zlib` built in, and a PNG is IHDR + IDAT + IEND with filter type 0 —
about fifty lines. The family constraint is to keep dependencies limited, and a
PNG encoder is not worth one.

Hand-written binary formats are exactly where quiet corruption lives, so it is
verified rather than reviewed:

1. **Round trip.** Inflate the emitted IDAT and assert it equals the exact
   scanlines fed in, byte for byte. Uses `node:zlib` alone.
2. **Fixture.** One byte-for-byte comparison against a known-good PNG, checked
   in, which catches header mistakes the round trip cannot see.

The round trip alone would pass on a file with a wrong bit depth or colour type.
Both checks are required.

## The component

`src/components/NativeImage.astro` — "the image as the machine stored it". It
sits with `Figure`, `Photo` and `AudioClip` in the existing vocabulary, and does
not favour one machine's format the way a name with *screen* in it would.

```astro
<NativeImage src="sinclair-zx-spectrum/assembly/gloaming/loading-screen/gloaming.scr"
             alt="The Gloaming loading screen: …"
             title="gloaming.scr — the loading screen the unit builds"
             source="Code198x" />
```

| Prop | Required | Meaning |
|---|---|---|
| `src` | yes | path to the file, relative to `CODE_SAMPLES_PATH` |
| `alt` | yes | alternative text; a picture of a game screen is not decoration |
| `title` | no | caption, as `AudioClip` uses it |
| `source` | no | attribution, as `AudioClip` uses it |
| `format` | conditionally | required when probing is not `Certain` — see below |

Build-time flow: read the file, `probe`, `decode_image`, encode PNG, emit
`<img>` with a `data:` URI.

**Data URI, with a budget.** One function decides how the PNG reaches the page.
The component **fails the build** if an encoded PNG exceeds **96 KiB**.

The number is chosen against the formats, not picked round. A 256×192 `SCREEN$`
and a 320×200 C64 bitmap encode to a few kilobytes; a 320×256 Amiga ILBM stays
well inside 96 KiB. **A hi-res interlaced ILBM — 640×512, HAM, photographic —
will exceed it**, and that is the budget working rather than failing: such a
picture should not be inlined into every reader's HTML, and the build says so by
name instead of quietly shipping a third of a megabyte of base64.

Exceeding the budget is therefore the trigger for emitted asset files, which are
out of this slice and are a change to this one function.

**Display size comes from `pixel_aspect`.** The PNG holds native pixels; the
`width` and `height` attributes carry the corrected display size, with
`image-rendering: pixelated` so integer scaling stays sharp.

### A weak identification must be declared

When `probe` returns `Probable`, the component **requires** an explicit `format`
prop and fails the build without one.

Today that means Art Studio, which has no magic number and no checksum — a wrong
call surfaces as a picture that looks wrong, not as an error. Requiring the
author to say so makes `Confidence` load-bearing rather than a value that rides
along in a struct. `Certain` needs nothing.

If `format` is given and disagrees with a `Certain` probe, that is a build
failure naming both.

## Errors fail the build

Every failure stops the build and names the file: missing, unidentified, probed
as a format that is not an image, a decoder rejection, a PNG over budget, a
`Probable` probe with no declared format.

A red build is cheap. A broken or wrong-looking picture discovered across the
site in October is not.

## Where the files live

Beside the code that produces them, in `code198x/code-samples`, which
`deploy.yml` already checks out and passes as `CODE_SAMPLES_PATH`.

The first consumer already exists:

```
code-samples/sinclair-zx-spectrum/assembly/gloaming/loading-screen/
    compose.py             the generator — source of truth
    loading-screen.png     2174 bytes — its output, a 256×192 constrained image
    gloaming.scr           6912 bytes — the SCREEN$, converted from the PNG
    README.md
```

**Note the direction.** The PNG is the *source* and the `.scr` is derived from
it by `build198x image`; the README documents the regeneration command, and for
this file the conversion is lossless — `mean_error 0.0`, the tool reporting
*"input appears already constrained."* These two are not a drifting duplicate,
and nothing here is deleted.

What `NativeImage` changes is which of them a page shows. Today a figure would
show `loading-screen.png`: the artwork before conversion. `NativeImage` shows
`gloaming.scr` — **the 6912 bytes the tape actually carries**, decoded by the
decoder the player uses. On this file the two images match, so the difference
costs nothing; on any file where the conversion is not lossless, the page stops
quietly showing the pre-conversion source in place of the shipped artifact.

The general case is the stronger one. Most machine files are authored directly
and have no PNG beside them, so a figure of one today means producing and
committing a rendering by hand. This removes that step, and with it the class of
staleness where the rendering and the artifact drift apart with nothing to say
which is current.

The 2,174-byte PNG also calibrates the budget above: a real Spectrum screen
lands two orders of magnitude inside 96 KiB.

This is what keeps the claim literally true: the reader is looking at the file
the unit builds, decoded by the decoder the player uses — not a picture of it
taken separately and liable to drift from the code beside it.

`src` resolves under `CODE_SAMPLES_PATH`. There is one location, deliberately:
a second one invites a figure to drift away from the code that produced it,
which is the whole claim this component makes.

Media with no sample behind it — a Vault illustration, say — has no home under
this spec and is out of scope. If that need turns out to be real, it is a
resolution rule in one function, not a change to anything here.

## The website build gains Rust

`deploy.yml` adds a checkout of `play198x/play198x`, a Rust toolchain with the
`wasm32-unknown-unknown` target, `wasm-pack`, and a `wasm-pack build --target
nodejs` step before `npm run build`. This follows the pattern the workflow
already uses for `code-samples`.

**The cost, stated rather than discovered:** the site publishes on a schedule,
so a Rust build failure would block a content drip. The build must therefore
cache the Rust toolchain and the wasm artifact, and the wasm step must run
before `npm run build` so a failure is attributed clearly.

**Verify the module shape before writing the loader.** `wasm-pack --target
nodejs` emits CommonJS glue while Astro frontmatter is ESM. The first
implementation task builds the package and reads what it emits, then chooses the
loader — rather than assuming `createRequire` and discovering otherwise.

## Publishing

**Not published to crates.io.** The core design says the shells are unpublished
because "they are applications, and nothing consumes them". Under this spec that
reason no longer holds — `play198x-web` is a library, and the website consumes
it from another org, which is the shape the same document flags as the
`emu198x/emu198x#1214` trap.

The conclusion survives on a different reason: the crate's public surface is
`#[wasm_bindgen]` functions returning JavaScript values, which no Rust caller
can use. Anything native wanting decoding uses `play198x-core`, already
published. Publishing this would create a registry entry nobody can call.

**npm is deferred, not rejected.** The website consumes the *built wasm*, which
is an npm-shaped artifact, and publishing it there would take Rust out of the
site's deploy path entirely. It is deferred because the family has no npm
presence at all, and establishing one two months before the CRASH! Live soft
launch buys a new trap surface for one consumer.

**Publish to npm when either becomes true:**

- a second consumer needs the built wasm, or
- a Rust build failure blocks a scheduled content drip.

Until then the build-time checkout is the mechanism. This is a decision with a
review condition, not an omission.

## Testing

**Rust.** `wasm-bindgen-test` on the `nodejs` target decodes a synthetic
`SCREEN$` — a known ink/paper/bright attribute over a known bitmap — and asserts
**exact RGBA at named pixel coordinates**, with the expected values taken from
`mediaspec198x` rather than from our own output. Also: `probe` on empty input
returns null rather than throwing; a truncated file produces the core's error
message unchanged.

**Node.** The PNG writer's two checks above. The component: a known `.scr`
yields an `<img>` whose `width`/`height` are the aspect-corrected values, and
each failure listed under *Errors fail the build* fails with a message naming
the file.

**CI.** `play198x` builds **both** targets. The `web` target has no consumer and
will rot silently otherwise.

## What this does not build

Named so the next slice's scope is unambiguous, not as an apology.

- **Any interactivity** — attribute-grid toggle, palette view, load animation.
  The boundary carries the data they need; nothing renders it.
- **Audio.** `AudioClip` works today with pre-rendered clips. Live module
  playback needs WebAudio queueing without `SharedArrayBuffer`, autoplay gating,
  and a decision about distributing complete musical works rather than clips.
- **The drop-target playground.** The `web` target is built but unused.
- **Retiring or changing `AudioClip`, `Figure` or `Photo`.**
- **Emitted asset files instead of data URIs.**

## Amendment to the core design

[`2026-08-25-data-driven-core-design.md`](2026-08-25-data-driven-core-design.md)
states: *"The GUI and web shells are **not** published — they are applications,
and nothing consumes them."*

The second clause is no longer true of `play198x-web`. That document should be
amended to say the shells are unpublished because their surfaces have no Rust
caller, and to record that `play198x-web` is a library with a consumer in
another org whose distribution question is answered here.
