# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## Repository overview

This repo contains a single-page web app: **IRS Survey Allocation Tracker**, a tool
for the SEA region (Singapore / Malaysia / Thailand / Philippines) team at the
Indian Register of Shipping to track vessel survey assignments.

There is no build system, package manager, or server — the entire application
lives in one file:

```
index.html   # everything: markup, CSS, and JS in a single file
```

Open `index.html` directly in a browser to run the app. There is nothing to
install, compile, lint, or test.

## Architecture

Everything is inline in `index.html`, structured in three parts:

1. **`<style>`** — a dark-themed, custom CSS design system (CSS variables in
   `:root`: `--bg`, `--surface`, `--accent`, `--green`, `--yellow`, `--red`, etc.).
   No CSS framework is used.
2. **HTML body** — header stats bar, a two-column layout (`.left-panel` for the
   email parser, `.right-panel` for the tracker table), plus three modals
   (Manual Add, Edit, Export).
3. **`<script>`** — vanilla JS, no framework, no modules. All functions are
   global and wired up via inline `onclick`/`oninput` handlers in the HTML.

### External dependencies (all loaded via CDN `<script>` tags, no local copies)

- `xlsx` (SheetJS) — Excel export
- `jspdf` + `jspdf-autotable` — PDF export
- Google Fonts (`JetBrains Mono`, `Syne`)

### Data model & persistence

- State is a single in-memory array `surveys`, persisted to
  `localStorage` under the key `irs_surveys` (see `save()`).
- There is no backend — all data lives in the browser's localStorage. Clearing
  browser storage wipes all tracked surveys.
- Each survey entry has the shape:
  ```js
  {
    id, vesselName, vesselType, imoNumber, eta, port,
    surveyType, surveyor, status, notes, createdAt
  }
  ```
- `surveyor` is one of: `WYC`, `CHMA`, `AP (Self)`, `Unassigned`.
- `status` is one of: `Pending`, `Confirmed`, `Rescheduled`, `Completed`.

### Key functions (all in the `<script>` block at the bottom of `index.html`)

- `parseEmail()` — sends pasted email text to the Anthropic Messages API
  (`https://api.anthropic.com/v1/messages`, model `claude-sonnet-4-20250514`)
  to extract vessel/survey fields as JSON, then shows a preview via
  `showParsePreview()`.
- `addParsedEntry()` / `addManualEntry()` / `saveEdit()` / `deleteEntry()` —
  CRUD on the `surveys` array, always followed by `save()` + `renderTable()`.
- `renderTable()` / `getFiltered()` / `setFilter()` / `filterSearch()` — table
  rendering, filtering (by status or surveyor) and search.
- `updateStats()` — recomputes the header stat counters (total / pending /
  this week / confirmed).
- `exportExcel()` / `exportPDF()` — build downloadable reports from
  `getExportData()` (scoped to a selected month or the full tracker).

## Known issue worth knowing before editing `parseEmail()`

The `fetch` call to `https://api.anthropic.com/v1/messages` in `parseEmail()`
sends no `x-api-key` / `anthropic-version` headers and no API key, so it will
not succeed against the real Anthropic API as written (browser-side calls to
that endpoint also can't carry a secret key safely). If asked to fix or wire
up the AI parsing feature for real use, flag this rather than silently
assuming it already works, and avoid embedding a real API key directly in
client-side JS — that would leak it to anyone viewing page source.

## Conventions to follow when editing this file

- **Single-file constraint**: keep everything in `index.html` unless the user
  explicitly asks to split into multiple files/a build step. Don't introduce a
  bundler, framework, or `package.json` unless requested.
- **No comments in JS/CSS** beyond the existing section-divider comments
  (`// ─── Section ──`) — match that style if adding a new section, otherwise
  avoid adding comments that just restate the code.
- **Styling**: reuse existing CSS variables (`var(--accent)`, `var(--green)`,
  etc.) and existing class patterns (`.btn`, `.badge`, `.status-badge`,
  `.form-input`) rather than introducing new ad-hoc styles.
- **State mutations**: any change to `surveys` must be followed by `save()`
  and `renderTable()` (see existing CRUD functions for the pattern), otherwise
  the UI and localStorage will drift out of sync.
- **IDs**: new entries use `Date.now().toString()` as `id`. Keep this pattern
  for consistency (it's simple and collision-safe enough for this use case).

## Testing / verification

There is no test suite. To verify changes:
1. Open `index.html` in a browser (or serve it with any static file server).
2. Manually exercise the flow you changed — e.g. add a manual entry, edit it,
   filter/search, and try both export formats — since this is a UI-only app
   with browser-based localStorage state.
