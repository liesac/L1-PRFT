# Technical Specification: Sport Activity File Reader

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Functional Requirements & User Stories](#2-functional-requirements--user-stories)
3. [Backend Architecture & API Specifications](#3-backend-architecture--api-specifications)
4. [Frontend Architecture & State Management](#4-frontend-architecture--state-management)
5. [Testing Strategy & Code Quality Guardrails](#5-testing-strategy--code-quality-guardrails)
6. [User Interface Design](#6-user-interface-design)

---

## 1. System Overview

### Product Summary

The **Sport Activity File Reader** is a locally deployed web application for parsing, analyzing, and visualizing sports activity data from `.fit` files. It is a single-user, on-demand tool with no authentication, no database persistence, and no network runtime calls. All data processing happens within the local process on each request.

### Core Objectives & Scope

| Objective | Description |
|---|---|
| File-local processing | All `.fit` parsing and analysis occurs entirely in-memory on the server; no data is persisted between requests |
| Single-page experience | No client-side routing; the entire UI lives in one HTML page with dynamic section visibility |
| Zero persistence | No database, no session storage, no disk writes of processed data |
| Local-only execution | The application is designed to run on `localhost`; no production deployment configuration is required |
| Telemetry visualization | Render six performance metrics on a single unified chart with interactive controls |
| Route visualization | Display the GPS track from the activity on an interactive map |

### Deployment Architecture

The application is deployed as a **single unit**:

- The FastAPI backend serves the compiled frontend SPA as static files from the root path (`/`).
- API endpoints are scoped exclusively under `/api/v1/` to prevent route collisions with static assets.
- This co-location eliminates Cross-Origin Resource Sharing (CORS) overhead in production; CORS configuration is only needed during development via the Vite proxy.

```
Browser
  └── GET / → FastAPI StaticFiles (compiled frontend)
  └── POST /api/v1/activities/analyze → FastAPI route handlers
```

---

## 2. Functional Requirements & User Stories

### 2.1 File Upload Workflow

**User Story:** As a user, I want to upload a `.fit` activity file and immediately see my performance data visualized.

#### Step-by-Step Flow

1. **Landing State** — The page renders with an upload zone (drag-and-drop area + "Browse" button). All visualization sections are hidden.
2. **File Selection** — User drags a file onto the drop zone or clicks to open a file picker. Only `.fit` files are accepted (validated by extension and MIME type).
3. **Client-Side Validation** — Before any network call, the frontend validates:
   - File extension is `.fit`
   - File size does not exceed the configured maximum (e.g., 50 MB)
   - File is not empty (size > 0 bytes)
4. **Upload Request** — The file is sent as `multipart/form-data` to `POST /api/v1/activities/analyze`.
5. **Loading State** — A progress indicator is shown; all interactive controls are disabled during the request.
6. **Response Handling:**
   - **Success (200):** The response JSON is stored in the centralized state store. All visualization sections become visible. Charts, map, widgets, and insights are rendered from the state data.
   - **Error (4xx/5xx):** An error banner is displayed with the user-facing message and the `requestId` for debugging. The upload zone remains interactive for retry.
7. **Reset** — A "Clear / Upload New File" action resets the entire state and hides all visualization sections, returning to the landing state.

#### Client-Side Validation Rules

| Rule | Constraint |
|---|---|
| File type | Must end in `.fit`; MIME type must be `application/octet-stream` or `application/fit` |
| File size | Maximum 50 MB |
| File content | Non-empty (size > 0) |

---

### 2.2 Unified Telemetry Graph

**User Story:** As a user, I want to see all key performance metrics plotted together on a single interactive timeline so I can correlate events across metrics.

#### Metrics

| Metric | Unit | Dataset Color (placeholder) |
|---|---|---|
| Cadence | rpm | Distinct series color 1 |
| Elevation | m | Distinct series color 2 |
| Heart Rate | bpm | Distinct series color 3 |
| Power | W | Distinct series color 4 |
| Speed | km/h | Distinct series color 5 |
| Temperature | °C | Distinct series color 6 |

Each metric is rendered as an independent `line` dataset within a single Chart.js instance. Missing metrics (not present in the FIT file) are omitted from the chart silently.

#### X-Axis Toggle (Time vs. Distance)

- A UI toggle control (radio buttons or segmented control) switches the X-axis between **Elapsed Time** (seconds formatted as `HH:MM:SS`) and **Distance** (kilometers, formatted as `0.00 km`).
- The toggle updates the chart's `data.labels` array and re-renders without re-fetching from the backend; both arrays are present in the state store after the initial load.
- Default: **Elapsed Time**.

#### Dynamic X-Axis Label Auto-Skipping

- The Chart.js `ticks.autoSkip` option must be `true`.
- The `ticks.maxTicksLimit` must be derived dynamically from the chart container's rendered pixel width, not a hardcoded constant.
- Implementation: a `ResizeObserver` monitors the chart container; on width change, it recalculates `maxTicksLimit` as `Math.floor(containerWidth / MIN_LABEL_WIDTH_PX)` where `MIN_LABEL_WIDTH_PX` is a constant (e.g., `80`).
- This prevents label overlap at any viewport width without disabling all intermediate ticks.

#### Y-Axis Scale Normalization

- Each metric dataset has its own Y-axis scale registered in Chart.js to prevent cross-metric distortion (e.g., cadence at 90 rpm must not visually dominate power at 250 W).
- Each Y-axis `display` property is set to `false` to hide axis labels (the legend identifies each series).
- The `min`/`max` for each axis are computed from the dataset's actual minimum and maximum values with a 10% padding applied to both bounds.

#### Unified Hover Tooltip

- Chart.js `interaction.mode` is set to `'index'` and `interaction.intersect` to `false`.
- The tooltip callback aggregates all visible dataset values at the hovered index into a single tooltip panel.
- Tooltip format per line: `{MetricLabel}: {value} {unit}` (e.g., `Heart Rate: 142 bpm`).
- Missing data at a specific index (null values) are displayed as `—`.

---

### 2.3 Interactive Mapping

**User Story:** As a user, I want to see the GPS track of my activity on an interactive map.

#### Implementation Constraints

- Map rendered using **Leaflet.js** inside a dedicated `<div>` container.
- Tile provider: OpenStreetMap (`https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png`). No API key required.
- GPS coordinates are supplied by the backend as an ordered array of `[latitude, longitude]` pairs.
- The route is rendered as a `L.polyline` overlay on the map.
- On load, the map view is fitted to the bounds of the polyline using `map.fitBounds(polyline.getBounds())`.
- If GPS data is absent from the FIT file, the map section is hidden entirely.
- The Leaflet map instance must be destroyed and re-initialized on each new file upload to prevent stale tile/overlay state.

---

### 2.4 Performance Widgets

**User Story:** As a user, I want to see at-a-glance statistics for each key metric without inspecting the chart.

#### Widget Definitions

| Widget Group | Displayed Stats |
|---|---|
| **Cadence** | Average, Maximum, Minimum (rpm) |
| **Heart Rate** | Average, Maximum, Minimum (bpm) |
| **Speed** | Average, Maximum, Minimum (km/h) |
| **Temperature** | Average, Maximum, Minimum (°C) |
| **Elevation** | Total Gain (m), Maximum (m), Minimum (m) |
| **Activity Summary** | Total Duration (HH:MM:SS), Total Distance (km) |

- All statistics are computed server-side and returned in the API response.
- Widget values format to two decimal places for floating-point metrics.
- If a metric is unavailable (null in the response), the widget displays `N/A`.

---

### 2.5 Automated Insights

**User Story:** As a user, I want a brief natural-language summary of my activity highlights so I can understand my performance at a glance.

#### Logic Guidelines

The backend generates a structured list of insight strings based on the following rules:

| Condition | Insight |
|---|---|
| Average heart rate > 160 bpm | Flag high cardiovascular intensity |
| Average power > 250 W | Flag high power output |
| Total elevation gain > 500 m | Flag significant climbing |
| Average cadence < 75 rpm (cycling) | Suggest cadence efficiency improvement |
| Average speed > 35 km/h | Flag high-speed effort |
| Heart rate variance high (max − avg > 40 bpm) | Flag inconsistent effort |

- Insights are generated as discrete strings in the `insights` array of the API response.
- The frontend renders each insight as a separate list item; no client-side computation of insights.
- The insight logic lives in the **domain layer** as a pure function, not in the route or use case.

---

## 3. Backend Architecture & API Specifications

### 3.1 Directory Structure

```text
backend/
├── src/
│   ├── domain/
│   │   ├── entities/
│   │   │   └── activity.py          # Core Activity entity (dataclasses, no framework deps)
│   │   ├── models/
│   │   │   └── telemetry.py         # Value objects: DataPoint, GpsCoordinate, MetricStats
│   │   └── services/
│   │       ├── activity_normalizer.py   # Maps raw FIT data → Activity entity
│   │       ├── statistics_calculator.py # Computes avg/min/max/gain from data points
│   │       └── insights_generator.py    # Pure function: Activity → list[str]
│   ├── application/
│   │   └── use_cases/
│   │       └── process_activity_upload.py  # Orchestrates parse → normalize → analyze
│   ├── infrastructure/
│   │   └── fit_parser/
│   │       ├── fit_parser_port.py    # Abstract base class (port interface)
│   │       └── fit_parser_adapter.py # fitdecode implementation
│   ├── presentation/
│   │   ├── routes/
│   │   │   └── activity_route.py    # POST /api/v1/activities/analyze
│   │   └── schemas/
│   │       ├── request_schemas.py   # Multipart file upload schema
│   │       └── response_schemas.py  # Pydantic response models
│   └── main.py                      # FastAPI app, route registration, StaticFiles mount
└── tests/
    ├── fixtures/
    │   └── sample_activity.fit      # Minimal valid FIT file for tests
    ├── domain/
    │   ├── test_statistics_calculator.py
    │   ├── test_activity_normalizer.py
    │   └── test_insights_generator.py
    ├── application/
    │   └── test_process_activity_upload.py
    └── presentation/
        └── test_activity_route.py
```

### 3.2 REST API Contract

#### `POST /api/v1/activities/analyze`

Accepts a `.fit` file upload and returns the fully analyzed activity data.

**Request**

```
Content-Type: multipart/form-data

Field: file (binary, required)
  - The raw .fit file content
  - Max size enforced by FastAPI's request size limit (configurable, default 50 MB)
```

**Success Response — `200 OK`**

```json
{
  "summary": {
    "total_duration_seconds": 3600,
    "total_distance_km": 42.2,
    "sport": "cycling"
  },
  "statistics": {
    "heart_rate": {
      "average": 142.5,
      "maximum": 178.0,
      "minimum": 98.0
    },
    "cadence": {
      "average": 88.3,
      "maximum": 120.0,
      "minimum": 45.0
    },
    "speed": {
      "average": 28.4,
      "maximum": 58.2,
      "minimum": 0.0
    },
    "power": {
      "average": 210.0,
      "maximum": 650.0,
      "minimum": 0.0
    },
    "temperature": {
      "average": 22.5,
      "maximum": 27.0,
      "minimum": 18.0
    },
    "elevation": {
      "total_gain_m": 320.0,
      "maximum_m": 850.0,
      "minimum_m": 530.0
    }
  },
  "telemetry": {
    "time_series": [0, 1, 2, 3],
    "distance_series": [0.0, 0.0028, 0.0056, 0.0083],
    "heart_rate": [null, 120, 122, 125],
    "cadence": [null, 85, 87, 88],
    "speed": [null, 10.2, 10.5, 10.8],
    "power": [null, 180, 185, 190],
    "temperature": [22, 22, 22, 23],
    "elevation": [530.0, 530.2, 530.5, 531.0]
  },
  "gps_track": [
    [48.8566, 2.3522],
    [48.8567, 2.3524]
  ],
  "insights": [
    "High cardiovascular intensity: average heart rate was 142 bpm.",
    "Significant elevation gain of 320 m recorded."
  ]
}
```

**Field Nullability Rules:**
- All `statistics.*` sub-objects may be `null` if that metric was absent from the file.
- All `telemetry.*` arrays may contain `null` entries at indices where data was not recorded.
- `gps_track` may be `null` or an empty array if no GPS data was found.
- `insights` is always an array (may be empty).

**Error Response — `4xx / 5xx`**

```json
{
  "error": {
    "code": "INVALID_FILE_TYPE",
    "message": "Only .fit files are supported.",
    "details": "Received file with extension .gpx",
    "requestId": "a3f2b1c4-..."
  }
}
```

**Error Codes**

| Code | HTTP Status | Description |
|---|---|---|
| `INVALID_FILE_TYPE` | 422 | File extension or content type is not `.fit` |
| `EMPTY_FILE` | 422 | Uploaded file has zero bytes |
| `FILE_TOO_LARGE` | 413 | File exceeds size limit |
| `UNPARSEABLE_CONTENT` | 422 | File is corrupt or uses an unsupported FIT variant |
| `INTERNAL_SERVER_ERROR` | 500 | Unexpected failure |

---

### 3.3 Pydantic Schemas

#### Response Schemas (`response_schemas.py`)

```python
from pydantic import BaseModel

class MetricStats(BaseModel):
    average: float | None
    maximum: float | None
    minimum: float | None

class ElevationStats(BaseModel):
    total_gain_m: float | None
    maximum_m: float | None
    minimum_m: float | None

class ActivityStatistics(BaseModel):
    heart_rate: MetricStats | None
    cadence: MetricStats | None
    speed: MetricStats | None
    power: MetricStats | None
    temperature: MetricStats | None
    elevation: ElevationStats | None

class ActivitySummary(BaseModel):
    total_duration_seconds: int
    total_distance_km: float
    sport: str | None

class TelemetryData(BaseModel):
    time_series: list[int]
    distance_series: list[float]
    heart_rate: list[float | None]
    cadence: list[float | None]
    speed: list[float | None]
    power: list[float | None]
    temperature: list[float | None]
    elevation: list[float | None]

class ActivityAnalysisResponse(BaseModel):
    summary: ActivitySummary
    statistics: ActivityStatistics
    telemetry: TelemetryData
    gps_track: list[tuple[float, float]] | None
    insights: list[str]

class ErrorDetail(BaseModel):
    code: str
    message: str
    details: str | None
    requestId: str

class ErrorResponse(BaseModel):
    error: ErrorDetail
```

---

### 3.4 FIT Parsing Mechanism

#### Port (Abstract Interface)

```python
# infrastructure/fit_parser/fit_parser_port.py
from abc import ABC, abstractmethod

class FitParserPort(ABC):
    @abstractmethod
    def parse(self, file_bytes: bytes) -> RawActivityData:
        """Parse raw FIT bytes into a normalized intermediate structure."""
        ...
```

#### Adapter (fitdecode Implementation)

- `FitParserAdapter` implements `FitParserPort`.
- Receives raw file bytes; constructs an in-memory `BytesIO` stream.
- Uses `fitdecode.FitReader` to iterate over all FIT messages.
- Collects `record` messages (per-second telemetry) and the `session` message (activity summary).
- Maps FIT field names to the `RawActivityData` intermediate structure.
- Handles `fitdecode.FitParseError` and re-raises as a domain exception (`UnparseableFileError`).
- No global state; every call to `parse()` is fully isolated.

#### Data Flow

```
POST /api/v1/activities/analyze
  → ActivityRoute.analyze()
    → ProcessActivityUploadUseCase.execute(file_bytes)
      → FitParserAdapter.parse(file_bytes)      # Infrastructure
        → RawActivityData                        # Intermediate DTO
      → ActivityNormalizer.normalize(raw_data)   # Domain service
        → Activity entity
      → StatisticsCalculator.calculate(activity) # Domain service
        → ActivityStatistics
      → InsightsGenerator.generate(activity, stats) # Domain service
        → list[str]
      → ActivityAnalysisResponse                 # Pydantic schema
    ← HTTP 200 JSON
```

---

### 3.5 Static File Serving

```python
# main.py (excerpt)
from fastapi.staticfiles import StaticFiles

app.mount("/", StaticFiles(directory="static", html=True), name="static")
```

- The `static/` directory inside `backend/` contains the Vite production build output.
- The mount is registered **after** all API routers to avoid shadowing `/api/v1/*` routes.
- In development, this mount is not active; Vite's dev server handles the frontend independently.

---

## 4. Frontend Architecture & State Management

### 4.1 Directory Structure

```text
frontend/
├── src/
│   ├── core/
│   │   ├── api/
│   │   │   ├── activityApi.js               # fetch() wrapper for /api/v1/activities/analyze
│   │   │   └── activityResponseMappers.js   # API JSON → normalized UI state shape
│   │   ├── events/
│   │   │   └── eventBus.js                  # Pub/sub event bus (no DOM events)
│   │   └── state/
│   │       └── appState.js                  # Centralized state store with subscriptions
│   ├── features/
│   │   ├── upload/
│   │   │   ├── uploadController.js          # Orchestrates file selection, validation, submission
│   │   │   └── uploadView.js                # Drop zone DOM manipulation
│   │   ├── summary/
│   │   │   ├── summaryController.js         # Subscribes to state, triggers widget renders
│   │   │   └── summaryView.js               # Renders metric stat widgets
│   │   ├── charts/
│   │   │   ├── chartController.js           # Manages Chart.js lifecycle, resize observer
│   │   │   ├── chartConfigFactory.js        # Builds Chart.js config objects (pure function)
│   │   │   └── chartView.js                 # Chart canvas initialization
│   │   ├── maps/
│   │   │   ├── mapController.js             # Manages Leaflet instance lifecycle
│   │   │   └── mapView.js                   # Leaflet container initialization
│   │   ├── tables/
│   │   │   ├── tableController.js           # (optional) Tabular telemetry data view
│   │   │   └── tableView.js
│   │   └── status/
│   │       ├── statusController.js          # Subscribes to state, renders loading/error banners
│   │       └── statusView.js
│   ├── shared/
│   │   ├── components/
│   │   │   ├── statWidget.js                # Reusable metric stat card renderer
│   │   │   └── toggle.js                    # Reusable toggle UI component
│   │   └── utils/
│   │       ├── formatters.js                # Duration, distance, metric value formatters
│   │       └── validators.js                # File validation pure functions
│   └── main.js                              # Module wiring entry point
├── public/
│   └── index.html
├── tests/
│   ├── core/
│   ├── features/
│   └── shared/
├── vite.config.js
├── .eslintrc.js
└── .prettierrc
```

---

### 4.2 Event-Driven Lifecycle

#### Centralized State Store (`appState.js`)

The state store is the single source of truth for all UI data. It exposes:

```js
// State shape
const initialState = {
  status: 'idle',          // 'idle' | 'loading' | 'success' | 'error'
  error: null,             // { code, message, requestId } | null
  activity: null,          // Normalized activity data | null
  xAxisMode: 'time',       // 'time' | 'distance'
};

// API
appState.get()             // Returns a frozen copy of current state
appState.set(partial)      // Merges partial into state and notifies subscribers
appState.subscribe(fn)     // Registers a listener called with (newState, prevState)
appState.unsubscribe(fn)   // Removes a listener
```

No global `window.*` assignment; the store is imported as an ES module wherever needed.

#### Event Bus (`eventBus.js`)

Used for decoupled side-effect signaling between features (e.g., `upload:complete`, `chart:resize`):

```js
eventBus.on('upload:complete', handler)
eventBus.emit('upload:complete', payload)
eventBus.off('upload:complete', handler)
```

Features must not import each other directly. All cross-feature communication flows through `appState` subscriptions or `eventBus` events.

#### Lifecycle Sequence

```
User selects file
  → uploadController validates file
  → uploadController calls activityApi.analyze(file)
  → appState.set({ status: 'loading' })
    → statusController re-renders loading indicator

API responds (success)
  → activityResponseMappers.mapResponse(json) → normalized activity object
  → appState.set({ status: 'success', activity })
    → summaryController renders widgets
    → chartController renders telemetry chart
    → mapController renders GPS track
    → statusController hides loading indicator

API responds (error)
  → appState.set({ status: 'error', error })
    → statusController renders error banner
```

---

### 4.3 Chart.js Implementation Constraints

#### Configuration (built by `chartConfigFactory.js`)

```js
// chartConfigFactory.js — pure function, no side effects
export function buildChartConfig({ telemetry, xAxisMode }) {
  return {
    type: 'line',
    data: {
      labels: xAxisMode === 'time'
        ? telemetry.time_series.map(formatDuration)
        : telemetry.distance_series.map(formatDistance),
      datasets: buildDatasets(telemetry),
    },
    options: {
      animation: false,
      interaction: { mode: 'index', intersect: false },
      plugins: {
        tooltip: { callbacks: { label: tooltipLabelCallback } },
        legend: { position: 'top' },
      },
      scales: buildScales(telemetry),
    },
  };
}
```

#### Y-Axis Scale Normalization (`buildScales`)

```js
function buildScales(telemetry) {
  const scales = { x: { ticks: { autoSkip: true } } };
  METRICS.forEach((metric, i) => {
    const values = telemetry[metric].filter(v => v !== null);
    if (!values.length) return;
    const min = Math.min(...values);
    const max = Math.max(...values);
    const padding = (max - min) * 0.1 || 1;
    scales[`y-${metric}`] = {
      type: 'linear',
      display: false,
      min: min - padding,
      max: max + padding,
      grid: { drawOnChartArea: i === 0 },
    };
  });
  return scales;
}
```

Each dataset references its own Y-axis via `yAxisID: 'y-{metric}'`.

#### Dynamic Label Skipping

```js
// chartController.js
const resizeObserver = new ResizeObserver(([entry]) => {
  const width = entry.contentRect.width;
  const maxTicks = Math.floor(width / MIN_LABEL_WIDTH_PX);
  chart.options.scales.x.ticks.maxTicksLimit = maxTicks;
  chart.update('none'); // skip animation
});
resizeObserver.observe(chartContainer);
```

---

### 4.4 Leaflet Implementation Constraints

```js
// mapController.js
let mapInstance = null;
let polylineInstance = null;

function renderMap(gpsTrack) {
  if (mapInstance) {
    mapInstance.remove(); // destroy previous instance
    mapInstance = null;
  }
  mapInstance = L.map('map-container');
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap contributors',
  }).addTo(mapInstance);
  polylineInstance = L.polyline(gpsTrack, { color: '#3b82f6', weight: 3 });
  polylineInstance.addTo(mapInstance);
  mapInstance.fitBounds(polylineInstance.getBounds());
}
```

---

### 4.5 Vite Configuration

```js
// vite.config.js
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    outDir: '../backend/static',
    emptyOutDir: true,
  },
  server: {
    proxy: {
      '/api': {
        target: 'http://127.0.0.1:8000',
        changeOrigin: false,
      },
    },
  },
});
```

- `outDir` points directly into `backend/static/` so `npm run build` produces the static assets ready for FastAPI to serve.
- The dev proxy routes all `/api/*` requests to the FastAPI process, eliminating CORS during development.

---

## 5. Testing Strategy & Code Quality Guardrails

### 5.1 Backend Testing

#### Unit Tests — Domain Layer

Each domain service is tested as pure functions with no mocks:

| Test File | What It Tests |
|---|---|
| `test_statistics_calculator.py` | Avg/min/max/gain computations for all metrics; null/missing data handling |
| `test_activity_normalizer.py` | Mapping of raw FIT dict → Activity entity; missing field handling |
| `test_insights_generator.py` | Each threshold rule produces the correct insight string |

#### Integration Tests — Use Case Layer

```python
# test_process_activity_upload.py
# Uses a real minimal .fit fixture, not mocked parser
def test_full_analysis_with_fixture():
    use_case = ProcessActivityUploadUseCase(adapter=FitParserAdapter())
    result = use_case.execute(Path("tests/fixtures/sample_activity.fit").read_bytes())
    assert result.summary.total_distance_km > 0
    assert result.statistics.heart_rate is not None
```

#### Integration Tests — API Layer

```python
# test_activity_route.py
# Uses httpx.AsyncClient with the live FastAPI app
async def test_analyze_returns_200_for_valid_fit():
    async with AsyncClient(app=app, base_url="http://test") as client:
        with open("tests/fixtures/sample_activity.fit", "rb") as f:
            response = await client.post("/api/v1/activities/analyze",
                                         files={"file": f})
    assert response.status_code == 200
    assert "summary" in response.json()

async def test_analyze_returns_422_for_non_fit_file():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/api/v1/activities/analyze",
                                     files={"file": ("test.gpx", b"<gpx/>")})
    assert response.status_code == 422
    assert response.json()["error"]["code"] == "INVALID_FILE_TYPE"
```

#### Coverage Requirement

Minimum **90%** line coverage enforced via `pytest-cov`:

```ini
# pyproject.toml
[tool.pytest.ini_options]
addopts = "--cov=src --cov-fail-under=90"
```

---

### 5.2 Frontend Testing

#### Component Rendering Tests (Vitest + jsdom)

```js
// tests/features/upload/uploadView.test.js
import { renderDropZone } from '@/features/upload/uploadView';

test('drop zone renders with correct aria attributes', () => {
  const el = renderDropZone();
  expect(el.getAttribute('aria-label')).toBe('File upload area');
  expect(el.getAttribute('role')).toBe('region');
});
```

#### State Store Tests

```js
// tests/core/state/appState.test.js
test('subscriber is called with new and previous state on set()', () => {
  const listener = vi.fn();
  appState.subscribe(listener);
  appState.set({ status: 'loading' });
  expect(listener).toHaveBeenCalledWith(
    expect.objectContaining({ status: 'loading' }),
    expect.objectContaining({ status: 'idle' })
  );
});
```

#### API Client Tests

```js
// tests/core/api/activityApi.test.js
vi.mock('@/core/api/activityApi');

test('analyze() calls correct endpoint with form data', async () => {
  activityApi.analyze.mockResolvedValueOnce({ summary: {} });
  const result = await activityApi.analyze(mockFile);
  expect(fetch).toHaveBeenCalledWith('/api/v1/activities/analyze',
    expect.objectContaining({ method: 'POST' }));
});
```

#### Coverage Thresholds

```js
// vite.config.js (coverage section)
test: {
  coverage: {
    thresholds: {
      lines: 85,
      functions: 85,
      statements: 85,
      branches: 75,
    },
  },
},
```

---

### 5.3 Code Quality & Enforcement

#### Backend

| Tool | Configuration | Enforcement |
|---|---|---|
| **Ruff** | `pyproject.toml` — `[tool.ruff]` | `ruff check .` must pass with zero warnings |
| **Ruff Format** | Same config | `ruff format --check .` must pass |
| **MyPy** | `strict = true` in `pyproject.toml` | `mypy src` must produce zero errors |
| **Pytest-cov** | Fail under 90% | CI fails if coverage drops |

MyPy strict requirements:
- All function signatures annotated (parameters and return types).
- No `Any` unless explicitly narrowed.
- No untyped `dict` or `list` without generics.

#### Frontend

| Tool | Configuration | Enforcement |
|---|---|---|
| **ESLint** | `.eslintrc.js` — `eslint:recommended` + JSDoc rules | `npm run lint` must pass |
| **Prettier** | `.prettierrc` | `npm run format --check` must pass |
| **Vitest coverage** | `vite.config.js` | `npm run test:coverage` fails below thresholds |

JSDoc requirements:
- All exported functions must have `@param` and `@returns` annotations.
- No implicit `any` equivalents in JSDoc (`*` types must be justified with a comment).

---

## 6. User Interface Design

### Wireframe / Layout Reference

The following approved designs serve as the visual reference for the SPA. Implementation must match the structural layout, section hierarchy, and widget groupings shown.

**Dark Theme:**

![Application Reference Design – Dark](docs/assets/ui-reference-design-dark.png)

**Light Theme:**

![Application Reference Design – Light](docs/assets/ui-reference-design-light.png)

### Layout Sections

| Section | Visibility | Content |
|---|---|---|
| **Upload Zone** | Always visible | Drag-and-drop area, file picker button, format hint |
| **Status Bar** | Conditional | Loading spinner (during upload), error banner (on failure) |
| **Activity Summary** | After success | Duration, distance, sport type widgets |
| **Performance Widgets** | After success | Stat cards for all metrics (avg/max/min) |
| **Telemetry Chart** | After success | Unified multi-metric line chart with X-axis toggle |
| **Route Map** | After success (GPS present) | Leaflet map with polyline overlay |
| **Insights** | After success | Bulleted list of automated insight strings |

### Theming

- TailwindCSS with `dark:` variants for all color tokens.
- Light/dark mode follows `prefers-color-scheme` media query by default.
- No manual toggle required (but may be added as an enhancement without spec change).

### Accessibility Requirements

- Upload drop zone must have `role="region"` and a descriptive `aria-label`.
- Chart canvas must have `aria-label` describing the chart type.
- All interactive controls (toggle, file picker button) must be keyboard-accessible.
- Error banners must use `role="alert"` for screen reader announcement.
- Color is never the sole differentiator for metric identity in the chart (legend labels always accompany color).
