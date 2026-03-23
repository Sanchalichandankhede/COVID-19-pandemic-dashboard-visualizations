# COVID-19 Global Analytics Dashboard

A fully self-contained, single-file HTML dashboard for visualizing COVID-19 pandemic trends — cases, mortality, vaccination progress, and geographic spread — with no build step or server required.

---

## Quick Start

```bash
# Clone or download the file, then just open it:
open covid19_dashboard.html          # macOS
start covid19_dashboard.html         # Windows
xdg-open covid19_dashboard.html     # Linux
```

Or serve it locally for best results (world map requires network fetch):

```bash
python -m http.server 8080
# then open http://localhost:8080/covid19_dashboard.html
```

> **Note:** The Global Map tab fetches world topology from a CDN. An internet connection is needed for that tab only; all other tabs work fully offline.

---

## Features

### Five Dashboard Tabs

| Tab | What's inside |
|---|---|
| **Overview** | Global KPI cards · Top 10 countries bar chart · CFR by region · Sparkline trends for cases, deaths, and positivity rate |
| **Trends** | Full 2020–2023 time-series (daily cases + 7-day avg + deaths) · Cumulative cases by income group · Wave comparison chart |
| **Heatmap** | Monthly cases-per-million heatmap (12 countries × 36 months) · Weekly death-rate heatmap · Regional severity index by quarter |
| **Vaccination** | Country coverage bars · Doses administered over time · Vaccine brand distribution (doughnut) · Vaccination vs case reduction curve · Coverage by income group |
| **Global Map** | D3 choropleth rendered from real TopoJSON · Three modes: Total Cases / Deaths per 100k / Vaccination % · Top vs bottom 5 countries bar · Continent polar-area chart |

### Global KPI Cards

| Metric | Value |
|---|---|
| Total cases | 769 M |
| Deaths | 6.96 M (CFR 0.91 %) |
| Vaccinated | 5.57 B (70.4 % of world) |
| Recovered | 741 M (96.4 % recovery rate) |

---

## Technology Stack

