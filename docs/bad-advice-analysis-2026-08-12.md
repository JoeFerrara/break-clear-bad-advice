# Bad-advice analysis — can Break Clear recommend a bad destination? (2026-08-12)

**The question** (Joe, 2026-08-12, gating the free-launch go): *"I don't want to recommend a
bad destination."* The acceptable worst case is an honest "no escapes within 6 driving hours."
Routing danger is explicitly out of scope as a data problem — Apple Maps warns drivers, and
language on site and app is the control (§6 audits that language).

**Method.** 14-agent workflow against `main` (HEAD `25e4e4d`/`4d5daf8`, 2026-08-12): four
mappers (degraded paths, ranking inputs, client presentation, empty-reason ladder), four
scenario stress-tests (regional mega-smoke, model error at the destination, peak-load
degradation, staleness), then an independent adversarial verifier per claimed wrong-advice
path, instructed to refute against current code. Several claims were partially refuted — the
refutations are reported alongside the confirmations. Every claim below carries file:line
evidence in the workflow transcript
(`subagents/workflows/wf_66789eab-322/journal.jsonl` in the session directory).

---

## Verdict

**The empty path is honest. The architecture mostly fails safe. But the honest empty is not
the worst case today.** Five wrong-advice paths were CONFIRMED by adversarial verification —
three already tracked as open priority-high issues (#58, #75, #73), **two previously
untracked** (the fresh-plume decay artifact and the stale-anchor vacuous confirmation), plus
one confirmed client-side fabrication ("Not in the next 3 days" from zero data). The fix
surface is concentrated: a launch gate of ~9 bounded changes (§5) converts the worst case to
an honest empty or an honestly-labeled partial. **Recommendation: the free-launch go stands,
conditioned on the §5 gate items shipping before public release.**

---

## 1. What holds up (verified sound)

- **Mega-smoke produces an honest empty.** When nothing clears the improvement gate,
  the escapes endpoint returns `EMPTY_HONEST` ("No destination beats your air by enough right now —
  staying put is the honest answer"), never a least-bad list; `nearestQualifying` must itself
  clear the same gate. The empty-reason ladder is complete, contract-ordered, and rendered
  verbatim by the client.
- **#24 is closed and its fixes are verifiably in the code**: hourly AQI is NowCast-derived
  from concentration (not `us_aqi`), the additive ramp-fabricating bias correction was
  replaced by the multiplicative 24h-half-life form, and the anchor is PM2.5-only
  (`cb39d2f`, `ceb22e9`, `b47cb67`).
- **~15 degraded paths, most honest**: labeled 429/503s; the origin-forecast degraded body is
  `no-store` + `x-breakclear-degraded` and never colo-cached; four labeled empty reasons.
- **The iOS client is unusually honest** in its own right: five typed failure states,
  verbatim server empty reasons, day-qualified freshness stamps, a "Couldn't update — showing
  earlier data" label, URLCache bypass (`eea961c` — this **closed the iOS half of #49**), and
  a widget staleness ladder that refuses to show aged numbers.
- **A real monitor cross-check exists at destinations** and feeds the ranking (hour-0 pin +
  multiplicative correction), not just the display.
- **Live production evidence** (implementation session, 2026-08-12): the honest-empty path
  verified against production — Yakima at a 3h budget returns an empty with the "nearest that
  qualifies" sentence naming Pullman beyond the budget, rather than offering something worse.
- **Refutations worth recording** (adversarial verify working as intended): escape cards
  inside a degraded body are *full-confidence destination data* (the origin budget doesn't
  touch the catalog), so rendering them normally is correct; the iOS 4-hour browser-cache
  path claimed under #49 is closed by `eea961c`; #49's remaining harm is the web companion
  and header-honoring third-party clients, not the flagship app.

## 2. Confirmed wrong-advice paths

Each of these was CONFIRMED (or the marked one PARTIAL) by an independent verifier that tried
to refute it against current `main`.

### 2.1 Fresh plume + working monitor: the decay artifact flips the gate — CONFIRMED high, previously untracked

The bias correction pins hour 0 to the monitor and decays with a 24h half-life
(`ranking.ts:230-237`), but the stay window starts at arrival (drive ≥ ~43 min), so **the
pinned hour never enters the gated statistic** — every scored hour trends back toward the
plume-blind CAMS curve. Verified arithmetic: monitor NowCast AQI 158 vs CAMS hour-0 AQI 28
(ratio ~13, inside the [0.05, 20] clamp) over a 48h window arriving +4.5h → corrected hours
~135/91/68/52, stay-mean ~71 — sails through the gate (threshold 150 for origin 175) **on a
"clearing" that is 100% correction-decay artifact; the model contributed zero information
about the plume ending.** If the plume persists, the true stay-mean ~158 would have been
rejected. The card ships honestly labeled (current 158, arrival USG, `cleanThroughCheckout:
false`) — but the *decision* to ship it was fabricated. The half-life was designed for the
CAMS-overestimates direction (#24); the underestimate direction is unhandled.

### 2.2 Stale anchor vacuously confirms the clean model — CONFIRMED high, previously untracked

**No recency check exists anywhere on the anchor path**: `siteReadingFor` never reads
`observedAt` (`airnowsites.ts:316-374`), `biasAnchorAQI` takes no timestamp, the only anchor
gate is distance. Typical anchor age is 1–2.5h (45–75 min AirNow publication lag + 30-min
cron); worst case ~12h through the 6h walk-back + 6h KV TTL after an AirNow failure. A plume
arriving inside that gap makes ratio ≈ 1 — **the stale monitor confirms the clean model, the
card shows a clean `current`, and a fully confident wrong recommendation ships.** Verified
worse than claimed: if CAMS *correctly* forecasts the arriving plume, the stale-clean anchor
produces ratio < 1 and **scales the correct smoke forecast down** (up to 20x, decaying over
24h) — a stale monitor can actively erase a true forecast. Compounding:
`dataAges.airnowMinutes` is computed from the *areas* ingest wall-clock, so it reads fresh
even when the observation behind the anchor is 6h old. NowCast's 0.5 newest-hour weight
additionally halves any step change. Plume fronts move 20–50 km/h; the typical staleness gap
is exactly the distance they need.

### 2.3 The known gate cluster: #58 + #75 + #73 — CONFIRMED high, already open

All three open, priority-high, and verified still present:

- **#58**: `originRefAQI` is an instantaneous **max-across-pollutants** NowCast; the
  destination side is a **PM2.5-only 48h forecast mean**. Destination ozone is fetched but
  never scored. On an ozone-driven origin day the app can confidently recommend a drive into
  equally bad ozone, judged solely on PM2.5. (The cross-pollutant asymmetry is *in addition
  to* the temporal asymmetry the issue text centers.)
- **#75**: nearest-monitor selection has no pollutant filter — a non-PM2.5 monitor's AQI is
  published as the destination's air ("Whitefish reads Moderate during a 207"), and it also
  silently nulls the bias anchor, disabling correction for that destination. **A fix already
  exists on branch `fix/75-pm25-resolver` (`767a709`), unmerged.**
- **#73**: `.alreadyClear` fires whenever the first 3 hours are ≤100 regardless of what
  follows — "Already Good right now" hours before Very Unhealthy.

### 2.4 The client fabricates "Not in the next 3 days" from zero data — CONFIRMED high, previously untracked

`ClearingOutlook.compute(hours: [])` returns `.noneInForecast`
(`OriginOutlook.swift:39` — the guard maps an empty series to the same case as a genuine
72-hour never-clears forecast), whose copy is **"Not in the next 3 days"** — rendered under
"When does it clear?" on the detail screen, written into the home-screen widget
(`clearingSummary`), and spoken by the Live Activity. The trigger (empty `origin.hourly`) is
precisely the degraded state during budget exhaustion — i.e., **mid-smoke-event, peak load,
the moment the answer matters** — and a confident false "won't clear for 3 days" pushes
needless escape decisions. The worker's own headline carefully avoids this (#43 made it
duration-free); the client undoes that care. A second path: the detail view's
top-of-hour filter can shrink a stale non-empty series below `minimumRun`, same sentence.

### 2.5 Degradation is signaled only in a header the client discards — PARTIAL (mechanism confirmed; harm narrower than claimed)

`x-breakclear-degraded` is the *only* degradation signal; `BackendClient` reads only the
status code — zero matches for "degraded" in `ios/`. A degraded 200 renders identically to a
full-confidence one. The verifier refuted the broad harm (cards in a degraded body are real
data; the headline is server-side honest) but confirmed the real harm is 2.4 plus the absence
of any "forecast unavailable" labeling (there's a "forecast only" freshness label but no
inverse). Fix is client-side detectable today via `hourly.isEmpty`; a body-level `degraded`
field is the durable contract fix.

## 3. The relative gate — a product decision to make explicit, not a bug

There is **no absolute air-quality ceiling anywhere in the pipeline**: the gate is
origin−25 OR one-full-category-better. Verified consequences: AQI-140 is recommended against
origin 180; at origin 250 the gate admits stay-mean up to 225 (same-category Very Unhealthy
cards with lodging links); the OR-branch certifies a 1-point improvement at category band
edges (200 vs 201 passes); negative-score cards ship (the product's own comfort-vs-drive
metric concludes the drive isn't worth it, and is ignored); summary copy says "dips to Very
Unhealthy" when the stay *worsens*. Every card is honestly labeled per-card (arrival
category, `stayMeanAQI`, `cleanThroughCheckout:false`, EPA colors) — the letter is honest,
but nothing frames the *set* as harm reduction rather than escape.

This is defensible — in regional mega-smoke, "less bad" can genuinely be the right advice,
and the alternative (an absolute bar) would return empties while a materially better
destination exists. Recommendation: **keep the relative gate, make the framing structural** —
an `advisory: "still-unhealthy"` field whenever stay-mean > 100, a minimum-delta AND on the
category branch (kills the 1-point pass), a score floor, and "worsens to" as the verb.

## 4. The medium tail (real, bounded, mostly labeled-but-implicit)

- **Null-origin gate disablement**: no monitor ≤40 km + no observing area ≤80 km + origin
  forecast unavailable → `originRefAQI` null → the gate passes *everything*; up to 6
  confident cards beside "No air-quality reading for your location," and the client's
  `comfort < 0.6` fallback paints the smoky gradient from no data.
- **Sites-feed loss silently falls back to reporting-area readings** (an area publishes its
  *worst* monitor — measured a full EPA category high), inflating `originRefAQI` and
  loosening the gate.
- **Sustained AirNow outage (>6h)** removes the monitor cross-check everywhere while raw
  CAMS cards (documented 2.7–3.6x smoke overestimate, errors −73 to +258 AQI) keep serving —
  labeled only implicitly (`current: null`).
- **Fingerprint-stale catalog after a catalog-changing deploy** can assert `EMPTY_HONEST`
  without having scored newly added destinations (deploy-coupled, ≤4h window, silent).
- **Staleness**: no age check on the catalog blob (up to 4h + 300s colo); iOS renders
  "Checked just now" over a 4h-old model — forecast age is decoded but rendered only in the
  web companion; the destination-detail endpoint ships zero freshness metadata; `EMPTY_NO_FORECAST`
  is colo-cached 300s (unlike the degraded branch) and #49 stretches it for browsers.
- **Budget mechanics**: the UTC-day rollover is 5 PM PDT (a Pacific afternoon exhaustion
  recovers at 5 PM, then can re-drain); a full Open-Meteo outage burns the day's budget in
  ~3h via spend-then-try catalog rebuilds, which then *blocks recovery* until rollover.
- **The 6-hour promise is model-only**: haversine × 1.35 / 75 km/h admits cards; measured
  per-route circuity reaches 1.96, so a max-budget card can be a ~8h real drive. Clients
  correct the displayed number but never retract the card (`isPrecise: false` is on the wire).
- **seenEscapes fallback**: a destination that dropped out of results (often *because its air
  worsened*) keeps rendering its old numbers on the pushed detail view, unlabeled.
- Destination surfaces label observations "now" with no timestamp; the detail screen's
  time-only observation stamp lacks the day qualification the Today header has.
- **#79 (filed by implementation 2026-08-12, priority high)**: the escapes endpoint's cache key is
  the snapped 0.05° cell but distances (and `siteReadingFor`) compute from the *raw* query
  coordinate — two coordinates in one cell share a cache entry, so **the same cached body can
  publish a different monitor's AQI depending on which neighbour warmed the cell**, and
  `budgetMinutesNeeded` can name a budget that returns nothing. Reproduced in production.
  **No live user impact today** (iOS snaps client-side; the web companion is not deployed) —
  it goes live with the web app, so it gates the *web* launch, and it likely interacts
  with #75.


---

*Sections 5 (launch gate) and 6 (launch decision) of the original report are omitted from this public copy. References to §5 and §6 above point to those omitted sections. API paths are described in words.*
