# AirSaathi — Product Requirements

**Project:** AirSaathi (Urban Air Quality Assistant)
**Document:** `prd.md` — what it does, and how it looks and feels
**Version:** 1.0.0
**Status:** Approved for build
**Last updated:** 2026-11-19
**Depends on:** `scope.md` v1.0.0 (capabilities C1–C10, naming conventions §6.1)
**Feeds:** `spec.md` v1.0.0

## 1. Product principles

Five principles. Every design and copy decision in this document traces to one of them, and every disagreement is settled by whichever principle is higher in the list.

1. **One answer, then the evidence.** Every screen opens with a single decision the household can act on. The chart, the pollutant breakdown, and the arithmetic sit below it, reachable in one tap but never in the way.
2. **A number or nothing.** An action that cannot be quantified by the exposure engine is not shown as an action. Vague advice is worse than silence because it trains the household to ignore the app.
3. **Show the working.** Every number has a trace: the inputs, the formula name, the parameter version. A household that distrusts a number can open it; a household that trusts it never sees it.
4. **Cheap first.** The action catalogue is ordered by µg/m³·h avoided per rupee. A free action that works beats a purchase that works better, and the ranking makes that visible.
5. **Never pretend to be a doctor.** The product models air and arithmetic. Symptoms, medication, and diagnosis route to a human.

## 2. Personas and jobs to be done

### 2.1 Household Guardian — "Meera", 38, Lucknow

Two children (7 and 11), ground-floor rented flat, LPG stove with no chimney, no purifier, husband commutes by two-wheeler. Checks her phone at 06:40 while making tea.

| Job | Current workaround | What AirSaathi gives her |
|---|---|---|
| Decide if the children walk to school today | Looks at an AQI number, guesses | A per-member commute verdict with a swap and the µg avoided |
| Decide when to open windows | Opens them at dawn out of habit — the daily worst hour | A ventilation window, pushed when it opens |
| Decide whether to buy a purifier | Asks the building group, gets five opinions | A required CADR for her bedroom volume and a price band, with the exposure delta |
| Know whether any of it helped | No way to know | A weekly Protected Hours figure and a cigarette-equivalent change |

### 2.2 Sensitive Individual — "Arjun", 11, asthmatic (member of Meera's household)

Modelled as a `household_members` row with `age_band = CHILD_5_12` and `sensitivity_flags = ['ASTHMA']`. He is not a separate user account in v1. His flags lower every threshold applied to his routine blocks and raise his inhalation-rate multiplier during the `MODERATE` and `VIGOROUS` activity classes.

### 2.3 Community Organiser — "Faizan", 52, RWA secretary, Delhi-NCR

| Job | What AirSaathi gives him |
|---|---|
| Give 60 flats one consistent message each morning | A circle plan card, shareable as an image plus text to WhatsApp |
| Stop the nightly waste burning behind D-block | Member reports aggregate into a drafted, dated civic complaint with the recorded PM2.5 deltas |
| Show the RWA that the push on windows worked | Circle-aggregate Protected Hours, with no individual household identifiable |

## 3. Information architecture

```
AirSaathi
├── Today            (tab 1, default)  ─ the plan
│   ├── Now card                       ─ fused AQI, band, staleness, provenance sheet
│   ├── Today's verdict                ─ one sentence, generated
│   ├── Action list                    ─ time-windowed, checkable
│   ├── Hourly strip                   ─ 0–72 h, scrubbable
│   └── Impact strip                   ─ today's avoided exposure so far
├── Home            (tab 2)            ─ indoor air
│   ├── Indoor estimate card           ─ modelled indoor PM2.5 vs outdoor
│   ├── Toolkit                        ─ ranked interventions, cost-tiered
│   ├── Source checklist               ─ cooking, incense, coils, smoking, candles
│   └── Purifier sizing tool           ─ room volume → required CADR
├── Impact          (tab 3)            ─ the numbers
│   ├── Protected Hours (week)
│   ├── Exposure avoided (µg/m³·h, µg inhaled)
│   ├── Cigarette-equivalent trend
│   ├── CO₂e avoided
│   └── Trace viewer                   ─ per-action arithmetic
├── Ask             (tab 4)            ─ grounded Q&A
└── More            (tab 5)
    ├── Household                      ─ members, home, routines
    ├── Circles
    ├── Alerts
    ├── Language
    ├── Data & privacy                 ─ export, erase, consent ledger
    └── About                          ─ attributions, model version, disclaimers
```

