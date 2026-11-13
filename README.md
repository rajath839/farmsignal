# FarmSignal

Crop-risk reporting and early warning for **Mysuru (Mysore) district, Karnataka, India**.

Farmers report what they are seeing in the field. When several reports of the same crop,
sharing a symptom, cluster in one place inside a short window, FarmSignal flags a
**possible risk** for that area — never a diagnosis.

This repository is a complete, runnable full-stack project:

```
farmsignal/
├── backend/     Java 17 · Spring Boot 3 · H2 database · Maven wrapper
├── frontend/    React 18 · TypeScript · Vite · four themes
└── README.md    this file
```

---

## 1. Prerequisites

| Tool | Version | Check |
|---|---|---|
| **JDK** | 17 or newer (21 works) | `java -version` |
| **Node.js** | 20 LTS or newer | `node -v` |
| **npm** | 10 or newer (ships with Node 20) | `npm -v` |

You do **not** need Maven installed — the project includes the Maven wrapper
(`mvnw.cmd` on Windows, `mvnw` on macOS/Linux).

You do **not** need a database server — H2 runs embedded and writes a file.

---

## 2. Extract once

Extract the delivered ZIP **once**. You get a single folder:

```
farmsignal\
├── backend\
├── frontend\
└── README.md
```

Every command below is written to be run from that extracted `farmsignal` folder.
Open **Command Prompt** (or PowerShell) and `cd` into it:

```bat
cd path\to\farmsignal
```

---

## 3. Start the backend (Windows)

Open a **first** terminal window:

```bat
cd backend
mvnw.cmd clean package
mvnw.cmd spring-boot:run
```

The first `package` run downloads Maven and the dependencies, so it takes a few
minutes. Later runs are fast.

When it is up you will see:

```
Started FarmSignalApplication in ... seconds
Tomcat started on port 8080 (http)
```

Check it in a browser or with curl:

```bat
curl http://localhost:8080/api/health
```

```json
{"status":"ok","service":"farmsignal-backend","version":"1.0.0",
 "farmCount":16,"reportCount":16,"placeCount":17}
```

### If port 8080 is already in use

```bat
mvnw.cmd spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
```

Then set `VITE_PROXY_TARGET=http://localhost:8081` before starting the frontend
(see §4).

### Development profile (H2 web console)

The H2 console is **off by default** and only enabled in the `dev` profile:

```bat
mvnw.cmd spring-boot:run -Dspring-boot.run.profiles=dev
```

Then open <http://localhost:8080/h2-console> and connect with:

| Field | Value |
|---|---|
| JDBC URL | `jdbc:h2:file:./data/farmsignal` |
| User Name | `sa` |
| Password | *(leave empty)* |

---

## 4. Start the frontend (Windows)

Open a **second** terminal window (leave the backend running):

```bat
cd frontend
copy .env.example .env.local
npm install
npm run dev
```

Open <http://localhost:5173>.

### Frontend API configuration

`frontend/.env.local` (copied from `.env.example`):

| Variable | Default | Meaning |
|---|---|---|
| `VITE_API_BASE_URL` | *(empty)* | Base URL of the Java backend. **Leave empty** to use the Vite dev proxy, which forwards `/api/*` to `http://localhost:8080` — no CORS setup needed. Set it to e.g. `http://localhost:8080` to call the backend directly. |
| `VITE_DEMO_MODE` | `false` | Set to `true` **only** to run the UI against bundled sample records (e.g. a design review with no backend). Every screen then shows a "Demo data" badge. Never enable in production. |

If your backend runs on another port, also set the proxy target in the same file:

```
VITE_PROXY_TARGET=http://localhost:8081
```

**No secrets belong in this file.** Everything prefixed `VITE_` is shipped to the
browser. The weather provider key lives only in the backend environment (see §7).

### Production build

```bat
cd frontend
npm run build
```

Output goes to `frontend/dist/`. Serve it from the same origin as the API, or set
`VITE_API_BASE_URL` at build time.

---

## 5. Database behaviour

- **Engine:** H2, file mode. The database file is created at
  `backend/data/farmsignal.mv.db` on first start.
- **Persistence:** farms and reports survive a restart. Alerts are **not** stored —
  they are recomputed from the persisted reports on every request, so they can
  never go stale.
- **Schema:** `spring.jpa.hibernate.ddl-auto=update`, so existing rows are kept
  when the schema changes.
