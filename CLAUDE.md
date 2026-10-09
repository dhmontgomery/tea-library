# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A static, dependency-free site for browsing a personal tea collection, served via GitHub Pages at `tea.montgomerymclain.com` (set by `CNAME`). There is no build step, package manager, linter, or test suite — each page is a single self-contained HTML file with inline CSS and JS.

- `index.html` — the tea browser: search, filter chips (Type, Format, Tags, Stock), a Brand dropdown, a sort menu, and a card grid. Card subtitle is Brand · Format · Location, plus "Out of stock" only when a tea is out (no "In stock" label). Favorites get a gold outline (`--favorite`) and a ★ before the name.
- `qr.html` — a printable QR code pointing at the site; loads `qrcodejs` from cdnjs.

## Running locally

Open `index.html` directly in a browser, or serve the directory (e.g. `python3 -m http.server`). When opened as a `file://` URL the direct Google Sheets fetch may be CORS-blocked; the page automatically falls back to `corsproxy.io`.

Deploying is just pushing to `main` (GitHub Pages); there is no staging, so a push is live within a couple of minutes.

There is no Node in this environment. To syntax-check the inline script, extract it and parse it with macOS's JavaScriptCore:

```sh
sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > /tmp/site.js
osascript -l JavaScript -e "ObjC.import('Foundation'); new Function($.NSString.stringWithContentsOfFileEncodingError('/tmp/site.js',4,null).js); 'ok'"
```

To check what the live sheet contains (columns, values, counts), fetch the `SHEET_CSV` URL with `curl` and inspect it with Python's `csv` module.

## Data flow (index.html)

- **Source of truth is a Google Sheet**, not this repo. `SHEET_CSV` points at the sheet's "Publish to web → CSV" URL. Adding/editing teas happens in the sheet; no code change needed.
- Columns are looked up **by header name (case-insensitive), not position**: `Name`, `Brand`, `Type`, `Format`, `Notes`, `In Stock`, `Tags`, `Favorite`, `Location`. Any column may be missing (the Brand and Tags filter rows hide themselves when empty). `Location` (e.g. `Top shelf`) is display-only on cards, not a filter or sort. `Tags` is comma/semicolon-separated (slashes are part of a tag, e.g. `Sweet / dessert`); `Favorite` is any mark (`X`, `Yes`…) except blank/No. `In Stock` is truthy only when the cell is `Yes` (case-insensitive). Missing `Type` defaults to `Other`.
- `parseCSV` is a hand-rolled parser that handles quoted fields and `""` escapes but splits on newlines first, so **multi-line cell values will break parsing**.
- State is a few module-level globals (`allTeas`, `activeType`, `activeFormat`, `activeBrand`, `activeTag`, `activeStock` — defaults to `In Stock`). `buildFilters()` derives the Type/Format/Tags chip sets (Tags gets a synthetic `Favorites` chip) and the Brand dropdown options from the data once; `render()` re-filters, sorts, and rebuilds the whole grid via `innerHTML` on every change.
- User-supplied sheet text must go through `escHtml()` before being inserted into HTML, including filter chip labels/values and dropdown options.
- Claude cannot edit the Google Sheet. When a change needs new sheet data (e.g. a new column), prepare the values in the sheet's current row order and have the user paste them; then re-fetch the CSV to verify before relying on it. Rows aren't unique by name (two teas are both "Turmeric Bliss"), so match on Name + Brand.

## Local-only files

`tea-classify-input/` holds shelf photos used to build the inventory. It is gitignored (as is `.DS_Store`) and must never be committed — the repo is public.

## Styling conventions

- Colors live in CSS custom properties on `:root` (purple accent `--accent: #5b3d8a`). `qr.html` hardcodes the same palette values rather than sharing them.
- Type badges are styled by class `badge-<Type>` with whitespace stripped from the type name (e.g. `Pu-erh` → `.badge-Pu-erh`). **A new tea type in the sheet needs a matching `.badge-*` rule and color variable**, otherwise the badge renders with no background.