Navigation is a five-item bottom bar. Depth is capped at three levels from any tab. There is no hamburger menu.

## 4. User stories and acceptance criteria

Stories are grouped by the capability they implement. Acceptance criteria are written Given/When/Then and are the basis of the QA suite. Every criterion is objectively testable; none contains a vague verb.

### 4.1 C1 — Air quality now and ahead

**US-1.1** As a Household Guardian, I see my location's current air quality the moment the app opens, so that I do not have to search for it.

- **Given** a household with a saved primary location and a cached reading under 60 minutes old, **when** the app opens, **then** the Now card renders within 400 ms from cache with the reading's age shown as "Updated N min ago", and a background refresh replaces it without a layout shift.
- **Given** the cached reading is between 60 minutes and 3 hours old, **when** the Now card renders, **then** the age label is shown in the warning colour token `--color-warn`.
- **Given** the cached reading is older than 3 hours, **when** the Now card renders, **then** the numeral is replaced by the band label alone, a `STALE` badge is shown, and the tap target opens the provenance sheet.
- **Given** no cached reading and no network, **when** the app opens, **then** the Now card shows the offline empty state with the last known reading, its timestamp, and no generated verdict text.

**US-1.2** As a Household Guardian, I see where the number came from, so that I can judge whether to trust it.

- **Given** any rendered reading, **when** I tap the Now card, **then** the provenance sheet lists every contributing source with its type (`STATION` or `MODEL`), distance in km for stations, observation time, and the licence string for each OpenAQ provider, plus the `confidence` enum and the reason the engine assigned it.
- **Given** `confidence = LOW`, **when** the Now card renders, **then** no pollutant numeral is displayed anywhere on the Today tab; only bands and ranges are shown.

**US-1.3** As a Household Guardian, I scrub forward through the next three days, so that I can plan around a coming spike.

- **Given** a household with a forecast, **when** I drag the hourly strip, **then** the Now card's values, band, and colour follow the scrub position, the card header switches from "Now" to the scrubbed local time, and the action list dims to indicate it reflects the plan rather than the scrub.
- **Given** I release the scrub, **when** 300 ms elapse, **then** the card animates back to the present reading.
- **Given** a forecast hour beyond 72 hours, **when** I scrub, **then** the strip ends; the 5-day outlook is a separate daily-granularity row below.

### 4.2 C2 — Household model

**US-2.1** As a new user, I reach a usable plan in under 90 seconds, so that I find out whether this app is worth keeping.

- **Given** a fresh install, **when** I complete onboarding, **then** I have answered at most 7 screens, each with at most 5 options, and a plan has been generated.
- **Given** onboarding, **when** any screen renders, **then** a "Skip for now" affordance is present and every skipped field falls back to the documented default in `spec.md` §5.3.
- **Given** I skip the home-seal question, **when** the plan generates, **then** the plan is produced using `seal_quality = TYPICAL` and the Home tab shows a "Make this more accurate" prompt naming the missing field.
- **Given** location permission is denied, **when** onboarding continues, **then** I can enter a city and area by text search, and the plan generates against the area centroid with `confidence` capped at `MEDIUM`.

**US-2.2** As a Household Guardian, I record each member's recurring routine, so that advice lands on the right person at the right hour.

- **Given** the Household screen, **when** I add a routine block, **then** I set member, activity class (`REST`, `LIGHT`, `MODERATE`, `VIGOROUS`), microenvironment, transit mode where applicable, local start and end time, and days of week, and the block is rejected with an inline error if it overlaps another block for the same member.
- **Given** a member with `sensitivity_flags` non-empty, **when** their routine block is saved, **then** the Today tab shows that member's actions above all others, labelled with the member's nickname.