- **Sample data:** on first start only, the seeder writes the clearly-labelled
  Mysuru sample set — 17 villages, 16 farms, 16 reports. It is skipped on every
  later start, so your own reports are never overwritten.

### Reset the sample data

Stop the backend, delete the database folder, start again:

```bat
cd backend
rmdir /s /q data
mvnw.cmd spring-boot:run
```

The seeder runs again because the database is empty.

### Start with an empty database (no sample data)

```bat
set FARMSIGNAL_SEED_SAMPLE_DATA=false
mvnw.cmd spring-boot:run
```

### Put the database somewhere else

```bat
set FARMSIGNAL_DB_URL=jdbc:h2:file:D:/farmsignal-data/farmsignal
mvnw.cmd spring-boot:run
```

---

## 6. REST API

All endpoints are under `/api`. Errors use one JSON shape (see §6.1).

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/health` | Liveness + record counts |
| `GET` | `/api/dashboard` | Counters, recent reports, active alerts, weather |
| `GET` | `/api/farms` | All farms |
| `GET` | `/api/places` | Villages with their crops and coordinates |
| `GET` | `/api/reports` | All reports, newest first |
| `POST` | `/api/reports` | Submit a report → `201` with the created report |
| `GET` | `/api/alerts` | Active possible-risk areas |
| `GET` | `/api/alerts/{id}` | One alert plus the reports in it |
| `POST` | `/api/alerts/recalculate` | Recompute alerts from current reports |
| `GET` | `/api/weather` | Weather outlook (sample data unless configured) |

### 6.1 Error shape

Every failure returns the same body, with **both** `fieldErrors` (a map) and
`errors` (an array), so any client can read either:

```json
{
  "timestamp": "2026-11-13T04:37:55.809Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Please check the highlighted fields and try again.",
  "path": "/api/reports",
  "fieldErrors": {
    "lat": "That location is outside the Mysuru (Mysore) district boundary. ..."
  },
  "errors": [
    { "field": "lat", "message": "That location is outside the Mysuru (Mysore) district boundary. ..." }
  ]
}
```

### 6.2 Report submission

```bat
curl -X POST http://localhost:8080/api/reports ^
  -H "Content-Type: application/json" ^
  -d "{\"crop\":\"Tomato\",\"locationName\":\"Kadakola\",\"symptoms\":[\"Leaf spots\"],\"dateObserved\":\"2026-11-12\"}"
