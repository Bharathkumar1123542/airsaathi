# AirSaathi — Technical Specification

**Project:** AirSaathi (Urban Air Quality Assistant)
**Document:** `spec.md` — how it is built, piece by piece
**Version:** 1.0.0
**Status:** Approved for build
**Last updated:** 2026-11-19
**Depends on:** `scope.md` v1.0.0 (fixed decisions §6, conventions §6.1), `prd.md` v1.0.0 (acceptance criteria §4)
**Feeds:** `checklist.md` v1.0.0

This document is implementable as written. Every table has its DDL, every endpoint its request and response shape, every formula its parameters and units, and every milestone its entry and exit criteria. Where a value is a judgement call, it lives in the `model_config` table with a provenance string rather than in code.

## 1. Repository layout

Monorepo, `Ibtidaa-Tech/airsaathi`.

```
airsaathi/
├── apps/
│   ├── api/                          # airsaathi-api — FastAPI
│   │   ├── src/airsaathi_api/
│   │   │   ├── main.py               # app factory, middleware, routers
│   │   │   ├── config.py             # pydantic-settings, env parsing
│   │   │   ├── deps.py               # DI: db session, redis, current household
│   │   │   ├── routers/
│   │   │   │   ├── health.py  locations.py  air_quality.py
│   │   │   │   ├── households.py  plans.py  actions.py
│   │   │   │   ├── impact.py  ask.py  circles.py  reports.py
│   │   │   │   ├── devices.py  privacy.py
│   │   │   ├── domain/
│   │   │   │   ├── aqi.py            # §6.2 sub-index + overall AQI
│   │   │   │   ├── fusion.py         # §6.3 station + model blend
│   │   │   │   ├── indoor.py         # §7.3 mass balance
│   │   │   │   ├── exposure.py       # §7.4 exposure + inhaled dose
│   │   │   │   ├── impact.py         # §7.6 counterfactual + CO2e + cig-eq
│   │   │   │   ├── planner.py        # §7.7 candidate generation + ranking
│   │   │   │   └── ventilation.py    # §7.5 window detection
│   │   │   ├── llm/
│   │   │   │   ├── client.py         # Anthropic client, retries, budget
│   │   │   │   ├── schemas.py        # pydantic output contracts
│   │   │   │   ├── narrator.py       # §8.3 plan narration
│   │   │   │   ├── advisor.py        # §8.4 grounded Q&A
│   │   │   │   ├── router.py         # §8.2 intent classification
│   │   │   │   ├── validator.py      # §8.5 numeric + policy guard
│   │   │   │   └── prompts/          # versioned .md prompt files
│   │   │   ├── providers/
│   │   │   │   ├── open_meteo_aq.py  open_meteo_weather.py  openaq.py
│   │   │   ├── rag/
│   │   │   │   ├── embed.py  retrieve.py  ingest.py
│   │   │   ├── models/               # SQLAlchemy 2.0 ORM
│   │   │   └── schemas/              # API pydantic models (camelCase alias)
│   │   ├── migrations/               # alembic
│   │   ├── tests/
│   │   │   ├── unit/ integration/ golden/ adversarial/
│   │   ├── pyproject.toml  Dockerfile
│   ├── worker/                       # airsaathi-worker — arq jobs
│   │   └── src/airsaathi_worker/{jobs,schedules}.py
│   └── app/                          # airsaathi-app — Flutter
│       ├── lib/
│       │   ├── main.dart  app.dart  router.dart
│       │   ├── core/{api,db,cache,i18n,theme,analytics}/
│       │   ├── features/{today,home,impact,ask,more,onboarding,circles}/
│       │   └── widgets/
│       ├── l10n/{app_en.arb,app_hi.arb,app_mr.arb}
│       └── test/  integration_test/
├── packages/
│   └── config-india/                 # the India configuration pack (§5.4)
│       ├── model_config.v1.yaml      # exposure parameters + provenance
│       ├── aqi_breakpoints.cpcb.yaml
│       ├── action_catalogue.yaml
│       └── emission_factors.yaml
├── corpus/                           # RAG source documents + manifest
├── infra/{fly.api.toml,fly.worker.toml,grafana/}
├── docs/{scope.md,prd.md,spec.md,checklist.md}
└── .github/workflows/{ci.yml,deploy.yml}
```

## 2. Architecture

```
┌────────────────────────────── Flutter client (Android) ─────────────────────────────┐
│  Riverpod providers → Repository → Dio (ETag, retry) ──┐    Drift (SQLite) offline  │
│                                                         │    mirror of plans,       │
│                                      FCM ◀──────────┐   │    actions, readings      │
└─────────────────────────────────────────────────────┼───┼────────────────────────────┘
                                                      │   │ HTTPS, Firebase ID token
                                                      │   ▼
┌───────────────────────────── airsaathi-api (FastAPI) ──────────────────────────────┐
│  routers → deps(auth, household scope, rate limit) → domain/ (pure functions)      │
│                                      │                                              │
│   llm/  ── Anthropic API ────────────┤   rag/ ── pgvector ──┐                       │
│   providers/ ── Open-Meteo, OpenAQ ──┤                      │                       │
└──────────────────────────────────────┼──────────────────────┼───────────────────────┘
                                       ▼                      ▼
                        ┌──────────── Redis 7 ────────┐  ┌─ PostgreSQL 16 ─┐
                        │ AQ cache (TTL 15m)          │  │ postgis         │
                        │ rate-limit buckets          │  │ pgvector (HNSW) │
                        │ arq queue                   │  └─────────────────┘
                        └──────────────┬──────────────┘
                                       ▼
┌──────────────────────── airsaathi-worker (arq) ───────────────────────────────────┐
│  fuse_grid_cells (*/15m) · refresh_forecasts (hourly) · generate_daily_plans (05:30│
│  per tz) · detect_material_change (*/15m) · send_alerts (*/5m) · weekly_impact     │
│  (Sun 19:00) · aggregate_reports (hourly) · rollup_exposure (daily 00:20)          │
│  · reembed_corpus (manual) · purge_expired (daily 03:00)                           │
└───────────────────────────────────────────────────────────────────────────────────┘
```

**Separation of concerns, stated as an invariant:** `domain/` contains pure, deterministic, side-effect-free functions. Every number the product shows is produced there. `llm/` consumes `domain/` output and produces only prose. No module in `llm/` may compute, derive, round, or restate a quantity that `domain/` did not produce. This is enforced by the validator in §8.5 and by a lint rule that forbids arithmetic operators in `llm/`.

## 3. Technology inventory

| Component | Version | Notes |
|---|---|---|
| Python | 3.12.x | `uv` for dependency management |
| FastAPI | 0.115+ | |
| Pydantic | 2.9+ | `alias_generator=to_camel`, `populate_by_name=True` |
| SQLAlchemy | 2.0+ | async engine, `asyncpg` driver |
| Alembic | 1.13+ | one migration per milestone, never edited after merge |
| PostgreSQL | 16 | extensions: `postgis`, `vector`, `pg_trgm` |
| pgvector | 0.7+ | HNSW index, `vector_cosine_ops` |
| Redis | 7.2 | |
| arq | 0.26+ | |
| anthropic (Python SDK) | 0.40+ | models `claude-sonnet-5-5`, `claude-haiku-5-5` |
| sentence-transformers | 3.x | `BAAI/bge-m3`, 1024-dim, normalised |
| httpx | 0.27+ | upstream calls, `http2=True`, timeouts in §4.2 |
| Flutter | 3.27+ | Dart 3.6+ |
| Riverpod | 2.5+ | `riverpod_generator` |
| Drift | 2.20+ | |
| Dio | 5.7+ | `dio_cache_interceptor` |
| fl_chart | 0.69+ | hourly strip and impact charts |
| firebase_auth / firebase_messaging | latest stable | |
| OpenTelemetry | Python + Dart SDKs | traces to Grafana Cloud OTLP |

## 4. Configuration

### 4.1 Environment variables

All parsed by `config.py` through `pydantic-settings`. The service refuses to start if any required variable is absent — there are no silent defaults for anything in this table.

| Variable | Required | Example | Purpose |
|---|---|---|---|
| `AIRSAATHI_ENV` | yes | `production` | `local` \| `staging` \| `production` |
| `DATABASE_URL` | yes | `postgresql+asyncpg://…` | Neon Postgres, Mumbai |
| `REDIS_URL` | yes | `rediss://…` | Upstash |
| `ANTHROPIC_API_KEY` | yes | — | LLM access |
| `ANTHROPIC_MODEL_PRIMARY` | yes | `claude-sonnet-5-5` | Narration, Q&A |
| `ANTHROPIC_MODEL_ROUTER` | yes | `claude-haiku-5-5` | Intent classification |
| `OPENAQ_API_KEY` | yes | — | `X-API-Key` header |
| `OPEN_METEO_AQ_BASE` | yes | `https://air-quality-api.open-meteo.com/v1/air-quality` | Keyless |
| `OPEN_METEO_WEATHER_BASE` | yes | `https://api.open-meteo.com/v1/forecast` | Keyless |
| `FIREBASE_PROJECT_ID` | yes | — | ID-token verification |
| `FCM_SERVICE_ACCOUNT_JSON` | yes | — | Push, base64-encoded |
| `MODEL_CONFIG_VERSION` | yes | `india-1.0.0` | Active `model_config` row |
| `CORPUS_VERSION` | yes | `2026-11-01` | Active corpus snapshot |
| `LLM_DAILY_TOKEN_CAP_PER_HOUSEHOLD` | yes | `12000` | §8.6 budget guard |
| `ASK_DAILY_LIMIT_PER_HOUSEHOLD` | yes | `15` | §8.6 |
| `SENTRY_DSN` | no | — | Absent disables Sentry |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | no | — | Absent disables tracing |

### 4.2 Upstream timeouts, retries, and budgets

| Upstream | Connect | Read | Retries | Backoff | Circuit breaker |
|---|---|---|---|---|---|
| Open-Meteo AQ | 2 s | 6 s | 2 | 0.5 s, 1.5 s jittered | Open after 5 consecutive failures, half-open after 60 s |
| Open-Meteo Weather | 2 s | 6 s | 2 | same | same |
| OpenAQ v3 | 2 s | 8 s | 2 | 1 s, 3 s jittered | same; plus token bucket 50 req/min against the published 60 req/min and 2,000 req/hour limits |
| Anthropic | 3 s | 45 s (stream) | 1 | 1 s | Open after 3 consecutive 5xx, half-open after 30 s |

On an open breaker the API serves the last persisted value with the appropriate `confidence` and staleness, per `prd.md` §5.4. No upstream failure is ever surfaced as a technical error string to the client.

## 5. Data model

### 5.1 Enumerations

Defined once as Postgres enum types and mirrored in Python and Dart. All values `UPPER_SNAKE_CASE`.

```sql
CREATE TYPE aqi_band       AS ENUM ('GOOD','SATISFACTORY','MODERATE','POOR','VERY_POOR','SEVERE');
CREATE TYPE confidence     AS ENUM ('HIGH','MEDIUM','LOW');
CREATE TYPE source_kind    AS ENUM ('STATION','MODEL','USER_SENSOR');
CREATE TYPE age_band       AS ENUM ('INFANT_0_4','CHILD_5_12','TEEN_13_17','ADULT_18_59','SENIOR_60_PLUS');
CREATE TYPE sensitivity    AS ENUM ('ASTHMA','COPD','HEART_CONDITION','PREGNANCY','ALLERGIC_RHINITIS','NONE');
CREATE TYPE microenv       AS ENUM ('OUTDOOR','HOME_INDOOR','TRANSIT','INDOOR_OTHER');
CREATE TYPE activity_class AS ENUM ('REST','LIGHT','MODERATE','VIGOROUS');
CREATE TYPE transit_mode   AS ENUM ('WALK_SIDE_STREET','WALK_MAIN_ROAD','CYCLE','TWO_WHEELER',
                                    'AUTO_RICKSHAW','CAR_WINDOWS_OPEN','CAR_RECIRC_FILTER',
                                    'BUS','METRO_UNDERGROUND','METRO_ELEVATED','NONE');
CREATE TYPE seal_quality   AS ENUM ('LOOSE','TYPICAL','TIGHT');
CREATE TYPE cooking_fuel   AS ENUM ('LPG','PNG','ELECTRIC_INDUCTION','KEROSENE','BIOMASS','COAL');
CREATE TYPE kitchen_vent   AS ENUM ('NONE','WINDOW_ONLY','EXHAUST_FAN','DUCTED_CHIMNEY');
CREATE TYPE indoor_source  AS ENUM ('COOKING','INCENSE','MOSQUITO_COIL','SMOKING_INDOORS',
                                    'CANDLES','DIYA','FRYING','SWEEPING_DRY','BIOMASS_HEATING');
CREATE TYPE cost_tier      AS ENUM ('FREE','LOW_COST','PURCHASE');
CREATE TYPE action_kind    AS ENUM ('AVOID_OUTDOOR','SHIFT_OUTDOOR','VENTILATE','SEAL',
                                    'FILTER','STOP_SOURCE','VENT_SOURCE','MASK','COMMUTE_SWAP',
                                    'COOKING_SWAP','WASTE_SWAP');
CREATE TYPE alert_type     AS ENUM ('BAND_DETERIORATION','VENTILATION_WINDOW','FORECAST_SPIKE',
                                    'SEVERE_IMMINENT','WINDOW_WITHDRAWN','WEEKLY_IMPACT','CIRCLE_REPORT');
CREATE TYPE intent_class   AS ENUM ('AIR_QUALITY','INDOOR','PRODUCT','BEHAVIOUR','POLICY',
                                    'MEDICAL','OFF_TOPIC','UNSAFE');
CREATE TYPE report_source  AS ENUM ('OPEN_WASTE_BURNING','CROP_RESIDUE_BURNING','CONSTRUCTION_DUST',
                                    'INDUSTRIAL_SMOKE','VEHICLE_IDLING','GENERATOR_EXHAUST','ROAD_DUST');
CREATE TYPE consent_kind   AS ENUM ('TERMS','LOCATION','HEALTH_SENSITIVITY','PUSH','ANALYTICS','CIRCLE_SHARING');
```

