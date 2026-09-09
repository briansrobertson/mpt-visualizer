# MPT Data Explorer — Design Decisions

Distillation of the choices made building the MPT distribution explorer (`index.html`, `field-definitions.html`, alongside `mpt_histdata.xlsx`). For future-you or anyone else who touches this repo.

## Core approach

**Table + filters + CSV export, not a chart.** The ask was "give me rows I can pull into R," not "show me a trend." A visualization would have been the wrong deliverable — this is long-format panel data where the useful unit is a filtered row set.

**Static single-page app, no backend.** `index.html` fetches `mpt_histdata.xlsx` via relative path and parses it client-side with SheetJS. No build step, no server — just drop files in a GitHub Pages repo directory. Tradeoff: the full 6.6MB file gets parsed on every page load (a couple seconds). Fine at 295K rows; would need a leaner format (columnar JSON, duckdb-wasm + parquet) if the dataset grows another 5–10x.

## Filter design

- **Date range, reference-window, and target-range bounds are computed from the data at load time** — not hardcoded. The parser tracks min/max date and collects unique reference windows / target ranges in a single pass over the rows. Update the xlsx, reload the page, filters reflect reality. No file to go edit.
- **`field` isn't exposed as a raw 69-item checkbox list.** It's parsed into `field_type` (Rate/Prob) + `field_stat` at load time, so the UI gives Rate a clean 4-item stat picker (percentiles/mean/mode) and Prob a cut/hike/range-bucket picker with a numeric bps min/max — same filtering power, far less UI noise.
- **No value-range filter.** Started with one, removed it — filtering on the column you're trying to observe just gives you a way to accidentally hide the thing you came for.
- Sortable columns, paginated rendering (perf — 295K rows would choke a naive full-DOM render).
- Export button dumps the **entire filtered set**, not just the visible page, as CSV in the original 5-column schema.

## Testing

Not just written-and-assumed-correct. Stood up a local server, drove the page with a headless browser, and cross-checked filter counts against independent pandas computation on the actual file (`Prob: cut` alone → 10,205 rows; narrowed to a date window → 1,386; CSV row count matched on-screen count). Caught one real bug this way: SheetJS's `dateNF` option formatted the `date` column correctly but silently left `reference_start` in US `m/d/yy` format — inconsistent behavior across columns in the same parse call. Fixed by reading raw `Date` objects and formatting them manually instead of trusting SheetJS's string formatter.

## Visual design

Navy / burnt-orange palette, IBM Plex Sans (UI) + IBM Plex Mono (data), matching the existing EFFR timeline tool's identity rather than introducing a new look. Deliberately avoided the generic "warm cream + terracotta" AI-generated-page aesthetic — cooler grey background, orange used sparingly as an accent rather than a base tone.

## License modal

- The workbook's `LICENSE` sheet has no cell values — the text lives in floating Excel drawing objects (text boxes), which is why a plain `openpyxl`/`pandas` read comes back empty. Extracted directly from the underlying XML.
- Reproduced verbatim in the modal — it's a legal disclaimer, not something to paraphrase.
- Uses `localStorage` to remember acceptance, so it's a one-time gate per browser rather than a nag on every reload.
- The xlsx fetch/parse kicks off in the background while the modal is up, so the tool is ready instantly once accepted rather than making the user wait twice.

## Field definitions page

Separate `field-definitions.html`, opens in a new tab (so filtering state isn't lost), same visual branding as the main tool. Sourced from the `DICTIONARY` sheet with two small edits worth flagging:
- Deduped one exact-duplicate row (`Rate: mean` was listed twice with identical text — clear copy-paste artifact in the source sheet).
- Normalized one inconsistent word (the 75th-percentile description said "average" where the other three rate stats said "rate" — made it consistent).

## Known constraints

- Full dataset parsed client-side on every load — no pagination/streaming from a backend. Revisit if the file grows substantially.
- `localStorage` gates the license modal per-browser, not per-user — clearing site data or a new browser/incognito session will show it again. That's expected behavior, not a bug.
