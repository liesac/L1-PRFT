# Implementation Plan: Sport Activity File Reader

> **TDD Contract:** For every implementation task, the test file must be committed (or at minimum written and failing) **before** any production code is written. No exceptions.

---

## Phase 1: Project Setup & Workspace Initialization

### 1.1 Backend Python Environment

- [ ] Create `backend/` directory with the following layout stubs:
  ```
  backend/
  ├── src/
  │   └── __init__.py
  ├── tests/
  │   └── __init__.py
  ├── pyproject.toml
  └── static/           ← empty; populated by frontend build
  ```
- [ ] Write `backend/pyproject.toml` with all dependency declarations:
  - **Runtime:** `fastapi`, `uvicorn[standard]`, `pydantic>=2`, `fitdecode`, `python-multipart`
  - **Dev:** `pytest`, `pytest-asyncio`, `pytest-cov`, `httpx`, `ruff`, `mypy`
  - Coverage config: `--cov=src --cov-fail-under=90`
  - MyPy config: `strict = true`
- [ ] Run `python -m pip install -e .[dev]` and verify the environment resolves cleanly.
- [ ] Add `backend/src/presentation/main.py` with a minimal FastAPI app (no routes yet) and a `StaticFiles` mount for `backend/static` at path `/`:
  ```python
  app = FastAPI()
  app.mount("/", StaticFiles(directory="static", html=True), name="static")
  ```
- [ ] Verify `uvicorn src.presentation.main:app --reload` starts without errors.

---

### 1.2 Frontend Vite Workspace

- [ ] Create `frontend/` directory with the following layout stubs:
  ```
  frontend/
  ├── src/
  │   └── main.js
  ├── public/
  │   └── index.html
  ├── tests/
  ├── package.json
  ├── vite.config.js
  ├── tailwind.config.js
  ├── .eslintrc.js
  └── .prettierrc
  ```
- [ ] Write `frontend/package.json` with all dependency declarations:
  - **Runtime:** `chart.js`, `leaflet`
  - **Dev:** `vite`, `tailwindcss`, `postcss`, `autoprefixer`, `vitest`, `@vitest/coverage-v8`, `jsdom`, `@testing-library/dom`, `eslint`, `prettier`, `eslint-config-prettier`
  - Scripts: `dev`, `build`, `preview`, `test`, `test:watch`, `test:coverage`, `lint`, `format`
- [ ] Write `frontend/vite.config.js` with:
  - `build.outDir: '../backend/static'` and `build.emptyOutDir: true`
  - `server.proxy: { '/api': { target: 'http://127.0.0.1:8000', changeOrigin: false } }`
  - `test.environment: 'jsdom'` and coverage thresholds (85% lines/functions/statements, 75% branches)
- [ ] Write `frontend/tailwind.config.js` with `content` paths covering `src/**/*.js` and `public/index.html`.
- [ ] Write `frontend/public/index.html` as the SPA shell: single `<div id="app">`, TailwindCSS CDN for dev, `<script type="module" src="/src/main.js">`.
- [ ] Write `frontend/src/main.js` as a minimal placeholder that logs `"App initialized"`.
- [ ] Run `npm install` and verify `npm run dev` starts without errors.

---

### 1.3 Multi-Project Linkage Verification

