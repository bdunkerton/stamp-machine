# Minification goals

How small can Stamp Machine get? Two aims:

1. **Fewer dependencies**: replace third-party code with small, hand-written versions.
2. **Fewer bytes**: shrink the app file, what it downloads at runtime, and the PDFs it produces.

This is a starting point. Add, cross off and re-measure as we go.

## Baseline

Measured on `main` after the font subset (PR #1).

| What | Bytes | Gzipped |
|---|---|---|
| `stamp-machine.html` | 47,231 | 22,677 |
| ↳ stamp font (base64) | 19,546 | |
| ↳ JavaScript (excluding font) | 17,237 | |
| ↳ CSS | 9,742 | |
| pdf.js `pdf.min.js` (CDN) | 320,004 | 88,489 |
| pdf.js `pdf.worker.min.js` (CDN) | 1,087,212 | 290,466 |
| pdf-lib `pdf-lib.min.js` (CDN) | 525,099 | 206,660 |
| Caveat, 4 weights (Google Fonts) | not measured | |

Right now the HTML file is about 2% of what a first visit downloads. The libraries are the other 98%.

## Dependencies

- [ ] **Replace pdf-lib with a hand-written PDF writer.** We only use `load`, `copyPages`, `addPage`, `embedPng`, `drawImage` and `save`. Writing a PDF is easy. Reading the template is the hard part, because modern PDFs often store their structure in compressed cross-reference streams. The browser's built-in `DecompressionStream` can inflate those without a library. Consider an incremental update (appending to the original file) so the template never has to be fully rebuilt.
- [ ] **Replace or remove pdf.js.** It only draws the preview that the name box is placed on. Options, from smallest to most work:
  - Show the browser's built-in PDF viewer in an `<iframe>` with the box overlaid. Almost zero bytes, but box alignment is fiddly and mobile Safari is unreliable.
  - Write a minimal renderer covering only the kinds of worksheets actually used (scans, simple vector pages).
  - Keep pdf.js as the one remaining dependency.
- [ ] **Remove the Google Fonts request for Caveat.** Four weights load for a title and a few labels. Options: subset Caveat to just the characters the UI uses and embed it, draw the title as an SVG path, or fall back to a system font.

## Stamp font

- [x] Subset Quicksand Bold to Latin, embed as WOFF2 and rename to StampFont per the OFL (105 KB → 19.5 KB of base64). PR #1.
- [ ] Decide whether accented names are needed. An ASCII-only subset is about 10.5 KB, but accented letters would fall back to a system font.
- [ ] **Draw names as vector paths instead of a font.** Store just the glyph outlines and kerning needed, and write them straight into the PDF as shapes. This removes the font, the canvas and the PNG step all at once (see Output PDFs).

## App code

- [ ] Add a small build step that strips whitespace and comments and outputs a minified copy, keeping a readable source file alongside it. Indentation alone is about 7.9 KB today.
- [ ] Shorten repeated CSS: `font-family: 'Caveat', cursive` appears 7 times, and similar rules could share classes or custom properties.
- [ ] Look for dead code and repeated helpers once the dependencies are gone.

## Output PDFs

- [ ] **Load the template once, not once per name.** `generateStamp` calls `PDFDocument.load` and `copyPages` inside the loop, so every page carries its own copy of the template's contents. With a large scanned worksheet and a full class, that multiplies quickly. Store the template once and have every page reference it.
- [ ] **Replace the 3× PNG with vector text.** Each name is rasterized at 3× and embedded as a PNG (a 3-name test came out at about 25 KB). Vector paths would be smaller and print sharper.

## Small fixes found along the way

- [ ] The download Blob type is misspelled `'applicatoin/pdf'`.
- [ ] `document.fonts.load('10px StampFont')` probably does nothing, since StampFont only exists inside the SVG, not on the page. Confirm, then remove.
- [ ] All-caps names sit low in the box, because they're centered on the full line height including room below the baseline.
- [ ] Long names get squeezed hard because the text always fills the box height.

## How we measure

After each change, record in the PR:

- `stamp-machine.html` size in bytes, raw and gzipped
- Total bytes downloaded on a first visit
- Size of a generated PDF from the same template and names
- A check that stamped output still looks right