| Library | Version | Purpose |
|---|---|---|
| [Chart.js](https://www.chartjs.org/) | 4.4.1 | All canvas charts (line, bar, doughnut, polar area) |
| [D3.js](https://d3js.org/) | 7.8.5 | SVG world choropleth + map projections |
| [TopoJSON](https://github.com/topojson/topojson) | 3.0.2 | Topology decoding for the world map |

All three libraries are loaded from `cdnjs.cloudflare.com` — no npm, no bundler.

---

## File Structure

Everything lives in a single `covid19_dashboard.html` file:

```
covid19_dashboard.html
├── <head>
│   ├── CSS custom properties (light + dark theme tokens)
│   ├── Layout styles (grid, panels, cards, legend, heatmap)
│   └── <script> CDN imports for Chart.js, D3, TopoJSON
│
└── <body>
    ├── .wrapper (max-width 1280px)
    │   ├── .hdr — title + tab navigation
    │   ├── .metrics — 4 KPI cards
    │   ├── #tab-overview
    │   ├── #tab-trends
    │   ├── #tab-heatmap
    │   ├── #tab-vaccination
    │   └── #tab-map
    │
    └── <script> (inline)
        ├── Helpers (isDark, textCol, gridCol, BASE_OPTS)
        ├── switchTab() — lazy tab rendering
        ├── Overview charts (country bars, CFR, sparklines)
        ├── buildTrends() — time-series, income, wave charts
        ├── buildHeatmap() + buildWeeklyHeatmap()
        ├── buildVacc() — all vaccination charts
        └── drawWorldMap() — D3 choropleth + side charts
```

---

## Tab Rendering

Tabs are **lazily rendered** — each tab's charts are built only the first time that tab is visited. Boolean flags (`mapDrawn`, `hmDrawn`, `trendsDrawn`, `vaccDrawn`) prevent duplicate chart creation.

```js
function switchTab(t) {
  // show the selected tab section
  // set active nav button
  if (t === 'map'         && !mapDrawn)    drawWorldMap();
  if (t === 'heatmap'     && !hmDrawn)     { buildHeatmap(); buildWeeklyHeatmap(); hmDrawn = true; }
  if (t === 'trends'      && !trendsDrawn) { buildTrends();  trendsDrawn = true; }
  if (t === 'vaccination' && !vaccDrawn)   { buildVacc();    vaccDrawn = true; }
}
```

---

## Theming

The dashboard supports automatic **light and dark mode** via CSS `prefers-color-scheme`. All colors are defined as CSS custom properties in `:root`:

```css
:root {
  --bg-primary:    #ffffff;
  --bg-secondary:  #f4f4f2;
  --text-primary:  #1a1a18;
  --text-secondary:#666660;
  --border-light:  rgba(0,0,0,0.08);
  /* ... color ramps: blue, teal, red, coral, amber, purple, pink, green */
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg-primary:   #1e1e1c;
    --text-primary: #f0f0ec;
    /* ... */
  }
}
```

Chart.js colors are set via the `isDark` flag at runtime so canvas elements also adapt.

---

## Customizing the Data

All chart data is hardcoded as JavaScript arrays/objects inside the `<script>` block. To swap in real data:

### Country Case Counts (Overview tab)
```js
// Around line 410 in the script block
const countries = [
  { name: 'United States', cases: 103e6, color: '#378ADD' },
  // Add / modify entries here
];
```

### Time-Series (Trends tab — inside buildTrends())
```js
labels: ['Jan 20','Feb 20', ...],      // x-axis date labels
datasets: [{ data: [12000, 15000, ...] }]  // daily values
```

### Vaccination Coverage (Vaccination tab — inside buildVacc())
```js
const vaccData = [
  { name: 'China',  pct: 92, col: '#9FE1CB' },
  // ...
];
```

### World Map Color Values (Global Map tab)
```js
// ISO 3166-1 numeric codes → value
const casesByISO  = { '840': 103, '356': 45, ... };   // millions
const deathsByISO = { '840': 230, '356':  53, ... };   // per 100k
const vaccByISO   = { '840':  70, '356':  67, ... };   // percent
```

---

## Connecting a Real Data Source

To pull live data, replace the hardcoded arrays with a `fetch()` call to a public API before building the charts:

```js
// Example: Our World in Data CSV
const CSV_URL = 'https://covid.ourworldindata.org/data/owid-covid-data.csv';

fetch(CSV_URL)
  .then(r => r.text())
  .then(csv => {
    const rows = parseCSV(csv);           // parse with PapaParse or d3.csvParse
    const usaRows = rows.filter(r => r.iso_code === 'USA');
    const labels  = usaRows.map(r => r.date);
    const data    = usaRows.map(r => +r.new_cases_smoothed);
    buildTrendChart(labels, data);        // pass into your chart function
  });
```

Useful free sources:

| Source | URL |
|---|---|
| Our World in Data | `https://github.com/owid/covid-19-data` |
| Johns Hopkins (archived) | `https://github.com/CSSEGISandData/COVID-19` |
| WHO | `https://data.who.int/dashboards/covid19` |
| disease.sh API | `https://disease.sh/v3/covid-19` |

---

## Browser Compatibility

| Browser | Support |
|---|---|
| Chrome / Edge 90+ | ✅ Full |
| Firefox 88+ | ✅ Full |
| Safari 14+ | ✅ Full |
| Mobile (iOS/Android) | ✅ Responsive layout |
| IE 11 | ❌ Not supported |

---

## Project Structure (if extracted)

If you choose to split the file into separate assets:

```
project/
├── index.html
├── css/
│   └── dashboard.css
├── js/
│   ├── overview.js
│   ├── trends.js
│   ├── heatmap.js
│   ├── vaccination.js
│   └── map.js
└── data/
    └── covid-data.json     ← replace hardcoded values here
```

---

## License

This project is released for educational and demonstration purposes. Pandemic statistics shown are approximate figures based on publicly reported data from 2020–2023. Always cite primary sources (WHO, CDC, OWID) for research or publication.