```

`lat` / `lng` are **optional**. When a known village is named and coordinates are
omitted, the server resolves them from the village. If you do send coordinates,
they must be inside the Mysuru district boundary.

---

## 7. Configuration reference (backend)

All settings have working defaults; override with environment variables.

| Variable | Default | Meaning |
|---|---|---|
| `SERVER_PORT` | `8080` | HTTP port |
| `FARMSIGNAL_DB_URL` | `jdbc:h2:file:./data/farmsignal` | H2 JDBC URL |
| `FARMSIGNAL_DB_USER` | `sa` | DB user |
| `FARMSIGNAL_DB_PASSWORD` | *(empty)* | DB password |
| `FARMSIGNAL_SEED_SAMPLE_DATA` | `true` | Seed the Mysuru sample set on first start |
| `FARMSIGNAL_CORS_ALLOWED_ORIGINS` | `localhost:5173`, `127.0.0.1:5173`, `localhost:4173`, `127.0.0.1:4173` | Comma-separated browser origins allowed to call the API |
| `FARMSIGNAL_WEATHER_PROVIDER` | *(empty)* | Set to `openweather` to use a live provider |
| `FARMSIGNAL_WEATHER_API_KEY` | *(empty)* | Provider key — **server-side only, never commit it** |

### Weather

`GET /api/weather` returns **sample data** (`"sampleData": true`) unless
`FARMSIGNAL_WEATHER_PROVIDER` and `FARMSIGNAL_WEATHER_API_KEY` are both set. The
frontend shows a "Sample data" badge on every card while that is the case, and it
never holds a provider key.

---

## 8. How detection works

A possible-risk area is flagged when **all** of these hold:

1. At least **3** reports of the **same crop**.
2. Every report is within **5 km** of **every other** report in the group.
3. All reports share at least **one symptom**.
4. All reports were observed within the last **14 days**.

Severity: **3 reports = Watch**, **4 = Moderate**, **5 or more = High**.

Two rules exist specifically to stop false alarms:

- **No chains.** The 5 km limit applies to the whole group, not just to
  neighbouring reports. Four reports each 4 km apart in a line span 12 km and do
  **not** form one area.
- **One shared symptom.** A group where A shares a symptom with B and B with C,
  but nothing is common to all three, is not an area.

If the same reports qualify under two different shared symptoms, only the larger
group is shown, so one area never appears twice with two different counts.

---

## 9. Location validation and boundary attribution

Report coordinates are validated against the **real administrative boundary** of
Mysuru district — a polygon, not a rectangular lat/lng box. The polygon is loaded
from `backend/src/main/resources/mysuru-district.json` and tested with a
ray-casting point-in-polygon check (with a ~30 m edge tolerance so a village on
the district line is not rejected).

**Boundary data source and attribution**

| | |
|---|---|
| Dataset | geoBoundaries gbOpen, India ADM2 (districts) |
| boundaryID | `IND-ADM2-76128533` |
| shapeID | `76128533B95879808095476` |
| boundaryYearRepresented | 2021 |
| Source | <https://www.geoboundaries.org/api/current/gbOpen/IND/ADM2/> (simplified geometry) |
| Upstream | Pathways Data Pvt. Ltd., lgdirectory.gov.in |
| Licence | Open Data Commons Open Database License 1.0 (**ODbL-1.0**) |

The same attribution is shown in the app under **About → Data and attribution**.

---

## 10. Frontend structure

```
frontend/
├── index.html                  theme applied before first paint
├── package.json / package-lock.json
├── tsconfig.json / tsconfig.app.json / tsconfig.node.json
├── vite.config.ts              dev proxy /api → :8080
├── .env.example
└── src/
    ├── main.tsx                entry
    ├── App.tsx                 shell: nav, theme picker, banner, footer
    ├── theme/ThemeProvider.tsx four themes, persisted, default Field Journal
    ├── styles/farmsignal.css   design system + all four themes
    ├── api/
    │   ├── client.ts           the ONLY module that calls fetch()
    │   ├── types.ts            DTOs + view models
    │   ├── mappers.ts          DTO → view model
    │   ├── errors.ts           ApiError, field errors, connection errors
    │   ├── index.ts            dataProvider + refreshAfterReport()
    │   ├── hooks.ts            useResource / useCombinedResource
    │   ├── AsyncState.tsx      loading / empty / error / field error
    │   ├── ConnectionBanner.tsx  global /api/health probe + Retry
    │   ├── demo.ts             opt-in demo data
    │   └── demo-data.json
    ├── components/             icons, ThemePicker, DistrictMap, ui
    ├── map/mapData.ts          district outline, towns, projection
    ├── lib/format.ts           dates, severity labels, disclaimer
    └── screens/                Dashboard, Map, Reports, ReportForm,
                                Alerts, AlertDetail, Weather, About
```

Every screen reads through `src/api/index.ts`. There is no local sample array in
any production path — demo data is reachable only when `VITE_DEMO_MODE=true`, and
then every screen is badged.

### Themes

Four selectable themes, **Field Journal** by default:

| Theme | Character |
|---|---|
| **Field Journal** | Warm paper, serif headings, earthy greens (default) |
| **Agri Intelligence** | Dark surfaces, bright signal colours |
| **Harvest Modern** | Bold type, saturated accents, high contrast |
| **Clean Botanical** | Airy white and sage, evergreen accents |

Risk meanings are identical in all four: **High / Moderate / Watch** always refer
to the same report counts. The choice is saved in `localStorage` and applied
before first paint, so there is no flash of the wrong theme.

### Accessibility

- Visible focus ring on every focusable element, in every theme.
- 44 px minimum touch targets on touch devices.
- `prefers-reduced-motion` disables all animation and smooth scrolling.
- The connection error is `role="alert"` + `aria-live="assertive"`.
- Every map marker is a focusable `role="button"` with a **descriptive** name
  (crop, village, symptoms, hectares), and a text list of every marker follows
  the map so the map is never the only route to the data.
- Form errors are tied to their field with `aria-describedby` and `aria-invalid`.

---

## 11. Verification

These are the commands that were actually run, and their results.

### Backend

```bat
cd backend
mvnw.cmd test
mvnw.cmd clean package
```

```
Tests run: 23, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

