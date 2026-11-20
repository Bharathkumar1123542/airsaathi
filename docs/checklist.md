# AirSaathi — Build Checklist

**Document:** `checklist.md` — the ordered task list for the whole project
**Version:** 1.0.0
**Last updated:** 2026-11-19
**Legend:** `[ ]` not started · `[~]` in progress · `[x]` done · `[!]` blocked (name the blocker inline)

Work top to bottom. A milestone is closed only when its exit criteria in `spec.md` §11 pass. Nothing below a milestone's gate starts before that gate is green.

---

## M0 — Foundations · 3 days

- [ ] Create `Ibtidaa-Tech/airsaathi` with the `spec.md` §1 directory tree
- [ ] `uv` project for `apps/api`; FastAPI app factory, routers package, `config.py` with all `spec.md` §4.1 variables (fail-fast on absent)
- [ ] Flutter project in `apps/app`; Riverpod, Dio, Drift, `firebase_core` wired
- [ ] `docker-compose`: Postgres 16 + `postgis` + `vector` + `pg_trgm`, Redis 7
- [ ] `/v1/health` and `/v1/health/ready` (checks DB, Redis, active `model_config`)
- [ ] CI: Ruff, mypy strict on `domain/` and `llm/`, pytest, `flutter analyze`, `flutter test`
- [ ] Provision Fly.io api + worker, Neon Postgres, Upstash Redis — all Mumbai region
- [ ] Secrets into Fly.io; `.env.example` with names only
- [ ] **Gate:** staging `/v1/health/ready` returns 200; CI green; themed Flutter shell runs

## M1 — Data model and configuration pack · 4 days

- [ ] Alembic migration: every enum in `spec.md` §5.1
- [ ] Alembic migration: every table in `spec.md` §5.2 with all constraints and indexes
- [ ] SQLAlchemy 2.0 models; Pydantic API schemas with `to_camel` aliases
- [ ] `packages/config-india/model_config.v1.yaml` — all `spec.md` §7.2 parameters, each with provenance
- [ ] `packages/config-india/aqi_breakpoints.cpcb.yaml` — all six pollutants per `spec.md` §6.2
- [ ] `packages/config-india/emission_factors.yaml` — including `grid_gco2e_per_kwh`
- [ ] `packages/config-india/action_catalogue.yaml` — all 22 actions from `spec.md` §7.8, English copy
- [ ] Data migration loading the pack and activating `india-1.0.0`
- [ ] Tests: exactly one active `model_config`; every `effect_model` resolves; breakpoint bands contiguous with no gap or overlap
- [ ] **Gate:** `alembic upgrade head` then `downgrade base` clean; all three tests pass

## M2 — Air quality ingestion and AQI · 4 days

- [ ] `providers/open_meteo_aq.py` — keyless, 5-day hourly, CO converted µg/m³ → mg/m³
- [ ] `providers/open_meteo_weather.py` — wind, temperature, boundary-layer height
- [ ] `providers/openaq.py` — `X-API-Key`, three-step resource walk, 50 req/min token bucket
- [ ] Timeouts, retries, circuit breakers for all three per `spec.md` §4.2
- [ ] `domain/aqi.py` — sub-index interpolation, overall AQI, validity rule, 500 clamp, band mapping
- [ ] `domain/fusion.py` — IDW station term, model blend, `alpha`, confidence rules, `sources` array
- [ ] Worker jobs: `refresh_stations` (daily), `fuse_grid_cells` (*/15 min), `refresh_forecasts` (hourly)
- [ ] Redis caching, 15-minute TTL; active-cell resolution from households and circles
- [ ] `/v1/locations/search`, `/v1/locations/resolve`, `/v1/air-quality/{current,hourly,daily,stations}`
- [ ] Tests: every band boundary × six pollutants; validity-rule fallback; respx fixtures for success / timeout / 429 / malformed per upstream; unit-conversion assertions
- [ ] Soak test: OpenAQ stays under 50 req/min at 2,000 synthetic households
- [ ] **Gate:** all five seeded cities return a fused reading with populated `sources` and correct `confidence`

## M3 — Exposure engine · 5 days