**US-2.3** As a parent, I describe my child without surrendering their identity.

- **Given** adding a household member, **when** the form renders, **then** the only required fields are age band and relationship; a nickname field is optional and free text; no date of birth, full name, school, or photo field exists anywhere in the product.

### 4.3 C3 — Daily Clean-Air Plan

**US-3.1** As a Household Guardian, I read one sentence and know what today demands of me.

- **Given** a generated plan, **when** the Today tab renders, **then** the verdict is at most 160 characters, names at most one band, at most one time window, and at most one action, and contains no numeral absent from the plan's facts object.
- **Given** the verdict generation fails validation twice, **when** the plan renders, **then** the deterministic template verdict is shown instead, and the event `plan_narration_fallback` is recorded. The user sees no error.

**US-3.2** As a Household Guardian, actions arrive attached to a time, so that I know when to act.

- **Given** a generated plan, **when** the action list renders, **then** every action carries a local time window, an avoided-exposure figure in µg/m³·h, a cost tier badge (`FREE`, `LOW_COST`, `PURCHASE`), and the member it applies to where member-specific.
- **Given** the action list, **when** it renders, **then** actions are ordered by window start time ascending, and within the same window by avoided exposure per rupee descending.
- **Given** an action whose window has passed, **when** the list renders, **then** the action moves to a collapsed "Earlier today" group retaining its checked state.
- **Given** I check an action, **when** the tap registers, **then** the Impact strip increments optimistically within 100 ms, and the mutation is queued for sync if offline.

**US-3.3** As a Household Guardian, the plan changes when the air changes.

- **Given** an active plan, **when** a fusion run finds the forecast has changed materially — defined as any of: next-12 h peak PM2.5 differs by ≥ 25 µg/m³, the peak band changes, or a ventilation window appears or disappears — **then** the plan is regenerated, its `revision` increments, and a push notification is sent unless quiet hours apply.
- **Given** a regenerated plan, **when** the Today tab renders, **then** a "Plan updated" chip names what changed in at most 80 characters, and dismissing it is persistent for that revision.
- **Given** a forecast change below all three thresholds, **when** the fusion run completes, **then** no regeneration occurs and no notification is sent.

**US-3.4** As a Household Guardian, I use yesterday's plan when my phone has no signal.

- **Given** a plan cached locally, **when** the app opens with no network, **then** the full plan, all actions, and all impact figures render from the local Drift store with an offline banner, and checking actions remains available and syncs on reconnect.

### 4.4 C4 — Indoor air toolkit

**US-4.1** As a Household Guardian, I see what the air is like inside my own home, not just outside.

- **Given** a complete home profile, **when** the Home tab renders, **then** the indoor estimate card shows modelled indoor PM2.5 in µg/m³ beside outdoor PM2.5, the ratio as a percentage, and the sentence pattern "Modelled, not measured" with a trace link.
- **Given** an incomplete home profile, **when** the Home tab renders, **then** the indoor estimate shows a range rather than a point value, and the card names each missing field as a tappable prompt.

**US-4.2** As a Household Guardian, I get interventions I can afford, cheapest-effective first.

- **Given** the toolkit, **when** it renders, **then** interventions are grouped `FREE`, `LOW_COST` (≤ ₹2,000), `PURCHASE` (> ₹2,000); within each group they are ordered by avoided exposure descending; and each shows avoided exposure in µg/m³·h per day, cost in ₹, and effort as a time estimate in minutes.
- **Given** an intervention already recorded as adopted in the household profile, **when** the toolkit renders, **then** it appears in an "Already doing" section with its ongoing contribution and is excluded from the ranked lists.

**US-4.3** As a Household Guardian, I size a purifier correctly instead of guessing.

