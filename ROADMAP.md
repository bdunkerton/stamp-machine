# Roadmap

How small can Stamp Machine get?

The goal is a single HTML file with **no third-party dependencies** that is **as small as possible**, and that produces **lean PDFs**. This roadmap is a living plan. Phases fill in as we explore each idea, record what we learn, and measure the savings.

**Status key:** ✅ Done · 🚧 In progress · 🔍 Exploring · 📋 Planned · 💡 Idea

## Where we are

[![stamp-machine.html size](https://img.shields.io/github/size/bdunkerton/stamp-machine/stamp-machine.html?branch=main&label=stamp-machine.html)](stamp-machine.html)

| What a first visit downloads | Bytes | Gzipped |
|---|---|---|
| `stamp-machine.html` | 47,173 | 22,613 |
| pdf.js + worker (CDN) | 1,407,216 | 378,955 |
| pdf-lib (CDN) | 525,099 | 206,660 |
| Caveat font, 3 weights (Google Fonts) | not measured | |

The HTML file is about 2% of a first visit today. The libraries are the other 98%.

Inside `stamp-machine.html`: stamp font 19,546 · JavaScript 17,179 · CSS 9,742 bytes.

### Size history

Add a row whenever a change lands.

| Date | Change | `stamp-machine.html` | First visit total | PR |
|---|---|---|---|---|
| 2026-10-29 | Starting point | 132,508 | ~2.1 MB | |
| 2026-10-29 | Subset stamp font to WOFF2 | 47,231 | ~2.0 MB | #1 |
| 2026-10-30 | Phase 1 quick wins | 47,173 | ~2.0 MB | #7 |

## Phase 1: Quick wins

Small, low-risk changes that cut bytes without changing how the app works.

| Item | Status | Saves | PR |
|---|---|---|---|
| Subset Quicksand to Latin, embed as WOFF2, rename to StampFont | ✅ | 85 KB | #1 |
| Stop requesting the unused Caveat 700 weight | ✅ | Small (CSS only) | #7 |
| Fix the misspelled `'applicatoin/pdf'` download type | ✅ | Bug fix | #7 |
| Remove `document.fonts.load('10px StampFont')`, a no-op | ✅ | 58 B | #7 |
| Decide whether accented names are needed (ASCII-only font is ~10.5 KB) | 💡 | ~9 KB | |

**Notes & findings**
- Font subset: 808 → 373 glyphs, rendering is pixel-identical. The OFL requires the rename because "Quicksand" is a Reserved Font Name. The copyright and license notice stay in the font and the README.
- Confirmed: `document.fonts.load('10px StampFont')` matched zero fonts, because StampFont only exists inside the SVG used to draw each name, and that SVG loads its own font before it's drawn. Removing it left the generated PDFs pixel-identical.
- Browsers only download the font files a page actually uses, so the Caveat 700 file was never downloaded. Dropping it only shortens the stylesheet Google sends back.

## Phase 2: Lean app code

Make the HTML itself as small as it can be.

| Item | Status | Saves | PR |
|---|---|---|---|
| Build step that outputs a minified copy, keeping readable source | 💡 | | |
| Consolidate repeated CSS (e.g. `font-family: 'Caveat'` ×7) | 💡 | | |
| Remove dead code and duplicate helpers | 💡 | | |

**Notes & findings**
- Indentation alone is ~7.9 KB today.
- Open question: commit the minified file as the app, or keep both source and output in the repo?

## Phase 3: Lean output PDFs

Make the PDFs the app generates smaller and sharper.

| Item | Status | Saves | PR |
|---|---|---|---|
| Load the template once, not once per name | 📋 | | |
| Draw names as vector paths instead of 3× PNG images | 💡 | | |

**Notes & findings**
- `generateStamp` reloads and copies the template inside its loop, so every page carries a full copy. That multiplies with large scanned worksheets and full classes.
- A 3-name test came out at ~25 KB, mostly PNG data. Vector names would also remove the need for the font, the canvas and the PNG step.

## Phase 4: Replace pdf-lib

Write the PDF ourselves instead of loading a 525 KB library.

| Item | Status | Saves | PR |
|---|---|---|---|
| Minimal PDF writer for the pages, images or paths we add | 💡 | | |
| Template reader (xref tables and compressed xref streams) | 💡 | | |
| Try an incremental update (append to the original file) | 💡 | | |

**Notes & findings**
- We only use `load`, `copyPages`, `addPage`, `embedPng`, `drawImage` and `save`.
- Writing a PDF is easy. Reading arbitrary templates is the hard part. The browser's built-in `DecompressionStream` can inflate compressed sections without a library.

## Phase 5: Replace pdf.js

Show the worksheet preview without a 1.4 MB library. This is the hardest phase.

| Item | Status | Saves | PR |
|---|---|---|---|
| Find out what kinds of worksheets are actually used | 💡 | | |
| Prototype: browser's built-in PDF viewer with the box overlaid | 💡 | | |
| Prototype: minimal renderer for just those worksheet types | 💡 | | |

**Notes & findings**
- pdf.js only draws the preview that the name box is placed on.
- Built-in viewer: almost zero bytes, but box alignment is fiddly and mobile Safari is unreliable.
- Keeping pdf.js as the one remaining dependency is an acceptable fallback.

## Phase 6: No outside requests

The app makes no network requests at all.

| Item | Status | Saves | PR |
|---|---|---|---|
| Replace the Google Fonts Caveat request (embed a subset, SVG title, or system font) | 💡 | | |

**Notes & findings**
- The UI uses Caveat at 400, 500 and 600 for the title and a few labels.

## Ideas to explore

Unsorted ideas go here until they earn a place in a phase.

- 💡

## Along the way

Bugs and polish noticed while working on minification.

- [ ] All-caps names sit low in the box. They're centered on the full line height, including room below the baseline.
- [ ] Long names get squeezed hard because the text always fills the box height.

## How we measure

For each change, record in the PR and in [Size history](#size-history):

- `stamp-machine.html` size in bytes, raw and gzipped
- Total bytes downloaded on a first visit
- Size of a generated PDF from the same template and names
- A check that stamped output still looks right
