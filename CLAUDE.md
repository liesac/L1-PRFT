# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

### Sport Activity File Reader

A Python and JavaScript web application (Single Page Application) designed to parse, analyze, and visualize sports activity files in the `.fit` format.

#### Key Features

- **File Processing:** Seamless upload and parsing of standard `.fit` activity files.
- **Unified Telemetry Graph:** Visualizes core performance metrics overlaid on a single, interactive chart:
  - **Cadence** (rpm)
  - **Elevation** (m)
  - **Heart Rate** (bpm)
  - **Power** (Watts)
  - **Speed** (km/h)
  - **Temperature** (°C)
- **Interactive Mapping:** Built-in map visualization rendering the GPS track of the recorded activity.
- **Performance Widgets:** Comprehensive breakdowns displaying **Average, Maximum, and Minimum** values for:
  - Cadence, Heart Rate, Speed, and Temperature.
  - **Elevation** metrics including Total Elevation Gain, Max, and Min.
  - **Summary Data** covering Total Time and Overall Distance.
- **Automated Insights:** An overall activity summary highlighting performance trends and key metric analysis to give users immediate, actionable insights into their training data.

#### Tech Stack

Target stack once implementation begins:

- Python 3.12+
- FastAPI
- Pydantic
- fitdecode
- Pytest
- Ruff
- MyPy
- No frontend framework, Vanilla JavaScript (ES6 Modules)
- vite
- Chart.js
- Leaflet
- TailwindCSS
- Vitest
- ESLint
- Prettier

## Conventions

### General Architecture

The application must follow Clean Architecture principles, emphasizing separation of concerns, modularity, testability, and maintainability. Business logic must remain independent from framework-specific implementations and UI rendering details.

The solution consists of:
- A Python/FastAPI backend responsible for processing uploaded .fit files and exposing activity data through a REST API.
- A JavaScript Single Page Application (SPA) responsible for file uploads, data visualization, and user interactions.
- No database persistence. Activity data is processed on demand and returned directly to the frontend.
- No client-side routing. The entire UI is implemented as a single-page experience.

### Backend Conventions

The backend should be organized using a layered structure that separates business logic from infrastructure and API concerns.

Recommended target structure:
```text
backend/
├── src/
│   ├── domain/
│   │   ├── entities/
│   │   └── models/
│   ├── application/
│   │   └── use_cases/
│   ├── infrastructure/
│   │   └── fit_parser/
│   ├── presentation/
│   │   ├── routes/
│   │   └── schemas/
│   └── main.py
└── tests/
```

Backend guidelines:

- The domain layer must contain core business concepts and remain independent from FastAPI and third-party frameworks.
- The application layer must implement use cases and application workflows.
- The infrastructure layer must handle FIT file parsing and external dependencies.
- The presentation layer must expose API endpoints and request/response schemas.
- API routes should remain thin and delegate all business operations to use cases.
- Pydantic models must be used for validation and serialization.
- Use dependency injection whenever appropriate.
- All code should use Python type hints.
- Keep functions and classes focused on a single responsibility.
- Business rules must never be implemented directly inside API routes.
- Unit tests must be provided for domain logic and use cases.
- Integration tests should validate API endpoints and request workflows.

### Frontend Conventions

The frontend is a Single Page Application built with Vanilla JavaScript using ES Modules. No frontend framework and no routing library should be used.

Recommended target structure:

```text
frontend/
├── src/
│   ├── core/
│   │   ├── api/
│   │   ├── events/
│   │   └── state/
│   ├── features/
│   │   ├── upload/
│   │   ├── summary/
│   │   ├── charts/
│   │   ├── maps/
│   │   └── tables/
│   ├── shared/
│   │   ├── components/
│   │   └── utils/
│   └── main.js
├── public/
└── tests/
```

Frontend guidelines:

- Organize code by feature whenever possible.
- Separate rendering logic from API communication and state management.
- Use ES Modules for all frontend code.
- Avoid global variables.
- Maintain a centralized application state when shared data is required.
- Encapsulate DOM manipulation inside dedicated feature modules.
- Reusable UI elements should be placed in the shared/components directory.
- Utility functions should be placed in the shared/utils directory.
- Features should communicate through events or shared state instead of direct dependencies.
- Keep modules small, focused, and easy to test.
- Follow accessibility and responsive design best practices.

### Data Visualization

The application should present activity information in multiple formats:

- Summary widgets for key metrics.
- Interactive maps for route visualization.
- The chart component will render all available data streams. The X-axis will map either time or distance, controlled by an in-app toggle config. To ensure layout responsiveness, the X-axis labels must dynamically scale and auto-skip based on the visualization container's width, preventing a 1:1 render of every single data value to avoid overlapping. The Y-axis will represent the scale for each metrics dataset; while data sources will utilize distinct value ranges, they must maintain absolute scale normalization between them. Additionally, a hover state (mouseover) will trigger a dynamic tooltip aggregating and displaying all data metrics at that specific data point coordinate.

Recommended libraries:

- Chart.js for charts and statistics visualization.
- Leaflet for route and map visualization.

### Testing

Backend:

- Use Pytest for unit and integration testing.
- Test all use cases and domain logic.
- Validate API behavior through integration tests.
- Mock external dependencies when appropriate.

Frontend:

- Use Vitest for unit testing.
- Use Testing Library for DOM and interaction testing.
- Test feature modules independently.
- Test user interactions, rendering logic, and state updates.

### Code Quality

Backend:

- Use Ruff for linting and formatting.
- Use MyPy for static type checking.
- Follow PEP 8 standards.

Frontend:

- Use ESLint for linting.
- Use Prettier for formatting.
- Public functions fully typed; no `any`
- Follow modern JavaScript best practices.

General guidelines:

- Prioritize readability over clever implementations.
- Prefer composition over unnecessary inheritance.
- Keep files small and focused.
- Document non-obvious business rules and architectural decisions.
- Write maintainable and testable code.
- Maintain high automated test coverage across the project.

## Documentation Reference
- Activity file guidelines follow [Activity File](docs/fitActivityFile.md) as reference.
- Activity file decoding FIT activity files guidelines follow [Decoding FIT Activity Files](docs/decodingFitActivityFiles.md) as reference.
- For additional documentation about FIT files look on your knowledge base or here [Flexible and Interoperable Data Transfer (FIT) SDK](https://developer.garmin.com/fit/)overview/

## Do

- Write the test ** before ** the implementation when implementing a task
- When creating the first project files, establish only the minimum structure needed for the requested task.
- If a requested change depends on a new runtime dependency, stop and ask for approval first.

### Documentation

The README.md must be kept up to date throughout the project lifecycle.

It should include:

- Project overview and purpose.
- High-level architecture description.
- Backend and frontend technology stack.
- Key libraries and dependencies.
- Directory and project structure.
- Local development setup instructions.
- Build and deployment instructions.
- Testing strategy and commands.
- Environment configuration requirements.
- Design decisions and architectural rationale.
- Any notable implementation details or project conventions.

Whenever a new technology, dependency, architectural pattern, or significant implementation detail is introduced, the README.md must be updated accordingly to reflect the current state of the project.

## Don't

- Don't add new dependencies without a note in `plan.md`
- Don't create a framework where a 20-line function works
- Don't access the network at runtime - this is a local tool

## When generating code

- For frontend code, prefer configured absolute imports from `src/` once `tsconfig.json` or equivalent path mapping exists.
- Prefer pure functions;
- When uncertain, ask a clarifying question in the Chat before writing code

## Human Gates

Claude MUST stop and ask for explicit human approval before:

- Adding any new runtime dependency not already in `package.json`