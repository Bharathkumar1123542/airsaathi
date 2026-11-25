# AGENTS.md

Commands, conventions, and boundaries for coding agents working on **AirSaathi**. For product background read [README.md](README.md). For requirements and design read `docs/`. This file is loaded automatically by Kiro and is always in context, so the rules below apply to every task.

## Project overview

AirSaathi is an urban air quality assistant. From a household's location, home setup, and daily routine it produces an hour-by-hour clean-air plan: when to go out, when to open or seal windows, which indoor sources to stop, which commute to swap, and which purchase pays off. Plans are available in English, Hindi, and Marathi, and are readable offline.

Spec v1.0.0 is approved. Implementation proceeds through milestones **M0 to M9**:

| Milestone | Delivers |
|---|---|
| M0 | Scaffold, CI, configuration, health endpoints |
| M1 | Schema, India configuration pack |
| M2 | Provider adapters, CPCB AQI, fusion, air-quality endpoints |
| M3 | Exposure engine and traces |
| M4 | Deterministic plans end to end (first fully usable product, **no LLM in the path**) |
| M5 | Flutter Today and Home tabs, onboarding, offline cache |
| M6 | LLM narration with the numeric validator |
| M7 | Grounded Q&A |
| M8 | Alerts, circles, reports, Impact tab |
| M9 | Privacy endpoints, tracing, load test, accessibility audit, staged rollout |

Do not build ahead of the milestone the current task belongs to.

## Non-negotiable rules

1. **The language model never produces a quantity.** It receives a facts object and returns prose only. Every number comes from `domain/` (package `airsaathi_api.domain`).
2. **`domain/` is pure.** Pure functions, no I/O, no clock or randomness read inside. Pass time and inputs in. It must stay `mypy --strict` clean with 95% line and branch coverage.
3. **Every number carries a trace**: each input with its unit and `model_config` provenance, plus the caveats. A new calculation without a trace is incomplete.
4. **The validator rejects any numeral that is not in the facts object.** Two rejections fall back to the Jinja2 template verdict. The household never sees an error from narration.
5. **No personal identifiers reach the LLM.** The facts object is anonymous.
6. **No magic numbers.** Model parameters, CPCB breakpoints, the action catalogue, and emission factors live in `packages/config-india/` with provenance and are loaded through the `model_config` version (`MODEL_CONFIG_VERSION`).
7. **Fusion is per grid cell, never per household.**
8. **Every query is household-scoped.** Identity comes from a verified Firebase ID token.
9. **Medical questions are refused and redirected.** AirSaathi does not diagnose, track symptoms, or answer medication questions.

## Source of truth

Read the relevant document before changing code. Do not guess formulas, contracts, or copy.

| File | Use it for |
|---|---|
| `docs/scope.md` | Who it is for, what v1 includes, what is cut, success metrics |
| `docs/prd.md` | Behaviour, screens, copy, acceptance criteria |
| `docs/spec.md` | Data model, formulas, API contracts, security (section 10), testing (section 12), milestones, worker jobs (section 2), fixtures (e.g. Lucknow, section 7.9) |
| `docs/checklist.md` | The ordered build tasks |

Workflow for each task:

1. Take the next task from `docs/checklist.md` and read the `prd.md` and `spec.md` sections it references.
2. Implement only what the task and its milestone cover. Do not implement anything `scope.md` lists as cut.
3. Add or update tests in the same change, even if nobody asked.
4. Run the checks in [Testing instructions](#testing-instructions) until they are green.
5. If the docs conflict or are silent on something you need, stop and ask. Do not invent behaviour.

Requirements, design, and tasks already live in `docs/`. When using Kiro's spec workflow, reference those files instead of generating parallel specs under `.kiro/specs/`.

## Repository layout

```
airsaathi/
├── apps/
│   ├── api/        # airsaathi-api: FastAPI; domain/ (every number), llm/ (prose only), providers/, rag/
│   ├── worker/     # airsaathi-worker: arq jobs and schedules
│   └── app/        # airsaathi-app: Flutter (features: today, home, impact, ask, more, onboarding, circles)
├── packages/config-india/   # model parameters with provenance, CPCB breakpoints, action catalogue, emission factors
├── corpus/         # cited RAG sources and manifest
├── infra/          # Fly.io configs, Grafana dashboards, Postgres image
├── docs/           # scope.md, prd.md, spec.md, checklist.md
└── .github/workflows/   # ci.yml, deploy.yml
```

Inside `apps/api`, the dependency direction is: routers, then deps (auth, household scope, rate limit), then `domain/`, `llm/`, `rag/`, `providers/`. `domain/` imports from none of the others. `llm/` never imports arithmetic helpers to produce figures.

## Tech stack

- **App:** Flutter 3.27+ (Dart 3.6+), Riverpod, Drift (SQLite), Dio, fl_chart. Android first, iOS-clean codebase.
- **API:** Python 3.12, FastAPI 0.115+, Pydantic 2.9+, SQLAlchemy 2.0+ (async), Alembic.
- **Worker:** arq 0.26+ on Redis 7.2.
- **Database:** PostgreSQL 16 with `postgis`, `pgvector`, `pg_trgm`.
- **LLM:** Anthropic API. `claude-sonnet-5-5` for narration and Q&A, `claude-haiku-5-5` for intent routing. Read model IDs from `ANTHROPIC_MODEL_PRIMARY` and `ANTHROPIC_MODEL_ROUTER`; never hardcode them.
- **Embeddings:** `BAAI/bge-m3` (1024 dimensions) via `sentence-transformers`.
- **Data:** Open-Meteo Air Quality (CAMS forecast), OpenAQ v3 (station ground truth), Open-Meteo Forecast (weather).
- **Identity and push:** Firebase Auth (phone OTP, anonymous first), Firebase Cloud Messaging.
- **Hosting and observability:** Fly.io, Neon, Upstash (Mumbai); OpenTelemetry to Grafana Cloud, Sentry.

## Dev environment tips

- Run `uv` commands from the package directory (`apps/api` or `apps/worker`) and pass `--env-file ../../.env` for anything that reads configuration.
- Use `uv sync` to install; `uv python install 3.12` if Python 3.12 is missing. Do not use `pip install` directly.
- Start dependencies from the repo root with `docker compose up -d --wait`. The Postgres image is built from `infra/postgres/Dockerfile` so that `postgis`, `vector`, and `pg_trgm` exist. A different local Postgres will fail with `extension "vector" is not available`.
- `.env.example` lists every variable name. The API refuses to start if any required variable is missing. Required: `AIRSAATHI_ENV`, `DATABASE_URL`, `REDIS_URL`, `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL_PRIMARY`, `ANTHROPIC_MODEL_ROUTER`, `OPENAQ_API_KEY`, `OPEN_METEO_AQ_BASE`, `OPEN_METEO_WEATHER_BASE`, `FIREBASE_PROJECT_ID`, `FCM_SERVICE_ACCOUNT_JSON`, `MODEL_CONFIG_VERSION`, `CORPUS_VERSION`, `LLM_DAILY_TOKEN_CAP_PER_HOUSEHOLD`, `ASK_DAILY_LIMIT_PER_HOUSEHOLD`. Optional: `SENTRY_DSN`, `OTEL_EXPORTER_OTLP_ENDPOINT`.
- The four secrets are `ANTHROPIC_API_KEY`, `OPENAQ_API_KEY`, `FIREBASE_PROJECT_ID`, and `FCM_SERVICE_ACCOUNT_JSON` (base64 on one line). Never print, log, or commit them.
- After editing Drift tables or annotated Dart classes, run `dart run build_runner build --delete-conflicting-outputs`. After editing ARB files, run `flutter gen-l10n`. Do not hand-edit generated files.
- From an Android emulator the host is `10.0.2.2`, not `localhost`. For a USB device use `adb reverse tcp:8000 tcp:8000`.

### Run commands

```bash
# Infrastructure (repo root)
docker compose up -d --wait && docker compose ps

# API (apps/api), from M1
uv sync
uv run --env-file ../../.env alembic upgrade head
uv run --env-file ../../.env uvicorn airsaathi_api.main:create_app --factory --reload --port 8000
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8000/v1/health/ready   # expect 200

# Worker (apps/worker), from M2
uv sync
uv run --env-file ../../.env arq airsaathi_worker.schedules.WorkerSettings

# Q&A corpus (apps/api), from M7
uv run --env-file ../../.env python -m airsaathi_api.rag.ingest

# App (apps/app), from M5
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter gen-l10n
flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8000
```

## Testing instructions

CI is defined in `.github/workflows/ci.yml`. Run the same checks locally before every commit:

```bash
# apps/api
uv run ruff check . && uv run ruff format --check .
uv run mypy src
uv run python scripts/check_llm_purity.py
uv run pytest tests/unit --cov=airsaathi_api.domain --cov-branch --cov-fail-under=95
uv run --env-file ../../.env pytest tests/integration tests/golden tests/adversarial

# apps/app
flutter analyze
flutter test
```

- While iterating, run one target: `uv run pytest tests/unit/<path>::<test_name>` or `flutter test test/<path>_test.dart`.
- Integration tests need the Compose containers and `DATABASE_URL` / `REDIS_URL` from `.env`.
- **No test calls a live upstream.** Providers are replayed from recorded fixtures and the Anthropic client is stubbed. `tests/conftest.py` replaces the four secrets with dummy values.
- `tests/golden` is the regression suite pinned to `india-1.0.0`. A changed golden value means a changed calculation: explain it, do not just regenerate the fixture.
- `tests/adversarial` (60+ fixtures, prompt-injection included) must stay at 100% rejection.
- Use Hypothesis for invariants in `domain/`.
- Fix every test, type, and lint error until the whole suite is green. Do not skip, delete, or loosen a test, the coverage gate, or a lint rule to get there.
- After moving files or changing imports, rerun Ruff, mypy, and `scripts/check_llm_purity.py`.

## Code style

**Python**
- Full type hints. `mypy --strict` for `src`. Ruff for lint and format; do not hand-format.
- Pydantic 2 models at every boundary (requests, responses, the facts object, the LLM text-only schema).
- SQLAlchemy 2.0 async style. Every schema change is an Alembic migration. Use bound parameters, never string-built SQL.
- Put units in names or types where a quantity is stored (PM2.5 in µg/m³, exposure in µg/m³·h, inhaled mass in µg).
- AQI is the CPCB AQI; the WHO 24-hour guideline is the health reference line.

**Dart / Flutter**
- Layering: Riverpod providers, then Repository, then Dio and the Drift offline mirror. Widgets never call Dio directly.
- Dio uses ETag and retry. The Drift mirror keeps the full plan readable offline.
- The client displays numbers from the API and shows their trace. It never recomputes exposure.
- All user-facing strings go through l10n for **English, Hindi, and Marathi**. No hardcoded strings.
- Accessibility (WCAG 2.2 AA): AQI always shown as colour plus numeral plus band label, 48 dp touch targets, usable at 200% text scale, and TalkBack reads the Now card as one utterance.
- Debug builds use plain HTTP; release builds pin the API certificate.

## Area rules

**API and domain**
- Routers stay thin. Business logic goes in `domain/`, I/O in `providers/` and the database layer.
- Every API contract follows `docs/spec.md`. Changing a request or response shape is a contract change.

**Worker**
- Jobs live in `apps/worker` and are scheduled in `airsaathi_worker.schedules.WorkerSettings`. Fusion runs every 15 minutes, forecasts refresh hourly, and `generate_daily_plans` runs at 05:30 local time per timezone. Job definitions are in `docs/spec.md` section 2.
- Alerts are capped at 3 per household per 24 hours.

**LLM (`llm/`)**
- Narrator output uses a text-only schema. The validator checks numerals and policy before anything is stored. The plan records its verdict source (model or template).
- Narration runs once per household per day. Budget: `LLM_DAILY_TOKEN_CAP_PER_HOUSEHOLD` (12,000 tokens). Q&A limit: `ASK_DAILY_LIMIT_PER_HOUSEHOLD` (15).
- Haiku routes intent; Sonnet narrates and answers.
- `scripts/check_llm_purity.py` is an AST check that keeps arithmetic out of `llm/`. Do not work around it.

**Providers**
- Adapters wrap Open-Meteo and OpenAQ v3 (`X-API-Key` header). Cache with a 15-minute Redis TTL. Handle 429 and outages by degrading confidence, not by failing.

**RAG**
- Hybrid retrieval over `pgvector` and `pg_trgm`. Answers cite the corpus (WHO, CPCB, US EPA indoor air, NIOSH, GRAP) and use the household's own numbers.
- Every document in `corpus/` needs a manifest entry. Re-ingest after changes at the version set by `CORPUS_VERSION`.

## Security and privacy

- Verify Firebase ID tokens on every request; scope every query to the household.
- Minimise location data: store and log only what `docs/spec.md` section 10 allows.
- DPDP Act 2023: consent, export, and erasure are product requirements (M9).
- No identifiers in LLM prompts, logs, traces, or Sentry events.
- Never commit `.env`, a Firebase service-account JSON, or `apps/app/android/app/google-services.json`.
- Keep the data attribution (Open-Meteo/CAMS, OpenAQ, WHO, CPCB) in the app's About screen and per-station licence on the station sheet.

## Designed degradations (not bugs)

Do not "fix" these by hiding them. They are specified behaviour, and the numbers stay correct in each case.

- `LOW` or `MEDIUM` confidence, or a staleness badge: OpenAQ key invalid or 429.
- Plain template verdict: Anthropic unreachable or invalid key, or validation failed twice.
- The plan pipeline must still produce a complete plan with the LLM disabled (this is the M4 product).

## PR instructions

- Title format: `[M<n>] <Title>`, for example `[M3] Add exposure trace for sealed-window action`.
- One checklist task (or a tightly related group) per PR. Link the `prd.md` and `spec.md` sections it implements.
- Before committing, run the full API and app checks from [Testing instructions](#testing-instructions). CI must be green before merge.
- Describe any golden-value change, config change, migration, or contract change in the PR body.

## Boundaries

**Always**
- Cite the spec section you are implementing; keep traces on every number; add tests; use recorded fixtures.
- Keep English, Hindi, and Marathi strings in sync.

**Ask first**
- Changing values in `packages/config-india/` or bumping `MODEL_CONFIG_VERSION` / `CORPUS_VERSION`.
- Adding or upgrading a dependency, or changing an LLM model ID.
- Changing an API contract, the data model, or anything in an approved `docs/` file.
- Editing `.github/workflows/`, `infra/`, or deployment settings.
- Anything `scope.md` marks as cut.

**Never**
- Let `llm/` compute, convert, round, or invent a number.
- Send personal identifiers to the LLM.
- Call a live upstream from a test.
- Commit secrets or credentials.
- Weaken the coverage gate, the adversarial suite, mypy, or Ruff to get green.
- Add medical advice, diagnosis, or symptom tracking.
- Show a raw error to a household when narration fails.

## Definition of done

- [ ] Behaviour matches `docs/prd.md` and `docs/spec.md`; nothing outside scope was added.
- [ ] Every new number has a trace and a `model_config` source.
- [ ] Tests added or updated; all API and app checks pass; `domain/` coverage is at or above 95%.
- [ ] Strings exist in English, Hindi, and Marathi; accessibility rules hold for UI changes.
- [ ] No secrets, identifiers, or live-upstream calls introduced.
