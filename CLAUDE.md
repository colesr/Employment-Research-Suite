# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file interactive dashboard (`index.html`, ~3700 lines) for exploring U.S. labor-market dynamics. Everything — CSS, JS, embedded data, the calibrated structural model, an offline AI knowledge base, and the human-capital pipeline datasets — lives in that one file. There is no build system, no package manager, no test runner, no server.

## Running it

Open `index.html` directly in a browser. The only external network dependency is Google Fonts (Fraunces / DM Sans / JetBrains Mono). For UI verification, either double-click the file or serve the directory (e.g. `python -m http.server`) and load `http://localhost:8000/`.

## Architecture

The file is laid out as a print-magazine pastiche with seven content tab panels + one slide-out AI overlay. Read the section banners in `<script>` — they're numbered 1–11 and map to the moving parts below.

- **Macro dataset (`DATA`, `RECESSIONS`, `SERIES`)** — 65 annual rows of `[year, unemp, gdp_growth, fed_funds, cpi_yy, lfp]` from FRED (UNRATE, A191RL1A225NBEA, FEDFUNDS, CPIAUCSL, CIVPART). `getSeries(key, fromY, toY)` is the single accessor.
- **Help Wanted datasets (`HW_*`)** — separate set of embedded constants for the human-capital panel: `HW_OCCUPATIONS` (BLS Employment Projections 2023–33), `HW_DEGREES_BY_FIELD` (NCES Table 322.10), `HW_STATE_DATA` (BLS LAUS/JOLTS/OEWS + Census ACS), `HW_STATE_POSITIONS` (12-col tile-cartogram coords), `HW_MAJOR_OUTCOMES` (ACS B23022 + College Scorecard), `HW_SUPPLY_DEMAND` (per-field openings vs. graduates). Each is labeled with its source and vintage in the comment header.
- **SVG chart primitives** (`setupAxes`, `ns`, `drawAxes`, `drawRecessions`, `pathFromPoints`) — all charts share these. There is no charting library; new charts should reuse these helpers rather than introducing one.
- **Seven content panels**, each with its own `render*` function:
  1. `renderOverview` — stats row, sparklines, big unemployment chart.
  2. `renderHistory` — dual-series viewer with twin y-axes and `correlation()`.
  3. Simulator — `VARIABLES` array defines every slider; `computeSim()` is the structural model `u = NATURAL_RATE + Σ coeff·(state - baseline)`. Presets in `applyPreset`.
  4. `runForecast` — four univariate baselines + structural IRF + √h confidence bands.
  5. `renderScatters` — Okun + Phillips scatters with era filter and `linreg()`.
  6. Methodology — static prose only.
  7. **Help Wanted** — `renderHelpWanted()` dispatches to `renderHWPoster`, `renderHWPipeline`, `renderHWGeography` (tile cartogram with metric selector + click-to-detail), `renderHWJourney` (top-occupations bars, wage curve, persistence — with compare mode), `renderHWSupplyDemand` (log-log scatter).
- **AI panel** (`section 10`) — slide-out overlay (`#aiPanel`). `KB` array holds ~75 retrieval-grounded entries; `scoreKB(query)` does keyword scoring with synonym expansion via `KB_SYNONYMS`. Optional Claude API mode if user provides a key (stored in `localStorage`). When you add a new panel/feature, also add a matching KB entry + synonyms — the AI is one of the platform's documented features.
- **Tab wiring (`wireTabs`)** is lazy — non-overview panels render on first activation and re-render on resize via the `resize` listener at the bottom. The AI tab is intercepted: it opens the overlay instead of swapping panels.

## Conventions when editing

- Keep everything in `index.html`. Splitting into separate files would break the "drop-in, no-build" premise. If you genuinely need a module, ask first.
- The model's coefficients in `VARIABLES`, `FC_VARS`, and the equations in the Methodology panel must stay consistent — the panel cites the numbers used by the code. Update both sides together.
- Use the existing SVG helpers; don't introduce D3, Chart.js, or React.
- The natural rate `NATURAL_RATE = 4.5` and trend growth `2.0` are referenced both in code and in the prose — change in lockstep.
- Animations rely on `path.getTotalLength()` + `strokeDashoffset` transitions; if you add a line chart, follow that pattern so the entrance animation matches.
- **Help Wanted data** is embedded snapshots. Re-source from the named primary documents (BLS EP, NCES Tbl 322.10, etc.) when refreshing — don't trust scraped or AI-summarized intermediaries. Vintage strings (`HW_VINTAGE`) and the §VIII.F live-refresh status messages must be updated alongside.
- **AI KB** must be expanded when you add a feature. The KB is one of the platform's documented surfaces; an undocumented feature is worse than a missing one.
- Live data refresh in §VIII.F only targets endpoints with browser CORS (DataUSA, College Scorecard, sometimes BLS). FRED is not refreshable from a static page — leave that explicit in the UI.
