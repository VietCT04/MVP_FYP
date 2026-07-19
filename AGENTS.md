# AGENTS.md

## Purpose

This file provides repository-level instructions for coding agents working on the Autonomous Supply Chain Reroute Agent.

The project is a local MVP that simulates supply-chain disruptions, identifies affected shipments, calculates capacity-aware alternative routes, and generates grounded OpenAI explanations for deterministic backend decisions.

These instructions apply to the entire repository unless a more specific `AGENTS.md` is added in a subdirectory.

## Start Here

Before changing code:

1. Read `README.md`.
2. Inspect the relevant implementation and tests.
3. Confirm the current API and data-model behavior before proposing structural changes.
4. Keep the requested change narrowly scoped.
5. Do not rewrite unrelated code merely to improve style.

## Repository Map

```text
backend/app/
  main.py          FastAPI application and endpoints
  config.py        Environment-backed application settings
  data.py          Synthetic locations, routes, shipments, and scenarios
  schemas.py       Pydantic request, response, and domain models
  services.py      Graph, disruption, routing, scoring, and metrics logic
  repository.py    SQLAlchemy persistence for simulation runs
  llm.py           Grounded OpenAI explanation generation

backend/tests/      Backend service tests
dashboard/          Streamlit and Plotly dashboard
README.md           Setup and demo instructions
requirements.txt    Python dependencies
Dockerfile          Backend container image
```

Update this map when major files or directories are introduced.

## Core Design Boundaries

### Deterministic backend owns decisions

NetworkX and backend service logic are the source of truth for:

- affected shipments
- blocked or congested routes
- candidate paths
- capacity feasibility
- route scoring
- selected routes
- delay, cost, risk, and aggregate metrics

Do not move routing, scoring, validation, or arithmetic into the LLM layer.

### The LLM only explains supplied facts

OpenAI may summarize a decision and propose operational next steps, but it must not invent or select:

- shipment IDs
- locations
- routes
- costs
- durations
- delay savings
- risk scores
- capacity values

Pass structured decision records to the model. Use strict structured output where practical and validate model-returned numeric values against backend-calculated values.

When adding LLM tests, mock the OpenAI call. Tests must not require a real API key or make paid network calls.

### Capacity affects route feasibility

A route is feasible only when every leg has enough remaining capacity for the shipment load. Batch rerouting processes higher-priority shipments first and updates route load after selecting a route.

Changes to capacity handling must include tests covering multiple shipments competing for the same route.

### Disruptions are graph mutations

The application currently creates fresh seed data and a fresh graph for each request. Disruption handling mutates those in-memory objects.

Be careful not to reuse mutated graph or seed objects across requests unless the lifecycle and concurrency behavior are deliberately redesigned.

## Data and Persistence

The current supply-chain network is synthetic and defined in `backend/app/data.py`.

SQLite persistence currently stores simulation runs and results. Locations, routes, shipments, and capacity reservations are not durable application records.

Do not imply that the MVP has production-grade inventory, shipment, or capacity persistence. A change that introduces durable operational state should define:

- ownership of the source of truth
- transaction boundaries
- concurrent update behavior
- migration strategy
- test isolation

Use timezone-aware UTC timestamps for persisted or externally returned times.

## API Guidance

Preserve existing endpoint behavior unless the task explicitly changes the contract.

Current API areas include:

- health and seed-data retrieval
- interactive route planning
- disruption simulation
- batch rerouting
- per-shipment analytics
- recommendation and metric history

For API changes:

- define request and response models in Pydantic
- return explicit status and reason fields for expected no-result cases
- use appropriate HTTP errors for missing resources and external-service failures
- avoid returning secrets or internal exception details
- add or update endpoint tests
- update `README.md` when setup or public usage changes

Prefer explicit imports in new code. Avoid adding new wildcard imports.

For collection defaults in new Pydantic models, prefer `Field(default_factory=list)` or an equivalent factory rather than shared mutable defaults.

