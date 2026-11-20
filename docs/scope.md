# AirSaathi — Scope

**Project:** AirSaathi (Urban Air Quality Assistant)
**Document:** `scope.md` — what it is, who it is for, and what is being cut
**Version:** 1.0.0
**Status:** Approved for build
**Last updated:** 2026-11-19
**Owner:** Product + Architecture
**Audience:** Engineering, design, QA, data, and any agent implementing v1

## 0. Reading order

| # | Document | Answers |
|---|---|---|
| 1 | `scope.md` (this document) | What is it, who is it for, what are we *not* building |
| 2 | `prd.md` | What does it do, how does it look and feel |
| 3 | `spec.md` | How is it built, piece by piece |
| 4 | `checklist.md` | What do I do next, in order |

Rule of precedence: if two documents disagree, `spec.md` wins on implementation detail, `prd.md` wins on behaviour and copy, `scope.md` wins on whether something is in v1 at all.

## 1. One-paragraph summary

AirSaathi is a mobile-first assistant for households in cities with severe seasonal smog. It takes a household's location, home setup, and daily routine, and returns a dated, hour-by-hour plan: when to go out and when not to, when to open windows and when to seal them, which indoor sources to stop, which commute to swap, and which purchase is worth the money. Every recommendation carries a number the household can check — the micrograms of PM2.5 they avoid breathing, expressed in plain units — produced by a deterministic exposure engine, never by a language model. The language model's job is to turn that arithmetic into advice a parent can act on in thirty seconds, in their own language.

## 2. Problem statement

Three failures stack up during a smog season:

1. **Numbers without meaning.** A household can read "AQI 412, Severe" on five different apps and still not know whether the school run is safe, whether to open the window, or whether the ₹18,000 purifier is worth it.
2. **Advice without specificity.** General guidance ("stay indoors", "wear a mask") ignores that the household cooks on an unvented LPG stove, that the child walks 1.2 km along an arterial road at 07:40, and that indoor PM2.5 in a typically sealed flat tracks outdoor PM2.5 at roughly 60% with no intervention.
3. **Action without feedback.** A family that changes three habits has no way to see what those changes bought them, so the habits do not survive past week two.

AirSaathi addresses all three: it localises the data, personalises against a household model, and quantifies the delta.

### 2.1 Reference thresholds used throughout the product

| Standard | Pollutant | Averaging | Value |
|---|---|---|---|
| WHO Air Quality Guideline (2021) | PM2.5 | Annual mean | 5 µg/m³ |
| WHO Air Quality Guideline (2021) | PM2.5 | 24-hour mean | 15 µg/m³ |
| WHO Air Quality Guideline (2021) | PM10 | 24-hour mean | 45 µg/m³ |
| CPCB NAAQS (India) | PM2.5 | Annual mean | 40 µg/m³ |
| CPCB NAAQS (India) | PM2.5 | 24-hour mean | 60 µg/m³ |
| CPCB NAAQS (India) | PM10 | 24-hour mean | 100 µg/m³ |

The product displays the **CPCB AQI** scale as primary (six bands: Good, Satisfactory, Moderate, Poor, Very Poor, Severe) and the WHO 24-hour guideline as the health reference line on all charts. US AQI is computed and stored but is not shown in the Indian locale. Band boundaries and breakpoints are specified in `spec.md` §6.2.

## 3. Who it is for

### 3.1 Primary users

| Persona | One-line description | Core need |
|---|---|---|
| **Household Guardian** | The adult who makes the household's daily decisions — school run, windows, cooking, purchases. | A single daily answer they can act on without reading a chart. |
| **Sensitive Individual** | An adult or child in the household with asthma, COPD, pregnancy, or age-related sensitivity (under 5 or over 65). | Thresholds tuned to them, and early warning before conditions turn. |
| **Community Organiser** | A resident who runs a building WhatsApp group, RWA, or school parent group. | Shareable local guidance and a way to escalate a recurring local source. |

### 3.2 Anti-personas — people v1 is explicitly not built for

| Anti-persona | Why excluded from v1 |
|---|---|
| Air quality researchers | Need raw station time series, QA flags, and export; v1 shows modelled household-relevant values, not research-grade data. |
| Industrial compliance officers | Need regulatory reporting and emissions inventories; different data, different liability. |
| Clinicians managing patients | Would require medical device framing and clinical validation; v1 ships explicit non-medical disclaimers. |
| Municipal policy teams | Need city-scale aggregation and intervention modelling; the community layer in v1 stops at neighbourhood circles. |

