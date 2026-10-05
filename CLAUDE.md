# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`Messbuch_2.7.html` is a single, fully self-contained offline web app (German UI) for taking one's own body
measurements alone, three times per measure, cross-checking them, and exporting a SeamlyMe `.smis` measurement
file for the "Hexen" sewing patterns (Bluse 1/2, Bolero, Strümpfe, Hexenhut …). It must keep working when opened
straight from disk (`file://`) on a phone: no build step, no dependencies, no network, no external assets.

Identifiers, comments and UI text are German; keep it that way.

## Working with the file — read this first

The file is ~12.6 MB but only ~2830 lines, with **CRLF line endings** (keep them; scripts that write the file must
emit `\r\n`). **One line alone is ~12.4 MB**: `const DATEN = {...};`, the first line of the `<script>` block, holding
JSON with 131 base64 images. Never `Read`/`cat`/`grep` without limits on the whole file or print that line — use line
ranges that skip it, `cut -c1-200`, or parse the data with Node. Line numbers shift with edits, so look them up:

```bash
D=$(grep -n -m1 '^const DATEN = ' Messbuch_2.7.html | cut -d: -f1)   # CSS and HTML skeleton above, app code below
grep -n 'function ' Messbuch_2.7.html | cut -c1-160

# Inspect/modify DATEN
node -e 'const fs=require("fs");const L=fs.readFileSync("Messbuch_2.7.html","utf8").split("\r\n");
const i=L.findIndex(l=>l.startsWith("const DATEN = "));
const D=JSON.parse(L[i].replace(/^const DATEN = /,"").replace(/;\s*$/,""));
console.log(Object.keys(D))'
```

When changing DATEN, rewrite that line as `const DATEN = ` + `JSON.stringify(D)` + `;` (one line) and keep all other
lines byte-identical.

## Verification (there are no tests)

```bash
# Syntax check of the script block (the one-line theme script in <head> doesn't match these patterns)
A=$(grep -n $'^<script>\r$' Messbuch_2.7.html | cut -d: -f1); B=$(grep -n $'^</script>\r$' Messbuch_2.7.html | cut -d: -f1)
awk -v a=$A -v b=$B 'NR>a && NR<b' Messbuch_2.7.html > /tmp/app.js && node --check /tmp/app.js
```

For behaviour, load the page in headless Chromium via Playwright (`file://` URL, phone-sized viewport), seed
`localStorage["messbuch_html_v1"]` with `{state:{…}}` via `addInitScript`, collect `pageerror`/console errors, fill an
input and check `#count`, `#pruefbox` and the export. Chromium is preinstalled at `/opt/pw-browsers/chromium`.
To mimic iOS Safari (no scroll anchoring), inject `*{overflow-anchor:none !important}`.

When touching a check in `pruefungen()`, compare its results (title/kind/status/text) against
`git show HEAD:Messbuch_2.7.html` over many randomly perturbed states — statuses must not change by accident.

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
- `defaults`: `[seamlyName, value]` pairs = the built-in measurement file used when no template is imported. Its
  names (the ones the patterns read) also count with a template: measured ones missing there are appended.
- `templateHead`: `.smis` XML header with `__NOTES__` placeholder.
- `armloch`: ids for the "Armloch und Ärmel" filter; `schnitte`: id → patterns that use the measure.

## App architecture (script block)