- [ ] With both servers running (backend on `:8000`, frontend Vite on `:5173`):
  - [ ] Confirm `http://localhost:5173` serves the SPA shell via Vite.
  - [ ] Confirm `http://localhost:5173/api/v1/` proxies correctly to FastAPI (expect 404 with FastAPI's JSON, not a Vite error and not CORS error in dev enviroment).
- [ ] Run `npm run build` from `frontend/` and confirm `backend/static/index.html` is produced.
- [ ] With only the backend running after the build, confirm `http://127.0.0.1:8000` serves the compiled `index.html` from FastAPI's `StaticFiles`.
- [ ] Confirm there are **zero CORS errors** in the browser console for the production-build path.

---

## Phase 2: Backend Domain and Infrastructure (FIT Parsing Logic)

### Task 2.1: Core Domain Entities and Value Objects

> **TDD first:** Write all tests in `tests/domain/` before touching `src/domain/`.

- [ ] **Write tests first** — `tests/domain/test_domain_entities.py`:
  - [ ] Test that `Activity` entity can be constructed with all fields populated.
  - [ ] Test that `Activity` entity can be constructed with optional fields as `None`.
  - [ ] Test that `DataPoint` value object rejects negative timestamp values.
  - [ ] Test that `GpsCoordinate` value object rejects latitude outside `[-90, 90]`.
  - [ ] Test that `MetricStats` value object rejects `minimum > maximum`.

- [ ] **Implement** `src/domain/entities/activity.py`:
  ```python
  @dataclass(frozen=True)
  class Activity:
      summary: ActivitySummary
      records: list[DataPoint]
      gps_track: list[GpsCoordinate] | None
  ```

- [ ] **Implement** `src/domain/models/telemetry.py`:
  - `DataPoint` — timestamp (int seconds), distance (float km), heart_rate, cadence, speed, power, temperature, elevation (all `float | None`)
  - `GpsCoordinate` — latitude (float), longitude (float); validated to [-90,90] / [-180,180]
  - `MetricStats` — average, maximum, minimum (all `float | None`)
  - `ElevationStats` — total_gain_m, maximum_m, minimum_m (all `float | None`)
  - `ActivitySummary` — total_duration_seconds (int), total_distance_km (float), sport (`str | None`)

- [ ] Run `pytest tests/domain/` — all tests must pass.
- [ ] Run `mypy src/domain/` — zero errors.

---

### Task 2.2: Statistics Calculator Domain Service

> **TDD first:** Tests before implementation.

- [ ] **Write tests first** — `tests/domain/test_statistics_calculator.py`:
  - [ ] Test average heart rate with a known list of `DataPoint` records.
  - [ ] Test maximum and minimum speed across records.
  - [ ] Test total elevation gain (cumulative positive deltas only).
  - [ ] Test all stats return `None` when all values for that field are `None`.
  - [ ] Test with a single-record list (edge case).
  - [ ] Test with empty record list returns `None` stats (no crash).

- [ ] **Implement** `src/domain/services/statistics_calculator.py`:
  - Pure function: `calculate(records: list[DataPoint]) -> ActivityStatistics`
  - `ActivityStatistics` holds `MetricStats | None` for each metric and `ElevationStats | None`.
  - Elevation gain = sum of `max(0, elevation[i] - elevation[i-1])` for all consecutive pairs.
  - No mutation of input data; no side effects.

- [ ] Run `pytest tests/domain/test_statistics_calculator.py` — all tests must pass.

---

### Task 2.3: Activity Normalizer Domain Service

> **TDD first:** Tests before implementation.

- [ ] **Write tests first** — `tests/domain/test_activity_normalizer.py`:
  - [ ] Test that a raw dict with all fields maps to a fully populated `Activity` entity.
  - [ ] Test that a raw dict missing `heart_rate` produces `DataPoint` records with `heart_rate=None`.
  - [ ] Test that a raw dict with no GPS records produces `Activity.gps_track = None`.
  - [ ] Test that the normalizer handles out-of-order timestamps by sorting records by timestamp.
  - [ ] Test that duplicate timestamps are deduplicated (keep first occurrence).

- [ ] **Implement** `src/domain/services/activity_normalizer.py`:
  - Pure function: `normalize(raw: RawActivityData) -> Activity`
  - `RawActivityData` is a typed dataclass (intermediate DTO from the parser adapter).
  - Handles all missing-field cases gracefully via `None`.
  - Sorts and deduplicates records by timestamp.

- [ ] Run `pytest tests/domain/test_activity_normalizer.py` — all tests must pass.

---

### Task 2.4: Insights Generator Domain Service

> **TDD first:** Tests before implementation.

- [ ] **Write tests first** — `tests/domain/test_insights_generator.py`:
  - [ ] Test high heart rate threshold (avg > 160 bpm) produces the expected insight string.
  - [ ] Test high power threshold (avg > 250 W) produces the expected insight string.
  - [ ] Test significant elevation gain threshold (gain > 500 m) produces the expected insight string.
  - [ ] Test low cadence threshold (avg < 75 rpm) produces the expected insight string.
  - [ ] Test high speed threshold (avg > 35 km/h) produces the expected insight string.
  - [ ] Test high heart rate variance (max − avg > 40 bpm) produces the expected insight string.
  - [ ] Test that when no thresholds are crossed, an empty list is returned.
  - [ ] Test that `None` statistics produce no insights (no crash).

- [ ] **Implement** `src/domain/services/insights_generator.py`:
  - Pure function: `generate(stats: ActivityStatistics) -> list[str]`
  - Each threshold check is an isolated `if` block returning one string entry.
  - No I/O, no mutation, no framework dependencies.

- [ ] Run `pytest tests/domain/test_insights_generator.py` — all tests must pass.

---

### Task 2.5: FIT Parser Infrastructure Adapter

> **TDD first:** Integration tests using a fixture `.fit` file before implementation.

- [ ] Add `tests/fixtures/sample_activity.fit` — a minimal valid `.fit` file containing at least:
  - One `session` message (sport, total distance, total elapsed time).
  - At least 10 `record` messages with timestamp, heart_rate, cadence, speed, power, temperature, altitude, position_lat, position_long.

- [ ] **Write tests first** — `tests/infrastructure/test_fit_parser_adapter.py`:
  - [ ] Test that `FitParserAdapter.parse()` returns a `RawActivityData` with non-empty records.
  - [ ] Test that parsed record count matches the number of `record` messages in the fixture.
  - [ ] Test that GPS coordinates are extracted into the `gps_track` field.
  - [ ] Test that `parse()` raises `UnparseableFileError` when given random bytes (not a valid FIT file).
  - [ ] Test that `parse()` raises `UnparseableFileError` when given an empty byte string.

- [ ] **Define** `src/infrastructure/fit_parser/fit_parser_port.py`:
  ```python
  from abc import ABC, abstractmethod
  class FitParserPort(ABC):
      @abstractmethod
      def parse(self, file_bytes: bytes) -> RawActivityData: ...
  ```

- [ ] **Define** `src/domain/exceptions.py`:
  - `UnparseableFileError(Exception)` — wraps `fitdecode.FitParseError`

- [ ] **Implement** `src/infrastructure/fit_parser/fit_parser_adapter.py`:
  - `FitParserAdapter(FitParserPort)` — wraps `fitdecode.FitReader` over `io.BytesIO(file_bytes)`.
  - Collects all `record` messages into a list of field dicts.
  - Extracts the `session` message for summary fields.
  - Converts `position_lat` / `position_long` from semicircles to decimal degrees.
  - Catches `fitdecode.FitParseError` and raises `UnparseableFileError`.

- [ ] Run `pytest tests/infrastructure/` — all tests must pass.

---

## Phase 3: Backend Presentation Layer (FastAPI API Design)

### Task 3.1: Pydantic Input/Output Schemas

- [ ] **Implement** `src/presentation/schemas/response_schemas.py` with all models from `spec.md §3.3`:
  - `MetricStats`, `ElevationStats`, `ActivityStatistics`
  - `ActivitySummary`, `TelemetryData`, `ActivityAnalysisResponse`
  - `ErrorDetail`, `ErrorResponse`
- [ ] **Implement** `src/presentation/schemas/request_schemas.py`:
  - File validation constants: `MAX_FILE_SIZE_BYTES = 50 * 1024 * 1024`, `ALLOWED_EXTENSION = ".fit"`
- [ ] Run `mypy src/presentation/schemas/` — zero errors.

---

### Task 3.2: Process Activity Upload Use Case

> **TDD first:** Use case tests before implementation.

- [ ] **Write tests first** — `tests/application/test_process_activity_upload.py`:
  - [ ] Test successful end-to-end: fixture `.fit` bytes → `ActivityAnalysisResponse` with non-null summary.
  - [ ] Test that `process_activity_upload` raises `UnparseableFileError` when the parser raises it.
  - [ ] Test that the use case returns an `insights` list (may be empty, not null).
  - [ ] Test that `telemetry.time_series` length equals `telemetry.heart_rate` length.
  - [ ] Test with a fixture missing GPS data — `gps_track` must be `None` or `[]`.

- [ ] **Implement** `src/application/use_cases/process_activity_upload.py`:
  ```python
  class ProcessActivityUpload:
      def __init__(self, parser: FitParserPort) -> None: ...
      def execute(self, file_bytes: bytes) -> ActivityAnalysisResponse: ...
  ```
  - Calls `parser.parse(file_bytes)` → `RawActivityData`
  - Calls `ActivityNormalizer.normalize(raw)` → `Activity`
  - Calls `StatisticsCalculator.calculate(activity.records)` → `ActivityStatistics`
  - Calls `InsightsGenerator.generate(stats)` → `list[str]`
  - Assembles and returns `ActivityAnalysisResponse`.

- [ ] Run `pytest tests/application/` — all tests must pass.

---

### Task 3.3: Activity API Route

> **TDD first:** API integration tests before the route implementation.

- [ ] **Write tests first** — `tests/presentation/test_activity_route.py`:
  - [ ] `POST /api/v1/activities/analyze` with valid fixture `.fit` returns `200` and body contains `summary`, `statistics`, `telemetry`, `gps_track`, `insights`.
  - [ ] Returns `422` with `error.code = "INVALID_FILE_TYPE"` when uploading a `.gpx` file.
  - [ ] Returns `422` with `error.code = "EMPTY_FILE"` when uploading a zero-byte file.
  - [ ] Returns `413` with `error.code = "FILE_TOO_LARGE"` when file exceeds 50 MB.
  - [ ] Returns `422` with `error.code = "UNPARSEABLE_CONTENT"` when uploading corrupted bytes with `.fit` extension.
  - [ ] Response body for all errors includes a non-empty `requestId` string.
  - [ ] Response body matches the `ActivityAnalysisResponse` Pydantic schema (no extra or missing keys).

- [ ] **Implement** `src/presentation/routes/activity_route.py`:
  - `router = APIRouter(prefix="/api/v1/activities")`
  - `POST /analyze` endpoint accepting `file: UploadFile`.
  - Validates file extension and size before passing bytes to the use case.
  - Handles `UnparseableFileError` → `422 UNPARSEABLE_CONTENT`.
  - Generates a `requestId` (UUID4) per request; logs it alongside any exception.
  - Returns `ActivityAnalysisResponse` on success.

- [ ] **Register** the router in `src/presentation/main.py`.
- [ ] Run `pytest tests/presentation/` — all tests must pass.
- [ ] Run `pytest --cov=src --cov-report=term-missing` — confirm ≥ 90% coverage.
- [ ] Run `mypy src` — zero errors.
- [ ] Run `ruff check . && ruff format --check .` — zero warnings.

---

## Phase 4: Frontend Core & Decoupled State Management

### Task 4.1: Event Bus

> **TDD first:** Unit tests before implementation.

- [ ] **Write tests first** — `tests/core/events/eventBus.test.js`:
  - [ ] Test that a registered handler is called when its event is emitted.
  - [ ] Test that a handler receives the emitted payload.
  - [ ] Test that `off()` removes a specific handler without affecting others on the same event.
  - [ ] Test that emitting an event with no registered handlers does not throw.
  - [ ] Test that multiple handlers on the same event are all called.
  - [ ] Test that a handler registered for event `A` is not called when event `B` is emitted.

- [ ] **Implement** `src/core/events/eventBus.js`:
  ```js
  const listeners = new Map();
  export const eventBus = {
    on(event, handler) { ... },
    off(event, handler) { ... },
    emit(event, payload) { ... },
  };
  ```
  - No global `window` assignment.
  - Pure in-memory pub/sub with no DOM event delegation.

- [ ] Run `npm run test -- tests/core/events/` — all tests must pass.

---

### Task 4.2: Centralized State Store

> **TDD first:** Unit tests before implementation.

- [ ] **Write tests first** — `tests/core/state/appState.test.js`:
  - [ ] Test `get()` returns an object with `status: 'idle'` in the initial state.
  - [ ] Test `set({ status: 'loading' })` updates the status field.
  - [ ] Test `set()` is non-destructive: unrelated fields are preserved.
  - [ ] Test that a subscriber is called with `(newState, prevState)` after `set()`.
  - [ ] Test that `unsubscribe()` removes the listener — it is not called on the next `set()`.
  - [ ] Test that `get()` returns a frozen (immutable) object — direct mutation throws.
  - [ ] Test that multiple subscribers are all notified on `set()`.

- [ ] **Implement** `src/core/state/appState.js`:
  ```js
  const initialState = {
    status: 'idle',   // 'idle' | 'loading' | 'success' | 'error'
    error: null,
    activity: null,
    xAxisMode: 'time',
  };
  export const appState = {
    get() { return Object.freeze({ ...state }); },
    set(partial) { ... /* merge, freeze, notify */ },
    subscribe(fn) { ... },
    unsubscribe(fn) { ... },
  };
  ```

- [ ] Run `npm run test -- tests/core/state/` — all tests must pass.

---

### Task 4.3: API Client and Response Mapper

> **TDD first:** Unit tests before implementation.

- [ ] **Write tests first** — `tests/core/api/activityApi.test.js`:
  - [ ] Test that `analyze(file)` calls `fetch` with `POST`, `multipart/form-data`, and the file attached as the `file` field.
  - [ ] Test that a `200` response resolves with the parsed JSON body.
  - [ ] Test that a `422` response rejects with an error object containing `code` and `requestId`.
  - [ ] Test that a network failure (fetch throws) rejects with an `INTERNAL_SERVER_ERROR`-shaped error.

- [ ] **Write tests first** — `tests/core/api/activityResponseMappers.test.js`:
  - [ ] Test that a complete API JSON response maps to the expected normalized state shape.
  - [ ] Test that `null` statistics fields in the API response map to `null` in the UI state.
  - [ ] Test that `gps_track: null` in the API response maps to `gpsTrack: null` in the UI state.
  - [ ] Test that insight strings are passed through verbatim.

- [ ] **Implement** `src/core/api/activityApi.js`:
  - `analyze(file: File): Promise<ActivityAnalysisResponse>` — wraps `fetch('/api/v1/activities/analyze', ...)`.
  - Constructs `FormData` with `file` field.
  - On non-2xx response: reads and throws the `error` envelope from the JSON body.

- [ ] **Implement** `src/core/api/activityResponseMappers.js`:
  - Pure function: `mapActivityResponse(json) → normalizedActivityState`
  - Converts `snake_case` API keys to `camelCase` UI keys.
  - Passes through all nullable fields as-is.

- [ ] Run `npm run test -- tests/core/api/` — all tests must pass.

---

## Phase 5: Frontend Feature Integration (UI Components)

### Task 5.1: File Upload Feature

> **TDD first:** Interaction tests before implementation.

- [ ] **Write tests first** — `tests/features/upload/uploadView.test.js`:
  - [ ] Test that `renderDropZone()` produces an element with `role="region"` and an `aria-label`.
  - [ ] Test that the drop zone element contains a file `<input>` with `accept=".fit"`.
  - [ ] Test that `setLoadingState(true)` disables the file input and shows a loading indicator.
  - [ ] Test that `setLoadingState(false)` re-enables the file input.

- [ ] **Write tests first** — `tests/features/upload/uploadController.test.js`:
  - [ ] Test that selecting a non-`.fit` file triggers an error state update via `appState.set()`.
  - [ ] Test that selecting a file over 50 MB triggers an error state update.
  - [ ] Test that selecting a valid `.fit` file calls `activityApi.analyze()`.
  - [ ] Test that a successful API response calls `appState.set({ status: 'success', activity: ... })`.
  - [ ] Test that a failed API response calls `appState.set({ status: 'error', error: ... })`.

- [ ] **Implement** `src/features/upload/uploadView.js`:
  - Renders drag-and-drop zone with file input, browse button, and format hint text.
  - Exports `setLoadingState(isLoading: boolean): void`.

- [ ] **Implement** `src/features/upload/uploadController.js`:
  - Imports `uploadView`, `activityApi`, `appState`, `validators`.
  - Wires file input `change` event and drag-and-drop events.
  - Calls `validators.validateFile(file)` before submission.
  - Dispatches `appState.set({ status: 'loading' })` before the API call.

- [ ] **Implement** `src/shared/utils/validators.js`:
  - Pure function: `validateFile(file: File): { valid: boolean; errorCode?: string }`
  - Checks extension (`.fit`), size (≤ 50 MB), and non-empty content.

- [ ] Run `npm run test -- tests/features/upload/` — all tests must pass.

---

### Task 5.2: Summary Widgets Feature

> **TDD first:** Rendering tests before implementation.

- [ ] **Write tests first** — `tests/features/summary/summaryView.test.js`:
  - [ ] Test that `renderWidgets(stats)` creates a card for each metric group.
  - [ ] Test that a widget card for heart rate displays average, maximum, and minimum values.
  - [ ] Test that a `null` metric value renders as `N/A` in the widget.
  - [ ] Test that values are formatted to two decimal places.

- [ ] **Implement** `src/shared/components/statWidget.js`:
  - Pure function: `createStatCard({ label, average, maximum, minimum, unit }): HTMLElement`
  - Displays `N/A` for null fields.

- [ ] **Implement** `src/features/summary/summaryView.js`:
  - `renderWidgets(stats: ActivityStatistics, summary: ActivitySummary): void`
  - Creates and mounts stat cards for each metric group including the Activity Summary widget.

- [ ] **Implement** `src/features/summary/summaryController.js`:
  - Subscribes to `appState`; calls `summaryView.renderWidgets()` when `status === 'success'`.
  - Hides the summary section when `status === 'idle'`.

- [ ] Run `npm run test -- tests/features/summary/` — all tests must pass.

---

### Task 5.3: Telemetry Chart Feature

> **TDD first:** Rendering tests before implementation.

- [ ] **Write tests first** — `tests/features/charts/chartConfigFactory.test.js`:
  - [ ] Test that `buildChartConfig({ telemetry, xAxisMode: 'time' })` returns `labels` formatted as `HH:MM:SS` strings.
  - [ ] Test that `buildChartConfig({ telemetry, xAxisMode: 'distance' })` returns `labels` formatted as `0.00 km` strings.
  - [ ] Test that the config contains one dataset per non-null metric in the telemetry object.
  - [ ] Test that each dataset references a unique `yAxisID`.
  - [ ] Test that the config sets `interaction.mode = 'index'` and `interaction.intersect = false`.
  - [ ] Test that the Y-axis for each metric has `min` and `max` derived from actual data values with 10% padding.
  - [ ] Test that metrics with all-null values are excluded from datasets.

- [ ] **Write tests first** — `tests/features/charts/chartController.test.js`:
  - [ ] Test that subscribing to `appState` with `status === 'success'` triggers chart initialization.
  - [ ] Test that changing `xAxisMode` in state calls `chart.update()` with new labels.
  - [ ] Test that a `ResizeObserver` callback updates `maxTicksLimit` based on container width.

- [ ] **Implement** `src/features/charts/chartConfigFactory.js`:
  - Pure function: `buildChartConfig({ telemetry, xAxisMode }): ChartConfiguration`
  - Calls `buildDatasets(telemetry)` and `buildScales(telemetry)` as private pure helpers.
  - Y-axis padding: `padding = (max − min) * 0.1 || 1`

- [ ] **Implement** `src/features/charts/chartView.js`:
  - Manages Chart.js canvas element creation and Chart instance lifecycle.
  - `initChart(canvas, config): Chart`
  - `destroyChart(chart): void`

- [ ] **Implement** `src/features/charts/chartController.js`:
  - Subscribes to `appState`; creates/destroys Chart instances on status transitions.
  - Attaches `ResizeObserver` on the chart container.
  - Updates chart when `xAxisMode` changes in state.

- [ ] **Implement** `src/shared/components/toggle.js`:
  - `createToggle({ options, onChange }): HTMLElement`
  - On selection, calls `appState.set({ xAxisMode: selectedValue })`.

- [ ] Run `npm run test -- tests/features/charts/` — all tests must pass.

---

### Task 5.4: Interactive Map Feature

> **TDD first:** Rendering tests before implementation.

- [ ] **Write tests first** — `tests/features/maps/mapController.test.js`:
  - [ ] Test that `renderMap(gpsTrack)` calls `L.map()` and `L.polyline()` with the provided coordinates.
  - [ ] Test that calling `renderMap()` a second time destroys the previous Leaflet instance before creating a new one.
  - [ ] Test that when `gpsTrack` is `null` or empty, the map section container is hidden.
  - [ ] Test that `mapController.init()` subscribes to `appState` and calls `renderMap` on `status === 'success'`.

- [ ] **Implement** `src/features/maps/mapView.js`:
  - Manages the Leaflet map container DOM element.
  - `getMapContainer(): HTMLElement`

- [ ] **Implement** `src/features/maps/mapController.js`:
  - Module-scoped `mapInstance` variable for teardown tracking.
  - `renderMap(gpsTrack: [number, number][] | null): void`
  - Uses OpenStreetMap tile layer with attribution.
  - Calls `map.fitBounds(polyline.getBounds())` after rendering the polyline.

- [ ] Run `npm run test -- tests/features/maps/` — all tests must pass.

---

### Task 5.5: Status and Insights Features

> **TDD first:** Tests before implementation.

- [ ] **Write tests first** — `tests/features/status/statusController.test.js`:
  - [ ] Test that `status === 'loading'` renders a visible loading spinner.
  - [ ] Test that `status === 'error'` renders an error banner with the error message and `requestId`.
  - [ ] Test that the error banner has `role="alert"`.
  - [ ] Test that `status === 'idle'` hides both spinner and error banner.

- [ ] **Write tests first** — `tests/features/insights/insightsView.test.js`:
  - [ ] Test that `renderInsights(['insight 1', 'insight 2'])` creates two list items.
  - [ ] Test that `renderInsights([])` renders an empty list or a "no insights" placeholder.

- [ ] **Implement** `src/features/status/statusView.js` and `statusController.js`.
- [ ] **Implement** `src/features/insights/insightsView.js` and `insightsController.js`.

- [ ] **Wire all controllers** in `src/main.js`:
  ```js
  import { uploadController } from './features/upload/uploadController.js';
  import { summaryController } from './features/summary/summaryController.js';
  import { chartController } from './features/charts/chartController.js';
  import { mapController } from './features/maps/mapController.js';
  import { statusController } from './features/status/statusController.js';
  import { insightsController } from './features/insights/insightsController.js';

  uploadController.init();
  summaryController.init();
  chartController.init();
  mapController.init();
  statusController.init();
  insightsController.init();
  ```

- [ ] **Implement** `src/shared/utils/formatters.js`:
  - `formatDuration(seconds: number): string` — outputs `HH:MM:SS`
  - `formatDistance(km: number): string` — outputs `0.00 km`
  - `formatMetric(value: number | null, decimals: number): string` — outputs `N/A` for null

- [ ] Run `npm run test` — full test suite must pass.

---

## Phase 6: Code Quality, Verification, and Production Pipeline

### 6.1 Backend Quality Gate

- [ ] Run `ruff check .` from `backend/` — zero warnings or errors.
- [ ] Run `ruff format --check .` from `backend/` — zero formatting violations.
- [ ] Run `mypy src` from `backend/` (strict mode) — zero errors.
- [ ] Run `pytest --cov=src --cov-report=term-missing` from `backend/` — coverage ≥ **90%**.
- [ ] Fix any failures before proceeding.

### 6.2 Frontend Quality Gate

- [ ] Run `npm run lint` from `frontend/` — zero ESLint violations.
- [ ] Run `npm run format -- --check` from `frontend/` — zero Prettier violations.
- [ ] Run `npm run test:coverage` from `frontend/` — coverage thresholds met:
  - Lines ≥ 85%, Functions ≥ 85%, Statements ≥ 85%, Branches ≥ 75%.
- [ ] Fix any failures before proceeding.

### 6.3 Production Build

- [ ] Run `npm run build` from `frontend/`:
  - [ ] Confirm `backend/static/index.html` is produced.
  - [ ] Confirm `backend/static/assets/` contains hashed JS and CSS bundles.
  - [ ] Confirm no build warnings about unresolved imports.
- [ ] Confirm `backend/static/` is listed in `.gitignore` (compiled assets should not be committed).

### 6.4 End-to-End Single-Server Verification

- [ ] Start only the FastAPI server: `uvicorn src.presentation.main:app --host 127.0.0.1 --port 8000`
- [ ] Open `http://127.0.0.1:8000` in a browser:
  - [ ] SPA shell loads with no CORS errors in the console.
  - [ ] No 404 errors for JS/CSS assets.
  - [ ] Upload a valid `.fit` fixture file:
    - [ ] Loading indicator appears.
    - [ ] Activity summary widgets render with correct values.
    - [ ] Telemetry chart renders with all available metric series.
    - [ ] X-axis toggle switches between time and distance labels.
    - [ ] GPS map renders with the route polyline and correct auto-zoom.
    - [ ] Insights list renders (or is empty without error).
  - [ ] Upload a non-`.fit` file:
    - [ ] Error banner appears with a user-readable message.
    - [ ] `requestId` is visible in the banner.
  - [ ] Resize the browser window:
    - [ ] Chart labels auto-skip without overlapping.
    - [ ] Layout remains usable at narrow widths.
- [ ] Confirm no JavaScript exceptions in the browser console during the full golden-path workflow.

### 6.5 Final Checklist

- [ ] `README.md` reflects the current setup, build, and test commands.
- [ ] `spec.md` matches the implemented behavior (update if any deviations were made during implementation).
- [ ] All test files have no skipped tests (`xit`, `xtest`, `pytest.mark.skip`) unless documented.
- [ ] No `console.log` debug statements left in production frontend code.
- [ ] No `print()` debug statements left in production backend code.
