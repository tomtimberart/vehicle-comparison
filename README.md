# Vehicle Comparison Tool — Codebase Guide

Single self-contained file: `vehicle_comparison.html`. No build step, no dependencies except Google Fonts.

**Language**: Hebrew (RTL). All labels and UI text are in Hebrew.

---

## Architecture

Everything lives in one HTML file split into three sections:
- `<style>` — CSS with CSS variables for the dark theme
- `<body>` — Static shell + dynamic content areas rendered by JS
- `<script>` — All application logic

**State**: Two sources of truth:
1. `vehicles[]` — array of vehicle objects (synced from DOM on every `recalc()`)
2. DOM inputs — the actual editable values (globalRate, globalTerm, globalDown, holdYears, per-car fields)

---

## Data Structures

### Vehicle object
```js
{
  id: number,
  name: string,
  price: number,         // purchase price ₪
  fuel: number,          // monthly fuel cost (stored /12, displayed *12 as annual)
  maintenance: number,   // annual ₪
  repairs: number,       // annual ₪
  insurance: number,     // annual ₪
  registration: number,  // annual ₪
  depreciation: number,  // annual % (e.g. 15 = 15%)
  financing: boolean,    // whether this car uses financing
  rate: number,          // per-car annual interest % (used when perCarFinancing mode on)
  term: number,          // per-car loan term in years
  down: number,          // per-car down payment ₪
}
```

**Note on fuel**: stored in `vehicles[]` as monthly value (`v.fuel`), but displayed in the card as annual (`v.fuel * 12`). `getVehicleData()` reads the annual input and `recalc()` divides by 12 before storing.

### Global settings (read from DOM via `getGlobal()`)
```js
{ rate, term, downPct, holdYears }
```

---

## Key Functions

| Function | Purpose |
|---|---|
| `addVehicle(data?)` | Push to `vehicles[]`, call `renderVehicles()` + `recalc()` |
| `removeVehicle(id)` | Filter from `vehicles[]`, re-render |
| `getVehicleData(id)` | Read all inputs for one car from DOM → raw object |
| `calcVehicle(v)` | Pure calculation → `{annualOp, totalOp, totalFinancing, monthlyPayment, loan, totalDepreciation, saleValue, totalCost, annualTotal}` |
| `renderVehicles()` | Rebuild the entire vehicle cards grid |
| `recalc()` | Sync DOM → `vehicles[]`, then call `renderCompare()` + `renderDep()` |
| `renderCompare()` | Build summary cards, bar charts, and detailed comparison table |
| `renderDep()` | Build depreciation canvas line chart + year-by-year table |
| `switchTab(tab)` | Show/hide `#tab-input` / `#tab-compare` / `#tab-depreciation` |
| `toggleFinancingMode()` | Toggle per-car vs global financing — hides/shows DOM fields + calls `recalc()` |
| `fr(id, field, label, val, unit)` | Helper — returns a field-row HTML string for a vehicle card input |

---

## Financing Modes

Two modes controlled by `#perCarFinancing` checkbox in the global section.

**Global mode (default, unchecked)**
- `#globalRate`, `#globalTerm`, `#globalDown` are visible
- All cars use these values in `calcVehicle()`
- Per-car fields (`v${id}_rate`, `v${id}_term`, `v${id}_down`) are hidden in cards

**Per-car mode (checked)**
- Global rate/term/down fields are hidden (`display:none` on `#gf-rate`, `#gf-term`, `#gf-down`)
- Each car card shows its own rate/term/down fields inside `div#v${id}_perCarFields`
- `calcVehicle()` reads `v.rate`, `v.term`, `v.down` instead of globals
- Defaults: 5.5% / 5 years / ₪40,000

`calcVehicle()` decision logic:
```js
const perCar = document.getElementById('perCarFinancing')?.checked;
const rate = (perCar && v.rate != null) ? v.rate : g.rate;
const term = (perCar && v.term != null) ? v.term : g.term;
const down = (perCar && v.down != null) ? v.down : g.downPct;
```

---

## Save / Load System

Uses `localStorage` under key `vehicle_comparison_saves` as a JSON object: `{ [saveName]: snapshot }`.

### Snapshot shape
```js
{
  vehicles: [...],   // copy of vehicles[] with all fields
  global: {
    rate, term, downPct, holdYears,
    perCarFinancing: boolean  // financing mode persisted
  },
  savedAt: string    // Hebrew locale date string
}
```

Key functions: `getSnapshot()`, `applySnapshot(snap)`, `doSave()`, `loadSave(name)`, `deleteSave(name)`, `exportJSON()`, `importJSON(event)`.

---

## UI Structure

```
.header
.container
  .tabs  (input | compare | depreciation)
  .save-bar  (save / load / export / import buttons)
  #saveModal  (modal overlay)
  #loadModal  (modal overlay)
  #tab-input
    .vehicles-grid   ← vehicle cards rendered here
    .financing-section  ← global settings + perCarFinancing checkbox
  #tab-compare
    #summaryCards
    .charts-row  (#totalCostChart, #annualBreakChart)
    .comparison-section  (#compTable)
  #tab-depreciation
    canvas#depCanvas
    #depTable
```

---

## CSS Variables
```css
--bg: #0a0d12        /* page background */
--surface: #111520   /* card background */
--surface2: #171d2e  /* card header / table header */
--border: #1e2a42
--accent: #3b82f6    /* blue — primary */
--accent2: #f59e0b   /* amber */
--accent3: #10b981   /* green — "best" highlight */
--danger: #ef4444
--text: #e8edf8
--muted: #6b7a9a
```

Bar chart colors cycle through `COLORS = ['#3b82f6','#f59e0b','#10b981','#a855f7','#ef4444','#06b6d4']`.
