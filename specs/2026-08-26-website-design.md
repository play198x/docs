# Play198x website — design

**Status:** Approved 2026-08-26.

**Depends on:**
[`2026-08-25-data-driven-core-design.md`](2026-08-25-data-driven-core-design.md).
The site hosts the player, so it cannot ship before the player runs — but the
constraint it places on the core exists from day one and is recorded there.

**Goal:** `play198x.github.io` — a landing page that *is* the player. Drop a
module or a loading screen on it and it plays or shows, in the browser, with no
install.

---

## Why the player rather than a brochure

Every sibling has a landing page. Play198x can have something none of them can:
the thing itself, running, on the page. `play198x-core` compiles to
`wasm32-unknown-unknown`, so decode and mixing are the *same code* the desktop
app runs — only the shell differs.

It also **demonstrates** the binding decision's embeddable clause (*"lightweight
and embeddable — WASM media previews in the curriculum"*) instead of promising
it: the WASM build the site needs is the same one a curriculum preview or a
Cat198x preview surface would later embed. One core, several shells.

## The rights position, which is why this shape is chosen

**The site ships no media. The visitor supplies their own, and nothing leaves
their browser.**

This is not incidental — it is the reason a gallery was rejected. A gallery of
loading screens and ILBM art would be the strongest-looking option and would
publish other people's artwork, engaging
[`publishing-third-party-imagery.md`](../../../decisions/publishing-third-party-imagery.md)
with its per-entry acknowledgement, resolution limits and artist attribution.
A drop target engages none of it: the file is read by WASM in the visitor's own
browser, decoded there, and never uploaded, stored, or transmitted. It is the
same principle the family already holds — *the fact travels, the artefact does
not* — applied to a web page.

Two things follow and must not be softened later:

- **No demo media is bundled with the site**, however tempting a
  "try it without a file of your own" button is. That button is a distribution
  of somebody's module.
- **No upload.** The player is client-side only. There is no endpoint, so there
  is nothing to secure and nothing to log.

The standing rights notice goes in the footer regardless, because the page
renders third-party works even though it does not serve them.

## Repository and stack

`play198x/play198x.github.io`, mirroring the **current**
`emu198x.github.io` — Astro 7 with `@astrojs/sitemap`, and the `198x-ui` shared
kit fetched at a pinned tag into a git-ignored `_198x-ui/` by
`scripts/fetch-ui.sh` (`UI_REF`, currently `v0.3.1`), with `prebuild` resetting
that checkout to the tag.

⚠ **`~/knowledge/family-github-pages-site-playbook.md` is out of date and must
not be followed verbatim.** It describes the pre-rebuild house style — paper
`#f8f7f2`, Inter, teal accent, shared files copied by `rsync`. Emu198x's site
was rebuilt onto the `198x-ui` kit with different tokens and typefaces, so
copying per the playbook produces a site matching the old look. Its CI and
Pages-enablement steps remain correct. The playbook is corrected as part of this
work.

Deployment follows the proven pattern: `.github/workflows/pages.yml` on push to
`main`, build then `deploy-pages`, `dist/` never committed — and **Pages enabled
by API before the first push**, or the first deploy fails:

```sh
gh api -X POST repos/play198x/play198x.github.io/pages -f build_type=workflow
```

The site is added to the org container's `.github/CLAUDE.md` repo map, committed
in the `.github` repo.

## Page content

One page. Above the fold, the drop target and the player. Below it:

- **What Play198x is**, in the decision's own terms — it renders media, it does
  not boot machines. The distinction is the reason the project exists and is the
  first thing a reader needs.
- **What it plays today**, honestly scoped: SCR, Koala, Art Studio, ILBM,
  ProTracker MOD, from plain files, ZIPs and ADFs, PowerPacker included. And
  **what it does not** — SID and AY are named as not-yet, because a media player
  that silently lacks SID would disappoint precisely the visitor most likely to
  arrive.
- **Downloads** for the desktop app, generated from the release rather than
  hand-listed.

Content is read from the flagship repo, never invented — the same rule the rest
of the family's sites hold.

## The web shell

**The desktop app does not run in the browser.** The desktop shell is GPUI,
which has no web target and no roadmap for one. That is a deliberate cost,
accepted in the core spec: the browser gets a **second, much smaller shell**,
`play198x-web`, and the two are not comparable in size. The desktop app needs a
whole browsing and metadata surface; the web shell needs a drop target, an image
canvas and play/pause.

`play198x-web` uses **no Rust UI framework**. The site is already Astro, so:

| Concern | Provided by |
|---|---|
| Chrome, drop zone, controls, metadata | HTML and CSS, from the Astro page |
| Decode, containers, mixing | `play198x-core` via `wasm-bindgen` |
| Image display | `<canvas>` |
| Audio | WebAudio, fed engine frames |

This keeps the bundle small. A Rust UI toolkit compiled to WASM would be most of
the download for a UI this thin, and the page's own HTML is better at the
metadata panel than any of them.

File access is the browser's drag-and-drop and file-picker APIs. **ZIP and ADF
containers work unchanged** — they are byte parsing, not filesystem access — so
a visitor can drop an unmodified TOSEC zip or an Amiga music disk and it opens.
That is the case that makes the tool feel real rather than a demo.

**Risk, reduced but not eliminated.** Browser audio is now WebAudio driven
directly through `web-sys` rather than `cpal`'s WASM backend, which removes the
callback-model mismatch that would have needed a scheduling shim: the engine
already produces frames on demand, which is exactly what an `AudioWorklet`
wants. It remains the least-proven part of the site and should still be spiked
before the page is built around it, so that failure degrades the site to
images-only rather than sinking it.

## Testing

The gates `emu198x.github.io` already runs, since the kit and CI come from
there: route checks, internal-link checks, source and rendered spacing checks,
and the vale prose gate against House198x. The prose gate must read the built
HTML and count alerts from JSON rather than trusting vale's exit code, which
tracks errors only — a lesson already paid for on the Emu198x site.

Site-specific: the WASM bundle must load and the drop target must decode a
generated synthetic module in a headless browser. That test uses a **synthetic**
module for the same reason the core's tests do — no media in the repository.

## Out of scope

Cataloguing or browsing a collection (Cat198x, by the binding decision).
Server-side anything. Bundled demo media. Documentation beyond the landing page
— that belongs in `play198x/docs`.