- **Given** the purifier sizing tool, **when** I enter room length, width, and ceiling height in metres, **then** the tool returns room volume in m³, required smoke CADR in m³/h for 5 air changes per hour, the resulting modelled indoor PM2.5 at the current outdoor level, and the DIY box-fan alternative with its own modelled figure and parts cost in ₹.
- **Given** the computed required CADR, **when** the result renders, **then** it is shown as a specification to shop against, with no brand name and no purchase link.

**US-4.4** As a Household Guardian, the app tells me when to air the house rather than telling me not to.

- **Given** a day with at least one ventilation window, **when** the action renders, **then** its copy offers the cleanest hour to air the house and its figure is stated as the saving against airing at the household's habitual time, named in the body — it never claims that opening the windows lowers exposure.
- **Given** a day whose cleanest forecast hour exceeds 120 µg/m³, **when** the plan renders, **then** no ventilation window is offered, and the plan instead carries the sealing action and the source-venting actions, with one line explaining that today the kitchen exhaust is the way to clear smoke.

### 4.5 C5 — Sustainable swaps

**US-5.1** As a Household Guardian, I see swaps that help my lungs and the city's air at once.

- **Given** a household with at least one `TRANSIT` routine block, **when** the swaps list renders, **then** each swap shows the member affected, avoided personal exposure in µg/m³·h, avoided inhaled mass in µg, avoided CO₂e in g, and the practical cost or time difference.
- **Given** a swap where personal exposure rises while CO₂e falls — for example moving from a filtered car to a bicycle on a `SEVERE` day — **when** it renders, **then** the trade-off is stated explicitly in the card and the swap is demoted below all swaps that improve both, never hidden.

### 4.6 C6 — Impact Explainer

**US-6.1** As a Household Guardian, I see what my choices bought me this week.

- **Given** at least one completed action in the trailing 7 days, **when** the Impact tab renders, **then** it shows Protected Hours for the week, total avoided exposure in µg/m³·h, total avoided inhaled mass in µg, the cigarette-equivalent change, and avoided CO₂e in g, each with the counterfactual baseline stated.
- **Given** no completed actions in the trailing 7 days, **when** the Impact tab renders, **then** it shows the modelled baseline exposure and the single highest-value available action, and states plainly that nothing has been logged yet. It does not show a zero-filled chart.

**US-6.2** As a sceptical user, I check the arithmetic.

- **Given** any impact figure anywhere in the product, **when** I tap it, **then** the trace sheet shows: every input with its value and unit, the named formula applied, every parameter with its value and its `model_config` provenance string, the intermediate results, and the `model_version` identifier.
- **Given** the cigarette-equivalent figure, **when** it renders anywhere, **then** the caveat string is adjacent and not behind a tap: it names the 22 µg/m³ 24-hour-mean basis and states that the comparison covers particulate exposure only.

### 4.7 C7 — Grounded Q&A

**US-7.1** As a Household Guardian, I ask in my own words and get an answer about my own home.

- **Given** the Ask tab, **when** I submit a question, **then** the first token streams within 1,500 ms p95, and the completed answer cites every factual claim either to a corpus passage with a titled source or to a named household fact.
- **Given** a question whose retrieval returns no passage above the relevance floor, **when** the answer renders, **then** it states that the corpus does not cover the question, offers the nearest covered topic, and makes no claim of its own.
- **Given** a question containing a numeral about the household's own air, **when** the answer renders, **then** every numeral in the answer is present in the facts object or a cited passage.

**US-7.2** As a user who mentions a symptom, I am routed to a person, not a model.

- **Given** a question classified `MEDICAL` by the intent router, **when** the answer renders, **then** the product does not answer the clinical question, states that it cannot, names the air-quality facts relevant to the household, and surfaces the escalation copy. This holds on repeat and rephrased asking within the same session.

### 4.8 C8 — Alerts

**US-8.1** As a Household Guardian, I am told before it gets bad, not after.

