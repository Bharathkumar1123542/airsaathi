# AirSaathi

[![CI](https://github.com/Ibtidaa-Tech/airsaathi/actions/workflows/ci.yml/badge.svg)](https://github.com/Ibtidaa-Tech/airsaathi/actions/workflows/ci.yml)

**An urban air quality assistant that turns a household's location, home setup, and daily routine into an hour-by-hour clean-air plan.**

Cities such as Delhi-NCR, Lucknow, Patna, Mumbai, and Pune spend weeks each winter under severe smog. A household can read "AQI 412, Severe" on five apps and still not know whether the school run is safe, whether to open the window, or whether an ₹18,000 purifier is worth buying.

AirSaathi answers those questions for one specific household. It returns a dated plan: when to go out and when not to, when to open windows and when to seal them, which indoor sources to stop, which commute to swap, and which purchase pays off. Every recommendation carries a number the household can check, the micrograms of PM2.5 they avoid breathing, computed by a deterministic exposure engine. The language model turns that arithmetic into advice in English, Hindi, or Marathi. It is never allowed to produce a number.

Status: specification v1.0.0 is approved for build. Implementation proceeds through milestones M0 to M9 (see [Build status](#build-status)).

## What a household gets

| ID | Capability | Result |
|---|---|---|
| C1 | Air quality now and ahead | Fused PM2.5, PM10, NO₂, O₃, SO₂, and CO with CPCB AQI, a 72-hour hourly forecast, and a 5-day outlook |
| C2 | Household model | Members (age band and sensitivity flags), home setup, cooking fuel, and routine blocks |
| C3 | Daily Clean-Air Plan | Time-windowed actions: outdoor windows, ventilation windows, commute and mask guidance, indoor-source stops |
| C4 | Indoor air toolkit | Ranked, costed interventions: free, low-cost, and purchase (purifier sized to room volume) |
| C5 | Sustainable swaps | Commute, cooking, and waste alternatives with exposure and CO₂e effects side by side |
| C6 | Impact Explainer | PM2.5 exposure avoided (µg/m³·h), inhaled mass avoided (µg), cigarette equivalent, and CO₂e avoided, each with a "how was this computed" trace |
| C7 | Grounded Q&A | Answers from a cited corpus (WHO, CPCB, US EPA indoor air, NIOSH, GRAP) plus the household's own numbers; medical questions are refused |
| C8 | Alerts | Push notifications on band changes, ventilation windows, forecast spikes, and a weekly summary, capped at 3 per household per 24 h |
| C9 | Neighbourhood circles | Invite-only groups with aggregate actions and a drafted civic complaint |
| C10 | Bilingual, low-bandwidth | English, Hindi, Marathi; the plan is cached and readable offline |

### Example: one action and its trace

For the Lucknow fixture in [`docs/spec.md`](docs/spec.md) section 7.9, sealing windows between 06:00 and 12:00 yields this trace, which the app shows under "How was this computed". Values are rounded as in the spec.

| Step | Value |
|---|---|
| Outdoor PM2.5, mean over 06:00–12:00 | 252.2 µg/m³ |
| Baseline indoor mean | 148.0 µg/m³ |
| Indoor mean with windows sealed | 103.7 µg/m³ |
| Difference | 44.3 µg/m³ |
| Hours affected | 6.0 h |
| **Exposure avoided** | **265.6 µg/m³·h** |

The trace also lists every input with its unit and its `model_config` provenance, plus the caveats (single well-mixed zone, sealing raises the cooking term, forecast values beyond the current hour are not measured).

## How it works

The product rests on one invariant: **the language model receives facts and returns prose. It never produces a quantity.**

1. The worker fuses station readings and the CAMS forecast per grid cell, never per household.
2. `domain/` computes exposure, ranks actions, and attaches a trace to every number.
3. The model receives a facts object with no personal identifiers and returns text only.
4. A validator rejects any numeral that is not in the facts object. Two rejections fall back to a Jinja2 template, and the household never sees an error.

### Architecture

```mermaid
flowchart LR
    subgraph CLIENT["Flutter client (Android)"]
        PROV["Riverpod providers"] --> REPO["Repository"]
        REPO --> DIO["Dio: ETag + retry"]
        REPO <--> DRIFT[("Drift SQLite<br/>offline mirror")]
    end

    subgraph API["airsaathi-api (FastAPI)"]
        ROUTERS["routers"] --> DEPS["deps: auth, household scope,<br/>rate limit"]
        DEPS --> DOMAIN["domain/<br/>pure functions: every number"]
        DEPS --> LLM["llm/<br/>prose only, validated"]
        DEPS --> RAG["rag/<br/>hybrid retrieval"]
        DEPS --> PROVIDERS["providers/<br/>upstream adapters"]
    end

    subgraph WORKER["airsaathi-worker (arq)"]
        JOBS["fuse_grid_cells, refresh_forecasts,<br/>generate_daily_plans, send_alerts,<br/>and the other jobs in spec section 2"]
    end

    DIO -->|"HTTPS + Firebase ID token"| ROUTERS
    FCM["Firebase Cloud Messaging"] -.->|"push"| CLIENT
    LLM --> ANTHROPIC["Anthropic API<br/>sonnet-5-5 and haiku-5-5"]
    PROVIDERS --> OPENMETEO["Open-Meteo<br/>air quality + weather"]
    PROVIDERS --> OPENAQ["OpenAQ v3"]
    API --> PG[("PostgreSQL 16<br/>postgis + pgvector")]
    API --> REDIS[("Redis 7.2<br/>cache, limits, queue")]
    WORKER --> PG
    WORKER --> REDIS
    WORKER -->|"send_alerts"| FCM
```

### Plan generation

```mermaid
sequenceDiagram
    autonumber
    participant W as Worker
    participant P as providers and fusion
    participant D as domain
    participant L as llm narrator
    participant V as llm validator
    participant DB as PostgreSQL
    Note over W: generate_daily_plans runs at 05:30 local time per timezone
    W->>P: readings and forecast for the grid cell
    P-->>W: fused series with confidence and sources
    W->>D: household model and series
    D-->>W: ranked actions, numbers, and traces (facts object)
    W->>L: facts object, no identifiers
    L-->>V: prose in a text-only schema
    alt every numeral is in the facts object and policy checks pass
        V-->>W: accepted
    else rejected twice
        V-->>W: fall back to the Jinja2 template verdict
    end
    W->>DB: persist plan, actions, facts, and verdict source
```

## Tech stack

| Layer | Choice |
|---|---|
| App | Flutter 3.27+ (Dart 3.6+), Riverpod, Drift (SQLite), Dio, fl_chart; Android first, iOS-clean codebase |
| API | Python 3.12, FastAPI 0.115+, Pydantic 2.9+, SQLAlchemy 2.0+ (async), Alembic |
| Worker | arq 0.26+ on Redis 7.2 |
| Database | PostgreSQL 16 with `postgis`, `pgvector`, `pg_trgm` |
| LLM | Anthropic API: `claude-sonnet-5-5` (narration, Q&A), `claude-haiku-5-5` (intent routing) |
| Embeddings | `BAAI/bge-m3` (1024 dimensions), self-hosted via `sentence-transformers` |
| Air quality data | Open-Meteo Air Quality (CAMS forecast), OpenAQ v3 (station ground truth), Open-Meteo Forecast (weather) |
| Identity and push | Firebase Auth (phone OTP, anonymous first), Firebase Cloud Messaging |
| Hosting | Fly.io (API and worker), Neon (Postgres), Upstash (Redis), all in Mumbai |
| Observability | OpenTelemetry to Grafana Cloud, Sentry |

## Quick start

These steps describe the finished v1. Each step names the milestone from which it works. Steps 1 to 3 and 8 need no Anthropic, OpenAQ, or Firebase account; steps 4 to 7 need the four secrets in `.env`.

### Prerequisites

| Tool | Version | Check |
|---|---|---|
| Git | 2.30+ | `git --version` |
| Docker Engine with Compose v2 | 24+ | `docker compose version` |
| uv | 0.5+ | `uv --version` |
| Python | 3.12 (uv installs it on demand) | `uv python install 3.12` |
| Flutter | 3.27+ (Dart 3.6+) | `flutter --version` |
| Android emulator or device | Android 11 (API 30) or later | `adb devices` |

Accounts and keys, needed from step 4 onward: an Anthropic API key, an OpenAQ API key, a Firebase project with Phone and Anonymous sign-in enabled, and a Firebase service-account JSON file with Cloud Messaging access.

### 1. Clone

```bash
git clone https://github.com/Ibtidaa-Tech/airsaathi.git
cd airsaathi
```

### 2. Configure the environment (from M0)

`.env.example` lists every variable name with no values. This block writes a local-development `.env` with every non-secret value filled in and the four secrets left empty:

```bash
cat > .env <<'EOF'
AIRSAATHI_ENV=local
DATABASE_URL=postgresql+asyncpg://airsaathi:airsaathi@localhost:5432/airsaathi
REDIS_URL=redis://localhost:6379/0
ANTHROPIC_MODEL_PRIMARY=claude-sonnet-5-5
ANTHROPIC_MODEL_ROUTER=claude-haiku-5-5
OPEN_METEO_AQ_BASE=https://air-quality-api.open-meteo.com/v1/air-quality
OPEN_METEO_WEATHER_BASE=https://api.open-meteo.com/v1/forecast
MODEL_CONFIG_VERSION=india-1.0.0
CORPUS_VERSION=2026-11-01
LLM_DAILY_TOKEN_CAP_PER_HOUSEHOLD=12000
ASK_DAILY_LIMIT_PER_HOUSEHOLD=15
ANTHROPIC_API_KEY=
OPENAQ_API_KEY=
FIREBASE_PROJECT_ID=
FCM_SERVICE_ACCOUNT_JSON=
EOF
```

Open `.env` and fill the four empty values. Produce the value for `FCM_SERVICE_ACCOUNT_JSON` (the service-account file, base64-encoded on one line) with:

```bash
base64 < service-account.json | tr -d '\n'
```

The API refuses to start when any variable in the table below is missing. `SENTRY_DSN` and `OTEL_EXPORTER_OTLP_ENDPOINT` are optional; leaving them out disables Sentry and tracing.

| Variable | Purpose |
|---|---|
| `AIRSAATHI_ENV` | `local`, `staging`, or `production` |
| `DATABASE_URL` | Postgres connection string (`asyncpg` driver) |
| `REDIS_URL` | Redis connection string |
| `ANTHROPIC_API_KEY` | LLM access |
| `ANTHROPIC_MODEL_PRIMARY` | Narration and Q&A model |
| `ANTHROPIC_MODEL_ROUTER` | Intent-classification model |
| `OPENAQ_API_KEY` | Sent as the `X-API-Key` header |
| `OPEN_METEO_AQ_BASE`, `OPEN_METEO_WEATHER_BASE` | Keyless upstream base URLs |
| `FIREBASE_PROJECT_ID` | Firebase ID-token verification |
| `FCM_SERVICE_ACCOUNT_JSON` | Push notifications, base64-encoded |
| `MODEL_CONFIG_VERSION` | Active `model_config` row |
| `CORPUS_VERSION` | Active corpus snapshot |
| `LLM_DAILY_TOKEN_CAP_PER_HOUSEHOLD` | Daily token budget per household |
| `ASK_DAILY_LIMIT_PER_HOUSEHOLD` | Daily Q&A question limit per household |

### 3. Start Postgres and Redis (from M0)

```bash
docker compose up -d --wait
docker compose ps
```

`docker compose ps` lists `postgres` and `redis` with status `healthy`. Postgres is built from `infra/postgres/Dockerfile` so that `postgis`, `vector`, and `pg_trgm` are all available.

### 4. Run the API (from M1)

```bash
cd apps/api
uv sync
uv run --env-file ../../.env alembic upgrade head
uv run --env-file ../../.env uvicorn airsaathi_api.main:create_app --factory --reload --port 8000
```

In a second terminal, confirm readiness (database, Redis, and the active `model_config`):

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8000/v1/health/ready
```

The command prints `200`. The OpenAPI schema is at `http://localhost:8000/docs`.

### 5. Run the worker (from M2)

In a third terminal:

```bash
cd apps/worker
uv sync
uv run --env-file ../../.env arq airsaathi_worker.schedules.WorkerSettings
```

The worker fuses air quality every 15 minutes and refreshes forecasts hourly. From M4 it also generates daily plans at 05:30 local time per timezone.

### 6. Load the Q&A corpus (from M7)

```bash
cd apps/api
uv run --env-file ../../.env python -m airsaathi_api.rag.ingest
```

The first run downloads `BAAI/bge-m3` (about 2.3 GB) and embeds every document in `corpus/` at the version set in `CORPUS_VERSION`.

### 7. Run the app (from M5)

Register an Android app in your Firebase project using the `applicationId` declared in `apps/app/android/app/build.gradle*`, download `google-services.json`, and save it as `apps/app/android/app/google-services.json`. Then start an emulator or connect a device and run:

```bash
cd apps/app
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter gen-l10n
flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8000
```

`10.0.2.2` is the emulator's alias for the host machine. For a USB-connected device, run `adb reverse tcp:8000 tcp:8000` and pass `--dart-define=API_BASE_URL=http://localhost:8000`. Debug builds use plain HTTP; release builds pin the API certificate.

### 8. Run the tests (from M0)

```bash
cd apps/api
uv run ruff check . && uv run ruff format --check .
uv run mypy src
uv run python scripts/check_llm_purity.py
uv run pytest tests/unit --cov=airsaathi_api.domain --cov-branch --cov-fail-under=95
uv run --env-file ../../.env pytest tests/integration tests/golden tests/adversarial

cd ../app
flutter analyze
flutter test
```

No test calls a live upstream. Every provider is replayed from recorded fixtures and the Anthropic client is stubbed. `tests/conftest.py` replaces the four secrets with dummy values, so the empty entries in `.env` do not matter. Integration tests read `DATABASE_URL` and `REDIS_URL` from `.env` and need the containers from step 3.

### Troubleshooting

| Symptom | Cause and fix |
|---|---|
| API exits at start and names a missing variable | Every variable in the step 2 table is required. Check `.env` and run through `uv run --env-file ../../.env`. |
| `extension "vector" is not available` | The database is not the Compose Postgres. Stop any local Postgres and rerun step 3. |
| `address already in use` on port 5432 or 6379 | Stop the local Postgres or Redis service, or change the host port in `docker-compose.yml` and `.env`. |
| The emulator cannot reach the API | Use `http://10.0.2.2:8000`, not `localhost`. A physical device needs `adb reverse` as in step 7. |
| Readings carry `LOW` or `MEDIUM` confidence, or a staleness badge | `OPENAQ_API_KEY` is invalid or OpenAQ returned 429. This is the designed degradation; set the key and wait one 15-minute fusion cycle. |
| Plans show a plain template verdict | `ANTHROPIC_API_KEY` is invalid, the Anthropic API is unreachable, or validation failed twice. This is the designed fallback; the numbers are unaffected. |
| First `rag.ingest` run is slow | It downloads about 2.3 GB of embedding weights once, then reuses the local cache. |

## Repository layout

```
airsaathi/
├── apps/
│   ├── api/        # airsaathi-api: FastAPI, domain/ (every number), llm/ (prose only), providers/, rag/
│   ├── worker/     # airsaathi-worker: arq jobs and schedules
│   └── app/        # airsaathi-app: Flutter (features: today, home, impact, ask, more, onboarding, circles)
├── packages/config-india/   # model parameters with provenance, CPCB breakpoints, action catalogue, emission factors
├── corpus/         # cited RAG sources and manifest
├── infra/          # Fly.io configs, Grafana dashboards, Postgres image
├── docs/           # scope.md, prd.md, spec.md, checklist.md
└── .github/workflows/   # ci.yml, deploy.yml
```

## Documentation

| File | Read it for |
|---|---|
| [`docs/scope.md`](docs/scope.md) | Who it is for, what v1 includes, what is cut, success metrics |
| [`docs/prd.md`](docs/prd.md) | Behaviour, screens, copy, and acceptance criteria |
| [`docs/spec.md`](docs/spec.md) | Data model, formulas, API contracts, security, milestones, testing |
| [`docs/checklist.md`](docs/checklist.md) | The ordered build tasks |
| [`AGENTS.md`](AGENTS.md) | Commands, conventions, and boundaries for coding agents |

Claude Code reads `CLAUDE.md` rather than `AGENTS.md`. To point it at the same instructions, run `echo '@AGENTS.md' > CLAUDE.md`.

## Build status

| Milestone | Delivers |
|---|---|
| M0 | Scaffold, CI, configuration, health endpoints |
| M1 | Schema, India configuration pack |
| M2 | Provider adapters, CPCB AQI, fusion, air-quality endpoints |
| M3 | Exposure engine and traces |
| M4 | Deterministic plans end to end (the first fully usable product, with no LLM in the path) |
| M5 | Flutter Today and Home tabs, onboarding, offline cache |
| M6 | LLM narration with the numeric validator |
| M7 | Grounded Q&A |
| M8 | Alerts, circles, reports, Impact tab |
| M9 | Privacy endpoints, tracing, load test, accessibility audit, staged rollout |

## Where to look for evidence

| Criterion | Where it lives |
|---|---|
| Code quality | Pure `domain/` with `mypy --strict`, Ruff, an AST check that keeps arithmetic out of `llm/`, 95% line and branch coverage gate on `domain/` |
| Problem statement alignment | [`docs/scope.md`](docs/scope.md) sections 2 and 4: three failures (numbers without meaning, advice without specificity, action without feedback) mapped to C1 to C10 |
| Security and privacy | [`docs/spec.md`](docs/spec.md) section 10: Firebase-verified identity, household scoping in every query, location minimisation, DPDP Act 2023 consent, export, and erasure, no identifiers sent to the LLM, prompt-injection suite |
| Efficiency | Fusion per grid cell, 15-minute Redis TTL, narration once per household per day, a daily token cap of 12,000 tokens per household, plan generation p95 at or under 4,000 ms |
| Testing | [`docs/spec.md`](docs/spec.md) section 12: golden regression suite pinned to `india-1.0.0`, 60 or more adversarial fixtures at 100% rejection, Hypothesis invariants, recorded-fixture provider tests |
| Accessibility | WCAG 2.2 AA; AQI always shown as colour plus numeral plus band label; 48 dp targets; 200% text scale; TalkBack reads the Now card as one utterance; English, Hindi, Marathi; full plan readable offline |

## Data sources and attribution

- Forecasts: CAMS (Copernicus Atmosphere Monitoring Service) data served by the [Open-Meteo](https://open-meteo.com/) Air Quality and Forecast APIs. Attribution ships in the app's About screen.
- Station measurements: [OpenAQ](https://openaq.org/) v3, including CPCB feeds. The per-provider licence appears on each station's detail sheet.
- Standards: WHO Air Quality Guidelines (2021) and the CPCB National Air Quality Index. The app shows the CPCB AQI as primary and the WHO 24-hour guideline as the health reference line.

AirSaathi is not a medical product. It does not diagnose, track symptoms, or answer medication questions, and its Q&A refuses and redirects medical queries.