The 23 tests cover: the district polygon (vertex count, all 17 seeded villages
inside, points outside rejected even when inside the old bounding box), the
anti-chain rule, the shared-symptom rule, crop separation, the 14-day window,
severity escalation, subset-cluster removal, and the full HTTP contract
(health, dashboard, farms, places, reports GET/POST, alerts, alert detail,
recalculate, weather, validation errors, 404).

### Frontend

```bat
cd frontend
npm install
npx tsc --noEmit -p tsconfig.app.json
npm run build
```

```
tsc: no errors
vite v5.4.21 building for production...
✓ 55 modules transformed.
dist/assets/index-*.css   34.21 kB │ gzip:  7.81 kB
dist/assets/index-*.js   247.24 kB │ gzip: 76.89 kB
✓ built in 3.39s
```

### End-to-end, against the running backend

| Check | Result |
|---|---|
| `GET /api/health` | `200`, 16 farms / 16 reports / 17 places |
| `POST /api/reports` with village, no coordinates | `201`, lat/lng resolved to 12.22 / 76.72 |
| `POST` with Delhi coordinates | `400`, `fieldErrors.lat` = outside district boundary |
| `POST` with empty crop/location/symptoms/date | `400`, four field errors |
| `POST` with only `lat` | `400`, "Enter both latitude and longitude…" |
| `GET /api/alerts/{unknown}` | `404` with a clear message |
| `POST /api/alerts/recalculate` | `200` |
| `GET /api/weather` | 4 risks, all `sampleData: true` |
| New report merged into the Tomato cluster | 4 → 5 reports, severity `moderate` → `high` |
| Restart the backend | 17 reports still present; seeder did not duplicate |
| CORS preflight from `http://localhost:5173` | `200`, `Access-Control-Allow-Origin` echoed |
| `/h2-console` in the default profile | not served (dev profile only) |

### In a real browser (Chromium)

| Check | Result |
|---|---|
| Dashboard loads live data | 1 active area, 5 reports, 8 recent reports, 4 weather cards |
| Alert card report count | shown **once** ("5 reports") — the v6 duplicate is gone |
| Map | district outline renders, 26 focusable markers with descriptive names |
| Theme switch to Agri Intelligence | `data-theme` changes, dark palette applies |
| Submit a report through the form | success screen, backend count 17 → 18 |
| Stop the backend, reload | visible "Cannot reach the FarmSignal server" + Retry, **no** sample data substituted |
| Restart backend, click Retry | dashboard loads normally |

### What was not verified

- **A live weather provider.** No API key was configured, so `/api/weather`
  returns labelled sample data. The adapter path is present but untested against
  a real provider.
- **Windows itself.** All commands were run on Linux with the same wrapper
  scripts; the Windows commands above use `mvnw.cmd`, `copy` and `rmdir`, which
  were not executed on a Windows machine.
- **The original v6 bundle's screens.** `farmsignal_v6.zip` is a compiled,
  minified build with no source maps and no `.tsx` files. It was used as a
  **reference** for the design system, sample data and map geometry; the React
  source in `frontend/` is a rebuild, not a decompilation. It is editable source
  and it is wired to the Java API.

---

## 12. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Port 8080 was already in use` | Another process holds the port | `mvnw.cmd spring-boot:run -Dspring-boot.run.arguments=--server.port=8081`, and set `VITE_PROXY_TARGET=http://localhost:8081` |
| Frontend shows "Cannot reach the FarmSignal server" | Backend not running, or wrong base URL | Start the backend; check `VITE_API_BASE_URL` / `VITE_PROXY_TARGET`; click Retry |
| `mvnw.cmd` is not recognised | Wrong folder | Run it from inside `backend\` |
| `npm install` fails behind a proxy | Corporate network | Configure `npm config set proxy ...`, or use a registry mirror |
| Reports disappeared | Database folder deleted | Expected — `backend\data\` holds all data |
| Weather always says "Sample data" | No provider configured | Set `FARMSIGNAL_WEATHER_PROVIDER` and `FARMSIGNAL_WEATHER_API_KEY` on the server |

---

## 13. Disclaimer

FarmSignal flags **possible risk**, not a diagnosis. A report on its own confirms
nothing, and a flagged area does not mean a disease or pest is present. Always
inspect the crop and confirm with your local agriculture extension officer or KVK
before acting.