### 3.3 Launch geography

v1 launches for **India**, with Delhi-NCR, Lucknow, Patna, Mumbai, and Pune as the seeded cities. The architecture is country-agnostic — AQI scale, standards, language set, and emission factors are all configuration — but only the India configuration pack ships in v1. Rationale: the seasonal smog problem, the CPCB station density, and the Hindi/English/Marathi language set are all concentrated there, and a single-country launch keeps the exposure-model defaults defensible.

## 4. What AirSaathi is — v1 capability list

Each capability is normative. If it is listed here, it ships in v1.

| ID | Capability | What the household gets |
|---|---|---|
| C1 | **Air quality now and ahead** | Current fused PM2.5/PM10/NO₂/O₃/SO₂/CO and CPCB AQI for the household's location, plus an hourly forecast to 72 hours and a 5-day daily outlook. |
| C2 | **Household model** | A stored profile: members (age band + sensitivity flags), home (floor, seal quality, room volumes, purifier, ventilation), cooking fuel and vent, heating, and each member's recurring routine blocks. |
| C3 | **Daily Clean-Air Plan** | A dated plan with time-windowed actions: outdoor windows ranked best-to-worst, ventilation windows, commute guidance per member, mask guidance, and indoor-source stops. Regenerates when the forecast materially changes. |
| C4 | **Indoor air toolkit** | A ranked, costed set of indoor interventions from the action catalogue — free (stop incense, vent the kitchen, close the window at 07:00), low-cost (DIY box-fan filter, door sealing), and purchase (HEPA unit sized to room volume). |
| C5 | **Sustainable swaps** | Commute, cooking, and waste-disposal alternatives with both an exposure effect and a CO₂e effect, so the household sees the personal and the collective number side by side. |
| C6 | **Impact Explainer** | For every action and for the household's day as a whole: PM2.5 exposure avoided (µg/m³·h), inhaled mass avoided (µg), cigarette-equivalent change, and CO₂e avoided — each with a one-tap "how was this computed" trace. |
| C7 | **Grounded Q&A** | Free-text questions answered from a curated, cited corpus (WHO, CPCB, US EPA indoor air, NIOSH respirator guidance, GRAP) plus the household's own live numbers. Refuses and redirects on medical questions. |
| C8 | **Alerts** | Push notifications on band deterioration, a good ventilation window opening, forecast spikes 12–24 hours ahead, and a weekly impact summary. Quiet hours and per-type opt-out. |
| C9 | **Neighbourhood circles** | Invite-code circles (building, street, school) sharing a location, aggregate circle actions, and a drafted civic complaint when members repeatedly report a local source such as open waste burning. |
| C10 | **Bilingual, low-bandwidth delivery** | English, Hindi, Marathi. The plan is cached and fully readable offline. Plan text is generated once per day per household, not per view. |

## 5. What is being cut from v1

Each cut names the reason and the condition that would bring it back. Nothing here is "maybe later" without a trigger.

| Cut | Reason | Revisit trigger |
|---|---|---|
| **Personal sensor hardware integration** (PurpleAir, Atmotube, AirGradient) | Adds device pairing, BLE, calibration drift handling, and firmware support to a v1 that has no revenue. The exposure engine already accepts a measured indoor concentration as an input, so this is additive, not structural. | 1,000 weekly active households, or a hardware partnership. |
| **In-app purchasing of purifiers or filters** | Payments, returns, GST invoicing, and liability for product claims. v1 states the required CADR for the room and the price band, and links out. | Post-launch, once recommendation accuracy is validated. |
| **Clinical / symptom tracking** | Symptom logs invite diagnostic interpretation. v1 carries sensitivity *flags* for threshold tuning, not symptom history. | Only with clinical review and a regulatory assessment. Not an engineering decision. |
| **Route-level routing ("cleanest path to school")** | Requires street-canyon-resolution dispersion that the 11 km CAMS grid and sparse CPCB stations cannot support. Claiming street-level precision would be the product's first lie. | Dense low-cost sensor coverage, or a hyperlocal model partnership. |
| **iOS launch** | Flutter targets both, but a second App Review track, TestFlight loop, and iOS push certificate triples release overhead for a market that is majority Android. The Flutter codebase stays iOS-clean. | Android retention ≥ 25% at week 4. |
| **Wearable / Wear OS surfaces** | Surface area without a distinct job to be done in v1. | Post-launch. |
| **Public user-generated feed** | Open posting means moderation, which means a trust-and-safety function. Circles are invite-only and capped to keep v1 moderatable by a single reviewer. | A staffed moderation process. |
| **Live LLM-generated numbers** | Permanently cut, not deferred. All quantities come from the deterministic engine in `spec.md` §7. The model may only restate numbers present in its input facts. | Never. This is an architectural invariant. |
| **Multi-household / landlord accounts** | One account owns one household in v1. | Enterprise or school-program demand. |
| **Historical data beyond 90 days** | Retention cost against thin v1 value. Daily rollups are kept for 13 months; hourly for 90 days. | Year-over-year comparison becomes a requested feature. |
| **Languages beyond en/hi/mr** | Each language is a full content-review pass, not a string file. | City expansion. |

