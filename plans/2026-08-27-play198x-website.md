# Play198x Website Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `play198x.github.io` — a landing page that **is** the player: drop a module or a loading screen on it and it plays or shows, in the visitor's own browser, with nothing uploaded and nothing installed.

**Architecture:** Astro 7 static site on GitHub Pages, mirroring the current `emu198x.github.io` (shared `198x-ui` kit at a pinned tag, `pages.yml`, the same gates). `play198x-core` compiled to `wasm32-unknown-unknown` via `play198x-web`'s `--target web` build does all decoding and mixing; HTML and CSS provide the chrome, `<canvas>` shows images, WebAudio plays modules.

**Tech Stack:** Astro 7, `@astrojs/sitemap`, `198x-ui` `v0.3.1`, Node 24, Rust 1.98 + `wasm-pack`, `web-sys`, Playwright, vale.

**Spec:** [`../specs/2026-08-26-website-design.md`](../specs/2026-08-26-website-design.md)

**Repo:** `play198x/play198x.github.io` — **already created, empty, with Pages enabled** (`build_type=workflow`, set by API on 2026-08-27; the spec records that doing this after the first push makes the first deploy fail). The org container's repo map already lists it, so no task does that.

## Global Constraints

- **The site ships no media, and nothing leaves the visitor's browser.** This is the reason the drop target was chosen over a gallery — a gallery would publish other people's artwork and engage `publishing-third-party-imagery.md`. Every test fixture is generated in code.
- **No media committed to this repo. Ever.** Not a sample module, not a screenshot of one.
- **GitHub Pages cannot set COOP/COEP headers**, so `SharedArrayBuffer` is unavailable. Any design that needs it is wrong for this site.
- **Rust 1.98.0.** `RUSTUP_TOOLCHAIN` is exported locally and overrides `rust-toolchain.toml` — every cargo command needs `env -u RUSTUP_TOOLCHAIN`.
- **`198x-ui` is pinned at `UI_REF=v0.3.1`** and fetched into a git-ignored `_198x-ui/`. Never vendored. Tracking `main` would let one change break three sites at once.
- **Content is read from the flagship repo, never invented** — the rule the rest of the family's sites hold.
- **The prose gate counts alerts from vale's JSON**, never trusts its exit code, which tracks errors only. A lesson already paid for on the Emu198x site.
- Conventional commits.

## Two decisions to make before Task 2

**1. npm — resolved 2026-08-27. `@play198x/web` is published.**

