# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file interactive dashboard (`index.html`, ~1900 lines) for exploring U.S. labor-market dynamics 1960–2024. Everything — CSS, JS, embedded annual data, and a calibrated structural model — lives in that one file. There is no build system, no package manager, no test runner, no server.

## Running it

Open `index.html` directly in a browser. The only external network dependency is Google Fonts (Fraunces / DM Sans / JetBrains Mono). For UI verification, either double-click the file or serve the directory (e.g. `python -m http.server`) and load `http://localhost:8000/`.

## Architecture

The file is laid out as a print-magazine pastiche with six tab panels driven by a small custom SVG plotting layer. Read the section banners in `<script>` — they're numbered 1–10 and match the moving parts below.

- **Embedded dataset (`DATA`, `RECESSIONS`, `SERIES`)** — 65 annual rows of `[year, unemp, gdp_growth, fed_funds, cpi_yy, lfp]` sourced from FRED (UNRATE, A191RL1A225NBEA, FEDFUNDS, CPIAUCSL, CIVPART). `SERIES[key].idx` is the column index; `getSeries(key, fromY, toY)` is the single accessor.
- **SVG chart primitives** (`setupAxes`, `ns`, `drawAxes`, `drawRecessions`, `pathFromPoints`) — all charts share these. There is no charting library; new charts should reuse these helpers rather than introducing one.
- **Six tab panels**, each with its own `render*` function and `wire*` event handler:
  1. `renderOverview` — stats row, sparklines, big unemployment chart.
  2. `renderHistory` — dual-series viewer with twin y-axes and `correlation()`.
  3. Simulator — `VARIABLES` array defines every slider (key, channel, coeff, baseline). `computeSim()` is the structural model: `u = NATURAL_RATE + Σ coeff·(state - baseline)` summed into cyclical/structural/frictional channels. Presets live in `applyPreset`.
  4. `runForecast` — picks one of four univariate baselines (`ar1Forecast`, `naiveForecast`, `trendForecast`, `expSmoothForecast`) and adds a per-quarter impulse response from `FC_VARS[var].irf`. CI bands use `sigma · √h · z`.
  5. `renderScatters` — Okun + Phillips scatters with era filter and OLS line via `linreg()`.
  6. Methodology — static prose only.
- **Tab wiring (`wireTabs`)** is lazy — non-overview panels render on first activation and re-render on resize via the `resize` listener at the bottom.

## Conventions when editing

- Keep everything in `index.html`. Splitting into separate files would break the "drop-in, no-build" premise. If you genuinely need a module, ask first.
- The model's coefficients in `VARIABLES`, `FC_VARS`, and the equations in the Methodology panel must stay consistent — the panel cites the numbers used by the code. Update both sides together.
- Use the existing SVG helpers; don't introduce D3, Chart.js, or React.
- The natural rate `NATURAL_RATE = 4.5` and trend growth `2.0` are referenced both in code and in the prose — change in lockstep.
- Animations rely on `path.getTotalLength()` + `strokeDashoffset` transitions; if you add a line chart, follow that pattern so the entrance animation matches.