## 6. Fixed technical decisions

These are settled. `spec.md` elaborates; it does not reopen them.

| Area | Decision | Rationale |
|---|---|---|
| Client | Flutter 3.27+ (Dart 3.6+), Android first, iOS-clean codebase | One codebase, strong offline story, mature local persistence. |
| State management | Riverpod 2.x | Compile-safe dependency graph; testable without widget tests. |
| Local store | Drift (SQLite) | Relational offline cache mirroring server entities; the plan must be readable with no network. |
| API | Python 3.12 + FastAPI, Pydantic v2, Uvicorn | Exposure engine and LLM orchestration are both Python-native; one language server-side. |
| Database | PostgreSQL 16 with `pgvector` and `postgis` | Relational household model, vector retrieval, and geospatial station lookup in one engine. |
| Cache / queue | Redis 7 | Response cache for upstream AQ data, rate limiting, and task broker. |
| Async work | `arq` (Redis-backed) | Lighter than Celery for ~6 recurring job types. |
| LLM | Anthropic API — `claude-sonnet-5-5` for plan narration and Q&A, `claude-haiku-5-5` for intent routing and classification | Structured tool-use output with strict schemas; two tiers keep per-household cost inside budget. |
| Embeddings | `BAAI/bge-m3` (1024-dim) self-hosted via `sentence-transformers` | Multilingual (en/hi/mr) in one model; no second vendor; no per-query cost. |
| AQ forecast source | Open-Meteo Air Quality API — `https://air-quality-api.open-meteo.com/v1/air-quality`, no API key, CAMS-backed, hourly to 5 days | Free, keyless, forecast-capable. Attribution to CAMS and Open-Meteo is mandatory and ships in the About screen. |
| AQ ground-truth source | OpenAQ v3 — `https://api.openaq.org/v3`, `X-API-Key` header | Reference-grade station measurements including CPCB feeds. Free tier: 60 req/min, 2,000 req/hour — the fusion job is built to that budget. |
| Weather source | Open-Meteo Forecast API | Wind speed, temperature, and boundary-layer height for the ventilation-window and dispersion heuristics. |
| Auth | Firebase Auth, phone OTP, anonymous-first | Phone is the norm in the launch market; anonymous-first lets a household get a plan before signing up. |
| Push | Firebase Cloud Messaging | Single provider for Android v1. |
| Hosting | API and worker as containers on Fly.io; Postgres on Neon; Redis on Upstash | Managed, cheap at low scale, Mumbai region available. |
| CI | GitHub Actions — lint, type-check, test, build, migrate, deploy | Repo is `Ibtidaa-Tech/airsaathi`. |
| Observability | OpenTelemetry traces → Grafana Cloud; Sentry for client and server errors | One trace per plan generation, spanning fetch → fuse → compute → narrate. |

### 6.1 Naming and unit conventions — binding across all four documents