- [ ] `domain/indoor.py` — analytic mass-balance step (`C_ss` + exponential decay), window and seal states, bedroom zoning
- [ ] `domain/exposure.py` — microenvironment concentration, transit multipliers, mask FFE, `IR` by activity × age, exposure and inhaled dose
- [ ] `domain/ventilation.py` — all five window conditions, merging, 30-minute minimum, two-per-day cap, withdrawal rule
- [ ] `domain/impact.py` — avoided exposure, avoided inhaled mass, Protected Hours, cigarette equivalent, avoided CO₂e
- [ ] Registered effect functions for all 22 catalogue actions
- [ ] `spec.md` §7.9 trace emitter on every computation
- [ ] Unit tests: every microenvironment, indoor source, transit mode, mask class, age band, sensitivity, intervention
- [ ] Stability test: non-negative and bounded at `L` up to 40 h⁻¹
- [ ] Analytic test: step reproduces closed-form steady state within 1e-9
- [ ] Conservation test: zero avoided exposure with no action applied
- [ ] Hypothesis property tests per `spec.md` §12
- [ ] Golden fixtures pinned to `india-1.0.0`
- [ ] **Gate:** 95% line and branch coverage on `domain/`; mypy strict clean

## M4 — Plan generation, deterministic end to end · 4 days

- [ ] `domain/planner.py` — counterfactual run, per-action re-run, 25 µg/m³·h noise floor, ranking, tier ordering, member rank boost, 7-action cap with at most 2 `PURCHASE`
- [ ] Combined-effect run; store `combined` separately from the per-action figures
- [ ] Persist `plans` + `plan_actions` with the full trace and the facts object
- [ ] Jinja2 template verdict renderer, all three locales
- [ ] Worker jobs: `generate_daily_plans` (05:30 per timezone), `detect_material_change` (*/15 min)
- [ ] `/v1/households/me` (+ members, routine-blocks with overlap rejection, indoor-sources, onboarding/complete)
- [ ] `/v1/households/me/daily-plan` (+ regenerate, 3/day), `/v1/plan-actions/{id}/complete` (idempotent)
- [ ] `/v1/households/me/{toolkit,purifier-sizing,swaps,impact}`, `/v1/traces/{traceRef}`
- [ ] Tests: byte-stable plan across runs; material change fires on each of the three thresholds and stays silent below all; `protected_hours` reconciles to 1e-6; template verdict passes the validator in all locales
- [ ] **Gate:** a seeded household gets a complete, correct, quantified plan with no LLM in the path

## M5 — Flutter client: Today and Home · 7 days

- [ ] Theme tokens from `prd.md` §5.1–§5.2, light and dark; verify every band fill/text pair at 4.5:1
- [ ] Riverpod graph; Dio with ETag, retry, certificate pinning
- [ ] Drift schema mirroring `aq_readings`, `plans`, `plan_actions`, `action_completions`; offline mutation queue with `clientMutationId`
- [ ] Onboarding: 7 screens, ≤ 5 options each, skip on every screen, all `spec.md` §5.3 defaults, text-location fallback
- [ ] Today: Now card with all eight states; verdict; action list with optimistic check; hourly strip with scrub and return; impact strip; "Plan updated" chip
- [ ] Home: indoor estimate card (point value and range forms), toolkit grouped by tier, source checklist, purifier sizing tool
- [ ] Provenance sheet; trace sheet rendering `spec.md` §7.9 directly
- [ ] ARB files `app_en`, `app_hi`, `app_mr`; `i18n_missing_key` event; English fallback never shows a raw key
- [ ] Analytics events from `prd.md` §10
- [ ] Tests: every `prd.md` §4.1–§4.4 criterion as an integration test; airplane-mode render and sync; 200% text scale; TalkBack on the Now card
- [ ] **Gate:** cold start to painted Now card ≤ 1,200 ms p95 on a 2 GB Android 11 device

## M6 — LLM narration with guardrails · 4 days

- [ ] `llm/client.py` — Anthropic SDK, budget accounting, prompt caching, retries, breaker
- [ ] `llm/schemas.py` — text-only output contracts; no numeric field anywhere
- [ ] `llm/narrator.py` + `prompts/narrator.v1.md`; golden few-shots per language
- [ ] `llm/validator.py` — numeral extraction, allowed set with 2% and significant-figure tolerance, all seven policy checks, single retry with named failure, template fallthrough
- [ ] Ruff rule forbidding arithmetic, `round`, `sum`, `statistics`, `numpy` under `llm/`; CI enforces
- [ ] `plan_narration_fallback` event; `verdict_source` persisted
- [ ] Adversarial suite, ≥ 60 fixtures: fabricated number, brand, clinical claim, two CTAs, uncited claim, bad citation span
- [ ] 40-fixture golden eval across six bands × three languages × sensitive and non-sensitive members
- [ ] Domain reviewer signs off narration copy in all three locales
- [ ] **Gate:** 100% adversarial rejection; golden eval passes; projected cost under ₹6.00 per household per month

## M7 — Grounded Q&A · 5 days