The trigger recorded in `2026-08-27-web-shell-build-time-images.md` — *publish when
a second consumer appears* — fired when this site became that consumer, and the
package now exists: [`@play198x/web`](https://www.npmjs.com/package/@play198x/web)
0.1.0, 24.5 kB, published from `play198x/play198x`.

**Task 3 therefore takes branch 1a**, and 1b is dead. The site adds a dependency
and copies files; **no Rust toolchain enters this repository's build at all.**

Two consequences for later tasks:

- Publishing is now **secretless and token-proof**. The package is set to
  *require 2FA and disallow tokens*, so publish authority flows only through
  `npm.yml` on a `play198x-web-v*` tag. Nobody can publish this package from a
  laptop, including you.
- **0.1.0 carries no provenance attestation**, because the bootstrap publish had
  to use a token. The next release will have one. That is a visible difference on
  the package page and is the cheapest proof that the trusted publisher is
  actually being used rather than silently failing over to something else.

**The published 0.1.0 has `probe` and `decode_image` only.** `ModulePlayer`
(Task 7) and `open_container`/`read_entry` (Task 6) do not exist in it yet, so
each of those tasks ends with a version bump and a tag, and the site's dependency
moves with it.

**2. The audio approach is unproven and Task 1 settles it.** The spec names WebAudio through `web-sys` and flags it as the least-proven part, to be spiked *before the page is built around it*, so that failure degrades the site to images-only rather than sinking it. Task 1 is that spike. **Tasks 7 and 8 must not begin until it reports.**

---

## File Structure

`play198x/play198x.github.io`:

| File | Responsibility |
|---|---|
| `package.json`, `tsconfig.json`, `astro.config.mjs` | the Astro 7 site, `site: 'https://play198x.github.io'` |
| `scripts/fetch-ui.sh` | fetch `198x-ui` at `UI_REF` into git-ignored `_198x-ui/` |
| `scripts/build-wasm.mjs` | produce the `--target web` package the page imports |
| `.github/workflows/pages.yml` | build, gate, deploy — PRs get every check and no deploy |
| `src/pages/index.astro` | the one page |
| `src/components/DropTarget.astro` | the drop zone and file picker |
| `src/components/Player.astro` | canvas, transport, metadata panel |
| `src/scripts/player.ts` | the browser-side glue: probe, decode, draw, play |
| `src/scripts/audio.ts` | whatever Task 1 concludes; nothing else knows how audio works |
| `scripts/check-*.mjs` | the gates, adapted from `emu198x.github.io` |
| `tests/*.test.mjs` | node tests |
| `tests/player.spec.ts` | Playwright: the wasm loads and decodes a synthetic module |

`play198x/play198x` (the crate gains a container boundary in Task 6 and an audio boundary in Task 7):

| File | Responsibility |
|---|---|
| `crates/play198x-web/src/lib.rs` | gains `open_container`/`read_entry` and `ModulePlayer` alongside `probe`/`decode_image` |

---

### Task 1: Spike — can WebAudio play wasm-rendered frames without `SharedArrayBuffer`?

**This is a spike. Its output is an answer and a recommendation, not code you keep.** Label everything you build throwaway. The plan's remaining audio tasks are written against your conclusion.

**Files:** none committed. Work in a scratch directory outside both repos.

**The question.** `play198x-core`'s `Engine::render(&mut [f32]) -> usize` is pull-based, which is what an `AudioWorklet` wants. But this site is static GitHub Pages: **no COOP/COEP headers, therefore no `SharedArrayBuffer`**, therefore no shared wasm memory between the main thread and the audio thread.

Three candidate shapes, in the order I would try them:

1. **wasm inside the worklet.** Compile a `WebAssembly.Module` on the main thread, `postMessage` it to the `AudioWorkletProcessor`, instantiate it there with its own memory. No sharing, so no `SharedArrayBuffer`. If this works it is the best answer: rendering happens on the audio thread, where glitches are least likely.
2. **Main thread renders, worklet consumes.** Render blocks on the main thread and post them to the worklet through a ring buffer built on a plain `ArrayBuffer`. Costs a copy and is exposed to main-thread jank.
3. **`AudioBufferSourceNode` queue.** Render a second or two ahead, schedule buffers back to back, refill on `ended` or a timer. Crudest, most portable, worst seek latency.

- [ ] **Step 1: Establish whether the constraint is real**

Confirm `crossOriginIsolated === false` and `typeof SharedArrayBuffer === 'undefined'` on a page served the way GitHub Pages serves one (plain `python3 -m http.server` is close enough — no COOP/COEP). Record the result. If `SharedArrayBuffer` turns out to be available without the headers, say so loudly — it changes the recommendation.

- [ ] **Step 2: Try shape 1**

Build the smallest thing that answers the question: a `play198x-web` `--target web` build exposing a function that fills an `f32` buffer with a sine wave, a worklet that instantiates the module from a `postMessage`d `WebAssembly.Module`, and a page that starts it on a click.

Use a sine wave, not a module — you are testing the transport, and a bug in module playback would be indistinguishable from a bug in the transport.

- [ ] **Step 3: Judge it honestly**

Play for **at least 60 seconds** and report: does it glitch, and how often? Does it survive a background tab? What is the latency from click to sound? A shape that works for five seconds and stutters at thirty has not worked.

If shape 1 fails, try shape 2, then shape 3, applying the same bar.

- [ ] **Step 4: Report**

Write to `<workspace>/task-1-spike.md`: which shapes you tried, what each did, the numbers behind "it works", and a recommendation with the sample rate and buffer size to use. If **no** shape holds up, say so plainly — the site then ships images-only, which the spec explicitly plans for, and that is a good outcome for this spike rather than a failure of it.

---

### Task 2: Scaffold the site, and get an empty page deployed

**Files:**
- Create in `play198x/play198x.github.io`: `package.json`, `tsconfig.json`, `astro.config.mjs`, `.gitignore`, `scripts/fetch-ui.sh`, `.github/workflows/pages.yml`, `src/pages/index.astro`, `README.md`

**Interfaces:**
- Produces: a deployed site at `https://play198x.github.io` with a placeholder page, and `_198x-ui/` available to components at `v0.3.1`.

Mirror `Emu198x/emu198x.github.io` — read its files rather than inventing equivalents.

- [ ] **Step 1: Copy the UI fetch script verbatim**

`scripts/fetch-ui.sh` from `emu198x.github.io` is correct as-is: `REF="${UI_REF:-v0.3.1}"`, clone or checkout into `_198x-ui`, `set -euo pipefail`. Copy it, keep its comments — they explain why the tag is pinned, which is the part that decays first.

Add `_198x-ui/`, `dist/`, `node_modules/` to `.gitignore`.

- [ ] **Step 2: `package.json` with the fetch wired to both entry points**

```json
{
  "name": "play198x.github.io",
  "type": "module",
  "scripts": {
    "ui:fetch": "./scripts/fetch-ui.sh",
    "predev": "npm run ui:fetch",
    "dev": "astro dev",
    "prebuild": "npm run ui:fetch",
    "build": "astro build",
    "preview": "astro preview",
    "test": "node --test 'tests/**/*.test.mjs'"
  }
}
```

`prebuild` and `predev` both fetch, so neither a local run nor CI needs to remember. Gates and the wasm build are added in Tasks 3 and 8 — do not add them yet.

Install Astro 7 and `@astrojs/sitemap` only. The Emu198x site's `@astrojs/mdx` and its markdown-link remark plugin exist for a docs tree this site does not have.

- [ ] **Step 3: `astro.config.mjs`**

```js
// @ts-check
import { defineConfig } from 'astro/config';
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  site: 'https://play198x.github.io',
  integrations: [sitemap()],
});
```

- [ ] **Step 4: A placeholder page that proves the kit resolves**

`src/pages/index.astro` importing `Plate.astro` and `SiteNav.astro` from `_198x-ui/components/` and `tokens.css`, with a heading and one sentence. Its only job is to prove the pinned kit is reachable and the tokens apply.

Read `_198x-ui/README.md` for the components' actual props rather than guessing them.

- [ ] **Step 5: `pages.yml`**

Copy `emu198x.github.io`'s workflow and **remove what does not apply**: the `emu198x-source` checkout and `EMU198X_SOURCE_ROOT`, and the vale and a11y steps (Task 9 adds this site's gates).