- **State & storage**: global `state` = `{measureId: [m1, m2, m3]}` (strings as typed). `zahl()` is strict: only a
  positive number with `,` or `.` decimals and an optional `cm` suffix counts (`37,` while typing too); anything else
  (`39,,5`, `37 5`, `0`) is NaN, so `unlesbar()` is true and `werteInKarte()` marks the field (class `unlesbar`,
  `aria-invalid`, text in `lesbar_<id>`: "nicht lesbar – zählt nicht mit"). Keep this reading rule in sync with the
  Android version. Persisted to `localStorage` under `messbuch_html_v1` — do not rename, existing users' data lives
  there. If storage is unavailable (some browsers for `file://`), falls back to `window.name` (prefix `MESSBUCH1:`)
  and shows a warning banner. Manual backup/restore is a JSON file `{app:"Messbuch", format:2, …}`; keep it backward
  compatible. `wach` (keep screen on), `zuletzt` (last edited measure, for "Weiter bei …"), `ungesichert`, `seit`/
  `seitZeit` (what changed since the last backup — measure ids, `notiz`, `vorlage`, `alles` — and since when) and
  `zeit` belong to the device: `standText()` writes them for `sichern()` only; they must not go into `sicherungText()`.
  - Entries are normalized by `eintrag()` (always three strings; a lone value becomes Messung 1, anything else empty)
    in `auspacken()`, `sicherungLaden()` and `gemessen()`, so a broken stored value cannot stop the app; the start
    `zeichne()` sits in a try/catch that still offers saving/loading a backup.
  - Ids this version doesn't know (from a newer one) are kept in `state` and in the backup; everything counts and
    renders via `M`/`BY`, so they are simply carried along. Don't delete unknown ids.
  - `laden()` reads localStorage and `window.name` and takes the newer one (`zeit`); if `window.name` wins it saves
    at once. `gespeichert` is the last string this tab wrote or read: a `storage` event for `KEY` (not `KEY+"_test"`/
    `"_vorher"`) or becoming visible with a different stored string runs `uebernehmen()` (reload, redraw, back to
    the same place via `ankerJetzt()`/`zumAnker()`, short `meldung()`); `uebernimmt` keeps `sichern()` from writing
    back meanwhile, so two tabs can't ping-pong.
  - Undo: before "Sicherung laden", "Messungen löschen" and "Alles zurücksetzen", `merkeVorher(art)` keeps the whole
    `standText()` in memory and under `KEY+"_vorher"`; the Sicherung section offers "Rückgängig: Stand vor …"
    (`rueckgaengig()`, which itself keeps the current state the same way).
  - Backup reminders stay quiet: `geaendert(was)` marks changes, `gesichertStand(datum)` clears them (saving a
    backup, loading one — then `gesichert` is the backup's date). The Ergebnis says what changed since (`seitText()`),
    after an export a button offers the backup (`nachExport`, no second automatic download), and at start
    `mahnungZeigen()` shows `#sicherungsmahnung` once if changes are older than a day. No header pill while typing.
  - Theme: the page starts dark (`<html data-theme="dark">`, dark `theme-color`); a one-line script in `<head>`
    switches to light before the first paint if the stored state says so. `themaAnwenden()` sets both later.
- **Rendering**: no framework. `zeichne()` rebuilds `#main` completely from DATEN + state; helpers `el()`, `knopf()`,
  `karte()` (one measure card; the line above the inputs starts with the measure name, `massname`, since with the
  keyboard open often only that line is visible). Small updates go through `werteInKarte()`, `pillsZeichnen()`,
  `menueZeichnen()`, `weiterZeigen()`. As soon as any value exists, Schritt 1 and 2 (`schritt1`/`schritt2` and their
  blocks) sit in a collapsed `details#vorbereitung`; `springe()` and `zumAnker()` open it, so anything inside must be
  reached through them. Above it, `#weiter` ("Weiter bei: …", also the first menu entry) jumps to `zuletzt` via
  `zeigeMass()`. Header pills follow the check list (neu messen, Konflikte, prüfen, offen, passen); `pillsRand()`
  toggles the fade mask `.pills.ueberlauf` while pills are out of sight — it must not change the header height
  (`kopfhoehe()` sets `scroll-padding-top` only on resize, filter changes and redraws), and `.mehr` is the image row.
  The filter key `offen` is labelled "Nur ohne Wert" (keys stay: the filter is stored).
- **Overlays & history**: menu, image zoom and dialog use `overlayAuf()`/`overlayZu()` with history entries so
  Android "back" closes them. Closed by a button, `overlayZu(false, danach)` goes back one step itself; the popstate
  of that step is marked by `zurueckEigen` (not the back key), and `danach` (e.g. the jump from the menu) runs only
  after it via `zurueckFertig()` (fallback after 1000 ms) — scrolling before it lands in the overlay's entry. Never
  call `history.back()` and scroll right away.
- **Screen on**: `wachSetzen(an, vonHand)` requests the wake lock (Schritt 1 and menu; `wachZeigen()` syncs all
  `input.wach` switches). The setting persists and is re-requested at start and on `visibilitychange`. A refusal
  after a tap resets the switch and shows `wachMeldung` (screen timeout hint); a refusal at start keeps the setting
  and retries on the next click (`wachNochmal`). Without `navigator.wakeLock` Schritt 1 shows only the hint.
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
  (card badge, class `neumessen`, header pill cycling through them via `neuSchritt(ids)` — the next card after
  `neuLetzt`, the last one jumped to, by DOM position, since a corrected card drops out of the list; the Ergebnis line
  "Davon N als „Neu messen“ markiert" uses it too — filter `neu`), WARN suspects =
  "Nachmessen empfohlen"; other measured cards show "✓ bestätigt", "grob geprüft" or "nicht gegengeprüft"
  (`befund_<id>`; `befundArt(id, F)` gives this one word for cards and jump buttons alike). "Neu messen" and
  "Nachmessen" cards add that all fields are averaged (overwrite the wrong value, don't add one next to it);
  "Neu messen" also offers "Werte leeren" (`leeren()`: empties the three fields, the old values stay visible as
  `vorher_<id>` until reload, focus goes to Messung 1). A new check that compares two independent ways to the same
  quantity belongs in `GLEICHUNG`; an inequality or a check with an open-ended tolerance does not.
- **Repeat measurements**: `mittel()` averages all filled fields (same as the Android version — no median or outlier
  removal). `streuung()` is the spread rounded to 0.1 as displayed (card: "Spanne x cm", orange above `grenze()`);
  `ausreisser()`/`streuHinweis()` name the one of three values that deviates from both others, for the
  "Wiederholgenauigkeit" text only.
- **Marking & navigation**: `markierungenZeichnen()` gives each card a traffic-light class like the check entries:
  `bad` (in any BAD check), `warn` (WARN only), `ok` (measured, nothing flagged), none without a value. Flagged cards
  get an index line per check at the top (`idx_<id>`: K/P chip + the other measures involved — same chip = measures
  that belong together) and, below the inputs, the checks with their numbers and jump buttons to those measures
  (`auff_<id>`; the full text where this measure is a suspect, else `ersterSatz()`; for `Messqualität` only this
  measure's part via `streuKarte()`, nothing for `Richtwert` — their texts list every measure of the app).
  `pruefboxInhalt()` gives check entries the status class and jump buttons to their measures. `massKnoepfe(ids, text,
  F)` marks jump buttons with the befund (`masssprung neu|nach|stark`: red/yellow border, "✓"). Tint colours are the
  `--badBg`/`--warnBg`/`--okBg` (+ `--onBad`/`--onWarn` for chip text) tokens in both themes. Always jump with `springe(id)`: images are `loading="lazy"` without a
  reserved size and shift the page while loading, and Safari has no scroll anchoring, so `halte()` (ResizeObserver
  on `#main`) re-aligns the target until the user touches, scrolls or types. `kopfAktualisieren()` likewise keeps the
  focused input in place when markings above it change, and remembers it (`feldZiel`/`feldOben`) so that `halte()`
  also puts it back when an image above loads afterwards — until the user touches or scrolls or leaves the field.
  All measures of a flagged check are tinted, not only the "Neu messen" suspects (decided by the user; keep it).
- **Back after a jump**: `springe(id, anker)` replaces the current history entry with `{anker:{id, dy}}` — the element
  at the top under the header (`ankerJetzt()`: card, check, heading, plus its offset to `ankerLinie()`) — and pushes
  a new entry; the back key (popstate with `anker`) returns there via `zumAnker()` (`halteDy` keeps the offset while
  images load). It stores the element, not the scroll position, since lazy images move the page by thousands of
  pixels. Take the anchor before anything that shifts the page, i.e. before `alleZeigen()` (see `zeigeMass()` and
  `neuSchritt()`); from the menu `menueAnker` (state when it was opened) is used. `springe(id, false)` adds no
  history entry (small correction inside the same card, e.g. `leeren()`). Menu entries lift a filter that hides
  their target. Test visibility with `zuSehen()`/`sichtbar()`, not with the height alone: Chrome lays out the content
  of a closed `<details>` with real sizes (hidden only via content-visibility), at positions over the cards.
  `zumAnker()` first drops the previous jump target, even when the anchor is gone.
- **Shoulder slope**: `winkel()` derives `shoulder_slope_neck_side_angle` from depth/drop, falling back to heights;
  exported rounded to 0.5°.
- **`.smis` export**: `baueSmis()` either fills `defaults` into `templateHead`, or — if the user imported their own
  `.smis` (`vorlage`) — `ausVorlage()` replaces only the `value` of each one-line `<m …/>` it has a value for (`mZeile()`: name and
  value in any order, further attributes kept; `parseSmis()` reads the same lines) and, before
  `</body-measurements>`, appends what the template lacks: measured `defaults` names, then the shoulder slope, then
  book measures with a SeamlyMe id. No name goes in twice (any `<m … name="…"` element of the template counts, even over several lines). It keeps
  the template's line endings (per line; an inserted `<notes>` gets CRLF if the file has any). `<notes>` gets
  `notizText()` (date, how many file values are measured, which still come from the template or are defaults, book
  measures outside the file names, the user's `notiz`); in the finished string an own earlier export note ("Eigene
  Messung vom …") is replaced, a foreign note stays as it was behind `NOTIZ_VORLAGE` (" – Notiz der Vorlage: "),
  and a missing `<notes>` is inserted before the first of `<unit>`/`<pm_system>`/`<personal>`/`<body-measurements>`.
  Insert user text with `split().join()`, not `replace()` (`$&`, `$'` would be replacement patterns). Local date is
  used on purpose (matches the Android version).
- **Template (`vorlage`)**: must be in cm — `smisEinheit()` (no `<unit>` = cm, case-insensitive); `vorlageFehler()`
  gives the reason a text is no usable template (no `<m>` lines, or not cm) and rejects it on import (`#datei`) and in
  `sicherungLaden()`; `exportieren()` refuses a stored non-cm template with a message. Nothing is converted.
  `dateiNamen()` (→ `DATEI`, set in `zeichne()`) = `defaults` names (value `"liste"`) ∪ template names (`"vorlage"`),
  so `standardListe()`/`inDatei()` include the built-in names with a template too; `fehltInVorlage(k)` is a built-in
  name missing from the template — unmeasured it is not in the file at all (the export question and the Ergebnis
  list these separately). `sicherungLaden()` replaces the template only when the backup brings a usable one
  (string, cm); otherwise the imported one stays, and its confirm text says which happens. "Vorlage entfernen" in the
  Ergebnis (after a confirm) sets `vorlage = null` and returns to the same place via `ankerJetzt()`/`zumAnker()`.
- **Export question & Ergebnis**: `exportieren()` asks once (`confirm`; it warns, never blocks). `dateiVerdacht()`
  collects the "Neu messen" values that go into the file (`DATEI[id]` or a `seamlyId`), and `verdachtText()` lists
  first those a pattern reads (`DATEN.schnitte`) with value and patterns, then the shoulder slope as its own line —
  counting only the inputs of the route `winkelWeg()` currently takes (`winkelEingaben()`) — then the rest as "von
  deinen Schnitten nicht gelesen"; "Nachmessen empfohlen" appears only as a count. The unmeasured file values
  (`standardListe()`, split by `fehltInVorlage()`) follow in the same question. `ergebnisBlock()` shows "Davon N als
  „Neu messen“ markiert (Verdacht)" (`V.datei` file names, slope counted once) with a button cycling through the cards
  behind them via `neuSchritt(V.karten)`, and under the shoulder slope "Beruht auf Werten mit Verdacht …".

## Conventions

- There is a parallel Android version; behaviour like the export format should stay in sync with it. Open there:
  the template export changed here (measured `defaults` names and the shoulder slope appended, `<notes>` replaced or
  inserted, template line endings kept, cm-only templates) — the Android version has to follow. The `.smis` without
  a template is byte-identical to before.
- The version appears in the file name and in `texts.fassung` ("Fassung 2.7 vom …") — update both on a release.
- `CHANGELOG.md` (German, for the user) lists what changed per version, what the Android version has to follow, known
  limits and decisions; add to it with every change.
- Page references "Buch S. …" refer to the book "Measurement Taking"; keep sources attributed in `texts.fassung`.