### 5.2 Schema

```sql
-- ─── identity and household ────────────────────────────────────────────────
CREATE TABLE users (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  firebase_uid    text UNIQUE NOT NULL,
  phone_e164      text,                       -- nullable: anonymous-first
  is_anonymous    boolean NOT NULL DEFAULT true,
  locale          text NOT NULL DEFAULT 'en' CHECK (locale IN ('en','hi','mr')),
  created_at      timestamptz NOT NULL DEFAULT now(),
  last_seen_at    timestamptz NOT NULL DEFAULT now(),
  erasure_requested_at timestamptz
);

CREATE TABLE locations (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  geohash5        char(5) NOT NULL,            -- fusion cell key, ~5 km
  geohash7        char(7) NOT NULL,            -- display precision, ~150 m
  centroid        geography(Point,4326) NOT NULL,  -- cell centroid, never the user's point
  exact_point     geography(Point,4326),       -- ONLY if consent_kind LOCATION granted at EXACT
  precision_source text NOT NULL DEFAULT 'GPS_TRUNCATED'
                   CHECK (precision_source IN ('GPS_TRUNCATED','TEXT_SEARCH','EXACT_CONSENTED')),
  area_label      text NOT NULL,
  city            text NOT NULL,
  state           text NOT NULL,
  country_code    char(2) NOT NULL DEFAULT 'IN',
  timezone        text NOT NULL DEFAULT 'Asia/Kolkata',
  created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX locations_geohash5_idx ON locations (geohash5);
CREATE INDEX locations_centroid_gix ON locations USING gist (centroid);

CREATE TABLE households (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_user_id      uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  primary_location_id uuid NOT NULL REFERENCES locations(id),
  floor_number       smallint NOT NULL DEFAULT 1 CHECK (floor_number BETWEEN -2 AND 60),
  seal_quality       seal_quality NOT NULL DEFAULT 'TYPICAL',
  home_volume_m3     numeric(7,2) NOT NULL DEFAULT 90.00 CHECK (home_volume_m3 > 0),
  bedroom_volume_m3  numeric(7,2) NOT NULL DEFAULT 32.00 CHECK (bedroom_volume_m3 > 0),
  cooking_fuel       cooking_fuel NOT NULL DEFAULT 'LPG',
  kitchen_vent       kitchen_vent NOT NULL DEFAULT 'WINDOW_ONLY',
  has_purifier       boolean NOT NULL DEFAULT false,
  purifier_cadr_m3h  integer CHECK (purifier_cadr_m3h IS NULL OR purifier_cadr_m3h BETWEEN 50 AND 1200),
  purifier_room      text CHECK (purifier_room IS NULL OR purifier_room IN ('BEDROOM','LIVING','WHOLE_HOME')),
  biomass_heating    boolean NOT NULL DEFAULT false,
  monthly_budget_inr integer CHECK (monthly_budget_inr IS NULL OR monthly_budget_inr >= 0),
  adopted_actions    text[] NOT NULL DEFAULT '{}',   -- action_catalogue.code values
  onboarding_complete boolean NOT NULL DEFAULT false,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT purifier_consistency CHECK (
    (has_purifier = false AND purifier_cadr_m3h IS NULL AND purifier_room IS NULL)
    OR (has_purifier = true AND purifier_cadr_m3h IS NOT NULL AND purifier_room IS NOT NULL))
);
CREATE UNIQUE INDEX households_one_per_owner_idx ON households (owner_user_id);

CREATE TABLE household_members (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  household_id  uuid NOT NULL REFERENCES households(id) ON DELETE CASCADE,
  nickname      text CHECK (nickname IS NULL OR char_length(nickname) <= 24),
  age_band      age_band NOT NULL,
  relationship  text NOT NULL,
  sensitivity_flags sensitivity[] NOT NULL DEFAULT '{NONE}',
  created_at    timestamptz NOT NULL DEFAULT now()
  -- No date_of_birth, full name, school, or photo column exists, by design (prd.md US-2.3).
);
CREATE INDEX household_members_household_idx ON household_members (household_id);

CREATE TABLE routine_blocks (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  member_id     uuid NOT NULL REFERENCES household_members(id) ON DELETE CASCADE,
  label         text NOT NULL,
  microenv      microenv NOT NULL,
  activity      activity_class NOT NULL,
  transit       transit_mode NOT NULL DEFAULT 'NONE',
  start_local   time NOT NULL,
  end_local     time NOT NULL,
  days_of_week  smallint[] NOT NULL CHECK (array_length(days_of_week,1) BETWEEN 1 AND 7), -- 1=Mon
  created_at    timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT block_order CHECK (end_local > start_local),
  CONSTRAINT transit_consistency CHECK (
    (microenv = 'TRANSIT' AND transit <> 'NONE') OR (microenv <> 'TRANSIT' AND transit = 'NONE'))
);
CREATE INDEX routine_blocks_member_idx ON routine_blocks (member_id);

CREATE TABLE indoor_source_habits (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  household_id  uuid NOT NULL REFERENCES households(id) ON DELETE CASCADE,
  source        indoor_source NOT NULL,
  minutes_per_day smallint NOT NULL CHECK (minutes_per_day BETWEEN 0 AND 1440),
  typical_start_local time,
  vented        boolean NOT NULL DEFAULT false,
  UNIQUE (household_id, source)
);

-- ─── air quality data ──────────────────────────────────────────────────────
CREATE TABLE aq_stations (
  id            bigint PRIMARY KEY,            -- OpenAQ locations_id
  name          text NOT NULL,
  provider      text NOT NULL,
  licence       text NOT NULL,
  point         geography(Point,4326) NOT NULL,
  parameters    text[] NOT NULL,
  last_seen_at  timestamptz,
  is_active     boolean NOT NULL DEFAULT true
);
CREATE INDEX aq_stations_point_gix ON aq_stations USING gist (point);

CREATE TABLE aq_observations (                 -- raw, per station, hourly
  station_id    bigint NOT NULL REFERENCES aq_stations(id),
  observed_at   timestamptz NOT NULL,
  pm25          numeric(7,2), pm10 numeric(7,2), no2 numeric(7,2),
  o3            numeric(7,2), so2 numeric(7,2), co numeric(6,3),  -- co in mg/m3
  PRIMARY KEY (station_id, observed_at)
);

CREATE TABLE aq_readings (                     -- fused, per grid cell, hourly
  geohash5      char(5) NOT NULL,
  observed_at   timestamptz NOT NULL,
  pm25          numeric(7,2) NOT NULL,
  pm10          numeric(7,2), no2 numeric(7,2), o3 numeric(7,2),
  so2           numeric(7,2), co numeric(6,3),
  pm25_24h      numeric(7,2),                  -- rolling means used for the AQI (§6.2)
  pm10_24h      numeric(7,2), no2_24h numeric(7,2), so2_24h numeric(7,2),
  o3_8h         numeric(7,2), co_8h numeric(6,3),
  aqi           smallint NOT NULL CHECK (aqi BETWEEN 0 AND 500),
  band          aqi_band NOT NULL,
  dominant      text NOT NULL,
  aqi_hours_available smallint NOT NULL DEFAULT 0,
  confidence    confidence NOT NULL,
  station_count smallint NOT NULL DEFAULT 0,
  nearest_station_km numeric(6,2),
  alpha         numeric(4,3) NOT NULL,         -- station weight actually applied (§6.3)
  sources       jsonb NOT NULL,                -- [{kind,id,distanceKm,observedAt,licence}]
  model_config_version text NOT NULL,
  PRIMARY KEY (geohash5, observed_at)
);
CREATE INDEX aq_readings_recent_idx ON aq_readings (geohash5, observed_at DESC);

CREATE TABLE aq_forecasts (                    -- per grid cell, hourly, horizon 120 h
  geohash5      char(5) NOT NULL,
  valid_at      timestamptz NOT NULL,
  issued_at     timestamptz NOT NULL,
  pm25          numeric(7,2) NOT NULL,
  pm10          numeric(7,2), no2 numeric(7,2), o3 numeric(7,2),
  so2           numeric(7,2), co numeric(6,3),
  aqi           smallint NOT NULL, band aqi_band NOT NULL,
  wind_speed_ms numeric(5,2), temp_c numeric(5,2),
  boundary_layer_m integer,
  confidence    confidence NOT NULL,
  PRIMARY KEY (geohash5, valid_at, issued_at)
);
CREATE INDEX aq_forecasts_latest_idx ON aq_forecasts (geohash5, valid_at, issued_at DESC);

-- ─── configuration (versioned, never hard-coded) ───────────────────────────
CREATE TABLE model_config (
  version       text PRIMARY KEY,              -- e.g. 'india-1.0.0'
  params        jsonb NOT NULL,                -- full parameter tree (§7.2)
  provenance    jsonb NOT NULL,                -- per-parameter source string
  reviewed_by   text NOT NULL,
  reviewed_at   timestamptz NOT NULL,
  is_active     boolean NOT NULL DEFAULT false
);
CREATE UNIQUE INDEX model_config_single_active_idx ON model_config (is_active) WHERE is_active;

CREATE TABLE aqi_breakpoints (
  standard      text NOT NULL,                 -- 'CPCB'
  pollutant     text NOT NULL,
  averaging_h   smallint NOT NULL,
  c_low         numeric(8,3) NOT NULL, c_high numeric(8,3) NOT NULL,
  i_low         smallint NOT NULL,  i_high smallint NOT NULL,
  provenance    text NOT NULL,
  PRIMARY KEY (standard, pollutant, c_low)
);

CREATE TABLE action_catalogue (
  code          text PRIMARY KEY,              -- e.g. 'SEAL_WINDOWS_PEAK'
  kind          action_kind NOT NULL,
  cost_tier     cost_tier NOT NULL,
  cost_inr      integer NOT NULL DEFAULT 0 CHECK (cost_inr >= 0),
  effort_minutes smallint NOT NULL DEFAULT 0,
  effect_model  text NOT NULL,                 -- function name in domain/ (§7.8)
  effect_params jsonb NOT NULL,
  preconditions jsonb NOT NULL,                -- household predicate (§7.8)
  recurring     boolean NOT NULL DEFAULT true,
  copy_en       jsonb NOT NULL,                -- {title, body, why}
  copy_hi       jsonb, copy_mr jsonb,
  review_status text NOT NULL DEFAULT 'DRAFT' CHECK (review_status IN ('DRAFT','REVIEWED')),
  is_active     boolean NOT NULL DEFAULT true
);

CREATE TABLE emission_factors (
  key           text PRIMARY KEY,              -- transit_mode or cooking key
  gco2e_per_pkm numeric(7,2),
  gco2e_per_hour numeric(8,2),
  provenance    text NOT NULL,
  config_version text NOT NULL REFERENCES model_config(version)
);

-- ─── plans, actions, impact ────────────────────────────────────────────────
CREATE TABLE plans (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  household_id  uuid NOT NULL REFERENCES households(id) ON DELETE CASCADE,
  plan_date     date NOT NULL,
  revision      integer NOT NULL DEFAULT 1,
  verdict_text  text NOT NULL,
  verdict_source text NOT NULL CHECK (verdict_source IN ('LLM','TEMPLATE')),
  language      text NOT NULL,
  facts         jsonb NOT NULL,                -- the facts object handed to the LLM (§8.3)
  peak_pm25     numeric(7,2) NOT NULL,
  peak_band     aqi_band NOT NULL,
  baseline_exposure_ugm3h numeric(10,2) NOT NULL,
  model_config_version text NOT NULL,
  prompt_version text NOT NULL,
  generated_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (household_id, plan_date, revision)
);
CREATE INDEX plans_household_date_idx ON plans (household_id, plan_date DESC, revision DESC);

CREATE TABLE plan_actions (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id       uuid NOT NULL REFERENCES plans(id) ON DELETE CASCADE,
  action_code   text NOT NULL REFERENCES action_catalogue(code),
  member_id     uuid REFERENCES household_members(id) ON DELETE SET NULL,
  window_start_local time NOT NULL,
  window_end_local   time NOT NULL,
  avoided_exposure_ugm3h numeric(10,2) NOT NULL,
  avoided_inhaled_ug     numeric(10,2) NOT NULL,
  avoided_gco2e          numeric(10,2) NOT NULL DEFAULT 0,
  cost_inr      integer NOT NULL,
  rank          smallint NOT NULL,
  title_text    text NOT NULL,
  body_text     text NOT NULL,
  trace         jsonb NOT NULL,                -- full arithmetic trace (§7.9)
  CONSTRAINT window_order CHECK (window_end_local > window_start_local)
);
CREATE INDEX plan_actions_plan_idx ON plan_actions (plan_id, rank);

CREATE TABLE action_completions (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_action_id uuid NOT NULL REFERENCES plan_actions(id) ON DELETE CASCADE,
  household_id  uuid NOT NULL REFERENCES households(id) ON DELETE CASCADE,
  completed_at  timestamptz NOT NULL DEFAULT now(),
  client_mutation_id uuid NOT NULL,            -- idempotency for offline sync
  UNIQUE (client_mutation_id)
);
CREATE INDEX action_completions_household_idx ON action_completions (household_id, completed_at DESC);

CREATE TABLE exposure_estimates (              -- daily rollup per member
  household_id  uuid NOT NULL REFERENCES households(id) ON DELETE CASCADE,
  member_id     uuid NOT NULL REFERENCES household_members(id) ON DELETE CASCADE,
  est_date      date NOT NULL,
  baseline_exposure_ugm3h numeric(10,2) NOT NULL,
  actual_exposure_ugm3h   numeric(10,2) NOT NULL,
  baseline_inhaled_ug     numeric(10,2) NOT NULL,
  actual_inhaled_ug       numeric(10,2) NOT NULL,
  protected_hours         numeric(6,2) NOT NULL,
  cig_eq_baseline         numeric(6,2) NOT NULL,
  cig_eq_actual           numeric(6,2) NOT NULL,
  avoided_gco2e           numeric(10,2) NOT NULL DEFAULT 0,
  model_config_version    text NOT NULL,
  PRIMARY KEY (household_id, member_id, est_date)
);

-- ─── RAG corpus ────────────────────────────────────────────────────────────
CREATE TABLE knowledge_documents (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  title         text NOT NULL,
  publisher     text NOT NULL,
  url           text,
  licence       text NOT NULL,
  published_on  date,
  corpus_version text NOT NULL,
  language      text NOT NULL DEFAULT 'en'
);

CREATE TABLE knowledge_chunks (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id   uuid NOT NULL REFERENCES knowledge_documents(id) ON DELETE CASCADE,
  ordinal       integer NOT NULL,
  heading_path  text NOT NULL,
  content       text NOT NULL,
  token_count   smallint NOT NULL,
  embedding     vector(1024) NOT NULL,
  UNIQUE (document_id, ordinal)
);
CREATE INDEX knowledge_chunks_hnsw_idx ON knowledge_chunks
  USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
CREATE INDEX knowledge_chunks_trgm_idx ON knowledge_chunks USING gin (content gin_trgm_ops);

-- ─── Q&A ───────────────────────────────────────────────────────────────────
CREATE TABLE ask_sessions (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  household_id  uuid NOT NULL REFERENCES households(id) ON DELETE CASCADE,
  started_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE ask_messages (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id    uuid NOT NULL REFERENCES ask_sessions(id) ON DELETE CASCADE,
  role          text NOT NULL CHECK (role IN ('USER','ASSISTANT')),
  body          text NOT NULL,
  intent        intent_class,
  citations     jsonb NOT NULL DEFAULT '[]',
  input_tokens  integer, output_tokens integer,
  blocked_reason text,
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ask_messages_session_idx ON ask_messages (session_id, created_at);

-- ─── circles and reports ───────────────────────────────────────────────────
CREATE TABLE circles (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name          text NOT NULL CHECK (char_length(name) BETWEEN 3 AND 60),
  invite_code   char(8) UNIQUE NOT NULL,
  location_id   uuid NOT NULL REFERENCES locations(id),
  admin_user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  member_cap    smallint NOT NULL DEFAULT 150 CHECK (member_cap <= 150),
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE circle_members (
  circle_id     uuid NOT NULL REFERENCES circles(id) ON DELETE CASCADE,
  household_id  uuid NOT NULL REFERENCES households(id) ON DELETE CASCADE,
  joined_at     timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (circle_id, household_id)
);

CREATE TABLE reports (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  circle_id     uuid REFERENCES circles(id) ON DELETE SET NULL,
  household_id  uuid NOT NULL REFERENCES households(id) ON DELETE CASCADE,
  source_type   report_source NOT NULL,
  point         geography(Point,4326) NOT NULL,   -- geohash7 centroid, not the exact point
  observed_at   timestamptz NOT NULL,
  pm25_at_report numeric(7,2),
  note          text CHECK (note IS NULL OR char_length(note) <= 280),
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX reports_circle_time_idx ON reports (circle_id, observed_at DESC);

CREATE TABLE complaint_drafts (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  circle_id     uuid NOT NULL REFERENCES circles(id) ON DELETE CASCADE,
  source_type   report_source NOT NULL,
  report_ids    uuid[] NOT NULL,
  body_text     text NOT NULL,
  generated_at  timestamptz NOT NULL DEFAULT now()
);

-- ─── devices, notifications, consent, audit ────────────────────────────────
CREATE TABLE devices (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  fcm_token     text UNIQUE NOT NULL,
  platform      text NOT NULL CHECK (platform IN ('ANDROID','IOS')),
  app_version   text NOT NULL,
  quiet_start_local time NOT NULL DEFAULT '21:00',
  quiet_end_local   time NOT NULL DEFAULT '06:30',
  enabled_alerts alert_type[] NOT NULL DEFAULT
    '{BAND_DETERIORATION,VENTILATION_WINDOW,FORECAST_SPIKE,SEVERE_IMMINENT,WINDOW_WITHDRAWN,WEEKLY_IMPACT}',
  last_registered_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE notifications (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  household_id  uuid NOT NULL REFERENCES households(id) ON DELETE CASCADE,
  alert_type    alert_type NOT NULL,
  body_text     text NOT NULL,
  dedupe_key    text NOT NULL,
  sent_at       timestamptz,
  opened_at     timestamptz,
  quiet_overridden boolean NOT NULL DEFAULT false,
  created_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE (household_id, dedupe_key)
);
CREATE INDEX notifications_household_day_idx ON notifications (household_id, created_at DESC);

CREATE TABLE consents (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  kind          consent_kind NOT NULL,
  granted       boolean NOT NULL,
  detail        text,                            -- e.g. 'EXACT' | 'AREA' for LOCATION
  policy_version text NOT NULL,
  recorded_at   timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX consents_user_kind_idx ON consents (user_id, kind, recorded_at DESC);

CREATE TABLE audit_log (
  id            bigserial PRIMARY KEY,
  actor         text NOT NULL,                  -- 'user:<uuid>' | 'job:<name>' | 'admin:<id>'
  action        text NOT NULL,
  entity        text NOT NULL,
  entity_id     text,
  metadata      jsonb NOT NULL DEFAULT '{}',
  at            timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX audit_log_entity_idx ON audit_log (entity, entity_id, at DESC);
```