Keep exactly, and keep their comments:
- `on: pull_request` — without it a PR reports "no checks", which reads as nothing needing checking rather than nothing being checked
- `permissions: contents: read` at workflow level, with `pages: write` and `id-token: write` only on `deploy`
- `if: github.event_name != 'pull_request'` on `deploy` — a PR gets every check and publishes nothing

- [ ] **Step 6: Build locally, then verify the deploy**

```bash
npm ci && npm run build && npx astro preview
```

Push and confirm the Pages deploy goes green and `https://play198x.github.io` serves the placeholder. **Report the deployed URL's actual response** — Pages was enabled by API ahead of the first push specifically so this works; if it 404s, say so rather than assuming propagation.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "feat: scaffold the Play198x site on the shared kit"
```

---

### Task 3: Get the wasm into the page

**Files:**
- Create: `scripts/build-wasm.mjs`
- Modify: `package.json`, `.github/workflows/pages.yml`, `.gitignore`

**Interfaces:**
- Produces: a `--target web` package importable from browser code, and `npm run build:wasm` producing it.

**Steve rules on npm before this task starts** (see *Two decisions*). Both branches are specified; take the one he chose and record which in your report.

- [ ] **Step 1a — if npm was chosen:** add `@play198x/web` at its published version to `dependencies`, and make `scripts/build-wasm.mjs` a three-line copy of the package's `dist` into `public/wasm/`. No Rust in this repo's build at all.

- [ ] **Step 1b — if checkout-and-build was chosen:** write `scripts/build-wasm.mjs` taking `PLAY198X_PATH` and running `wasm-pack build <path>/crates/play198x-web --target web --out-dir pkg-web`, with a `try/catch` naming `wasm-pack` and `PLAY198X_PATH` on failure — a bare `spawnSync ENOENT` is a poor greeting for a contributor without Rust. Add the play198x checkout, Rust toolchain, `Swatinem/rust-cache` and `wasm-pack` install to `pages.yml` **before** the build step, so a Rust failure reads as a Rust failure rather than a mystery inside Astro.

- [ ] **Step 2: Load it from the page and prove it ran**

Import the module in a `<script>` on the index page, call `probe` on a `Uint8Array` you build in the browser, and log the result. A **6912-byte** array probes as `scr` with confidence `probable` — SCR has no magic number, its length is the whole signal.

- [ ] **Step 3: Verify in a real browser**

```bash
npm run build:wasm && npm run build && npx astro preview
```
Open the page, check the console. **Report what the console actually said.** A wasm module that 404s under Astro's asset handling is the likely failure, and it will not surface any other way.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "feat: load the decoder in the browser"
```