- **Given** a forecast spike crossing into `VERY_POOR` or `SEVERE` within the next 12–24 hours, **when** the alert job runs, **then one** notification is sent naming the band, the local onset time, and one action.
- **Given** quiet hours configured 21:00–06:30 local by default, **when** an alert would fire inside them, **then** delivery defers to the next permitted minute, except for `SEVERE` onset within 3 hours, which overrides quiet hours and is labelled as doing so.
- **Given** multiple alert conditions within 60 minutes, **when** the job runs, **then** they coalesce into one notification; the product never sends more than 3 notifications per household per day.

| Alert type | Trigger | Default | Quiet-hours override |
|---|---|---|---|
| `BAND_DETERIORATION` | Current band worsens by ≥ 1 and holds 30 min | On | No |
| `VENTILATION_WINDOW` | Window opens ≤ 2 h ahead, forecast confidence `HIGH` | On | No |
| `FORECAST_SPIKE` | `VERY_POOR`+ onset in 12–24 h | On | No |
| `SEVERE_IMMINENT` | `SEVERE` onset ≤ 3 h | On | Yes |
| `WINDOW_WITHDRAWN` | An issued ventilation window is invalidated | On | No |
| `WEEKLY_IMPACT` | Sunday 19:00 local | On | No |
| `CIRCLE_REPORT` | A circle member files a local-source report | Off | No |

### 4.9 C9 — Neighbourhood circles

**US-9.1** As a Community Organiser, I give my neighbours one message.

- **Given** I create a circle, **when** it is created, **then** I receive an 8-character invite code, the circle is capped at 150 members, and I am its only admin.
- **Given** a circle with ≥ 3 members, **when** the circle card renders, **then** it shows the circle's shared plan verdict, the count of members who completed each top action, and aggregate Protected Hours — and never any individual member's figures, actions, or household composition.
- **Given** the circle card, **when** I tap share, **then** the product produces a 1080×1080 image plus a text block of at most 400 characters carrying the verdict, the band, the date, and the attribution line, ready for WhatsApp.

**US-9.2** As a Community Organiser, repeated local reports become a usable complaint.

- **Given** ≥ 3 reports of the same `source_type` within 500 m and 7 days, **when** the aggregation job runs, **then** the circle admin is offered a drafted civic complaint containing the dated report times, the reported source type, the recorded PM2.5 readings at each report time, and the circle's area name — with no reporter identity included.
- **Given** the drafted complaint, **when** it renders, **then** it is editable before any export, and the product only copies it to the clipboard or shares it. The product never submits it on the household's behalf.

### 4.10 C10 — Bilingual, low-bandwidth delivery

**US-10.1** As a Hindi-first user, the whole product speaks Hindi, including the generated advice.

- **Given** `language = hi`, **when** any screen renders, **then** all UI strings, all action catalogue copy, all generated narrative, and all notification bodies are in Hindi, with numerals in Western Arabic digits and units in Devanagari-transliterated form per the locale file.
- **Given** a language change, **when** it is saved, **then** the current plan's narrative is regenerated in the new language within 5 seconds, and cached per language so switching back costs no generation.
- **Given** a string missing from a locale file, **when** the screen renders, **then** the English string is shown and the event `i18n_missing_key` is recorded. The UI never shows a raw key.

## 5. Visual design

### 5.1 CPCB AQI band tokens

Every band is encoded by three channels — fill, label, and numeral — so that colour is never the sole carrier of meaning.

| Band | AQI range | Fill token | Fill hex | Text on fill | Label (en) |
|---|---|---|---|---|---|
| `GOOD` | 0–50 | `--aqi-good` | `#009865` | `#FFFFFF` | Good |
| `SATISFACTORY` | 51–100 | `--aqi-satisfactory` | `#5A9E2F` | `#FFFFFF` | Satisfactory |
| `MODERATE` | 101–200 | `--aqi-moderate` | `#C9A227` | `#1A1A1A` | Moderate |
| `POOR` | 201–300 | `--aqi-poor` | `#E07B24` | `#1A1A1A` | Poor |
| `VERY_POOR` | 301–400 | `--aqi-very-poor` | `#D33A2C` | `#FFFFFF` | Very Poor |
| `SEVERE` | 401–500 | `--aqi-severe` | `#9B2423` | `#FFFFFF` | Severe |