### 5.3 Onboarding defaults

Every skippable onboarding field has exactly one documented default. These are the values that make `prd.md` US-2.1 testable.

| Field | Default | Effect of the default |
|---|---|---|
| `seal_quality` | `TYPICAL` | F_inf 0.60, air exchange 0.50 h⁻¹ |
| `home_volume_m3` | 90.00 | 2 BHK, 2.7 m ceiling |
| `bedroom_volume_m3` | 32.00 | 3.5 × 3.4 × 2.7 m |
| `floor_number` | 1 | No vertical adjustment applied |
| `cooking_fuel` | `LPG` | 1,500 µg/h unvented emission |
| `kitchen_vent` | `WINDOW_ONLY` | Vent factor 0.70 |
| `has_purifier` | `false` | λ_p = 0 |
| `biomass_heating` | `false` | No heating source term |
| Members | One `ADULT_18_59`, `{NONE}`, relationship `SELF` | Single-adult household model |
| Routine blocks | `HOME_INDOOR`/`REST` 22:00–06:00; `HOME_INDOOR`/`LIGHT` 06:00–09:00 and 18:00–22:00; `INDOOR_OTHER`/`LIGHT` 09:00–18:00 | A plausible day with no commute; commute advice is suppressed until a `TRANSIT` block exists |
| `indoor_source_habits` | `COOKING` 60 min/day starting 07:00, `vented=false` | Dominant Indian indoor source retained; all others absent |
| Location precision | `AREA` — the geohash5 cell centroid, from a device coordinate truncated on the device | No confidence penalty. A location obtained by text search instead (`precision_source = 'TEXT_SEARCH'`) caps `confidence` at `MEDIUM`, per §6.3 |
| `language` | Device locale if in {en,hi,mr}, else `en` | |
| Quiet hours | 21:00–06:30 local | |

### 5.4 The India configuration pack

`packages/config-india/` holds the YAML that is loaded into `model_config`, `aqi_breakpoints`, `action_catalogue`, and `emission_factors` by `alembic` data migrations. Nothing in this pack is duplicated in code. Every leaf value carries a `provenance` string naming its source document, and the domain reviewer signs the pack by writing `reviewed_by` and `reviewed_at` on the `model_config` row. Activating a pack is a single transaction that flips `is_active`; the previous row is retained forever so that any historical `exposure_estimates` row remains reproducible from its `model_config_version`.

## 6. Air quality ingestion, fusion, and AQI

### 6.1 Providers

**Open-Meteo Air Quality** — `GET https://air-quality-api.open-meteo.com/v1/air-quality`, no API key.

```
?latitude=26.85&longitude=80.95
&hourly=pm2_5,pm10,nitrogen_dioxide,ozone,sulphur_dioxide,carbon_monoxide,us_aqi
&current=pm2_5,pm10,nitrogen_dioxide,ozone,sulphur_dioxide,carbon_monoxide
&forecast_days=5&timezone=Asia%2FKolkata
```

Backed by the Copernicus Atmosphere Monitoring Service (CAMS) — the 11 km European and 40 km global composition forecasts — updated twice daily. Units: `pm2_5`, `pm10`, `nitrogen_dioxide`, `ozone`, `sulphur_dioxide` in µg/m³; `carbon_monoxide` in µg/m³, which the provider adapter divides by 1,000 to store `co` in mg/m³ for CPCB sub-index computation. Attribution to CAMS and to Open-Meteo is mandatory and ships in the About screen.

**Open-Meteo Weather** — `GET https://api.open-meteo.com/v1/forecast` with `hourly=wind_speed_10m,temperature_2m,boundary_layer_height`. Used only by the ventilation-window heuristic (§7.5).

**OpenAQ v3** — base `https://api.openaq.org/v3`, `X-API-Key` header, free tier 60 requests/minute and 2,000/hour. Three-step resource walk, which the adapter caches aggressively because the first two steps change rarely:

| Step | Call | Cache TTL |
|---|---|---|
| 1. Discover stations | `GET /v3/locations?coordinates={lon},{lat}&radius=25000&limit=100` (note: longitude first) | 24 h in Postgres `aq_stations` |
| 2. Map sensors | `GET /v3/locations/{locations_id}/sensors` | 24 h |
| 3. Latest values | `GET /v3/locations/{locations_id}/latest` | 15 min in Redis |
| 3b. Backfill history | `GET /v3/sensors/{sensors_id}/measurements/hourly?datetime_from=…&datetime_to=…` | Persisted in `aq_observations` |

Station discovery runs in the `refresh_stations` job (daily), not in the request path. A token bucket caps the adapter at 50 requests/minute.

### 6.2 CPCB AQI computation — `domain/aqi.py`

Breakpoints live in `aqi_breakpoints`, never in code. The India pack seeds the CPCB table below, each row carrying `provenance = 'CPCB National Air Quality Index, sub-index breakpoints'`:

| Pollutant | Avg (h) | Band boundaries (concentration → sub-index) |
|---|---|---|
| PM2.5 (µg/m³) | 24 | 0–30→0–50 · 30–60→51–100 · 60–90→101–200 · 90–120→201–300 · 120–250→301–400 · 250–500→401–500 |
| PM10 (µg/m³) | 24 | 0–50→0–50 · 50–100→51–100 · 100–250→101–200 · 250–350→201–300 · 350–430→301–400 · 430–600→401–500 |
| NO₂ (µg/m³) | 24 | 0–40→0–50 · 40–80→51–100 · 80–180→101–200 · 180–280→201–300 · 280–400→301–400 · 400–600→401–500 |
| O₃ (µg/m³) | 8 | 0–50→0–50 · 50–100→51–100 · 100–168→101–200 · 168–208→201–300 · 208–748→301–400 · 748–1000→401–500 |
| SO₂ (µg/m³) | 24 | 0–40→0–50 · 40–80→51–100 · 80–380→101–200 · 380–800→201–300 · 800–1600→301–400 · 1600–2000→401–500 |
| CO (mg/m³) | 8 | 0–1→0–50 · 1–2→51–100 · 2–10→101–200 · 10–17→201–300 · 17–34→301–400 · 34–50→401–500 |

**Averaging periods are not optional.** A CPCB sub-index is defined on the pollutant's averaging window, not on the instantaneous value. `domain/aqi.py` therefore takes rolling means from `aq_readings` history, never the spot reading: a 24-hour mean for PM2.5, PM10, NO₂, and SO₂, and an 8-hour mean for O₃ and CO. The averaged inputs are persisted alongside the result (`pm25_24h`, `o3_8h`, and siblings) so that any displayed AQI is reproducible from the stored row, and the client shows the spot concentration and the AQI side by side as two distinct quantities — which is why a reading can show PM2.5 at 248 µg/m³ and an AQI of 382 in the same card without contradiction.