---

### Task 4: The drop target

**Files:**
- Create: `src/components/DropTarget.astro`, `src/scripts/player.ts`
- Modify: `src/pages/index.astro`

**Interfaces:**
- Produces: `onFile(file: File): Promise<void>` in `player.ts`, called by both the drop handler and the file picker.

- [ ] **Step 1: The drop zone and the picker, both reaching the same function**

A visitor who cannot drag — keyboard, touch, screen reader — must still be able to open a file. The picker is not a fallback bolted on; it is the accessible path, and the a11y gate in Task 9 will fail without it.

Handle `dragover` (prevent default, show the active state), `dragleave`, and `drop`. Read with `file.arrayBuffer()`.

- [ ] **Step 2: Probe and route**

Call `probe`. Report by name what was dropped, and what it will do with it — an image format goes to Task 5's canvas, `protracker` to Task 8's player, `null` says so plainly.

Say what a file **is**, not only what failed: *"that's a 6912-byte file — probably a ZX Spectrum SCREEN$"* tells a visitor more than *"unsupported"*, and it is honest about `Probable` rather than hiding it.

- [ ] **Step 3: Test the routing with generated bytes**

`tests/probe-routing.test.mjs` — a 6912-byte array routes to the image path; 1084 bytes with `M.K.` at offset 1080 routes to audio; three bytes route to neither. **Generated in code. No fixtures on disk.**

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "feat: take a dropped file and say what it is"
```

---

### Task 5: Show a picture

**Files:**
- Create: `src/components/Player.astro`
- Modify: `src/scripts/player.ts`

- [ ] **Step 1: Decode to a canvas**

`decode_image` returns `width`, `height`, `rgba`, `pixel_aspect_w`, `pixel_aspect_h`, `palette`. Build an `ImageData` from `rgba` and `putImageData` it to a `<canvas>` sized `width × height`.

**Set the canvas's CSS size from `pixel_aspect`**, not its buffer size: mode pixels are not display pixels, and a shell that ignores this draws a C64 multicolour picture at half its real width. Add `image-rendering: pixelated` so scaling stays sharp.

- [ ] **Step 2: A weak identification must be admitted, not hidden**

A `.scr` always probes `Probable`. The page cannot ask an author to declare a format the way the curriculum component does — there is no author — so it must **say so**: show the format and that the identification is probable, and offer the other image formats as a manual override. A visitor who dropped something unusual can correct it; a visitor who dropped a screen sees a screen.

- [ ] **Step 3: Show the metadata**

`metadata::image_meta` reports what an interface should show. Render it beside the canvas.

- [ ] **Step 4: Verify with a generated image**

Build a 6912-byte SCREEN$ in the browser — all bitmap bytes clear, every attribute `0x28` (PAPER 5 cyan, INK 0 black) — and confirm the canvas is uniformly `rgb(0, 194, 194)`. That value is `mediaspec198x`'s index 5 at the normal `0xC2` level; it must come from the spec table, never from reading our own output back.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: show a dropped picture at its real shape"
```

---

### Task 6: Open a ZIP and an ADF