Every fill/text pair above meets WCAG 2.2 AA contrast at 4.5:1 for body text. The canonical CPCB yellow and light-green are darkened as shown to reach that ratio; the original CPCB hues are retained only in the legend on the station provenance sheet, where they appear as non-text swatches.

### 5.2 Type and layout

| Token | Value |
|---|---|
| Typeface | Inter (Latin) + Noto Sans Devanagari (hi, mr), variable weights 400/500/700 |
| Scale | 12 / 14 / 16 / 20 / 28 / 44 px, 1.5 line height for body, 1.2 for display |
| AQI numeral | 44 px, weight 700, tabular figures |
| Spacing | 4 px base unit; card padding 16 px; gutter 16 px; section gap 24 px |
| Radius | 12 px cards, 8 px chips, 999 px pills |
| Min touch target | 48 × 48 px |
| Elevation | Two levels only: flat (0) and raised (card, 1 dp with a 4% shadow) |
| Motion | 200 ms standard ease-out; 0 ms when the OS reduce-motion flag is set |
| `--color-warn` | `#B45309` on light, `#F59E0B` on dark — the aging-reading label |
| `--color-focus` | `#1D4ED8` on light, `#93B4FD` on dark — the 2 px focus ring |
| `--color-surface` | `#FFFFFF` on light, `#121212` on dark |
| `--color-text-primary` | `#1A1A1A` on light, `#F2F2F2` on dark |
| Theme | Light and dark, both authored; the band fills above hold in both, with the dark-theme surface at `#121212` |

### 5.3 Screen specifications

#### Today — Now card

```
┌─────────────────────────────────────────────┐
│ Gomti Nagar, Lucknow            Updated 6m ▸│
│                                             │
│   382        VERY POOR                      │
│   AQI        PM2.5  248 µg/m³               │
│              Main pollutant                 │
│                                             │
│  WHO 24-h guideline is 15 µg/m³ ───────     │
│  2 CPCB stations · 1 model     Confidence ●●○│
└─────────────────────────────────────────────┘
```

States: `loading` (skeleton, no spinner over 1 s), `fresh`, `aging` (warning age label), `stale` (band only, `STALE` badge), `low_confidence` (band and range only), `offline` (last known plus timestamp), `no_location` (location prompt).

#### Today — verdict and action list

```
┌─────────────────────────────────────────────┐
│ Today, 19 Nov                               │
│ Air stays Very Poor until evening. Keep      │
│ windows shut and send Arjun by auto today.   │
└─────────────────────────────────────────────┘

  06:00–12:00  ▸ Keep windows closed          FREE
               Avoids 266 µg/m³·h             ☐
               Arjun · asthma

  07:30–08:10  ▸ Swap the school walk for a   FREE
               shared auto                     ☐
               Avoids 148 µg/m³·h · 0 g CO₂e added

  14:00–15:30  ▸ Cleanest hour to air the      FREE
               house today                     ☐
               Avoids 101 µg/m³·h vs airing at 07:00

  Any time     ▸ Run the kitchen exhaust for   FREE
               10 min after cooking            ☐
               Avoids 240 µg/m³·h
```

Each row: time window, title, cost badge, avoided exposure, optional member tag, checkbox. Tapping the row opens the action detail sheet with the trace, the reasoning, and the "why this works" copy. Tapping the checkbox never navigates.

#### Home — indoor estimate card

```
┌─────────────────────────────────────────────┐
│ Inside your home right now                  │
│                                             │
│  Indoor  142 µg/m³    Outdoor  248 µg/m³    │
│          ███████░░░░░ 57% of outdoor         │
│                                             │
│  Modelled, not measured.  How? ▸            │
│  With a 250 m³/h purifier in the bedroom,    │
│  door closed: 12 µg/m³                      │
└─────────────────────────────────────────────┘
```

#### Impact — week summary