**Partial history.** When fewer than 18 of the 24 hours (or 6 of the 8) are available, the mean is taken over the available hours, `aqi_hours_available` records the count, and `confidence` is capped at `MEDIUM`. Below 6 hours of PM2.5 history no AQI is emitted and the client renders the spot concentration with its band only.

Sub-index by linear interpolation inside the containing band:

```
I_p = I_low + (I_high - I_low) / (C_high - C_low) * (C_p - C_low)
```

Concentrations above the top band clamp to 500. Overall AQI and validity:

```
AQI       = max(I_p for p in available)
dominant  = argmax_p(I_p)
```

**Validity rule:** an overall AQI is emitted only when (a) at least one of PM2.5 or PM10 is present, and (b) at least three pollutants are present. When the rule fails, `aq_readings.aqi` is computed from PM2.5 alone, `dominant = 'PM2_5'`, and `confidence` is capped at `MEDIUM`; the client then renders the band with a "PM2.5 only" qualifier.

Band mapping is fixed: 0–50 `GOOD`, 51–100 `SATISFACTORY`, 101–200 `MODERATE`, 201–300 `POOR`, 301–400 `VERY_POOR`, 401–500 `SEVERE`.

### 6.3 Fusion — `domain/fusion.py`

Fusion is per **geohash-5 grid cell** (≈ 5 km), never per household. One fetch therefore serves every household in the cell, which is what keeps the OpenAQ budget flat as households grow (risk R5 in `scope.md` §9).

For each active cell, every 15 minutes:

1. **Station term.** Take active stations within 25 km with an observation no older than 180 minutes. Inverse-distance weighting, exponent 2, with the distance floored to avoid a singularity:

   ```
   d_i      = max(haversine_km(cell_centroid, station_i), 0.5)
   w_i      = 1 / d_i^2
   C_station = Σ(w_i * C_i) / Σ(w_i)
   ```

2. **Model term.** `C_model` is the Open-Meteo value for the cell centroid at the matching hour.

3. **Blend weight.** The nearer the nearest station, the more the stations are trusted; the model never falls below 15% weight:

   ```
   d_near = distance to nearest contributing station, km
   alpha  = clamp(1 - d_near / 25, 0, 0.85)        # 0 when no station contributes
   C_fused = alpha * C_station + (1 - alpha) * C_model
   ```

4. **Confidence.**

   | Condition | `confidence` |
   |---|---|
   | ≥ 2 stations within 10 km, freshest observation ≤ 90 min | `HIGH` |
   | ≥ 1 station within 25 km, freshest ≤ 180 min | `MEDIUM` |
   | Model only, or stations stale | `LOW` |

   Two caps apply on top of the table. A location whose `precision_source` is `TEXT_SEARCH` caps `confidence` at `MEDIUM`, because the household's real position inside a 5 km cell is unknown. Partial AQI history caps it at `MEDIUM` per §6.2. Where several caps apply, the lowest wins.

5. **Persist** one `aq_readings` row with `alpha`, `station_count`, `nearest_station_km`, and a `sources` array carrying every contributor's kind, id, distance, observation time, and licence — which is exactly what the provenance sheet in `prd.md` US-1.2 renders.

Forecast rows are refreshed hourly into `aq_forecasts`, keyed by `(geohash5, valid_at, issued_at)` so that a forecast is never overwritten and the material-change detector in §7.7 can compare issues. Forecast `confidence` is `HIGH` for `valid_at` within 24 h, `MEDIUM` to 72 h, `LOW` beyond.

A cell is "active" if any household's `primary_location_id` or any circle's `location_id` resolves to it. Inactive cells are not fetched.

## 7. The exposure engine — `domain/`

This section is the heart of the product. It is deterministic, pure, and the sole producer of every quantity the user sees.

### 7.1 Model outline

A household member's day is a sequence of `routine_blocks`. Each block places the member in a microenvironment with an activity class for a span of local time. The engine computes, per hour:

1. The outdoor concentration for the cell (observed or forecast).
2. The indoor concentration, from a mass balance over the home (§7.3).
3. The concentration the member actually breathes in their microenvironment, with transit and mask modifiers (§7.4).
4. Exposure (µg/m³·h) and inhaled dose (µg) over the block.