The spec calls this *"the case that makes the tool feel real rather than a demo"*: a visitor drops an unmodified TOSEC zip or an Amiga music disk and it opens. `play198x-core` already reads all three container shapes and decrunches PowerPacked entries in passing — but `play198x-web` exposes none of it, so this is a boundary addition as well as a UI one.

Containers are **byte parsing, not filesystem access**, so they work in a browser unchanged. Nothing is extracted to disk and nothing leaves the page.

**Files:**
- Modify (in `play198x/play198x`): `crates/play198x-web/src/lib.rs`, `crates/play198x-web/tests/boundary.rs`
- Modify (in the site): `src/scripts/player.ts`, `src/components/DropTarget.astro`

**Interfaces:**
- Consumes: `play198x_core::container`
- Produces: `open_container(bytes) -> Option<ContainerListing>` with `len()` and `name(i) -> String`; `read_entry(bytes, index) -> Result<Vec<u8>, JsError>`

- [ ] **Step 1: Write the failing boundary tests**

```rust
#[wasm_bindgen_test]
fn a_plain_file_is_not_a_container() {
    assert!(play198x_web::open_container(&screen(0x28)).is_none());
}

#[wasm_bindgen_test]
fn a_zip_lists_the_entries_it_holds() {
    let zip = synthetic_zip(&[("screen.scr", &screen(0x28)), ("readme.txt", b"hello")]);
    let listing = play198x_web::open_container(&zip).expect("a zip is a container");
    assert_eq!(listing.len(), 2);
    assert_eq!(listing.name(0), "screen.scr");
}

#[wasm_bindgen_test]
fn an_entry_reads_back_the_bytes_that_went_in() {
    let original = screen(0x28);
    let zip = synthetic_zip(&[("screen.scr", &original)]);
    assert_eq!(play198x_web::read_entry(&zip, 0).unwrap(), original);
}

#[wasm_bindgen_test]
fn an_index_past_the_end_is_an_error_not_a_panic() {
    let zip = synthetic_zip(&[("screen.scr", &screen(0x28))]);
    assert!(play198x_web::read_entry(&zip, 99).is_err());
}
```

Build `synthetic_zip` in code — a stored (uncompressed) zip is a local header, the data, a central directory and an end record, which is short enough to write by hand and keeps the promise that no fixture files exist. **No media committed.**

- [ ] **Step 2: Run and watch them fail**

```bash
env -u RUSTUP_TOOLCHAIN wasm-pack test --node crates/play198x-web
```
Expected: FAIL — `open_container` not found. As everywhere in this crate, `cargo test --manifest-path` would run zero of these and report "ok".

- [ ] **Step 3: Implement the boundary**

Wrap `play198x_core::container`. Two rules carry over from the core and must not be relaxed here:

- **The archive size limit stays enforced.** The core caps what it will read because an unbounded read of a hostile archive is an allocation failure, which *aborts* rather than unwinding. A browser tab is exactly where someone will drop a 400 MB zip.
- **No panics.** An out-of-range index, a damaged central directory and a truncated entry are all errors carrying the core's own message.

- [ ] **Step 4: Pass, and rebuild both targets**

```bash
env -u RUSTUP_TOOLCHAIN wasm-pack test --node crates/play198x-web
wasm-pack build crates/play198x-web --target web --out-dir pkg-web
wasm-pack build crates/play198x-web --target nodejs --out-dir pkg-node
```

- [ ] **Step 5: Commit the crate change** — in `play198x/play198x`.

```bash
git commit -m "feat: open ZIP and ADF containers across the wasm boundary"
```

- [ ] **Step 6: Let a visitor choose an entry**

In the site: when `open_container` returns a listing, show the entries rather than guessing. One playable entry may open directly; several must be chosen from, because an Amiga disk holds many modules and picking one for the visitor is picking wrong most of the time.

Probe each entry so the list says what each one is, and grey out what cannot be opened rather than hiding it — a visitor who dropped a disk expecting a tune needs to see that the disk holds no tune.

- [ ] **Step 7: Verify with a generated archive**

In the browser, build the same stored zip the Rust test uses, drop it through the handler, confirm the entry list appears with both names and that choosing the `.scr` draws the picture from Task 5.