```
┌─────────────────────────────────────────────┐
│ This week                                   │
│  14.2  Protected Hours                       │
│  ────────────────────────────                │
│  Exposure avoided     3,120 µg/m³·h   How? ▸ │
│  Inhaled mass avoided   2,480 µg      How? ▸ │
│  Cigarette equivalent  7.5 → 6.7      How? ▸ │
│   Daily average. Based on a 22 µg/m³ 24-h     │
│   mean per cigarette; particulates only.      │
│  CO₂e avoided           4,080 g       How? ▸ │
│  Baseline: your routine with no actions taken │
└─────────────────────────────────────────────┘
```

#### Ask — answer surface

Streamed answer, then a citation list. Each citation is a chip with the source title and a tap to the passage. Household facts used appear as a distinct chip style labelled "Your home". A persistent one-line footer reads that AirSaathi is not a medical service.

### 5.4 Empty, error, and degraded states

| State | Trigger | Behaviour |
|---|---|---|
| No location | Permission denied and no text location | Full-screen prompt with city search; no plan tab content |
| No plan yet | Onboarding incomplete | Today tab shows the remaining onboarding steps as a checklist |
| Offline | No connectivity | Persistent banner; cached plan fully usable; mutations queued; generation disabled with an explanatory line |
| Upstream failure | Both AQ sources fail | Last known reading with `STALE` badge; no generated verdict; the failure is never surfaced as a technical error string |
| Low confidence | Confidence enum `LOW` | Bands and ranges only, product-wide; impact figures suppressed for the affected hours |
| Generation failure | LLM validation fails twice | Deterministic template copy; no user-visible error |
| Rate limited | Household Q&A cap reached | Remaining-quota line and the reset time in local time |

## 6. Content and tone rules

### 6.1 Voice

Plain, specific, unhurried. Second person. Present tense. Indian English conventions. No alarm language — the band already carries the severity, and adjectives on top of `SEVERE` read as panic. No praise for completing actions beyond a neutral acknowledgement; the number is the reward.

### 6.2 Hard constraints on generated text

Enforced programmatically, not by instruction alone. The validator in `spec.md` §8.5 rejects any violation.

1. No numeral may appear that is absent from the facts object.
2. No pollutant, standard, or threshold may be named that is absent from the facts object or the cited corpus.
3. No brand, model, product, or retailer name may appear.
4. No clinical term, diagnosis, medication, or dosage may appear.
5. No causal health claim about an individual may appear. Population-level framing drawn from a cited passage is permitted; "this will make Arjun ill" is not.
6. Length caps: verdict 160 characters, action title 60, action body 220, notification body 140.
7. Exactly one call to action per verdict.

### 6.3 Required standing copy

| Surface | Copy requirement |
|---|---|
| Ask footer, every answer | States AirSaathi is not a medical service and names who to contact for symptoms |
| Any cigarette-equivalent figure | The 22 µg/m³ basis and the particulate-only limitation, adjacent, never behind a tap |
| Indoor estimate | "Modelled, not measured" |
| About | CAMS and Open-Meteo attribution; OpenAQ attribution; per-provider licences; current `model_version` |
| Sensitive-member screens | Threshold-tuning explanation and the non-medical disclaimer |
| Civic complaint draft | A line stating the household drafted and sent it, not AirSaathi |

## 7. Localisation

| Item | Rule |
|---|---|
| Languages | `en` (base), `hi`, `mr` |
| String source | ARB files per locale; `en` is authoritative; no runtime machine translation |
| Generated narrative | Produced in the household's language directly, not translated from English; the facts object is language-neutral |
| Action catalogue copy | Authored per locale by the content reviewer; an action with an unreviewed locale string is not served in that locale |
| Numerals | Western Arabic digits in all locales |
| Units | Localised unit words; symbols (`µg/m³`) unchanged |
| Dates and times | 24-hour clock in all locales; `d MMM` short dates |
| Pluralisation | ICU plural rules via `intl` |
| RTL | Not required in v1; no layout hard-codes left or right |

## 8. Accessibility