Two runs are always performed: the **counterfactual** (the household's routine with no actions taken) and the **actual** (with the actions completed). Every impact figure is the difference. This is why the baseline is stated on every screen in `prd.md` §5.3 — it is a real computed quantity, not a rhetorical device.

### 7.2 Parameters — `model_config.params`, version `india-1.0.0`

All values below are the shipped India pack. Each is overridable per household where the household has told us something better, and each carries a provenance string in `model_config.provenance`.

**Building envelope, by `seal_quality`**

| `seal_quality` | Air exchange `a` (h⁻¹) | Penetration `P` | Implied F_inf |
|---|---|---|---|
| `LOOSE` | 1.00 | 0.85 | 0.71 |
| `TYPICAL` | 0.50 | 0.80 | 0.57 |
| `TIGHT` | 0.25 | 0.70 | 0.39 |

F_inf is not an independent parameter — it is `P · a / (a + k)`, the steady-state ratio implied by the three values to its left, and it is listed only so the envelope's effect is legible at a glance. The engine never reads it.

Windows-open state overrides both: `a_open = 4.00 h⁻¹`, `P_open = 1.00`.

**PM2.5 deposition:** `k = 0.20 h⁻¹` (indoor surface loss).

**Indoor source emission rates (µg/h of PM2.5, while active)**

| `indoor_source` | Rate | Vent factor applied when vented |
|---|---|---|
| `COOKING` on `LPG` or `PNG` | 1,500 | `NONE` 1.00 · `WINDOW_ONLY` 0.70 · `EXHAUST_FAN` 0.45 · `DUCTED_CHIMNEY` 0.25 |
| `COOKING` on `KEROSENE` | 6,000 | same |
| `COOKING` on `BIOMASS` | 25,000 | same |
| `COOKING` on `COAL` | 18,000 | same |
| `COOKING` on `ELECTRIC_INDUCTION` | 400 | same |
| `FRYING` (additive to cooking) | 2,500 | same |
| `INCENSE` | 3,000 | — |
| `MOSQUITO_COIL` | 4,500 | — |
| `SMOKING_INDOORS` | 12,000 | — |
| `CANDLES` | 500 | — |
| `DIYA` | 700 | — |
| `SWEEPING_DRY` | 2,000 | — |
| `BIOMASS_HEATING` | 20,000 | — |

**Purifier removal:** `λ_p = CADR_m3h / V_room_m3` (h⁻¹). The DIY box-fan-plus-MERV-13 build is modelled at `CADR = 300 m³/h` and `cost_inr = 1800`.

**Transit mode multipliers on ambient PM2.5**

| `transit_mode` | Multiplier |
|---|---|
| `WALK_SIDE_STREET` | 1.00 |
| `WALK_MAIN_ROAD` | 1.50 |
| `CYCLE` | 1.40 |
| `TWO_WHEELER` | 2.00 |
| `AUTO_RICKSHAW` | 1.80 |
| `CAR_WINDOWS_OPEN` | 1.60 |
| `CAR_RECIRC_FILTER` | 0.60 |
| `BUS` | 1.50 |
| `METRO_UNDERGROUND` | 1.20 |
| `METRO_ELEVATED` | 0.90 |

`INDOOR_OTHER` (workplace, school) uses the `TYPICAL` envelope with no indoor sources.

**Inhalation rate `IR` (m³/h), adult base**

| `activity_class` | `IR` |
|---|---|
| `REST` | 0.45 |
| `LIGHT` | 0.90 |
| `MODERATE` | 1.80 |
| `VIGOROUS` | 3.00 |

**Age factor on `IR`:** `INFANT_0_4` 0.45 · `CHILD_5_12` 0.65 · `TEEN_13_17` 0.95 · `ADULT_18_59` 1.00 · `SENIOR_60_PLUS` 0.90.

**Sensitivity multipliers.** Sensitivity never alters physics. It alters two things only: the **threshold** at which an action is triggered, and the **rank** the action receives.

| `sensitivity` | Threshold multiplier | Rank boost |
|---|---|---|
| `ASTHMA` | 0.60 | +2 |
| `COPD` | 0.55 | +2 |
| `HEART_CONDITION` | 0.60 | +2 |
| `PREGNANCY` | 0.70 | +1 |
| `ALLERGIC_RHINITIS` | 0.80 | +1 |
| `NONE` | 1.00 | 0 |

Age also tightens thresholds: `INFANT_0_4` 0.60, `CHILD_5_12` 0.75, `SENIOR_60_PLUS` 0.75, others 1.00. Where both apply, the lower multiplier wins — they do not compound.

**Mask fitted filtration efficiency (FFE), real-world**

| Mask | FFE |
|---|---|
| `N95_FITTED` | 0.90 |
| `N95_LOOSE` | 0.70 |
| `SURGICAL` | 0.35 |
| `CLOTH` | 0.15 |
| `NONE` | 0.00 |

**Cigarette equivalence:** `CIG_PM25_UGM3_PER_DAY = 22.0`. Used only for the display figure in §7.6, always with the caveat string required by `prd.md` §6.3.

**CO₂e emission factors (g per passenger-km), `emission_factors`**

| Key | g CO₂e/p-km |
|---|---|
| `CAR_PETROL_SOLO` | 170 |
| `CAR_DIESEL_SOLO` | 160 |
| `CAR_EV_SOLO` | 60 |
| `CAR_SHARED_4` | 43 |
| `TWO_WHEELER` | 72 |
| `AUTO_RICKSHAW` | 58 |
| `BUS` | 35 |
| `METRO` | 14 |
| `CYCLE` | 0 |
| `WALK` | 0 |

Grid intensity for `CAR_EV_SOLO` and `METRO` is itself a pack parameter (`grid_gco2e_per_kwh`), so a cleaner grid is a config change, not a code change.

### 7.3 Indoor concentration — `domain/indoor.py`

Single well-mixed zone per hour, solved analytically so the step is unconditionally stable regardless of how large the loss rate becomes (a 500 m³/h purifier in a 32 m³ bedroom gives `L > 15 h⁻¹`, which a naive forward-Euler step would send negative).

```
L      = a + k + λ_p                                    # total loss rate, h^-1
S      = P * a * C_out + (E_total * vent_factor) / V     # total source, µg/m^3/h
C_ss   = S / L                                           # steady state, µg/m^3
C_in(t+Δ) = C_ss + (C_in(t) - C_ss) * exp(-L * Δ)        # exact for constant inputs
```

with `Δ = 1 h`, `V` = `home_volume_m3` (or `bedroom_volume_m3` when the purifier is `BEDROOM` and the member is in a `REST` block), and `E_total` the sum of active `indoor_source_habits` rates for that hour.

**Bedroom zoning** uses `bedroom_volume_m3` only when the bedroom door is closed — that is, when `CLOSE_BEDROOM_DOOR_AT_NIGHT` is among the household's completed actions. With the door open, the purifier serves `home_volume_m3` and `λ_p` falls accordingly. This single assumption is the difference between a purifier modelled at 8 air changes per hour and the same unit at 3, so it is surfaced in the trace's caveats rather than buried.

Initial condition for a day: `C_in(00:00) = C_ss` computed from the 00:00 outdoor value with no indoor sources. The day is then stepped forward hour by hour; a 24-hour warm-up from the previous day's final value is used whenever the previous day is available.

**Window state per hour** is set by the plan: an hour inside an issued and completed `VENTILATE` window uses `a_open` and `P_open`; an hour inside a completed `SEAL` window uses the `TIGHT` envelope regardless of the household's `seal_quality`; otherwise the household's own envelope applies.

### 7.4 Member exposure and inhaled dose — `domain/exposure.py`

Per member, per hour `t`, given their routine block:

```
C_breathed(t) =
    C_out(t)                                   if microenv = OUTDOOR
    C_in(t)                                    if microenv = HOME_INDOOR
    C_out(t) * transit_multiplier[mode]        if microenv = TRANSIT
    C_in_other(t)                              if microenv = INDOOR_OTHER

C_effective(t) = C_breathed(t) * (1 - FFE[mask(t)])

IR(t) = IR_base[activity] * age_factor[age_band]

exposure_ugm3h += C_effective(t) * Δ
inhaled_ug     += C_effective(t) * IR(t) * Δ
```

Hours not covered by any routine block default to `HOME_INDOOR` / `REST`. Overlapping blocks are impossible — the `routine_blocks` overlap check in `prd.md` US-2.2 rejects them at write time.

Both totals are reported. Exposure answers "how dirty was the air around them"; inhaled dose answers "how much did they actually breathe in", and only the latter responds to a child's smaller lungs or a cyclist's harder breathing. The Impact tab shows both because they move differently and a household that sees only one will eventually catch the product out.

### 7.5 Ventilation windows — `domain/ventilation.py`

A ventilation window is **the least-bad hour to air the house**, not an hour when opening the windows lowers PM2.5. The distinction matters, and getting it the wrong way round is the most likely modelling error in this feature.

With a `TYPICAL` envelope the infiltration factor is 0.57 (§7.2), so on a smog day indoor PM2.5 sits *below* outdoor almost all the time. A rule that issued a window only when outdoor fell below indoor would therefore almost never fire — and would be answering a question nobody asked. Households ventilate regardless, because they must: to clear cooking smoke, humidity, carbon dioxide, and solvent vapour. The product's job is not to talk them out of it but to tell them **when** to do it, and what it costs.

So the window is defined by a counterfactual shift, and its benefit is computed against the time the household would otherwise have opened up.

An hour `t` is a candidate when all of the following hold:

1. `forecast_confidence(t) = HIGH` and `t` is within 12 hours of now.
2. `C_out(t) <= p25(C_out over the forecast day)` — the hour is in the cleanest quarter of the day.
3. `C_out(t) <= 120` µg/m³ — a hard ceiling. Above it no window is issued at all that day, and the plan instead carries `SEAL_WINDOWS_PEAK` plus source-venting actions, which clear cooking smoke through the exhaust rather than by admitting outdoor air.
4. `wind_speed_10m(t) >= 1.0` m/s — stagnant air does not flush a room.

Contiguous candidate hours merge into one window. A window shorter than 30 minutes is discarded. At most two windows are issued per day; where more qualify, the two with the lowest mean `C_out` win.

**The effect model is `shift_ventilation`.** The counterfactual is the household airing the house for 60 minutes at its habitual time — `indoor_source_habits` gives that time where known, and the default is 07:00, which in the launch market is both the customary hour and, as it happens, close to the daily peak. The actual is the same 60 minutes inside the issued window. During any ventilating hour the envelope switches to `a_open = 4.00 h⁻¹` and `P_open = 1.00`, which puts indoor concentration within 2% of outdoor after 60 minutes.

Worked example on the §8.3 facts day, `TYPICAL` envelope, no purifier:

| Quantity | Habitual 07:00 | Issued window 14:00 |
|---|---|---|
| `C_out` | 285 µg/m³ | 82 µg/m³ |
| Indoor while open, ≈ `C_out · a_open/(a_open + k)` | 271 µg/m³ | 78 µg/m³ |
| Indoor had the house stayed shut | 163 µg/m³ | 71 µg/m³ |
| Cost of ventilating for 1 h | 108 µg/m³·h | 7 µg/m³·h |

`avoided_exposure_ugm3h = 108 - 7 = 101`. That figure — the saving from moving the airing, not from the airing itself — is what the action carries, and the action's copy says so: it offers the cleanest hour to air the house, and never claims that opening the windows reduces exposure.

**Withdrawal.** If a later forecast issue invalidates an issued window that has not yet started, the window is removed from the plan and a `WINDOW_WITHDRAWN` notification is sent. A window already in progress is not withdrawn — telling someone to shut a window they opened twenty minutes ago buys nothing and costs trust.

### 7.6 Impact arithmetic — `domain/impact.py`

```
avoided_exposure_ugm3h = baseline_exposure_ugm3h - actual_exposure_ugm3h
avoided_inhaled_ug     = baseline_inhaled_ug     - actual_inhaled_ug

protected_hours        = Σ over hours of Δ where actual_C_effective(t) < baseline_C_effective(t)

cig_eq                 = mean_24h_exposure_ugm3 / 22.0
                         where mean_24h_exposure_ugm3 = daily_exposure_ugm3h / 24

avoided_gco2e          = Σ over swapped trips of
                         distance_km * (ef[from_mode] - ef[to_mode])
```

`cig_eq` is defined per member per day, because its 22 µg/m³ basis is a 24-hour mean and summing a week of exposure into it would inflate the figure sevenfold. The weekly Impact surface therefore shows the **daily average** cigarette equivalent, baseline against actual, and labels it as such. Worked example, consistent with the facts object in §8.3: a baseline of 3,980 µg/m³·h per day gives 3,980 ÷ 24 ÷ 22 = 7.5, and a day on which the household completes every surfaced action removes 1,284 µg/m³·h, giving 2,696 ÷ 24 ÷ 22 = 5.1.

`protected_hours` is fractional and is summed across members for the household figure, which is why a four-person household can show more Protected Hours than there are hours in the week — the unit is person-hours and the UI labels it as such.

Negative values are never hidden. If a household's actual exposure exceeds the baseline — they cycled on a `SEVERE` day — the figure is shown negative with the trade-off copy from `prd.md` US-5.1.

### 7.7 Plan generation — `domain/planner.py`

Pure function. Input: household, members, routine blocks, source habits, the 72-hour forecast, the action catalogue, the active `model_config`. Output: an ordered list of candidate actions with computed effects. No LLM involvement whatsoever.

```
1  Build the hourly outdoor series for plan_date from aq_forecasts (latest issue per valid_at).
2  Run the counterfactual: indoor series + per-member exposure with zero actions.
3  Detect ventilation windows (§7.5).
4  For each action in action_catalogue where preconditions(household) is true:
       a  Determine the applicable time window for plan_date.
       b  Re-run the indoor + exposure model with that action applied, all others at baseline.
       c  effect = counterfactual - with_action, per affected member.
       d  Discard the action if avoided_exposure_ugm3h < 25 µg/m³·h (the model's noise floor).
5  Rank the surviving candidates:
       score = (avoided_exposure_ugm3h / max(cost_inr, 1)) * sensitivity_rank_boost_factor
       - Sort FREE before LOW_COST before PURCHASE.
       - Within a tier, sort by score descending.
       - Apply the member rank boost from §7.2.
6  Cap at 7 actions per plan, at most 2 of them PURCHASE.
7  Compose the facts object (§8.3) and persist plans + plan_actions with the full trace.
```

Step 4b is deliberately one-action-at-a-time against the baseline rather than cumulative. Effects do not add linearly — sealing the windows and running a purifier overlap — so the plan also stores a `combined` figure computed by running every surfaced action together, and the Impact tab reports the combined figure, never the sum of the individual ones. The per-action number is labelled "on its own" in the UI, which is the honest framing.

**Material change detection** (`detect_material_change`, every 15 min) compares the newest forecast issue against the issue the active plan was built from. The plan regenerates when any of: next-12 h peak PM2.5 differs by ≥ 25 µg/m³; the next-12 h peak band changes; a ventilation window appears or disappears. These are exactly the thresholds in `prd.md` US-3.3, and they live in `model_config.params.material_change` so they are tunable without a deploy.

### 7.8 Action catalogue — effect models and preconditions

Each `action_catalogue` row names an `effect_model`, which is a registered pure function in `domain/`, plus `effect_params` and a `preconditions` predicate evaluated against the household. The v1 pack ships these 22 actions:

| `code` | `kind` | Tier | `effect_model` | Preconditions |
|---|---|---|---|---|
| `SEAL_WINDOWS_PEAK` | `SEAL` | `FREE` | `seal_envelope` | always |
| `OPEN_WINDOWS_CLEAN_WINDOW` | `VENTILATE` | `FREE` | `shift_ventilation` | a window exists today (§7.5) |
| `SHIFT_OUTDOOR_TO_CLEANEST` | `SHIFT_OUTDOOR` | `FREE` | `shift_block` | an `OUTDOOR` block exists |
| `SKIP_OUTDOOR_EXERCISE` | `AVOID_OUTDOOR` | `FREE` | `remove_block` | a `MODERATE`+ `OUTDOOR` block exists |
| `KITCHEN_EXHAUST_WHILE_COOKING` | `VENT_SOURCE` | `FREE` | `vent_factor` | `kitchen_vent` ≥ `EXHAUST_FAN` |
| `KITCHEN_WINDOW_WHILE_COOKING` | `VENT_SOURCE` | `FREE` | `vent_factor` | `kitchen_vent` = `WINDOW_ONLY` |
| `EXHAUST_10_MIN_AFTER_COOKING` | `VENT_SOURCE` | `FREE` | `extend_vent` | `kitchen_vent` ≥ `EXHAUST_FAN` |
| `STOP_INCENSE` | `STOP_SOURCE` | `FREE` | `remove_source` | `INCENSE` habit exists |
| `STOP_MOSQUITO_COIL` | `STOP_SOURCE` | `FREE` | `remove_source` | `MOSQUITO_COIL` habit exists |
| `NO_SMOKING_INDOORS` | `STOP_SOURCE` | `FREE` | `remove_source` | `SMOKING_INDOORS` habit exists |
| `DAMP_MOP_INSTEAD_OF_SWEEP` | `STOP_SOURCE` | `FREE` | `remove_source` | `SWEEPING_DRY` habit exists |
| `CLOSE_BEDROOM_DOOR_AT_NIGHT` | `SEAL` | `FREE` | `zone_bedroom` | `has_purifier` and room = `BEDROOM` |
| `CAR_RECIRC_ON_COMMUTE` | `COMMUTE_SWAP` | `FREE` | `swap_transit` | a `CAR_WINDOWS_OPEN` block exists |
| `SHIFT_COMMUTE_30_MIN` | `COMMUTE_SWAP` | `FREE` | `shift_block` | a `TRANSIT` block exists |
| `WALK_SIDE_STREET_ROUTE` | `COMMUTE_SWAP` | `FREE` | `swap_transit` | a `WALK_MAIN_ROAD` block exists |
| `N95_ON_COMMUTE` | `MASK` | `LOW_COST` | `apply_mask` | a `TRANSIT` or `OUTDOOR` block exists |
| `DIY_BOX_FAN_FILTER` | `FILTER` | `LOW_COST` | `add_purifier` | not `has_purifier` |
| `DOOR_WINDOW_WEATHER_SEAL` | `SEAL` | `LOW_COST` | `upgrade_seal` | `seal_quality` ≠ `TIGHT` |
| `SWAP_TWO_WHEELER_FOR_METRO` | `COMMUTE_SWAP` | `LOW_COST` | `swap_transit` | a `TWO_WHEELER` block and a metro city |
| `HEPA_PURIFIER_BEDROOM` | `FILTER` | `PURCHASE` | `add_purifier` | not `has_purifier` |
| `DUCTED_KITCHEN_CHIMNEY` | `VENT_SOURCE` | `PURCHASE` | `vent_factor` | `kitchen_vent` < `DUCTED_CHIMNEY` |
| `SWITCH_BIOMASS_TO_LPG` | `COOKING_SWAP` | `PURCHASE` | `swap_fuel` | `cooking_fuel` ∈ {`BIOMASS`,`COAL`,`KEROSENE`} |

Adding an action is a data change plus one registered effect function plus reviewed copy in three locales. It is never a change to the planner.

### 7.9 Trace format

Every `plan_actions.trace` and every impact figure's trace is this shape, and it is what the UI's "How?" sheet renders directly:

```json
{
  "formula": "indoor_mass_balance_then_exposure_delta",
  "modelConfigVersion": "india-1.0.0",
  "inputs": [
    {"name": "outdoorPm25WindowMean", "unit": "µg/m³", "value": 252.2,
     "note": "mean of the hourly forecast series over 06:00–12:00"},
    {"name": "homeVolume", "unit": "m³", "value": 90.0},
    {"name": "cookingEmission", "unit": "µg/h", "value": 1500.0,
     "provenance": "model_config india-1.0.0 sources.COOKING.LPG"},
    {"name": "cookingMinutesInWindow", "unit": "min", "value": 60.0},
    {"name": "airExchangeRate", "unit": "h⁻¹", "value": 0.50,
     "provenance": "model_config india-1.0.0 envelope.TYPICAL.a"},
    {"name": "penetration", "unit": "dimensionless", "value": 0.80,
     "provenance": "model_config india-1.0.0 envelope.TYPICAL.P"},
    {"name": "deposition", "unit": "h⁻¹", "value": 0.20,
     "provenance": "model_config india-1.0.0 pm25.k"},
    {"name": "sealedAirExchangeRate", "unit": "h⁻¹", "value": 0.25,
     "provenance": "model_config india-1.0.0 envelope.TIGHT.a"},
    {"name": "sealedPenetration", "unit": "dimensionless", "value": 0.70,
     "provenance": "model_config india-1.0.0 envelope.TIGHT.P"}
  ],
  "steps": [
    {"label": "Baseline loss rate L = a + k + λp", "value": 0.70, "unit": "h⁻¹"},
    {"label": "Baseline infiltration term P·a·C_out/L", "value": 144.1, "unit": "µg/m³"},
    {"label": "Baseline cooking term, averaged over the window", "value": 3.9, "unit": "µg/m³"},
    {"label": "Baseline indoor mean", "value": 148.0, "unit": "µg/m³"},
    {"label": "Sealed loss rate L = a + k + λp", "value": 0.45, "unit": "h⁻¹"},
    {"label": "Sealed infiltration term", "value": 98.1, "unit": "µg/m³"},
    {"label": "Sealed cooking term, averaged over the window", "value": 5.6, "unit": "µg/m³"},
    {"label": "Sealed indoor mean", "value": 103.7, "unit": "µg/m³"},
    {"label": "Difference", "value": 44.3, "unit": "µg/m³"},
    {"label": "Hours affected", "value": 6.0, "unit": "h"}
  ],
  "notes": [
    "Each cooking term is the time-average of the source contribution over the window, including its decay after the stove is off, not the source divided by the number of hours."
  ],
  "result": {"name": "avoidedExposure", "value": 265.6, "unit": "µg/m³·h"},
  "caveats": ["Modelled from a single well-mixed zone.",
              "Sealing raises the cooking term, because the same smoke clears more slowly.",
              "Outdoor values beyond the current hour are forecast, not measured."]
}
```

## 8. LLM orchestration — `llm/`

### 8.1 The invariant

> The language model receives facts and returns prose. It never produces a quantity.

Three mechanisms enforce this rather than trusting an instruction:

1. **Structural.** The model is called with tool-use and a strict output schema containing only text fields. There is no numeric field in any output schema in §8.3 or §8.4, so a number can only arrive embedded in prose, where the validator can find it.
2. **Validation.** Every generated string passes the numeric and policy validator in §8.5 before persistence. Rejection is retried once with the failure named, then falls through to a deterministic template.
3. **Lint.** A custom Ruff rule forbids arithmetic operators, `round()`, `sum()`, `statistics`, and `numpy` imports anywhere under `llm/`. CI fails on violation.

### 8.2 Intent router — `llm/router.py`, `claude-haiku-5-5`

Runs on every Ask submission before retrieval. Cheap, single-turn, strict output.

```python
class IntentResult(BaseModel):
    intent: IntentClass           # the intent_class enum from §5.1
    confidence: Literal["HIGH", "MEDIUM", "LOW"]
    rewritten_query: str          # retrieval-optimised, <= 200 chars
    mentions_household: bool      # needs the facts object injected
```

| `intent` | Routing |
|---|---|
| `AIR_QUALITY` | Retrieve + answer with facts object |
| `INDOOR` | Retrieve + answer with facts object and home profile |
| `PRODUCT` | Retrieve + answer; brand names blocked by validator; CADR specification allowed |
| `BEHAVIOUR` | Retrieve + answer with facts object |
| `POLICY` | Retrieve only; no facts object |
| `MEDICAL` | **No generation.** Deterministic refusal plus escalation copy plus the household's current air facts. Persisted with `blocked_reason = 'MEDICAL'` |
| `OFF_TOPIC` | Deterministic redirect, no model call |
| `UNSAFE` | Deterministic refusal, no model call, `audit_log` entry |

A `MEDICAL` classification is sticky for the session: once set, subsequent messages in the same `ask_sessions` row are re-checked against the medical lexicon directly and a rephrased clinical question cannot route around the refusal. This is the mechanism behind `prd.md` US-7.2.

### 8.3 Plan narrator — `llm/narrator.py`, `claude-sonnet-5-5`

Called once per plan generation. Input is the facts object, which is also persisted verbatim in `plans.facts` so that any narrative can be audited against exactly what the model saw.

**Facts object schema** (camelCase, language-neutral, no free text):

```json
{
  "factsVersion": "1.0.0",
  "modelConfigVersion": "india-1.0.0",
  "planDate": "2026-11-19",
  "language": "hi",
  "location": {"areaLabel": "Gomti Nagar", "city": "Lucknow", "timezone": "Asia/Kolkata"},
  "current": {"pm25": 248.0, "aqi": 382, "band": "VERY_POOR", "dominant": "PM2_5",
              "confidence": "HIGH", "observedAt": "2026-11-19T06:00:00Z"},
  "today": {"peakPm25": 291.0, "peakBand": "SEVERE", "peakLocalTime": "08:00",
            "minPm25": 82.0, "minLocalTime": "14:00",
            "meanPm25": 186.0,
            "bandHours": {"SEVERE": 3, "VERY_POOR": 9, "POOR": 8, "MODERATE": 4}},
  "indoor": {"modelledMeanPm25": 107.4, "ratioOfOutdoor": 0.58, "basis": "MODELLED"},
  "ventilationWindows": [{"startLocal": "14:00", "endLocal": "15:30", "outdoorPm25": 82.0}],
  "members": [{"memberId": "…", "nickname": "Arjun", "ageBand": "CHILD_5_12",
               "sensitivityFlags": ["ASTHMA"], "baselineExposureUgm3h": 3980.0,
               "baselineInhaledUg": 2640.0}],
  "actions": [{"actionCode": "SEAL_WINDOWS_PEAK", "kind": "SEAL", "costTier": "FREE",
               "costInr": 0, "windowStartLocal": "06:00", "windowEndLocal": "12:00",
               "avoidedExposureUgm3h": 265.6, "avoidedInhaledUg": 239.1,
               "avoidedGco2e": 0.0, "memberId": null, "rank": 1}],
  "combined": {"avoidedExposureUgm3h": 1284.0, "avoidedInhaledUg": 862.0,
               "avoidedGco2e": 1360.0, "protectedHours": 9.5},
  "thresholds": {"whoPm25Daily": 15.0, "cpcbPm25Daily": 60.0, "cigEqBasis": 22.0}
}
```

**Output schema** — text only, no numeric fields:

```python
class PlanNarration(BaseModel):
    verdict: str = Field(max_length=160)
    action_titles: dict[str, str]     # action_code -> title, each <= 60 chars
    action_bodies: dict[str, str]     # action_code -> body,  each <= 220 chars
```

**Prompt structure** (`llm/prompts/narrator.v1.md`, versioned; `plans.prompt_version` records which ran):

- Role: an air-quality communicator writing for a household in the named city.
- Hard rules, restated in the prompt and enforced by the validator: use only numerals present in the facts object; name no brand; no clinical claim; one call to action in the verdict; write in `language`.
- Style: plain, specific, no alarm adjectives, second person, present tense.
- The facts object as JSON.
- Three few-shot examples per language, each pinned to a facts fixture in `tests/golden/`.

**Fallback template** (`domain/templates/verdict.{locale}.j2`) renders from the same facts object with Jinja2 and zero model involvement. It is plainer but always correct, and it is what ships when validation fails twice. `plans.verdict_source` records which path produced the text, and the `plan_narration_fallback` analytics event fires.

### 8.4 Grounded Q&A — `llm/advisor.py`, `claude-sonnet-5-5`

Pipeline, in order:

1. **Route** (§8.2). Non-generating intents exit here.
2. **Retrieve.** Hybrid search over `knowledge_chunks`:
   - Dense: `bge-m3` embedding of `rewritten_query`, cosine over the HNSW index, top 24.
   - Lexical: `pg_trgm` similarity on `content`, top 24.
   - Fuse with Reciprocal Rank Fusion, `k = 60`, take the top 8.
   - **Relevance floor:** discard any chunk with cosine similarity < 0.32. If fewer than 2 chunks survive, the answer is the deterministic "not covered" response from `prd.md` US-7.1 and no model call is made.
3. **Assemble context.** The surviving chunks with their document titles and publishers, plus the facts object when `mentions_household` is true.
4. **Generate** with streaming on, output schema below.
5. **Validate** (§8.5). The allowed numeral set for Q&A is the union of the facts object's numerals and every numeral appearing in the retrieved chunks.
6. **Persist** to `ask_messages` with `citations`, token counts, and `intent`.

```python
class AdvisorAnswer(BaseModel):
    answer_markdown: str = Field(max_length=1400)
    citations: list[Citation]          # >= 1 when any factual claim is present
    used_household_facts: list[str]    # facts object key paths actually relied on

class Citation(BaseModel):
    chunk_id: UUID
    document_title: str
    publisher: str
    quoted_span: str = Field(max_length=180)   # must be a verbatim substring of the chunk
```

`quoted_span` is verified as a literal substring of the referenced chunk. A citation that fails this check is dropped, and if dropping leaves a factual claim uncited the answer is rejected under §8.5.

**Corpus** (`corpus/`, manifest-driven, `corpus_version` stamped on every chunk):

| Source | Use |
|---|---|
| WHO Global Air Quality Guidelines 2021 | Guideline values, health framing at population level |
| CPCB National Ambient Air Quality Standards and AQI methodology | Indian standards, AQI bands and breakpoints |
| US EPA indoor air quality guidance | Indoor sources, ventilation, filtration |
| US EPA residential air cleaner guidance | CADR sizing, filter classes |
| NIOSH / CDC respirator guidance | Mask classes and fit |
| CAQM Graded Response Action Plan (GRAP) schedules for Delhi-NCR | What restrictions apply at which stage |
| ISO 16890 and HEPA filter classification summaries | Filter specification vocabulary |

Each document's licence is recorded in `knowledge_documents.licence` and surfaced on the citation chip. Only documents whose licence permits quotation are chunked with `quoted_span` enabled; for the rest, the citation carries the title and publisher with no span.

### 8.5 The validator — `llm/validator.py`

Runs on every generated string before it is persisted or streamed to completion.

**Numeric check.**

```
1  Extract every numeral token from the generated text:
       regex  (?<![\w.])\d{1,3}(?:,\d{3})*(?:\.\d+)?|\d+(?:\.\d+)?
2  Normalise: strip thousands separators, parse as Decimal.
3  Build the allowed set A = every numeric leaf in the facts object (recursively),
   plus, for Q&A, every numeral in the retrieved chunks,
   plus clock values already present in the facts object's time fields.
4  A generated value g is accepted if any a in A satisfies:
       g == a
       OR abs(g - a) / max(abs(a), 1) <= 0.02          # 2% tolerance for rounding
       OR g == round_to_significant(a, 1) or round_to_significant(a, 2)
5  Any unaccepted numeral fails the check, naming the offending token.
```

**Policy checks**, each a hard fail:

| Check | Implementation |
|---|---|
| Brand or retailer name | Denylist in `llm/policy/brands.txt`, case-insensitive whole-word match |
| Clinical term | Lexicon in `llm/policy/clinical.txt` — diagnoses, drug classes, dosage patterns |
| Individual causal health claim | Pattern set matching a member nickname or pronoun within 8 tokens of a harm verb |
| Unlisted pollutant or standard | Any pollutant or standard name not present in the facts object or a cited chunk |
| Length | Per-field caps from the output schema and `prd.md` §6.2 |
| Call-to-action count | At most one imperative clause in `verdict`, detected by a verb-initial clause pattern |
| Citation coverage (Q&A only) | Every sentence containing a numeral or a factual assertion maps to a citation or a `used_household_facts` entry |

**On failure:** retry once, appending the specific failure (offending token or matched term) to the prompt as a correction turn. On a second failure, fall through to the deterministic path — template verdict for narration, "not covered" response for Q&A — record the event, and surface nothing to the user. The user never sees a validation error.

**Adversarial test set** (`tests/adversarial/`): at least 60 fixtures in which a stubbed model returns prose containing a fabricated number, a brand name, a clinical claim, a two-CTA verdict, an uncited claim, or a mis-transcribed citation span. The suite asserts a 100% rejection rate. This is release gate 4 in `scope.md` §11.

### 8.6 Budget controls

| Control | Value | Enforcement |
|---|---|---|
| Narration frequency | Once per household per day, plus once per material change, plus once per language switch | `plans` uniqueness on `(household_id, plan_date, revision)`; language-keyed cache |
| Q&A daily limit | `ASK_DAILY_LIMIT_PER_HOUSEHOLD` (15) | Redis counter, resets at local midnight; the client shows remaining quota per `prd.md` §5.4 |
| Daily token cap | `LLM_DAILY_TOKEN_CAP_PER_HOUSEHOLD` (12,000) | Pre-call estimate plus post-call accounting in Redis; on breach, Q&A degrades to retrieval-only extractive answers with citations and no generation |
| Router model | `claude-haiku-5-5` | Classification never runs on the primary model |
| Context size | Top 8 chunks, facts object only when `mentions_household` | Retrieval caps in §8.4 |
| Prompt caching | Static prompt prefix and corpus system block marked for caching | Reduces repeat input cost across a session |

Per-household monthly cost is monitored as a first-class metric against the ₹6.00 target in `prd.md` N10, with an alert at 80%.

### 8.7 Prompt versioning and evaluation

Prompts are files under `llm/prompts/` named `<role>.v<N>.md`. The active version per role is a `model_config.params.prompts` entry, so a prompt change is a config activation, not a deploy. Every `plans` row and `ask_messages` row records the prompt version that produced it.

An eval suite (`tests/golden/`) pins 40 facts-object fixtures spanning all six bands, all three languages, sensitive and non-sensitive members, and the degraded states. Each fixture asserts: validator pass, length compliance, correct language, the expected action set surfaced, and a stable snapshot of the template fallback. A prompt version cannot be activated until it passes the full suite.

## 9. API contracts

Base path `/v1`. All requests carry `Authorization: Bearer <Firebase ID token>`. All responses are JSON with `lowerCamelCase` keys. All timestamps are ISO 8601 UTC with a `Z` suffix.

### 9.1 Conventions

| Concern | Rule |
|---|---|
| Errors | `{"error": {"code": "SNAKE_CASE_CODE", "message": "human readable", "details": {}}}` with HTTP 4xx/5xx. Codes are a closed enum documented in the OpenAPI schema. |
| Idempotency | Every mutating endpoint accepts `Idempotency-Key`; `action_completions` additionally carries `clientMutationId` for offline replay. |
| Pagination | Cursor-based: `?cursor=<opaque>&limit=<=100`; response carries `nextCursor` or `null`. |
| Caching | `GET` responses carry `ETag` and `Cache-Control: private, max-age=<ttl>`; the client sends `If-None-Match`. |
| Rate limits | Per household: 120 req/min general, 15/day on `/ask`. `429` carries `Retry-After`. |
| Versioning | Additive changes only within `/v1`. A breaking change opens `/v2`. |
| Scope | Every household-scoped route is authorised against the token's user; a mismatch returns `404`, never `403`, so household existence is not disclosed. |

### 9.2 Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/health` | Liveness; no auth |
| `GET` | `/v1/health/ready` | Readiness: DB, Redis, active `model_config`; no auth |
| `POST` | `/v1/auth/session` | Exchange a Firebase ID token for a `users` row; creates anonymous users |
| `GET` | `/v1/locations/search?q=&limit=` | Area search for the no-permission path |
| `POST` | `/v1/locations/resolve` | `{latitude, longitude, precision}` → `locations` row at the consented precision |
| `GET` | `/v1/air-quality/current?locationId=` | Fused current reading with `sources`, `confidence` |
| `GET` | `/v1/air-quality/hourly?locationId=&hours=72` | Hourly observed-plus-forecast series |
| `GET` | `/v1/air-quality/daily?locationId=&days=5` | Daily outlook |
| `GET` | `/v1/air-quality/stations?locationId=` | Contributing stations with licence strings |
| `GET`/`PATCH` | `/v1/households/me` | Household profile |
| `GET`/`POST` | `/v1/households/me/members` | Members |
| `PATCH`/`DELETE` | `/v1/households/me/members/{memberId}` | |
| `GET`/`POST` | `/v1/households/me/members/{memberId}/routine-blocks` | Routine blocks; `409 ROUTINE_BLOCK_OVERLAP` on overlap |
| `PATCH`/`DELETE` | `/v1/households/me/routine-blocks/{blockId}` | |
| `PUT` | `/v1/households/me/indoor-sources` | Full replace of source habits |
| `POST` | `/v1/households/me/onboarding/complete` | Marks complete; triggers first plan synchronously |
| `GET` | `/v1/households/me/daily-plan?date=` | Latest revision for the date |
| `POST` | `/v1/households/me/daily-plan/regenerate` | Manual regeneration; rate-limited to 3/day |
| `POST` | `/v1/plan-actions/{planActionId}/complete` | Body `{clientMutationId}`; idempotent |
| `DELETE` | `/v1/plan-actions/{planActionId}/complete` | Un-complete |
| `GET` | `/v1/households/me/toolkit` | Ranked indoor interventions, grouped by tier |
| `POST` | `/v1/households/me/purifier-sizing` | `{lengthM, widthM, heightM}` → volume, required CADR, modelled outcomes, DIY alternative |
| `GET` | `/v1/households/me/swaps` | Commute, cooking, waste swaps with both effects |
| `GET` | `/v1/households/me/impact?period=week\|month` | Protected Hours and all impact figures |
| `GET` | `/v1/traces/{traceRef}` | The §7.9 trace for any figure |
| `POST` | `/v1/ask` | SSE stream; body `{sessionId?, question, language}` |
| `GET` | `/v1/ask/sessions/{sessionId}` | Message history |
| `POST` | `/v1/circles` · `POST /v1/circles/join` | Create / join by invite code |
| `GET` | `/v1/circles/{circleId}` | Aggregate circle card; never per-member figures |
| `POST` | `/v1/circles/{circleId}/share-card` | Returns a 1080×1080 PNG URL plus the text block |
| `POST` | `/v1/reports` | File a local-source report |
| `GET` | `/v1/circles/{circleId}/complaint-draft` | Draft if the §4.9 threshold is met, else `404` |
| `POST`/`DELETE` | `/v1/devices` | Register / deregister FCM token |
| `PATCH` | `/v1/devices/{deviceId}/preferences` | Quiet hours, enabled alert types |
| `GET` | `/v1/privacy/consents` · `POST /v1/privacy/consents` | Consent ledger |
| `POST` | `/v1/privacy/export` | Queues a JSON export; `202` |
| `POST` | `/v1/privacy/erase` | Queues erasure; `202`; confirmation within 7 days |
| `GET` | `/v1/meta/attributions` | CAMS, Open-Meteo, OpenAQ providers and licences |
| `GET` | `/v1/meta/model-version` | Active `model_config` and `corpus_version` |

### 9.3 Representative payloads

`GET /v1/air-quality/current?locationId=…`

```json
{
  "locationId": "9f1c…", "areaLabel": "Gomti Nagar", "city": "Lucknow",
  "observedAt": "2026-11-19T06:00:00Z", "ageMinutes": 6,
  "pm25": 248.0, "pm10": 392.0, "no2": 64.0, "o3": 22.0, "so2": 18.0, "co": 1.9,
  "aqi": 382, "band": "VERY_POOR", "dominant": "PM2_5",
  "aqiBasis": {"pm2524h": 226.4, "pm1024h": 341.0, "no224h": 58.0,
               "so224h": 16.0, "o38h": 26.0, "co8h": 1.7, "hoursAvailable": 24},
  "confidence": "HIGH", "confidenceReason": "TWO_STATIONS_WITHIN_10KM",
  "stationCount": 2, "nearestStationKm": 3.4, "alpha": 0.864,
  "stale": false,
  "sources": [
    {"kind": "STATION", "id": 2178, "name": "Lalbagh, Lucknow - CPCB",
     "distanceKm": 3.4, "observedAt": "2026-11-19T06:00:00Z",
     "licence": "CC BY 4.0", "provider": "CPCB"},
    {"kind": "MODEL", "id": "cams-global", "distanceKm": null,
     "observedAt": "2026-11-19T06:00:00Z",
     "licence": "CC BY 4.0", "provider": "Copernicus Atmosphere Monitoring Service via Open-Meteo"}
  ],
  "thresholds": {"whoPm25Daily": 15.0, "cpcbPm25Daily": 60.0},
  "modelConfigVersion": "india-1.0.0"
}
```

`GET /v1/households/me/daily-plan?date=2026-11-19`

```json
{
  "planId": "7c2a…", "planDate": "2026-11-19", "revision": 2,
  "language": "hi", "generatedAt": "2026-11-19T00:05:12Z",
  "verdict": "…", "verdictSource": "LLM",
  "peakPm25": 291.0, "peakBand": "SEVERE",
  "changeSummary": "Ventilation window moved to 14:00",
  "actions": [
    {"planActionId": "a1…", "actionCode": "SEAL_WINDOWS_PEAK", "kind": "SEAL",
     "costTier": "FREE", "costInr": 0,
     "windowStartLocal": "06:00", "windowEndLocal": "12:00",
     "title": "…", "body": "…",
     "avoidedExposureUgm3h": 265.6, "avoidedInhaledUg": 239.1, "avoidedGco2e": 0.0,
     "memberId": null, "memberNickname": null, "rank": 1,
     "completed": false, "traceRef": "tr_a1…"}
  ],
  "combined": {"avoidedExposureUgm3h": 1284.0, "avoidedInhaledUg": 862.0,
               "avoidedGco2e": 1360.0, "protectedHours": 9.5},
  "ventilationWindows": [{"startLocal": "14:00", "endLocal": "15:30", "outdoorPm25": 82.0}],
  "modelConfigVersion": "india-1.0.0", "promptVersion": "narrator.v1"
}
```

`POST /v1/households/me/purifier-sizing`

Request:

```json
{"lengthM": 3.5, "widthM": 3.4, "heightM": 2.7}
```

Response:

```json
{
  "roomVolumeM3": 32.13,
  "targetAirChangesPerHour": 5,
  "requiredCadrM3h": 161,
  "modelledIndoorPm25": {"withoutPurifier": 141.7, "withRequiredCadr": 17.4},
  "diyAlternative": {"description": "Box fan with a MERV 13 filter",
                     "cadrM3h": 300, "costInr": 1800, "modelledIndoorPm25": 9.9},
  "assumesBedroomDoorClosed": true,
  "traceRef": "tr_ps_…", "modelConfigVersion": "india-1.0.0"
}
```

`POST /v1/ask` streams Server-Sent Events: `event: token` frames carrying `{"delta": "…"}`, then one `event: citations` frame, then `event: done` with `{"messageId", "intent", "remainingQuota"}`. On a blocked intent the stream emits a single `event: blocked` frame with the deterministic body and `blockedReason`.

## 10. Security and privacy

### 10.1 Authentication and authorisation

Firebase ID tokens are verified against the project's JWKS on every request, cached 10 minutes. `deps.get_current_household()` resolves the token's `firebase_uid` to `users` and then to the single `households` row, and every household-scoped query is filtered by that id in the query itself — authorisation is never a post-fetch check. Circle reads are authorised through `circle_members`.

### 10.2 Location minimisation

| Precision | Stored | When |
|---|---|---|
| `AREA` (default) | `geohash5` and the cell centroid only | Always |
| `EXACT` | `locations.exact_point` | Only with an `EXACT`-detail `LOCATION` consent row |

The client never transmits raw coordinates at higher precision than the consented level — truncation happens on the device before the request. Reports store the `geohash7` centroid, not the reporter's point, so a report cannot be traced to a flat.

### 10.3 Personal data inventory and DPDP Act 2023 obligations

| Category | Fields | Basis | Retention |
|---|---|---|---|
| Identity | `users.phone_e164`, `firebase_uid` | Consent at sign-up | Until erasure |
| Location | `locations` rows linked to a household | Consent, purpose-limited to plan generation | Until erasure |
| Household composition | `household_members` age band, relationship, optional nickname | Consent | Until erasure |
| Health sensitivity | `household_members.sensitivity_flags` | Separate explicit `HEALTH_SENSITIVITY` consent; refusable without losing the product | Until erasure or withdrawal |
| Routine | `routine_blocks` | Consent | Until erasure |
| Behavioural | `action_completions` | Consent | 13 months |
| Q&A content | `ask_messages.body` | Consent | 90 days, then deleted |
| Derived | `exposure_estimates` | Derived from the above | 13 months |

**Children.** No date of birth, legal name, school, or photograph is collected or storable — the schema has no column for any of them (§5.2). A minor is represented as an age band plus optional nickname plus sensitivity flags, under the account of the adult who created the household.

**Withdrawal of health consent** deletes `sensitivity_flags` and re-runs plan generation without them, within the request. Thresholds revert to the unmodified values.

**Erasure** (`POST /v1/privacy/erase`) sets `users.erasure_requested_at`, and the `purge_expired` job hard-deletes the `users` row, cascading every dependent table, within 7 days. `audit_log` retains only the action and a salted hash of the user id, no payload. Circle membership is removed; circle aggregates are recomputed so no deleted household's contribution remains.

**Export** produces a single JSON document covering every category above plus the `model_config_version` for each derived figure, so an exported impact number is reproducible.

### 10.4 Data sent to the LLM provider

The facts object contains no `firebase_uid`, no phone number, no household id, no exact coordinates, and no stored member id — member references in the narration path are replaced by opaque per-request ordinals, and the nickname is included only when the household has one and only for the members the plan addresses. Q&A question bodies are sent as written, which is disclosed in the privacy notice. Zero-retention handling is requested at the API level where available, and `ANTHROPIC_API_KEY` is scoped to this service.

### 10.5 Application security

| Control | Implementation |
|---|---|
| Transport | TLS 1.2+ only; HSTS; certificate pinning in the Flutter client for the API host |
| Input validation | Pydantic on every request body; strict types; no free-form SQL anywhere, parameterised queries only |
| Output encoding | JSON only; the client treats all server text as data and never renders HTML from it |
| Prompt injection | Retrieved chunks and user questions are placed in clearly delimited, untrusted context blocks; the validator's policy checks run on the output regardless of what the input asked for; no tool in the Q&A path can mutate state |
| Secrets | Fly.io secrets; nothing in the repo; `.env.example` carries names and no values |
| Dependencies | `pip-audit` and `dart pub outdated` in CI; weekly Dependabot |
| Rate limiting | Redis token buckets per household and per IP |
| Audit | `audit_log` on every consent change, erasure, export, circle admin action, and `UNSAFE` classification |
| Headers | `X-Content-Type-Options`, `Referrer-Policy: no-referrer`, `Permissions-Policy` minimal |

## 11. Build in small steps

**Capability coverage.** Every capability in `scope.md` §4 lands in a named milestone, so nothing in the scope is orphaned and nothing in the build is unmotivated:

| Capability | Delivered in |
|---|---|
| C1 Air quality now and ahead | M2 (data), M5 (surface) |
| C2 Household model | M1 (schema), M4 (API), M5 (onboarding) |
| C3 Daily Clean-Air Plan | M3 (engine), M4 (planner), M5 (surface), M6 (narration) |
| C4 Indoor air toolkit | M3 (indoor model), M4 (toolkit and sizing), M5 (surface) |
| C5 Sustainable swaps | M3 (CO₂e), M4 (swaps endpoint), M8 (surface) |
| C6 Impact Explainer | M3 (impact arithmetic and traces), M4 (endpoint), M8 (Impact tab) |
| C7 Grounded Q&A | M7 |
| C8 Alerts | M8 |
| C9 Neighbourhood circles | M8 |
| C10 Bilingual, low-bandwidth delivery | M4 (template locales), M5 (ARB, offline), M6 (narration locales) |

Ten milestones. Each is independently shippable, has explicit entry and exit criteria, and ends in something demonstrable. No milestone depends on a later one. The ordering puts the deterministic engine before the LLM on purpose: the product is correct without the model, and the model only changes how it reads.

### M0 — Foundations (week 1, 3 days)

**Entry:** empty repository.
**Build:** monorepo scaffold per §1; `uv` project for the API; Flutter project; Docker images; `docker-compose` for local Postgres 16 with `postgis`, `vector`, `pg_trgm` and Redis 7; `config.py` with every §4.1 variable; `/v1/health` and `/v1/health/ready`; GitHub Actions running lint (Ruff), type-check (mypy strict on `domain/` and `llm/`), and tests; Flutter analyze and test; Fly.io apps created for api and worker; Neon and Upstash provisioned in Mumbai.
**Exit:** `GET /v1/health/ready` returns 200 from the deployed staging API; CI green on an empty test suite; `flutter run` shows a themed empty shell.

### M1 — Data model and configuration pack (week 1–2, 4 days)

**Entry:** M0 closed.
**Build:** every enum and table from §5.1–§5.2 as Alembic migrations; SQLAlchemy models; the `packages/config-india/` YAML for `model_config` (§7.2 parameters with provenance), `aqi_breakpoints` (§6.2), `emission_factors` (§7.2), and the 22-row `action_catalogue` (§7.8) with English copy; a data migration that loads the pack and activates `india-1.0.0`.
**Exit:** `alembic upgrade head` on a clean database produces the full schema; a test asserts exactly one active `model_config`; a test asserts every `action_catalogue.effect_model` resolves to a registered function; a test asserts every `aqi_breakpoints` band is contiguous with no gap or overlap per pollutant.

### M2 — Air quality ingestion and AQI (week 2, 4 days)

**Entry:** M1 closed.
**Build:** `providers/open_meteo_aq.py`, `open_meteo_weather.py`, `openaq.py` with the timeouts, retries, breakers, and the OpenAQ token bucket from §4.2; `domain/aqi.py` implementing §6.2 including the validity rule; `domain/fusion.py` implementing §6.3; worker jobs `refresh_stations` (daily), `fuse_grid_cells` (*/15 min), `refresh_forecasts` (hourly); Redis caching with a 15-minute TTL; `/v1/locations/*` and `/v1/air-quality/*`.
**Exit:** for the five seeded cities, `GET /v1/air-quality/current` returns a fused reading with a populated `sources` array and a correct `confidence`; AQI unit tests cover every band boundary for all six pollutants plus the clamp above 500 and the validity-rule fallback; a recorded-fixture test (`respx`) covers each upstream's success, timeout, 429, and malformed-payload path; a soak test confirms the OpenAQ call rate stays under 50/min with 2,000 synthetic households.

### M3 — Exposure engine (week 3, 5 days)

**Entry:** M2 closed.
**Build:** `domain/indoor.py` (§7.3 analytic step), `domain/exposure.py` (§7.4), `domain/ventilation.py` (§7.5), `domain/impact.py` (§7.6), and the registered effect functions behind every `action_catalogue.effect_model` (§7.8); the §7.9 trace emitter on every computation.
**Exit:** unit tests cover every microenvironment, every indoor source, every transit mode, every mask class, every age band, and every intervention; a stability test asserts the indoor step stays non-negative and bounded with `L` up to 40 h⁻¹; an analytic test asserts the step reproduces the closed-form steady state within 1e-9 for constant inputs; a conservation test asserts zero avoided exposure when no action is applied; golden-file regression fixtures are pinned to `india-1.0.0`; every public function in `domain/` has a trace and mypy-strict types.

### M4 — Plan generation, deterministic end to end (week 4, 4 days)

**Entry:** M3 closed.
**Build:** `domain/planner.py` (§7.7) including the combined-effect run and the noise floor; persistence to `plans` and `plan_actions`; the Jinja2 template verdict renderer in all three locales; worker jobs `generate_daily_plans` (05:30 per timezone) and `detect_material_change` (*/15 min); `/v1/households/*`, `/v1/plan-actions/*`, `/v1/households/me/toolkit`, `/v1/households/me/purifier-sizing`, `/v1/households/me/swaps`, `/v1/households/me/impact`, `/v1/traces/{traceRef}`.
**Exit:** a seeded household with a known forecast fixture produces a byte-stable plan across runs; the material-change detector fires on each of the three §7.7 thresholds and stays silent below all of them, with tests for each; `protected_hours` reconciles with the hourly series to within 1e-6; the template verdict passes the §8.5 validator in all three locales. **This milestone is the first fully usable product — correct, quantified, and with no LLM in the path.**

### M5 — Flutter client, Today and Home (week 5–6, 7 days)

**Entry:** M4 closed.
**Build:** theme tokens from `prd.md` §5.1–§5.2 in light and dark; Riverpod graph; Dio client with ETag and retry; Drift schema mirroring `aq_readings`, `plans`, `plan_actions`, `action_completions`; Today tab with the Now card and all eight states, the verdict, the action list with optimistic check and offline queue, the hourly strip with scrub, the impact strip; Home tab with the indoor estimate, toolkit, source checklist, and purifier sizing; the provenance sheet; the trace sheet; onboarding's seven screens with every §5.3 default and the skip path; ARB files for all three locales.
**Exit:** every acceptance criterion in `prd.md` §4.1–§4.4 passes as an integration test; airplane-mode test confirms the full plan renders from Drift and checks queue and sync; cold start to painted Now card ≤ 1,200 ms p95 on a 2 GB Android 11 device; 200% text scale shows no clipping on any screen; TalkBack reads the Now card as one utterance.

### M6 — LLM narration with guardrails (week 7, 4 days)

**Entry:** M4 and M5 closed. The product already works without this.
**Build:** `llm/client.py` with budget accounting and prompt caching; `llm/schemas.py`; `llm/narrator.py` and `narrator.v1.md` in three languages with golden few-shots; `llm/validator.py` implementing the complete §8.5 numeric and policy check set; the Ruff rule forbidding arithmetic under `llm/`; the fallback wiring and the `plan_narration_fallback` event.
**Exit:** the adversarial suite of ≥ 60 fixtures rejects 100%; the 40-fixture golden eval passes in all three languages; a chaos test forcing the model to emit a fabricated number confirms the user-visible output is the template with no error surfaced; measured narration cost per household per day projects under the ₹6.00 monthly target; the domain reviewer has signed off the narration copy in all three locales.

### M7 — Grounded Q&A (week 8, 5 days)

**Entry:** M6 closed.
**Build:** `corpus/` ingestion with the manifest, licences, and 600-token chunks at 100-token overlap; `bge-m3` embedding and the HNSW index; hybrid retrieval with RRF and the 0.32 relevance floor (§8.4); `llm/router.py` with the sticky `MEDICAL` path; `llm/advisor.py` with citation span verification; `POST /v1/ask` as SSE; the Ask tab with streaming, citation chips, household-fact chips, quota line, and the standing non-medical footer.
**Exit:** every criterion in `prd.md` §4.7 passes; a 120-question evaluation set scores ≥ 95% groundedness on manual review; every question in the medical set is refused, including eight rephrasings within one session; the "not covered" path triggers correctly when retrieval is thin; first token ≤ 1,500 ms p95.

### M8 — Alerts, circles, and community reporting (week 9, 5 days)

**Entry:** M7 closed.
**Build:** `/v1/devices/*` with quiet hours and per-type opt-out; worker jobs `send_alerts` (*/5 min) with dedupe keys, coalescing, the 3-per-day cap, and the `SEVERE_IMMINENT` quiet-hours override; `weekly_impact` (Sunday 19:00 local); the full alert matrix from `prd.md` §4.8; circles create, join, aggregate card, and share-card rendering; reports; `aggregate_reports` (hourly) and the complaint draft; the Impact tab in full.
**Exit:** every criterion in `prd.md` §4.6, §4.8, and §4.9 passes; a test asserts no household can receive more than 3 notifications in 24 hours; a test asserts the circle endpoint response contains no per-member figure under any query; the complaint draft contains no reporter identity; the share card renders at 1080×1080 in all three locales.

### M9 — Hardening and launch (week 10, 5 days)

**Entry:** M8 closed.
**Build:** privacy endpoints — consent ledger, export, erase — with the `purge_expired` job; the `rollup_exposure` job (daily 00:20); OpenTelemetry spans across fetch → fuse → compute → narrate as one trace per plan; Grafana dashboards and alerts on every guardrail in `scope.md` §7.3; Sentry on client and server; the staging load test at 2,000 synthetic households; the accessibility audit; the About screen with CAMS, Open-Meteo, and OpenAQ attributions and the active `model_config` version; Play Store listing and staged rollout.
**Exit:** every item in `scope.md` §11 holds; load test meets every target in `prd.md` §9; erasure and export verified end to end; accessibility audit passes; 10% staged rollout with crash-free sessions ≥ 99.5% over 72 hours.

## 12. Testing strategy

| Layer | Tool | Coverage requirement |
|---|---|---|
| `domain/` unit | pytest | 95% line and branch. Every formula, every parameter path, every enum value |
| `domain/` golden | pytest + pinned fixtures | Every change to `model_config` must either leave golden outputs unchanged or bump the version and update fixtures in the same commit |
| `domain/` property | Hypothesis | Invariants: indoor concentration is non-negative and bounded by `max(C_out, C_ss_with_sources)`; avoided exposure is zero when no action applies; exposure is monotonic in outdoor concentration; `protected_hours` ≤ 24 per member per day |
| Provider adapters | respx recorded fixtures | Success, timeout, 429, 5xx, malformed payload, and unit-conversion correctness for each upstream |
| API integration | pytest + httpx + a real Postgres in CI | Every endpoint: happy path, auth failure, cross-household access, validation failure, idempotent replay |
| LLM | stubbed client | Golden eval (40 fixtures) and adversarial suite (≥ 60 fixtures); no test calls the live API |
| Migrations | pytest | `upgrade head` then `downgrade base` on a clean database; data migration idempotency |
| Flutter widget | flutter_test | Every state of the Now card, action list, indoor card, impact card |
| Flutter integration | integration_test | Onboarding end to end; airplane-mode plan read and sync; language switch; scrub |
| Accessibility | flutter_test semantics + manual audit | Every criterion in `prd.md` §8 |
| Load | k6 against staging | 2,000 households, all guardrails in `scope.md` §7.3 |
| Security | pip-audit, dart pub outdated, manual injection suite | Zero high-severity findings; prompt-injection suite confirms the validator holds regardless of input |

## 13. Observability and operations

### 13.1 Tracing

One trace per plan generation, named `plan.generate`, with spans: `aq.fetch.openaq`, `aq.fetch.open_meteo`, `aq.fuse`, `exposure.counterfactual`, `exposure.candidates`, `plan.rank`, `llm.narrate`, `llm.validate`, `plan.persist`. Attributes on every trace: `household_id_hash`, `geohash5`, `model_config_version`, `prompt_version`, `verdict_source`. One trace per Ask request with `router`, `retrieve.dense`, `retrieve.lexical`, `retrieve.fuse`, `llm.generate`, `llm.validate`.

### 13.2 Metrics and alerts

| Metric | Alert threshold |
|---|---|
| `plan_generate_duration_ms` p95 | > 4,000 ms for 10 min |
| `aq_fusion_failure_ratio` | > 1% over 24 h |
| `openaq_requests_per_minute` | > 50 |
| `llm_validation_rejection_ratio` | > 5% over 1 h |
| `plan_narration_fallback_ratio` | > 10% over 1 h |
| `llm_cost_inr_per_household_month` | > ₹4.80 (80% of target) |
| `stale_reading_served_ratio` | > 2% over 1 h |
| `api_5xx_ratio` | > 0.5% over 5 min |
| `notifications_sent_per_household_day` | any household > 3 |
| `crash_free_sessions` | < 99.5% over 24 h |

### 13.3 Runbook

| Symptom | First action |
|---|---|
| OpenAQ returning 429 | Confirm the token bucket is active; reduce `fuse_grid_cells` to */30 min; readings degrade to `MEDIUM`/`LOW` confidence, which is a correct and visible degradation |
| Open-Meteo unavailable | Breaker opens; forecasts freeze at the last issue; ventilation windows stop being issued because forecast confidence drops below `HIGH`; no plan claims a window it cannot support |
| Anthropic unavailable | Breaker opens; every plan renders the template verdict; Q&A returns retrieval-only extractive answers with citations; no user-visible error |
| Validation rejection spike | Freeze the active prompt version, revert `model_config.params.prompts` to the previous version (a config activation, not a deploy), and inspect the rejected tokens recorded on the event |
| A parameter is found to be wrong | Author a new `model_config` version with the correction and its provenance, run the golden suite, activate in one transaction. Historical estimates stay reproducible against their own version; the product does not retroactively rewrite a household's past numbers |
| Plan generation backlog | Scale worker concurrency; `generate_daily_plans` is idempotent per `(household_id, plan_date, revision)` and safe to re-run |

### 13.4 Deployment

GitHub Actions on merge to `main`: lint → type-check → test → build images → `alembic upgrade head` against staging → deploy staging → smoke test → manual gate → `alembic upgrade head` against production → deploy production with a rolling release. Migrations are additive and backwards-compatible for one release, so the API and the database are never incompatible mid-deploy. Flutter ships via a Play Store staged rollout at 10% → 50% → 100%, gated on crash-free sessions at each step. Rollback is the previous image plus, if needed, reactivation of the previous `model_config` version; no migration is ever reversed in production.