| Context | Convention | Example |
|---|---|---|
| Product name | `AirSaathi`, one word, capital A and S | |
| Repository / package | `kebab-case` | `airsaathi-api` |
| Database tables, columns | `snake_case`, tables plural | `household_members.sensitivity_flags` |
| Python | `snake_case` functions, `PascalCase` classes | `compute_exposure()`, `ExposureEngine` |
| Dart | `lowerCamelCase` members, `PascalCase` classes | `dailyPlan`, `PlanAction` |
| JSON over the wire | `lowerCamelCase` | `{"pm25": 182.4, "generatedAt": "..."}` |
| Enum values | `UPPER_SNAKE_CASE` everywhere | `VERY_POOR`, `TWO_WHEELER` |
| API routes | `/v1/<plural-resource>`, kebab-case | `/v1/households/{householdId}/daily-plan` |
| Timestamps | ISO 8601, UTC, `Z` suffix, stored `timestamptz` | `2026-11-19T06:41:00Z` |
| Local time | IANA zone on the location record | `Asia/Kolkata` |
| PM / gas concentration | µg/m³ | `pm25 = 182.4` |
| CO concentration | mg/m³ | `co = 1.9` |
| Exposure (concentration × time) | µg/m³·h, field suffix `_ugm3h` | `exposure_ugm3h = 1842.0` |
| Inhaled mass | µg, field suffix `_ug` | `inhaled_ug = 1473.6` |
| Air exchange / removal rate | h⁻¹ | `0.5` |
| Purifier airflow | m³/h (CADR, smoke) | `250` |
| Room volume | m³ | `32.5` |
| Greenhouse gas | g CO₂e, field suffix `_gco2e` | `avoided_gco2e = 1360` |
| Distance | km (display), m (computation) | |
| Currency | INR, integer rupees, field suffix `_inr` | `cost_inr = 1800` |

Forbidden in all specification text and all generated user copy: "should" as a requirement verb (use "must" / "does"), "maybe", "etc.", "TBD" without an owner and a date, and any number without a unit.

## 7. Success metrics

### 7.1 North star

**Weekly Protected Hours per household** — the sum, across household members, of hours in the week during which modelled personal PM2.5 exposure was reduced below the counterfactual "no-action" baseline, as computed by the exposure engine. It rises only when a household actually changes behaviour, and it is computed from the same engine that drives the advice.

### 7.2 Supporting metrics with v1 targets

| Metric | Definition | Target by week 8 post-launch |
|---|---|---|
| Plan open rate | Households opening the Daily Plan on a given day ÷ active households | ≥ 55% |
| Action completion rate | Plan actions marked done ÷ plan actions surfaced | ≥ 30% |
| Week-4 retention | Households active in week 4 ÷ households onboarded in week 0 | ≥ 25% |
| Onboarding completion | Households reaching a first generated plan ÷ households starting onboarding | ≥ 70% |
| Impact trace open rate | Households tapping "how was this computed" at least once | ≥ 20% |
| Q&A groundedness | Sampled Q&A answers where every factual claim maps to a retrieved citation or a household fact | ≥ 95% |
| Numeric fidelity | Generated narratives where every number appears in the input facts object | 100% (enforced, not measured — see `spec.md` §8.5) |

### 7.3 Guardrail metrics — a release that breaches any of these is rolled back

| Guardrail | Limit |
|---|---|
| Plan generation p95 latency | ≤ 4,000 ms end to end |
| `/v1/air-quality/current` p95 latency | ≤ 400 ms (cache hit), ≤ 1,200 ms (cold) |
| Crash-free sessions | ≥ 99.5% |
| LLM cost per active household per month | ≤ ₹6.00 |
| Upstream fetch failure rate | ≤ 1% of scheduled fusion runs over 24 h |
| Stale-data display | No reading older than 3 h shown without a visible staleness badge |

## 8. Constraints

| Constraint | Value |
|---|---|
| Team | 1 backend, 1 Flutter, 1 designer (part-time), 1 content/domain reviewer (part-time) |
| Timeline | 10 working weeks to public beta, structured as milestones M0–M9 in `spec.md` §11 |
| Infrastructure budget | ≤ ₹8,000/month at 2,000 monthly active households |
| LLM budget | ≤ ₹12,000/month at 2,000 monthly active households; enforced by the per-household daily token cap in `spec.md` §8.6 |
| Data residency | All personal data in an India region. Neon Postgres and Fly.io in Mumbai; Anthropic API calls carry no direct identifiers (`spec.md` §9.3) |
| Legal | India DPDP Act 2023 — consent ledger, purpose limitation, data export, data erasure. Children's data minimised to age band plus sensitivity flag; no child name required, nickname only |
| Licensing | CAMS and Open-Meteo attribution in About; OpenAQ attribution and per-provider licence surfaced on the station detail sheet |
| Accessibility | WCAG 2.2 AA for contrast and target size; AQI never encoded by colour alone — always colour plus band label plus numeral |