| Requirement | Criterion |
|---|---|
| Contrast | WCAG 2.2 AA, 4.5:1 body, 3:1 for ≥ 24 px text and UI components |
| Colour independence | Every band carries fill + text label + numeral; charts carry a pattern fill in addition to hue |
| Touch targets | ≥ 48 × 48 px, ≥ 8 px apart |
| Screen reader | Every interactive element has a semantic label; the Now card announces band, pollutant, value, and freshness in one utterance; the hourly strip exposes an accessible table alternative |
| Text scaling | Layouts hold to 200% OS text scale with no clipping or overlap |
| Reduce motion | All animation durations collapse to 0 ms when the OS flag is set |
| Focus | Visible focus ring on every focusable element, 2 px, `--color-focus` |
| Charts | Every chart has a text summary sentence above it carrying the same information |

## 9. Non-functional requirements

| # | Requirement | Target |
|---|---|---|
| N1 | Cold app start to Now card painted, cached | ≤ 1,200 ms p95 on a 2 GB-RAM Android 11 device |
| N2 | Plan generation end to end | ≤ 4,000 ms p95 |
| N3 | Q&A first token | ≤ 1,500 ms p95 |
| N4 | `/v1/air-quality/current` | ≤ 400 ms p95 cached, ≤ 1,200 ms p95 cold |
| N5 | Offline capability | Full plan, actions, impact, and action-check mutations with zero connectivity |
| N6 | Payload on Today tab refresh | ≤ 40 KB gzipped |
| N7 | APK size | ≤ 25 MB |
| N8 | Crash-free sessions | ≥ 99.5% |
| N9 | API availability | ≥ 99.5% monthly |
| N10 | LLM cost per active household per month | ≤ ₹6.00 |
| N11 | Data erasure completion | ≤ 7 days from request, with confirmation |
| N12 | Data export | JSON, within 24 hours of request, self-service |
| N13 | No reading older than 3 h without a staleness badge | 100% |
| N14 | Battery | ≤ 1% per day attributable to AirSaathi background work |

## 10. Analytics events

Minimal and purposeful: every event maps to a metric in `scope.md` §7. No event carries exact coordinates, member nicknames, or free-text question bodies.

| Event | Properties |
|---|---|
| `onboarding_step_completed` | `stepId`, `skipped` |
| `onboarding_completed` | `durationMs`, `skippedCount` |
| `plan_viewed` | `planId`, `revision`, `source` (`push`, `cold_start`, `tab`) |
| `plan_action_checked` | `actionCatalogId`, `costTier`, `avoidedUgm3h`, `offline` |
| `plan_action_unchecked` | `actionCatalogId` |
| `plan_regenerated` | `planId`, `revision`, `reason` |
| `plan_narration_fallback` | `planId`, `failureStage` |
| `impact_trace_opened` | `figureType`, `surface` |
| `provenance_sheet_opened` | `confidence`, `stationCount` |
| `ask_question_submitted` | `intentClass`, `retrievedCount`, `language` |
| `ask_answer_ungrounded_blocked` | `intentClass` |
| `toolkit_item_adopted` | `actionCatalogId`, `costTier` |
| `purifier_sizing_computed` | `roomVolumeM3`, `requiredCadrM3h` |
| `swap_viewed` | `swapId`, `tradeOff` (bool) |
| `notification_sent` / `notification_opened` | `alertType`, `quietHoursOverridden` |
| `circle_created` / `circle_joined` | `circleId`, `memberCount` |
| `circle_share_generated` | `circleId`, `format` |
| `report_filed` | `sourceType` |
| `complaint_draft_generated` | `circleId`, `reportCount` |
| `i18n_missing_key` | `key`, `locale` |
| `data_export_requested` / `data_erasure_requested` | — |

## 11. Release gates

A build ships to beta only when: every acceptance criterion in §4 passes; §9 targets are met under the staging load test; §8 accessibility criteria pass audit; the §6.2 validator rejects the full adversarial set; and all three locales pass content review on the complete action catalogue.