- [ ] **Step 8: Commit the site change**

```bash
git add -A && git commit -m "feat: open a dropped archive and let a visitor pick from it"
```

---

### Task 7: The audio boundary in the crate

**Do not start until Task 1 has reported.** If it concluded no shape works, **skip this task and Task 8**, and say so in the ledger — the site ships images-only, which the spec plans for.

**Files:**
- Modify (in `play198x/play198x`): `crates/play198x-web/src/lib.rs`, `crates/play198x-web/tests/boundary.rs`

**Interfaces:**
- Consumes: `play198x_core::decode::module`, `play198x_core::engine::Engine`
- Produces: `ModulePlayer` with `new(bytes, sample_rate)`, `render(&mut [f32]) -> usize`, `set_playing(bool)`, `seek_order(usize)`, and position getters.

- [ ] **Step 1: Write the failing test**

```rust
#[wasm_bindgen_test]
fn a_player_renders_the_frames_it_is_asked_for() {
    let mut player = play198x_web::ModulePlayer::new(&synthetic_module(), 48_000).unwrap();
    let mut buffer = vec![0.0f32; 1024];
    assert_eq!(player.render(&mut buffer), 1024);
}

#[wasm_bindgen_test]
fn a_paused_player_renders_silence_rather_than_stopping() {
    let mut player = play198x_web::ModulePlayer::new(&synthetic_module(), 48_000).unwrap();
    player.set_playing(false);
    let mut buffer = vec![1.0f32; 256];
    let rendered = player.render(&mut buffer);
    assert_eq!(rendered, 256, "a paused player still fills its buffer");
    assert!(buffer.iter().all(|&s| s == 0.0), "and fills it with silence");
}
```

Pause rendering silence rather than returning zero matters: an audio callback starved of samples clicks, and a paused player is the most likely thing to starve it.

Build `synthetic_module()` in code — a minimal valid ProTracker module, `M.K.` at 1080. **No fixture files.**

- [ ] **Step 2: Run it and watch it fail**

```bash
env -u RUSTUP_TOOLCHAIN wasm-pack test --node crates/play198x-web
```
Expected: FAIL — `ModulePlayer` not found. **Never judge these tests with `cargo test --manifest-path`: it runs zero `#[wasm_bindgen_test]` tests and reports "ok".**

- [ ] **Step 3: Implement `ModulePlayer`**

Wrap `Engine`. `render` takes `&mut [f32]` so the caller owns the buffer and no allocation happens per callback — the core's render path is allocation-free and this must not undo that.

- [ ] **Step 4: Pass, then check both targets still build**

```bash
env -u RUSTUP_TOOLCHAIN wasm-pack test --node crates/play198x-web
env -u RUSTUP_TOOLCHAIN cargo clippy --manifest-path crates/play198x-web/Cargo.toml --all-targets -- -D warnings
wasm-pack build crates/play198x-web --target web --out-dir pkg-web
wasm-pack build crates/play198x-web --target nodejs --out-dir pkg-node
```

The `nodejs` target feeds the Code198x website. Breaking it here breaks that site's build.

- [ ] **Step 5: Commit** — in the **`play198x/play198x`** repo, not the site.

```bash
git commit -m "feat: play a module through the wasm boundary"
```

---

### Task 8: Play it

**Files:**
- Create: `src/scripts/audio.ts`
- Modify: `src/scripts/player.ts`, `src/components/Player.astro`

Implement the shape Task 1 recommended. **Nothing outside `audio.ts` knows how audio works** — if the shape has to change, one file changes.

- [ ] **Step 1: Start on a gesture, and say so before it**

Browsers refuse an `AudioContext` before a user gesture. Show the module's name and length with a play control, and start the context inside that click. A page that autoplays and fails silently looks broken; one that waits and says *press play* does not.

- [ ] **Step 2: Wire the transport**

Play, pause, and the position from `Engine::position()`. Pause calls `set_playing(false)` — the player keeps rendering silence, per Task 7.

- [ ] **Step 3: Verify by listening, and by measuring**

