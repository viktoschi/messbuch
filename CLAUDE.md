# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`Messbuch_2.6.html` is a single, fully self-contained offline web app (German UI) for taking one's own body
measurements alone, three times per measure, cross-checking them, and exporting a SeamlyMe `.smis` measurement
file for the "Hexen" sewing patterns (Bluse 1/2, Bolero, Strümpfe, Hexenhut …). It must keep working when opened
straight from disk (`file://`) on a phone: no build step, no dependencies, no network, no external assets.

Identifiers, comments and UI text are German; keep it that way.

## Working with the file — read this first

The file is ~12.5 MB but only ~1950 lines, with **CRLF line endings** (keep them; scripts that write the file must
emit `\r\n`). **One line alone is ~12.4 MB**: `const DATEN = {...};`, the first line of the `<script>` block, holding
JSON with 131 base64 images. Never `Read`/`cat`/`grep` without limits on the whole file or print that line — use line
ranges that skip it, `cut -c1-200`, or parse the data with Node. Line numbers shift with edits, so look them up:

```bash
D=$(grep -n -m1 '^const DATEN = ' Messbuch_2.6.html | cut -d: -f1)   # CSS and HTML skeleton above, app code below
grep -n 'function ' Messbuch_2.6.html | cut -c1-160

# Inspect/modify DATEN
node -e 'const fs=require("fs");const L=fs.readFileSync("Messbuch_2.6.html","utf8").split("\r\n");
const i=L.findIndex(l=>l.startsWith("const DATEN = "));
const D=JSON.parse(L[i].replace(/^const DATEN = /,"").replace(/;\s*$/,""));
console.log(Object.keys(D))'
```

When changing DATEN, rewrite that line as `const DATEN = ` + `JSON.stringify(D)` + `;` (one line) and keep all other
lines byte-identical.

## Verification (there are no tests)

```bash
# Syntax check of the script block
A=$(grep -n $'^<script>\r$' Messbuch_2.6.html | cut -d: -f1); B=$(grep -n $'^</script>\r$' Messbuch_2.6.html | cut -d: -f1)
awk -v a=$A -v b=$B 'NR>a && NR<b' Messbuch_2.6.html > /tmp/app.js && node --check /tmp/app.js
```

For behaviour, load the page in headless Chromium via Playwright (`file://` URL, phone-sized viewport), seed
`localStorage["messbuch_html_v1"]` with `{state:{…}}` via `addInitScript`, collect `pageerror`/console errors, fill an
input and check `#count`, `#pruefbox` and the export. Chromium is preinstalled at `/opt/pw-browsers/chromium`.
To mimic iOS Safari (no scroll anchoring), inject `*{overflow-anchor:none !important}`.