## Routing and Scoring Guidance

Candidate-route generation must:

- respect directed edges
- exclude blocked routes
- reject paths exceeding capacity
- handle missing nodes and disconnected paths explicitly
- remain deterministic for identical input state

Route scoring is lower-is-better and currently considers:

- duration
- cost
- risk
- capacity penalty

Priority-specific weighting is intentional. Changes to weights or normalization must include tests demonstrating how route ranking changes for `HIGH`, `MEDIUM`, and `LOW` priorities.

Do not silently alter business meaning while refactoring routing code.

## Dashboard Guidance

The Streamlit dashboard is an MVP client of the FastAPI service.

When changing the dashboard:

- keep backend calls behind the existing API helper or a clearly defined replacement
- handle backend failures visibly
- do not duplicate routing or scoring logic in Streamlit
- keep session-state keys stable or migrate them deliberately
- ensure displayed routes and metrics come from backend responses
- preserve readability of the geographic network view

If backend endpoint shapes change, update the dashboard in the same change unless compatibility is intentionally retained.

## Configuration and Secrets

Configuration is loaded from environment variables and `.env` for local development.

Never commit:

- API keys
- real credentials
- production connection strings
- private customer or shipment data

Keep `.env` ignored. Add safe placeholders to `.env.example` when introducing configuration.

The application should start without an OpenAI key for non-LLM operations. Endpoints that explicitly request an explanation may fail clearly when the key is unavailable.

## Dependencies

Add dependencies only when the standard library or current stack cannot reasonably solve the problem.

When adding or upgrading a dependency:

- update `requirements.txt`
- explain why it is needed in the PR
- consider compatibility with Python 3.12 and the Docker image
- run the relevant tests
- avoid introducing overlapping frameworks for an already-solved concern

Neo4j-related environment placeholders may exist, but Neo4j is not part of the active implementation unless code and dependencies are added explicitly.

## Commands

Create and activate a virtual environment, then install dependencies.

Windows PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
```

macOS or Linux:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Run the backend:

```bash
uvicorn backend.app.main:app --reload
```

Run the dashboard:

```bash
streamlit run dashboard/streamlit_app.py
```

Run tests:

```bash
pytest backend/tests
```

For routing, scoring, schema, repository, or API changes, run the full backend test suite. For documentation-only changes, tests may be skipped, but state that explicitly in the PR.

## Testing Expectations

Add regression tests for bug fixes and focused tests for new behavior.

At minimum, consider coverage for:

- disruption impact detection
- blocked-node and blocked-route avoidance
- congestion behavior
- directed-network reachability
- capacity rejection and reservation
- priority-based scoring
- no-base-route and no-feasible-route results
- database save/update/read behavior
- API validation and error responses
- LLM output validation using mocks

Tests should be deterministic and independent. Do not depend on execution order, external APIs, or a developer's existing SQLite database.

## Change Workflow

For each task:

1. Identify the smallest set of files required.
2. Read nearby tests before changing behavior.
3. Implement the deterministic core before presentation or explanation layers.
4. Add or update tests.
5. Run relevant validation commands.
6. Update documentation when behavior, setup, configuration, or API contracts change.
7. Review the diff for unrelated changes, secrets, generated files, and accidental database files.

## Pull Request Expectations

A pull request should explain:

- what changed
- why it changed
- user or developer impact
- important design decisions
- validation performed
- known limitations or follow-up work

Keep commits focused. Do not combine dependency upgrades, broad formatting, architecture changes, and feature work unless they are inseparable.

## Completion Checklist

Before marking work complete, verify:

- [ ] The change follows the deterministic-backend and grounded-LLM boundary.
- [ ] API and model changes are reflected in callers.
- [ ] Relevant tests pass.
- [ ] No secrets, `.env`, database files, or generated caches are committed.
- [ ] Documentation is updated where necessary.
- [ ] The final diff contains no unrelated changes.