Play a generated module for **60 seconds**. Report glitches, whether a background tab survives, and click-to-sound latency. "It played" is not a result; the numbers are.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "feat: play a dropped module in the browser"
```

---

### Task 9: The page's words, and the gates

**Files:**
- Modify: `src/pages/index.astro`, `package.json`, `.github/workflows/pages.yml`
- Create: `scripts/check-routes.mjs`, `scripts/check-internal-links.mjs`, `scripts/check-prose.mjs`, `tests/player.spec.ts`, `.vale.ini`

- [ ] **Step 1: Write the page below the fold**

Three sections, in the spec's own terms:

- **What Play198x is** — it renders media, it does not boot machines. The distinction is why the project exists and is the first thing a reader needs.
- **What it plays today** — SCR, Koala, Art Studio, ILBM, ProTracker MOD, from plain files, ZIPs and ADFs, PowerPacker included. **And what it does not: SID and AY, named as not-yet.** A media player that silently lacks SID disappoints exactly the visitor most likely to arrive. They are blocked on `emu198x/emu198x#1214`.
- **Downloads** — generated from the GitHub release, not hand-listed, so they cannot go stale. If no desktop release exists yet, say that rather than linking nothing.

Footer carries the standing rights notice: the page renders third-party works even though it never serves them.

- [ ] **Step 2: Port the gates**

Copy `check-routes.mjs`, `check-internal-links.mjs` and `check-prose.mjs` from `emu198x.github.io` and adapt paths. `expected-routes.txt` for a one-page site lists `/` and `/sitemap-index.xml`.

**`check-prose.mjs` must keep counting alerts from vale's JSON.** Its exit code tracks errors only, so trusting it would let every suggestion through — the exact thing the script exists to stop, and a lesson already paid for once.

- [ ] **Step 3: The site-specific test**

`tests/player.spec.ts` — Playwright drives the real page: build a synthetic module in the browser, hand it to the drop handler, assert the player reports the right length and the canvas or transport appears. This is the test that would catch the wasm failing to load at all, which every node test would miss.

- [ ] **Step 4: Wire everything into the build and the workflow**

Add the gates to `npm run build`, and to `pages.yml` add the vale install (pinned, failing loudly when missing) and `npx playwright install --with-deps chromium` plus the a11y sweep, mirroring Emu198x's ordering: gates and accessibility **between** building and publishing, because a page that fails WCAG AA is not one to deploy.

- [ ] **Step 5: Prove the gates gate**

Break each one deliberately — a bad internal link, a prose suggestion, a missing route — confirm the build fails, revert. **Report the evidence.** A gate nobody has seen fail is not known to work.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: say what Play198x is, and gate the page that says it"
```

---

### Task 10: Correct the stale playbook

**Files:**
- Modify: `~/knowledge/family-github-pages-site-playbook.md`

The spec flags this explicitly: the playbook describes the **pre-rebuild** house style — paper `#f8f7f2`, Inter, teal accent, shared files copied by `rsync` — and following it verbatim produces a site matching the old look. Its CI and Pages-enablement steps remain correct.

- [ ] **Step 1: Correct it from what this task actually did**

Replace the stale styling and `rsync` sections with the `198x-ui` kit at a pinned `UI_REF` fetched into a git-ignored `_198x-ui/`. **Delete the wrong guidance rather than annotating it** — a correction that leaves the false version readable is not a correction. Keep the CI and Pages steps.

Add what this build learned that the playbook did not know: that Pages must be enabled by API before the first push, and that on a static Pages site `SharedArrayBuffer` is unavailable, with Task 1's conclusion about what that means for audio.

- [ ] **Step 2: Check the entry's own frontmatter is still true**

Its `verified:` line must say how you know — the command run or the build it came from. If you cannot fill it honestly, the entry is not verified.

---

## What this plan does not build

- **Cataloguing or browsing a collection** — Cat198x's, by the binding decision.
- **Anything server-side.** The site is static and the visitor's file never leaves their browser.
- **Bundled demo media**, which is the whole rights position.
- **Documentation beyond the landing page** — that belongs in `play198x/docs`.
- **SID, AY, NSF, SAP** — blocked on `emu198x/emu198x#1214`, and named on the page as not-yet rather than omitted.
- **Animation** — ANIM, FLI/FLC.