## 9. Key risks

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Model invents a number and a household acts on it | Severe — trust and safety | Deterministic engine owns all arithmetic; post-generation numeric validator rejects any numeral absent from the facts object; template fallback renderer on rejection (`spec.md` §8.5) |
| R2 | Exposure model defaults are wrong for the launch market | High — all impact numbers skew | Every parameter lives in a versioned `model_config` row, stamped onto each estimate as `model_version`; domain reviewer signs off the India pack before M6; sensitivity ranges published in the impact trace |
| R3 | Advice construed as medical advice | High — legal and ethical | Hard-coded refusal path on symptom and medication intents; standing disclaimer on sensitive-member screens; escalation copy to a clinician reviewed by the content reviewer |
| R4 | Sparse station coverage yields a confident wrong reading | Medium-high | Every reading carries a `confidence` enum and a `sources` array; low confidence degrades the UI to bands without a numeral; no reading is shown without a provenance sheet |
| R5 | OpenAQ rate limit exceeded as households grow | Medium | Fusion is per *grid cell*, not per household: one fetch serves every household in the cell; Redis TTL 15 min; hard token bucket at 50 req/min against the 60 limit |
| R6 | LLM cost scales with engagement | Medium | Plan narration generated once per household per day plus on material forecast change (defined in `spec.md` §8.4); Q&A capped per household per day; routing on `claude-haiku-5-5` |
| R7 | Households act on a ventilation window that the forecast gets wrong | Medium | Ventilation windows require forecast confidence `HIGH` and are never issued beyond 12 h ahead; a window is withdrawn by push if conditions change |
| R8 | Flutter offline cache diverges from server state | Low-medium | Server is authoritative; client holds a read-through cache keyed by `plan_id` with a monotonic `revision`; conflicts resolve server-side |

## 10. Glossary

| Term | Definition |
|---|---|
| **AQI** | Air Quality Index. CPCB six-band Indian scale unless prefixed "US AQI". |
| **Band** | One CPCB AQI category: `GOOD`, `SATISFACTORY`, `MODERATE`, `POOR`, `VERY_POOR`, `SEVERE`. |
| **Microenvironment** | A place-and-activity context with its own concentration and inhalation rate: `OUTDOOR`, `HOME_INDOOR`, `TRANSIT`, `INDOOR_OTHER`. |
| **Exposure** | Concentration integrated over time, µg/m³·h. Independent of breathing rate. |
| **Inhaled dose** | Mass of pollutant breathed in, µg. Exposure × inhalation rate. |
| **F_inf** | Infiltration factor — the fraction of outdoor PM2.5 that reaches indoors at steady state with no intervention. |
| **CADR** | Clean Air Delivery Rate, m³/h, smoke rating. Used to size purifiers against room volume. |
| **Cigarette equivalent** | A communication device: 24-hour mean PM2.5 ÷ 22 µg/m³. Always displayed with its caveat string. |
| **Counterfactual** | The modelled no-action day used as the baseline against which avoided exposure is measured. |
| **Action catalogue** | The server-side table of candidate interventions, each with an effect model and a cost tier. |
| **Facts object** | The validated JSON payload of household state and computed numbers handed to the LLM. The model may not introduce a number outside it. |
| **Plan** | One dated `plans` row plus its ordered `plan_actions`, scoped to one household. |
| **Circle** | An invite-only neighbourhood group sharing a location and aggregate actions. |

## 11. Definition of done for v1

v1 is done when all of the following hold simultaneously:

1. All ten capabilities C1–C10 are implemented, with the acceptance criteria in `prd.md` §4 passing.
2. Milestones M0–M9 in `spec.md` §11 are closed, each against its stated exit criteria.
3. The deterministic exposure engine has unit tests covering every microenvironment, every indoor source, and every intervention in the action catalogue, with a golden-file regression suite pinned to `model_version`.
4. The numeric validator rejects 100% of injected synthetic hallucinations in the adversarial test set (`spec.md` §8.5).
5. The India configuration pack is signed off by the domain reviewer, with every parameter's source recorded in `model_config.provenance`.
6. All three languages pass content review for the full plan-action surface, not just UI strings.
7. Every guardrail metric in §7.3 is met in staging under a 2,000-household synthetic load.
8. DPDP obligations are demonstrable: a household can export its data and erase its account end to end, and both paths are tested.
9. Accessibility audit passes against §8's criteria.
10. `checklist.md` has every item checked, with no item marked blocked.
