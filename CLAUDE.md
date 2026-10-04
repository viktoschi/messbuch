# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`Messbuch_2.6.html` is a single, fully self-contained offline web app (German UI) for taking one's own body
measurements alone, three times per measure, cross-checking them, and exporting a SeamlyMe `.smis` measurement
file for the "Hexen" sewing patterns (Bluse 1/2, Bolero, Strümpfe, Hexenhut …). It must keep working when opened
straight from disk (`file://`) on a phone: no build step, no dependencies, no network, no external assets.

Identifiers, comments and UI text are German; keep it that way.

## Working with the file — read this first

The file is ~12.5 MB but only ~1800 lines. **Line 198 alone is ~12.4 MB**: `const DATEN = {...};` as one line of
JSON with 131 base64 images. Never `Read`/`cat`/`grep` without limits on the whole file or print line 198 — use
line ranges that skip it, `cut -c1-200`, or parse the data with Node:

```bash
# Layout: CSS ≈ 9–163, HTML skeleton ≈ 164–196, DATEN = 198, app script ≈ 199–1816
sed -n '199,400p' Messbuch_2.6.html
grep -n 'function ' Messbuch_2.6.html | cut -c1-160

# Inspect/modify DATEN
node -e 'const fs=require("fs");const L=fs.readFileSync("Messbuch_2.6.html","utf8").split("\n");
const D=JSON.parse(L[197].replace(/^const DATEN = /,"").replace(/;\s*$/,""));
console.log(Object.keys(D))'
```

When changing DATEN, rewrite line 198 as `const DATEN = ` + `JSON.stringify(D)` + `;` (one line) and keep all other
lines byte-identical.

## Verification (there are no tests)

```bash
# Syntax check of the script block
awk 'NR>=198 && NR<=1816' Messbuch_2.6.html > /tmp/app.js && node --check /tmp/app.js
```

For behaviour, load the page in headless Chromium via Playwright (`file://` URL, phone-sized viewport), collect
`pageerror`/console errors, fill an input and check `#count`, `#pruefbox` and the export. Chromium is preinstalled
at `/opt/pw-browsers/chromium`.

## Data (`DATEN`)

- `measures[]`: `{id, name, tip, group, extra, long, seamlyId}`. `id` is usually the SeamlyMe name; book-only
  measures use `b_…`/`x_…` ids and map to SeamlyMe via `seamlyId`. Groups starting with `extra_` are optional
  "Buchmaße" and don't count toward the `x / 44` progress. `long` widens the repeat tolerance (1.0 vs 0.5 cm).
- `groups[]`: section order, titles and intros. `texts`: all fixed page copy, including `fassung` (version line).
- `help[id]`: `{text, img, imageText, mehr:[{img, kurz, text}]}` — explanation and pictures from the book.
- `bilder`: image key → data URI. Keys are `sNNNN-NN` (book page/figure, referenced from `help` and
  `texts.prepImages`/`markImages`) and `sy_<fig>_h|d` (Seamly diagrams, **built dynamically** in `seamlyBlock()`
  from `seamly.karten[id].fig` — not findable by text search).
- `seamly.karten` / `seamly.figuren`: SeamlyMe code, description and diagram legend per measure.
- `defaults`: `[seamlyName, value]` pairs = the built-in measurement file used when no template is imported.
- `templateHead`: `.smis` XML header with `__NOTES__` placeholder.
- `armloch`: ids for the "Armloch und Ärmel" filter; `schnitte`: id → patterns that use the measure.

## App architecture (script block)

- **State & storage**: global `state` = `{measureId: [m1, m2, m3]}` (strings, comma decimals allowed via `zahl()`).
  Persisted to `localStorage` under `messbuch_html_v1` — do not rename, existing users' data lives there. If storage
  is unavailable (some browsers for `file://`), falls back to `window.name` (prefix `MESSBUCH1:`) and shows a warning
  banner. Manual backup/restore is a JSON file `{app:"Messbuch", format:2, …}`; keep it backward compatible.
- **Rendering**: no framework. `zeichne()` rebuilds `#main` completely from DATEN + state; helpers `el()`, `knopf()`,
  `karte()` (one measure card). Small updates go through `werteInKarte()`, `pillsZeichnen()`, `menueZeichnen()`.
  Overlays (menu, image zoom, dialog) use `overlayAuf()`/`overlayZu()` with history entries so Android "back" closes them.
- **Cross-checks**: `pruefungen()` is a long list of self-contained IIFEs, each calling
  `add(title, kind, status, text)` with status `OK | WARN | BAD | OPEN`. Use `mittel()`, `voll()`, `nm()`/`namen()`,
  `f1()`; emit `OPEN` (with what's needed) when inputs are missing. Results drive the header pills and menu.
- **Shoulder slope**: `winkel()` derives `shoulder_slope_neck_side_angle` from depth/drop, falling back to heights;
  exported rounded to 0.5°.
- **`.smis` export**: `baueSmis()` either fills `defaults` into `templateHead`, or — if the user imported their own
  `.smis` (`vorlage`) — `ausVorlage()` rewrites only the `<m name=… value=…/>` lines it has values for and appends
  book measures with a SeamlyMe id. Local date is used on purpose (matches the Android version).

## Conventions

- There is a parallel Android version; behaviour like the export format should stay in sync with it.
- The version appears in the file name and in `texts.fassung` ("Fassung 2.6 vom …") — update both on a release.
- Page references "Buch S. …" refer to the book "Measurement Taking"; keep sources attributed in `texts.fassung`.