When touching a check in `pruefungen()`, compare its results (title/kind/status/text) against
`git show HEAD:Messbuch_2.6.html` over many randomly perturbed states — statuses must not change by accident.

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
- **Cross-checks**: `pruefungen()` is a long list of self-contained checks, each calling
  `add(title, kind, status, text[, ids[, gut]])` **exactly once** with status `OK | WARN | BAD | OPEN`. Use `mittel()`,
  `voll()`, `nm()`/`namen()`, `f1()`; emit `OPEN` (with what's needed) when inputs are missing. The result list is
  sorted BAD → WARN → OPEN → OK; each entry carries `nr` (stable check number → element id `pruefung_<nr>`), `ids`
  (affected measures) and, for BAD/WARN, `zeichen` (`K1…` / `P1…`). `nr` is the call order, so new checks go at the
  end (before `gelesen = null;`); a check that is split keeps its first part at the old place (see `anteil(i)`:
  "Vorder- und Rückenanteil ergeben den Umfang: Brust" in place, Taille/Beckenkamm/Hüfte at the end).
- **Affected measures**: without an explicit `ids` argument, `add()` uses every measure the check read through
  `werte()` since the previous `add()` (tracked in `gelesen`). Pass an explicit list when only part of what was read is
  at fault (ordered chains, per-item lists, values read only for the explanation or the tolerance), and `[]` for fit
  hints that don't doubt a measurement. Ordered chains (heights, front/back centre, arm depths) use
  `kette(seq, richtig)`: `ids` = measures of all violated neighbour pairs, `gut` = chain members in order with every
  other member (all pairs checked, not only neighbours), text = first violation only. Kinds `Richtwert`/`Messqualität`
  judge each measure on its own (`einzeln()`), so their measures aren't shown as belonging together; a `Richtwert`
  check's `gut` is the measures within range.
- **What to re-measure** (`befunde()`): each check entry also carries `gut` — the measures a passing check vouches
  for (default: its compared measures; checks that read values only for the explanation text, or pass only in parts,
  pass `gut` explicitly as 6th `add()` argument). Passing checks listed in `GLEICHUNG` (two independent ways to the
  same length) confirm strongly; other passing checks (order, minimum lengths like "Vorderer Weg", ranges,
  Richtwerte) only roughly; `Messqualität` not at all. In a BAD/WARN check the suspects are the affected measures
  without strong confirmation, narrowed to those without any confirmation when possible; if all are confirmed, the
  least often confirmed (fewest `GLEICHUNG` checks) are suspects, and only if all are confirmed equally often, all
  are. `einzeln()` checks are never narrowed: a `Messqualität` measure is always a suspect ("Messungen liegen zu weit
  auseinander"), a `Richtwert` measure is one unless strongly confirmed — then it stays "✓ bestätigt" with
  "ungewöhnlicher Wert" (`F.ungewoehnlich`) instead of "nicht nachmessen". Each suspicion carries `wie`
  (`offen | selten | alle | streuung | richtwert`) for the reason text. BAD suspects = "Neu messen"
  (card badge, class `neumessen`, header pill cycling through them via `neuSchritt`, filter `neu`), WARN suspects =
  "Nachmessen empfohlen"; other measured cards show "✓ bestätigt", "grob geprüft" or "nicht gegengeprüft"
  (`befund_<id>`). A new check that compares two independent ways to the same quantity belongs in `GLEICHUNG`; an
  inequality or a check with an open-ended tolerance does not.
- **Marking & navigation**: `markierungenZeichnen()` gives each card a traffic-light class like the check entries:
  `bad` (in any BAD check), `warn` (WARN only), `ok` (measured, nothing flagged), none without a value. Flagged cards
  get an index line per check at the top (`idx_<id>`: K/P chip + the other measures involved — same chip = measures
  that belong together) and, below the inputs, the checks with jump buttons to those measures (`auff_<id>`).
  `pruefboxInhalt()` gives check entries the status class and jump buttons to their measures. Tint colours are the
  `--badBg`/`--warnBg`/`--okBg` (+ `--onBad`/`--onWarn` for chip text) tokens in both themes. Always jump with `springe(id)`: images are `loading="lazy"` without a
  reserved size and shift the page while loading, and Safari has no scroll anchoring, so `halte()` (ResizeObserver
  on `#main`) re-aligns the target until the user touches, scrolls or types. `kopfAktualisieren()` likewise keeps the
  focused input in place when markings above it change.
- **Shoulder slope**: `winkel()` derives `shoulder_slope_neck_side_angle` from depth/drop, falling back to heights;
  exported rounded to 0.5°.
- **`.smis` export**: `baueSmis()` either fills `defaults` into `templateHead`, or — if the user imported their own
  `.smis` (`vorlage`) — `ausVorlage()` rewrites only the `<m name=… value=…/>` lines it has values for and appends
  book measures with a SeamlyMe id. Local date is used on purpose (matches the Android version).

## Conventions

- There is a parallel Android version; behaviour like the export format should stay in sync with it.
- The version appears in the file name and in `texts.fassung` ("Fassung 2.6 vom …") — update both on a release.
- Page references "Buch S. …" refer to the book "Measurement Taking"; keep sources attributed in `texts.fassung`.
