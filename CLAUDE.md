# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A static, dependency-free site for browsing a personal tea collection, served via GitHub Pages at `tea.montgomerymclain.com` (set by `CNAME`). There is no build step, package manager, linter, or test suite — each page is a single self-contained HTML file with inline CSS and JS.

- `index.html` — the tea browser (search, filter chips, sort, card grid).
- `qr.html` — a printable QR code pointing at the site; loads `qrcodejs` from cdnjs.

## Running locally

Open `index.html` directly in a browser, or serve the directory (e.g. `python3 -m http.server`). When opened as a `file://` URL the direct Google Sheets fetch may be CORS-blocked; the page automatically falls back to `corsproxy.io`.

Deploying is just pushing to `main` (GitHub Pages).

## Data flow (index.html)

- **Source of truth is a Google Sheet**, not this repo. `SHEET_CSV` points at the sheet's "Publish to web → CSV" URL. Adding/editing teas happens in the sheet; no code change needed.
- Columns are looked up **by header name (case-insensitive), not position**: `Name`, `Brand`, `Type`, `Format`, `Notes`, `In Stock`, `Tags`, `Favorite`. Any column may be missing (the Brand and Tags filter rows hide themselves when empty). `Tags` is comma/semicolon-separated (slashes are part of a tag, e.g. `Sweet / dessert`); `Favorite` is any mark (`X`, `Yes`…) except blank/No. `In Stock` is truthy only when the cell is `Yes` (case-insensitive). Missing `Type` defaults to `Other`.
- `parseCSV` is a hand-rolled parser that handles quoted fields and `""` escapes but splits on newlines first, so **multi-line cell values will break parsing**.
- State is a few module-level globals (`allTeas`, `activeType`, `activeFormat`, `activeBrand`, `activeTag`, `activeStock`). `buildFilters()` derives the Type/Format/Brand/Tags chip sets (Tags gets a synthetic `Favorites` chip) from the data once; `render()` re-filters, sorts, and rebuilds the whole grid via `innerHTML` on every change.
- User-supplied sheet text must go through `escHtml()` before being inserted into HTML.

## Styling conventions

- Colors live in CSS custom properties on `:root` (purple accent `--accent: #5b3d8a`). `qr.html` hardcodes the same palette values rather than sharing them.
- Type badges are styled by class `badge-<Type>` with whitespace stripped from the type name (e.g. `Pu-erh` → `.badge-Pu-erh`). **A new tea type in the sheet needs a matching `.badge-*` rule and color variable**, otherwise the badge renders with no background.
