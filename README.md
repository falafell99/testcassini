# AquaGuard — Agricultural Water Overuse Detection System

> **Cassini Hackathon 2025 — Hungary Track**  
> Operational Intelligence Dashboard for Agricultural Water Management  
> Pest County, Hungary · OVIT Regional Water Authority

---

## Overview

AquaGuard is a real-time agricultural water monitoring dashboard built for water management inspectors at the OVIT (Országos Vízügyi Igazgatóság) Regional Water Authority in Pest County, Hungary. It integrates data from **ESA Copernicus satellite programs (Sentinel-1, Sentinel-2)**, **Galileo OSNMA meter authentication**, the **Hungarian MePAR agricultural parcel registry**, and live **Open-Meteo weather data** to detect irrigation overuse and generate actionable enforcement recommendations.

The system allows inspectors to:
- **Monitor** 847 agricultural fields across Pest County in real time
- **Detect anomalies** where fields are irrigating significantly above their satellite-derived Crop Water Requirement (CWR)
- **Issue enforcement actions** (cease irrigation orders, warning notices, inspection flagging) directly from the interface
- **Understand spatial distribution** of water use via an interactive map with satellite data overlays
- **Track inspector activity** with a timestamped action log

---

## Live Demo

🌐 **[testcassini.vercel.app](https://testcassini.vercel.app)**

---

## Key Features

| Feature | Description |
|---|---|
| 🗺️ **Interactive Map** | CartoDB Positron light-map with colored field polygons (green/amber/red by status), Leaflet tooltips, click-to-detail |
| 📊 **Fields Table** | Sortable, filterable table of all monitored fields with animated waste score bars and soil moisture badges |
| 📈 **Reports Dashboard** | Live weather strip (Open-Meteo), 4 Chart.js charts, water consumption distribution analysis, enforcement action table |
| ⚙️ **Settings** | Inspector profile, configurable alert thresholds, API connection status, notification preferences, refresh rate |
| 🔍 **Command Palette** | `Cmd+K` spotlight search across all fields — fuzzy search by ID, owner, crop, district |
| 🔔 **Toast Notifications** | Action feedback toasts with 4-second countdown bar, slide-in animation |
| 📋 **Activity Log** | Timestamped inspector action history (cease orders, inspections, logins) |
| ⏱️ **Live Simulation** | Meter drift every 60s, satellite freshness bar (30s cycle), live pass countdowns (seconds precision) |
| 📱 **Field Detail Panel** | Slides in from right: 7-day usage vs CWR chart, Sentinel gauges, Galileo OSNMA card, typewriter system verdict, action buttons |

---

## Data Sources & Methodology

### 1. Sentinel-2 L2A — Crop Water Requirement (CWR) Estimation
**Source:** ESA Copernicus / [Sentinel Hub](https://www.sentinel-hub.com/)  
**API:** Planet Labs API (configured in Settings)

Sentinel-2 multispectral imagery (10m resolution, ~2-day revisit) is processed to compute the **NDVI (Normalized Difference Vegetation Index)** for each registered field:

```
NDVI = (NIR − Red) / (NIR + Red)
```

NDVI is combined with the **FAO-56 Penman-Monteith evapotranspiration model** and Open-Meteo's reference ET₀ to estimate each field's **Crop Water Requirement** in litres per day. This CWR is the baseline against which actual meter readings are compared.

### 2. Sentinel-1 GRD — Soil Moisture (SAR Backscatter)
**Source:** ESA Copernicus / Sentinel-1 C-band SAR  
**Revisit:** ~6-day repeat for Pest County

Sentinel-1 SAR (Synthetic Aperture Radar) backscatter is used to independently assess soil moisture levels at field level. Fields whose soil is already at or above field capacity (`Saturated` / `High` status) but continue to irrigate are flagged as high-priority anomalies — irrigation at saturation has zero agronomic benefit and constitutes water waste.

### 3. Galileo OSNMA — Meter Authentication
**Source:** EU Space Programme / [Galileo OSNMA](https://www.gsc-europa.eu/galileo/services/galileo-open-service-navigation-message-authentication-osnma)

Each water meter in the MePAR network provides a Galileo OSNMA-signed GPS location stamp. The system verifies:
- Meter GPS coordinates match the registered MePAR field boundary
- Signature is cryptographically valid (prevents meter spoofing / tampering)
- Timestamp is recent (replay attack detection)

Fields where the meter GPS offset exceeds 14m from the registered boundary are flagged for possible unauthorized tap or meter tampering.

### 4. MePAR — Field Parcel Registry
**Source:** Hungarian Agricultural Parcel Registry (MePAR)  
**URL:** [mepar.nebih.gov.hu](https://mepar.nebih.gov.hu)

MePAR provides the authoritative field boundary polygons (GeoJSON), registered owner information, crop type declarations, and monthly water quota allocations for all agricultural parcels in Hungary. Field IDs in AquaGuard correspond to MePAR parcel codes.

### 5. Open-Meteo — Live Weather & ET₀
**Source:** [Open-Meteo](https://open-meteo.com/) / [odp.met.hu](https://odp.met.hu)  
**Endpoint:**
```
https://api.open-meteo.com/v1/forecast
  ?latitude=47.5&longitude=19.0
  &daily=temperature_2m_max,temperature_2m_min,
         precipitation_sum,et0_fao_evapotranspiration
  &timezone=Europe/Budapest
  &forecast_days=7
```

Used for the 7-day forecast strip in the Reports tab and for computing daily ET₀ (reference evapotranspiration) used in the CWR model. Completely free, no API key required.

### 6. data.vizugy.hu — Hungarian Water Authority Hydrology
**Source:** OVIT National Water Management Directorate  
**URL:** [data.vizugy.hu](https://data.vizugy.hu)

Provides hydrological telemetry including river gauge data, groundwater levels, and drought district level declarations. The drought alert system (vizhiany.vizugy.hu) feeds the active drought banner displayed in the Map tab.

### 7. KSH / Eurostat — Baseline Calibration
**File:** `data-files/mez0046.xlsx` (KSH Agricultural Statistics)  
**File:** `data-files/Waterbase_v2024_1_T_WISE6_AggregatedDataByWaterBody.csv` (EEA Waterbase)

Used during development to calibrate the mock CWR baselines and water consumption figures against real Hungarian agricultural statistics. The `mez0046` dataset provides county-level irrigation water use broken down by crop type.

---

## Anomaly Detection Logic

```
wastePercent = ((actualUse - CWR) / CWR) × 100

status = 'anomaly'  if wastePercent > 40%   AND soilMoisture ∈ {Saturated, High}
status = 'warning'  if wastePercent > 15%
status = 'ok'       if wastePercent ≤ 15%
```

**Enforcement urgency levels:**

| Urgency | Condition | Recommended Action |
|---|---|---|
| 🔴 Critical | waste > 80%, soil saturated | Issue Cease Irrigation Order |
| 🔴 High | waste > 50% | Issue Cease Irrigation Order |
| 🟡 Medium | waste 30–50% | Send Warning Notice |
| 🟡 Low | waste 15–30% | Monitor + Flag Next Cycle |
| ✅ None | waste < 15% | No Action Required |

---

## Project Structure

```
testcassini/
├── frontend/                        # React 18 + Vite SPA
│   ├── src/
│   │   ├── components/
│   │   │   ├── App.jsx              # Root: state, routing, intervals
│   │   │   ├── TopBar.jsx           # Fixed header: brand, freshness bar, anomaly badge
│   │   │   ├── Sidebar.jsx          # Dark navy collapsible navigation
│   │   │   ├── MapView.jsx          # Leaflet map + left panel (anomalies, layers)
│   │   │   ├── FieldDetailPanel.jsx # Slide-in right panel: charts, gauges, actions
│   │   │   ├── FieldsTable.jsx      # Sortable/filterable data table
│   │   │   ├── ReportsTab.jsx       # Weather strip, KPI cards, 4 charts, enforcement
│   │   │   ├── SettingsTab.jsx      # Accordion: profile, thresholds, API, notifications
│   │   │   ├── CommandPalette.jsx   # Cmd+K spotlight search
│   │   │   ├── Toast.jsx            # Notification toasts with countdown bar
│   │   │   ├── ActivityLog.jsx      # Inspector action history panel
│   │   ├── data/
│   │   │   └── fields.js            # All mock field data, county trend, summary stats
│   │   ├── hooks/
│   │   │   ├── useCounterAnimation.js  # Animates numbers from 0 → target on mount
│   │   │   └── useWeatherData.js       # Fetches Open-Meteo 7-day forecast
│   │   ├── styles/
│   │   │   └── theme.css            # CSS custom properties, keyframes, utility classes
│   │   └── main.jsx                 # React DOM entry point
│   ├── index.html                   # HTML shell with Google Fonts import
│   ├── package.json
│   └── vite.config.js
│
├── backend/
│   └── main.py                      # FastAPI: Planet Labs satellite imagery proxy
│
├── data-files/                      # Reference datasets (not served at runtime)
│   ├── mez0046.xlsx                 # KSH Hungarian agricultural water use statistics
│   └── Waterbase_v2024_1_T_WISE6_AggregatedDataByWaterBody.csv  # EEA Waterbase
│
├── requirements.txt                 # Python dependencies for backend
├── vercel.json                      # Vercel build + routing configuration
└── README.md
```

---

## Technology Stack

### Frontend
| Technology | Version | Purpose |
|---|---|---|
| React | 18 | UI component framework |
| Vite | 6+ | Build tool and dev server |
| React-Leaflet | 4+ | Interactive map rendering |
| Chart.js + react-chartjs-2 | 4+ | All dashboard charts |
| Lucide React | latest | Icon library |
| Tailwind CSS | v4 (config-free) | CSS utility baseline |
| DM Sans + DM Mono | Google Fonts | Typography |

### Backend
| Technology | Version | Purpose |
|---|---|---|
| FastAPI | 0.110.0 | Serverless API endpoint |
| Uvicorn | 0.27.1 | ASGI server (local dev) |
| Requests | 2.31.0 | Planet Labs API proxy |

### External APIs
| API | Auth | Used For |
|---|---|---|
| Open-Meteo | None (free) | 7-day weather forecast, ET₀ |
| Planet Labs | API Key | Satellite imagery (NDVI tiles) |
| CartoDB / CARTO | None (free tiles) | Map base layer (Positron light) |

### Deployment
| Service | Config |
|---|---|
| Vercel | `vercel.json` — static frontend + Python serverless functions |
| GitHub | `falafell99/testcassini` — auto-deploy on push |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Browser (SPA)                         │
│  React 18 + Vite                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │  MapView │ │  Fields  │ │ Reports  │ │ Settings │   │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────────┘   │
│       │            │            │                        │
│  ┌────▼────────────▼────────────▼──────────────────┐    │
│  │              App.jsx (root state)                │    │
│  │  - selectedField, activeTab, toasts, activityLog │    │
│  │  - setInterval: meter drift (60s)                │    │
│  │  - setTimeout: new anomaly demo (90s)            │    │
│  └────────────────────────────────────────────────┬─┘    │
└──────────────────────────────────────────────────┼──────┘
                                                   │
              ┌────────────────────────────────────┘
              │ fetch() calls
              ▼
┌─────────────────────────┐   ┌──────────────────────────┐
│  Open-Meteo (free)       │   │  Vercel Serverless        │
│  api.open-meteo.com      │   │  /api/* → backend/main.py │
│  Weather + ET₀           │   │  → Planet Labs API proxy  │
└─────────────────────────┘   └──────────────────────────┘
```

---

## Component Reference

### `App.jsx`
Root component managing all application state. Key state:
- `activeTab` — current visible tab (`map` | `fields` | `reports` | `settings`)
- `selectedField` — field object currently shown in detail panel (or `null`)
- `toasts` — array of notification objects
- `anomalyCount` — displayed in TopBar badge, increments on new detections
- `fields` — array of all field objects (updated by meter drift interval)

Key effects:
- **Meter drift** `setInterval(60s)` — randomly adjusts `actualUse` by ±18L to simulate live meter telemetry
- **New anomaly demo** `setTimeout(90s)` — fires a warning toast after 90 seconds to demonstrate live detection
- **`Cmd+K`** keyboard handler opens the Command Palette

### `MapView.jsx`
- Map tile: **CartoDB Positron** (`light_all`) — clean light-grey base, ideal for overlaying colored polygons
- Polygon colors: green (efficient), amber (warning), red (anomaly)
- Left panel: drought banner, county stats with animated counters, Copernicus layer toggles, live satellite pass countdowns (counts down in real seconds using `setInterval(1000ms)`), anomaly cards
- Clicking any polygon or anomaly card fires `onFieldSelect(field)` to open the detail panel

### `FieldDetailPanel.jsx`
Slides in from the right (380px width, `translateX` animation). Contains:
- 7-day Chart.js line chart: actual use (red) vs CWR estimate (dashed green)
- 4 metric cards: Actual Use, CWR, Waste Delta, Days Flagged
- Sentinel-2 NDVI gauge bar (teal)
- Sentinel-1 Soil Moisture bar (color-coded)
- Galileo OSNMA verification card (teal light background)
- System Verdict with typewriter animation (28ms/character)
- Action buttons: Cease Order / Warning Notice / Flag / Monitor (urgency-adaptive)

### `ReportsTab.jsx`
- Weather strip: fetches Open-Meteo API on mount, shimmer skeleton while loading
- KPI cards: animated counters (`useCounterAnimation` hook)
- Chart 1: Horizontal bar — water waste by field (L above CWR)
- Chart 2: Multi-line trend — actual vs CWR vs quota baseline, 7-day county view
- Chart 3: Doughnut — status distribution (Efficient / Warning / Anomaly)
- Chart 4: Stacked bar — monthly quota utilisation by field
- Water distribution section: visual breakdown of agricultural vs residential vs industrial water use
- Enforcement table: with confirm modal and `✓ Issued` state after execution

### `SettingsTab.jsx`
Five accordion sections (click header to expand/collapse):
1. **Inspector Profile** — form fields for name, badge ID, county, authority (open by default)
2. **Alert Thresholds** — range sliders for anomaly (40%) and warning (15%) thresholds
3. **API Connections** — status cards for all 5 APIs; Sentinel Hub card has key input field
4. **Notification Settings** — email and SMS toggles
5. **Data & Refresh Rate** — segmented control for polling interval

### `useWeatherData.js`
Fetches from Open-Meteo on mount. Returns `{ data, loading, error }`. `data` is the raw daily forecast object containing arrays for `time`, `temperature_2m_max`, `temperature_2m_min`, `precipitation_sum`, `et0_fao_evapotranspiration`, and `weathercode`. The `weatherIcon(code)` helper maps WMO codes to emoji.

### `useCounterAnimation.js`
Custom hook: animates a number from 0 to `target` over `duration`ms using `easeOutQuart` easing. Used for all KPI cards and county overview stats. Signature: `useCounterAnimation(target, duration, delay)`.

---

## Data Model — Field Object

Each field in `src/data/fields.js` has the following structure:

```js
{
  // Identity
  id: 'C-12',                  // MePAR parcel code
  owner: 'Nagy Bt.',           // Registered owner name
  crop: 'Sunflower',           // Declared crop type
  area: 6.1,                   // Hectares
  district: 'Gödöllő',        // Sub-district within Pest County

  // Geospatial
  coordinates: [[lat, lng], ...], // Polygon corners (WGS84)
  centroid: [lat, lng],           // Polygon centroid

  // Telemetry
  actualUse: 11200,            // Today's actual water use in litres
  cwr: 5850,                   // Satellite-derived CWR in litres
  wastePercent: 91,            // (actualUse - cwr) / cwr × 100
  soilMoisture: 'Saturated',   // Sentinel-1 SAR classification

  // Status
  status: 'anomaly',           // 'anomaly' | 'warning' | 'ok'
  daysAnomaly: 5,              // Consecutive days above threshold
  quotaUsed: 78,               // % of monthly quota consumed

  // Satellite data
  ndvi: 0.61,                  // Sentinel-2 NDVI value (0–1)

  // Galileo OSNMA
  galileoVerified: true,
  galileoHash: '3f7a...b91c',
  galileoTimestamp: '25 Apr 07:58',

  // Historical (7 days)
  history: [9800, 10400, ...], // actualUse per day
  cwrHistory: [5600, 5700, ...], // CWR estimate per day

  // Enforcement
  actionUrgency: 'critical',   // 'critical'|'high'|'medium'|'low'|'none'
  recommendedAction: 'Issue Cease Irrigation Order',
  waterRecoveryEst: 5350,      // Estimated recoverable litres/day

  // UI
  alertDescription: 'Soil saturated · Still pumping',
  systemVerdict: 'SYSTEM VERDICT: ...',  // Typewriter text
}
```

---

## County Summary Statistics

```js
SUMMARY = {
  total: 847,              // Fields monitored in Pest County (MePAR)
  efficient: 791,          // Fields within CWR + 15% threshold
  warning: 52,             // Fields 15–40% above CWR
  anomaly: 4,              // Fields > 40% above CWR with soil saturation
  waterSavedToday: 18240,  // L saved vs unmonitored baseline
  totalWasted: 27380,      // L/day excess extraction (anomaly fields)
  complianceRate: 93.8,    // % of fields within quota
}
```

---

## Installation & Local Development

### Prerequisites
- Node.js ≥ 18
- Python ≥ 3.11 (for backend, optional for frontend-only dev)
- npm or yarn

### Frontend (primary)

```bash
# 1. Navigate to frontend
cd frontend

# 2. Install dependencies
npm install

# 3. Start development server (hot reload)
npm run dev
# → http://localhost:5173

# 4. Build for production
npm run build
# → Output in frontend/dist/
```

### Backend (optional — for Planet API satellite imagery)

```bash
# From project root
pip install -r requirements.txt
cd backend
uvicorn main:app --reload --port 8000
# → http://localhost:8000
```

The Vite dev server proxies `/api/*` → `http://localhost:8000` during development (configured in `vite.config.js`).

Set the `PLANET_API_KEY` environment variable before starting the backend:
```bash
export PLANET_API_KEY=your_key_here
```

---

## Deployment (Vercel)

### Option A — CLI (quickest)

```bash
# From project root
npx vercel          # Preview deployment
npx vercel --prod   # Production deployment
```

Vercel reads `vercel.json` automatically — no manual settings needed.

### Option B — GitHub Integration (recommended for continuous updates)

1. Push your code to GitHub
2. Go to [vercel.com/dashboard](https://vercel.com/dashboard) → **Add New Project**
3. Import your repository
4. Vercel detects `vercel.json` automatically — no build settings to change
5. Under **Environment Variables**, add:
   - `PLANET_API_KEY` = your Planet Labs API key
6. Click **Deploy**

Every subsequent `git push` to the connected branch will trigger an automatic redeployment.

### `vercel.json` configuration explained

```json
{
  "buildCommand": "cd frontend && npm install && npm run build",
  "outputDirectory": "frontend/dist",
  "builds": [
    { "src": "backend/main.py", "use": "@vercel/python" }
  ],
  "rewrites": [
    { "source": "/api/(.*)", "destination": "/backend/main.py" }
  ]
}
```

- `buildCommand` — builds the Vite React app inside `frontend/`
- `outputDirectory` — serves static files from `frontend/dist/`
- `builds` — deploys `backend/main.py` as a Python serverless function
- `rewrites` — routes all `/api/*` traffic to the Python backend

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `PLANET_API_KEY` | Optional | Planet Labs API key for live NDVI satellite imagery |

The Open-Meteo weather API requires no key. All other data in the current version is either fetched from free endpoints or served from the local mock data in `src/data/fields.js`.

---

## Browser Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Cmd+K` / `Ctrl+K` | Open Command Palette (spotlight field search) |
| `↑` / `↓` | Navigate Command Palette results |
| `Enter` | Open selected field in detail panel |
| `Esc` | Close Command Palette / dismiss modal |

---

## Known Limitations & Future Work

| Item | Status | Notes |
|---|---|---|
| Live satellite imagery | 🔄 Demo mode | Requires Sentinel Hub API key + polygon-based query implementation |
| MePAR live feed | 🔄 Mock data | Production requires signed data sharing agreement with NÉBIH |
| Galileo OSNMA hardware | 🔄 Simulated | Requires physical meter hardware with OSNMA chip integration |
| SMS alerts | 📋 Planned Q3 2026 | Backend SMS provider integration (e.g., Twilio) |
| Historical data > 7 days | 📋 Planned | PostgreSQL time-series backend with 90-day field history |
| Mobile / tablet layout | 📋 Planned | Current layout optimised for desktop inspector workstation |
| Real meter API integration | 📋 Planned | MQTT or REST endpoint per-field from physical flow meters |

---

## Cassini Hackathon Context

This project was developed for the **Cassini Hackathon 2025 — Hungary Track**, which challenges teams to build applications using EU Space Programme data (Copernicus, Galileo, EGNOS) to address real-world challenges in Central and Eastern Europe.

**Problem addressed:** Agricultural water over-extraction is a major challenge in the Great Hungarian Plain (Alföld) and Pest County during summer drought periods. Current enforcement relies on manual reporting and periodic on-site inspections — there is no real-time remote monitoring system. AquaGuard proposes using freely available satellite data (already paid for by EU taxpayers via ESA) to give water authority inspectors a live operational picture, enabling targeted enforcement rather than blanket restrictions.

**Satellite data used:**
- 🛰️ **Sentinel-2** (Copernicus) — NDVI, Crop Water Requirement
- 🛰️ **Sentinel-1** (Copernicus) — SAR Soil Moisture
- 🔐 **Galileo OSNMA** (EU Space Programme) — Tamper-proof meter authentication

---

## License

MIT License — see LICENSE file for details.

---

## Team

Built at Cassini Hackathon 2025 · Hungary Track  
*AquaGuard — Making satellite data work for water inspectors*