- [ ] `corpus/` manifest with every source in `spec.md` §8.4 and its licence; quotation permitted flag per document
- [ ] `rag/ingest.py` — 600-token chunks, 100-token overlap, heading paths
- [ ] `rag/embed.py` — `bge-m3`, 1024-dim, normalised; HNSW index built
- [ ] `rag/retrieve.py` — dense top-24 + `pg_trgm` top-24, RRF `k=60`, top-8, 0.32 relevance floor, "not covered" path under 2 survivors
- [ ] `llm/router.py` — intent classes, `claude-haiku-5-5`, sticky `MEDICAL` with direct lexicon re-check
- [ ] `llm/advisor.py` — context assembly, citation span verification, `used_household_facts`
- [ ] `POST /v1/ask` as SSE (`token`, `citations`, `done`, `blocked`); `GET /v1/ask/sessions/{id}`
- [ ] Ask tab: streaming, citation chips, household-fact chips, quota line, standing non-medical footer
- [ ] Daily Q&A limit and token cap with extractive degradation
- [ ] Tests: every `prd.md` §4.7 criterion; 120-question groundedness set ≥ 95%; medical set refused including eight in-session rephrasings
- [ ] **Gate:** first token ≤ 1,500 ms p95; no ungrounded claim in the evaluation set

## M8 — Alerts, circles, reporting · 5 days

- [ ] `/v1/devices` register and deregister; `PATCH .../preferences` for quiet hours and alert types
- [ ] `send_alerts` (*/5 min): full `prd.md` §4.8 matrix, dedupe keys, 60-minute coalescing, 3-per-day cap, `SEVERE_IMMINENT` quiet-hours override with its label
- [ ] `WINDOW_WITHDRAWN` on invalidation of a not-yet-started window; never withdraw one in progress
- [ ] `weekly_impact` (Sunday 19:00 local)
- [ ] Circles: create with 8-character invite code, join, 150-member cap, aggregate card
- [ ] Share card: 1080×1080 PNG + ≤ 400-character text, all three locales
- [ ] Reports (geohash7 centroid only); `aggregate_reports` (hourly); complaint draft at ≥ 3 reports / 500 m / 7 days, editable, no reporter identity, copy-or-share only
- [ ] Impact tab in full with baselines stated and the cigarette-equivalent caveat adjacent
- [ ] Tests: 3-notification cap holds; circle responses expose no per-member figure under any query; complaint draft carries no identity
- [ ] **Gate:** every `prd.md` §4.6, §4.8, §4.9 criterion passes

## M9 — Hardening and launch · 5 days

- [ ] `/v1/privacy/consents` (get and post), `/v1/privacy/export`, `/v1/privacy/erase`
- [ ] `purge_expired` (daily 03:00): erasure within 7 days, cascade, salted-hash audit retention, circle aggregate recomputation
- [ ] Health-consent withdrawal path: flags deleted and plan re-run in-request
- [ ] `rollup_exposure` (daily 00:20) into `exposure_estimates`
- [ ] OpenTelemetry: `plan.generate` and `ask` traces with all `spec.md` §13.1 spans and attributes
- [ ] Grafana dashboards and alerts on every metric in `spec.md` §13.2; Sentry on client and server
- [ ] k6 load test at 2,000 synthetic households against every `prd.md` §9 target
- [ ] Accessibility audit against every `prd.md` §8 criterion
- [ ] About screen: CAMS + Open-Meteo attribution, OpenAQ attribution and per-provider licences, active `model_config` and `corpus_version`
- [ ] Privacy notice published; DPDP data inventory reviewed
- [ ] `/v1/meta/attributions` and `/v1/meta/model-version`
- [ ] Play Store listing; staged rollout 10% → 50% → 100% gated on crash-free sessions
- [ ] **Gate:** every item in `scope.md` §11 holds

---

## Standing tasks — run every milestone, not once

- [ ] Any new `model_config` value carries a provenance string and a golden-fixture update in the same commit
- [ ] Any new user-facing string exists in all three locales before the feature ships
- [ ] Any new `action_catalogue` row has a registered effect function, reviewed copy in three locales, and a unit test
- [ ] Any new generated-text surface is added to the validator's coverage and to the adversarial suite
- [ ] Any new endpoint has a cross-household access test asserting `404`, not `403`
- [ ] Coverage on `domain/` stays ≥ 95% line and branch
- [ ] `pip-audit` and `dart pub outdated` clean of high-severity findings

## Release gate — all four must be true to ship to beta

- [ ] Every acceptance criterion in `prd.md` §4 passes
- [ ] Every guardrail in `scope.md` §7.3 is met under the 2,000-household staging load
- [ ] The §8.5 validator rejects 100% of the adversarial set
- [ ] All three locales pass content review across the complete action catalogue
