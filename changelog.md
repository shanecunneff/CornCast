# CornCast — Development Changelog

## Project Goals

Build a mobile-first spring corn snow departure-time calculator for ski mountaineers in the Indian Peaks Wilderness and Rocky Mountain National Park. The app should:

- Predict the corn snow window (open and close time) for a given peak, aspect, and overnight forecast
- Calculate the trailhead departure time needed to arrive during prime corn
- Incorporate live NRCS SNOTEL snowpack data as a background modifier
- Present results in plain English with score drivers explained
- Work well on iOS Safari in a browser without a native app
- Be physically grounded in snowpack thermodynamics and solar geometry
- Remain a single self-contained HTML file with no build tools or dependencies beyond Leaflet

---

## Session History

### Session 1 — v1-v5 (Foundation)

Built the working application from scratch. Core deliverables:

- Physics model: Corn window computed from first principles -- overnight freeze quality, solar radiation by aspect, melt delay, window duration, terrain factor
- 24 peaks with per-aspect terrain rubrics (Snow Retention x Aspect Purity x Wind Shelter)
- SNOTEL integration: Live NRCS data fetch via CORS proxy chain; SWE % of median used as a background modifier on crust quality and window duration
- Leaflet map with pie-chart markers showing per-aspect score by compass direction, color-coded green/amber/red
- Plan / Result / Map / About tabs -- mobile-first layout
- Score normalization: Raw score normalized to S. Arapaho SE face in March under optimal conditions = 100
- Month maturity model: Explicit seasonal penalties for grain coarsening (0-22%) and isothermal probability (5-70%), based on Colbeck (1987) and Pomeroy & Brun (2001)
- Approach time calculator: Linear regression fit to personal skinning pace data; speed = -7.8085 x grade + 2.4481

---

### Session 2 — v6

- Removed Copeland Lake SNOTEL station (#412, 8,320') -- too low-elevation to represent the alpine ski zone
- Added "What's Driving the Score" narrative card in Results
- Moved all model derivation prose and formulas to About tab only
- Fixed iconAnchor on Leaflet map markers

---

### Session 3 — v7

- SWE stations: Wild Basin (#1042) restored at corrected elevation 9,410'
- SWE fetch robustness: Dual URL format fallback per station
- Snow surface model: Replaced single "days since snowfall" with two-input depth + age model
  - depthFactor = min(1.0, depth_in / 8.0)
  - ageFactor = exp(-days x 0.10)
  - crustQ = max(0.05, depthFactor x ageFactor x maturityScale)
- Live crust preview on Plan tab
- Peak coordinates: Paiute and Pawnee corrected to USGS summit positions
- Map: Full viewport height on iOS using 100dvh

---

### Session 4 — v8

- Proxy fallback chain extended: allorigins -> corsproxy.io -> codetabs -> thingproxy -> direct
- Graceful SNOTEL failure: Model runs without SWE when all proxies fail
- USGS Topo basemap added as primary; OSM as toggle
- Hillshade overlay at 0.35 opacity

---

### Session 5 — v9

- Solar geometry refinements: Per-aspect sun-hit times and monthly radiation multipliers verified for 40N latitude
- Terrain shade delays: Hand-coded per-face delays for bowls with known ridgeline blocking
- Wind scour model: Aspect-specific wind exposure coefficients; dual thresholds for freeze quality (12 mph) vs. window duration (18 mph)
- June NE physics: Radiation multipliers corrected for high sun angle and northerly sunrise azimuth

---

### Session 6 — v10 (Full Coordinate Verification)

Major errors corrected (>1 mile displacement):
- Apache, Navajo, Shoshoni -- placed ~3 miles south; corrected to Continental Divide
- Sawtooth Mtn -- was using SNOTEL coordinates; corrected to 40.1269, -105.6252
- Red Deer Mtn -- corrected to 40.1120, -105.6320
- Ogalalla Peak -- 18 km error; corrected to 40.1703, -105.6667
- Lone Eagle Peak -- 4 km error; corrected to 40.0714, -105.6603
- Mt. Toll -- 2 km error; corrected to 40.0883, -105.6339
- Paiute Peak -- corrected to 40.0980, -105.6332

Minor fixes (<0.5 mile): Pawnee, Arikaree, Kiowa, Mt. Albion, N/S Arapaho.

---

### Session 7 — v11 (Physics Pipeline About Tab)

No model changes. About tab restructured into a unified Physics Pipeline -- seven steps, three modifier cards. Plain-English explanation before every formula; full literature sources on each step.

---

### Session 8 — v12 (Pack Ripeness + Open-Meteo Fetch)

Pack ripeness model: Replaced static monthly isothermal lookup with a computed packRipeness score from the past 10 days of actual weather data (heat load, above-freeze nights, rain, cold recovery).

Open-Meteo weather fetch: Live fetch auto-populates Plan tab fields. All values lapse-corrected to peak elevation.

Weather tile strip and ripeness trajectory chart added.

---

### Session 9 — v13 (Two-Layer Pack Model)

Replaced single-layer packRipeness with a two-layer model:

- Surface layer (0-30cm): 14-day rolling window. Scored 0-1 where 0 = cold-core, 1 = near-isothermal.
- Column layer (30cm-full depth): Full season from March 1. 4.5x slower response than surface.
- Combined: effectiveIsoPrb = surface x 0.60 + column x 0.40

Physics motivation: Jennings et al. (2018) found non-zero column cold content delayed melt onset by up to 5.7 hours at Niwot Ridge.

Fetch architecture:
- ERA5 archive: March 1 to yesterday (finalized reanalysis)
- Forecast: today to today+15
- Merged via Promise.all; forecast wins on overlap

---

### Session 10 — v13 patch (Strip + Dawn Temp Fixes)

Three bugs fixed:

Strip showing only 3 days: fcastEnd was target+15, causing Open-Meteo to reject requests for distant targets. Fixed: fcastEnd = today+15 unconditionally.

Strip anchored to today, not target: Fixed to anchor at targetIdx-10 through targetIdx+4.

Dawn temp not populating: Hourly data only exists for today to today+15. Fixed: fallback dawnTemp = overnight_low + 2F.

Tile redesign: 34px -> 52px width; each tile shows hi, lo, freeze icon, and dominant weather event.

---

### Session 11 — v14 (About Tab Rewrite)

No model changes. About tab rewritten for a general audience -- all version references removed, physics explained as timeless principles. Two-Layer Ripeness, Weather Data, Known Limitations, and Disclaimer sections expanded and clarified.

---

### Session 12 — v15 (Chart Direction Fix + Pack Health Reframing)

Bug: X-axis showing wrong month names. Root cause: getMonth() is 0-indexed; array index was off by 2. Fixed with direct named array.

Chart reframed so both lines read the same direction (high = better conditions):
- Surface freeze quality = 1 - surfaceRipeness (100% = maximum freeze potential)
- Column cold reserve = columnColdContent directly (100% = full cold reserve)

Both lines now start high in March and decline through the season. Threshold line relabeled from "isothermal" to "marginal."

Strip label and driver rows updated: all values display as high=good. Internal model values are inverted at display time only -- no model logic changed.

Score question (design decision -- no change): A warm column with a weak surface freeze still scores low. Physically correct -- no overnight refreeze means no crust regardless of column state.

---

### Session 13 — v16 (UX Fixes: auto tag, Open-Meteo deep link)

Dawn temp "auto" tag made dynamic. Previously hardcoded in HTML and always visible. Now hidden by default; shown only when prefillForecast() writes a live value; disappears when user overrides manually.

Open-Meteo link upgraded to deep link. Now builds a URL to the forecast API explorer pre-loaded with the selected peak's exact coordinates and the relevant variables in correct units. Link text updated to "Open-Meteo forecast for [Peak Name]".

---

### Session 14 — v17 (Score Clarity + Snow Surface Metric Rename)

"Crust" metric renamed to "Snow Surface" in the Results conditions card. The old name implied a holistic quality verdict; the metric specifically represents snow depth and freshness -- the raw material available to refreeze. One-liner added below the three metrics explaining what each one means.

Poor scores suppress departure times. Scores below 20 previously showed departure times from the 0.20h duration floor -- a 12-minute window nobody can time on a mountain. Now:
- Score < 20: departure card replaced with red "Conditions too poor to plan around"
- Theoretical window shown dimmed (42% opacity), labeled "Theoretical window (not actionable)"
- Physics output preserved for transparency

About tab score band 0-24 updated: "theoretical window only; too short to plan around."

---

### Session 17 — v20 (SNOTEL SNWD storm detection + orographic multiplier)

**Motivation:** Open-Meteo gridded snowfall systematically underestimates alpine snowfall at 12,000'+ terrain due to orographic enhancement that the ~9km model grid cannot resolve. SNOTEL SNWD (snow depth) sensors sit at 9,400–11,480' and measure actual fallen snow, making them far more reliable for storm detection at mountain elevation.

**SNOTEL SNWD integration (primary change):** New `nrcsSnwdUrl`, `parseSnwdCSV`, and `loadSNWD` functions mirror the existing SWE fetch architecture. Each basin's highest-elevation station is used as the primary SNWD source (Niwot Ridge #663 at 11,480' for boulder, Sawtooth #1251 at 9,640' for stvrain, Joe Wright #551 at 10,020' for bigthompson), with fallback down the station list. `loadSNWD` is fired as a third Promise inside `fetchWeatherData` in parallel with the two Open-Meteo fetches, so it adds no latency to the critical path. `processWeatherData` receives `snwdData` as a new parameter.

**Storm detection logic:** For **past target dates** (target ≤ today), the Open-Meteo storm scan is replaced by SNOTEL SNWD daily depth-change deltas over a 14-day lookback. If SNOTEL shows no recent accumulation (no delta ≥ 0.5"), `snowDaysSince` and `snowDepthEst` are set to null — trusting the sensor reading over the gridded estimate. For **future target dates**, Open-Meteo remains the source with a new `OROG_MULT = 1.5` orographic uplift factor applied to forecast-flagged days only. Archive days are not multiplied (they are real ERA5 observations, not forecasts).

**isForecast flag preserved on allDays:** Previously stripped during the dayMap merge. Now preserved via `Object.assign` so the storm detection scan can distinguish forecast vs. archive days for the OROG_MULT decision.

**Source transparency in UI:** `snowSource` string added to the `wd` return object. `prefillForecast` updates the "Recent Snowfall" section header subtitle dynamically: SNOTEL source shows station name and elevation; Open-Meteo shows orographic note. A `snow-source-note` div below the crust preview shows a one-line explanation of where the depth estimate came from. Future dates get an explicit "verify against SNOTEL or OpenSnow" note.

**Temperature limitation documented (no model change):** Confirmed temperatures and wind are requested in correct imperial units. Lapse rate accuracy assessed: 3.5°F/1000ft is a reasonable climatological mean but has ±1.5°F/1000ft variability per event. Clear-calm nights (best corn nights) are most prone to temperature inversions that make the model over-predict summit coldness. Magnitude estimate: ±3-8°F systematic bias on optimal nights. SNOTEL TMIN from Niwot Ridge (#663) as a cross-check on auto-filled overnight lows is a noted future improvement.

---

### Session 16 — v19 (Unit audit: snowfall cm→in fix + dynamic lapse correction)

Two bugs corrected following observation of unrealistically cold snowpack state and inflated storm depth estimates in auto-filled fields.

**snowfall_sum unit bug (primary):** Open-Meteo's `snowfall_sum` field is always returned in centimeters regardless of `precipitation_unit=inch` — that parameter only converts liquid precipitation (`precipitation_sum`). Fixed in `parseDailyJson`: multiply by 0.3937 (cm → inches) at the parse point. Downstream thresholds (storm detection ≥ 0.5", CCC boost > 1.0", tile display ≥ 0.3", depth estimate) were all written for inches and are now correct. Effect: eliminates artificially elevated column cold content from spurious snowfall counts throughout the season; `snowDepthEst` auto-fill now shows realistic inch values (was ~2.5× overstated).

**Dynamic lapse correction:** `GRID_ELEV_FT = 10500` was hardcoded as the assumed ERA5 grid cell elevation. Open-Meteo returns `json.elevation` (meters) in every response — the actual grid cell elevation for the queried coordinates. `elevCorrectTemp()` updated to accept an optional `gridElevFt` parameter (falls back to 10500 if absent for safety). `parseDailyJson` reads `json.elevation * 3.28084` and passes it through for both tmin and tmax. `processWeatherData` does the same for the hourly dawn temp pulled from the forecast endpoint. Grid elevation for Indian Peaks / RMNP coordinates typically runs 8,800–11,200 ft; the dynamic value removes a potential 2–9°F lapse correction error depending on peak.

No model logic, thresholds, or physics coefficients changed. Temperature and wind unit requests (`temperature_unit=fahrenheit`, `wind_speed_unit=mph`) were confirmed correct in both API URLs — those were not the source of the issue.

---

### Session 15 — v18 (Driver Narrative Restructured to Physics Pipeline)

"What's Driving the Score" completely restructured from a flat list into seven labeled pipeline steps matching the About tab physics sequence.

Step 1 - Overnight Freeze Quality: overnight low + sky clarity + wind scour as separate rows. Wind shows aspect exposure % and freeze quality penalty in points.

Modifier - Snowpack Condition: surface freeze quality bar + column cold reserve bar, positioned after Step 1 where they enter the physics chain.

Steps 2-3 - Solar Input & Sun-Hit Time: aspect radiation % for the current month shown explicitly, then morning cloud attenuation to produce effective radiation.

Step 4 - Melt Onset -> Corn Opens: dawn temp + daytime high + morning rise rate interpreted as thermal debt payoff speed.

Step 5 - Snow Surface: depth/age rows; SWE modifier positioned here where it feeds into crustQ.

Step 6 - Window Duration -> Corn Closes: duration with column cold reserve callout; wind duration penalty (>18 mph) shown separately from the Step 1 scour penalty -- different physical mechanisms.

Step 7 - Terrain Factor: star rubric moved inline into the driver row.

CSS added: .driver-step-head -- teal left-border section label, uppercase tracking.

Bug fixed: unescaped apostrophe (won't) in a single-quoted JS string in buildDrivers silently killed the entire script, emptying the peak selector. Caught via Node.js Function() syntax check. Fixed to "will not."
---

### Session [N] — v21 (Jan 1 Column Cold Content Window)

**Motivation:** A warm January–February depletes column cold content before March 1, but the
previous archive fetch started on Mar 1, so the model never saw it. This year's Front Range
warm spell is a clear example where this materially affects accuracy.

**Changes:**
- Archive fetch start date extended from Mar 1 to Jan 1 (`cccStart = year + '-01-01'`).
- `processWeatherData` splits the extended archive into `preCCCDays` (Jan/Feb, CCC-only) and
  `seasonArchiveDays` (Mar+, display/strip/surface ripeness). The weather strip, ripeness
  trajectory chart, and `allDays` remain anchored to March 1 — no display behavior changed.
- `seasonDaysForCCC` now = `preCCCDays.concat(allDays.slice(0, tgtIdx))`, feeding the full
  Jan 1 → target window into `computeColumnColdContent`.
- `computeColumnColdContent` auto-detects pre-season data: if `seasonDays[0]` is in Jan or Feb,
  starts from baseline 0.95 (deep winter cold reserve) instead of the month-specific Mar/Apr/May
  baseline. The existing linear ageFactor (0.30 oldest → 1.00 most recent) already correctly
  discounts the older January days — no coefficient changes required.
- Archive URL hint text updated to reflect Jan 1 start.

**Physics rationale:** Pre-season warm spells reduce column cold content through the same
mechanism as in-season warmth — above-40°F maxima and above-freezing nights penetrate the
column slowly (4.5× slower than surface). A warm Jan/Feb genuinely depletes cold reserve
that would otherwise persist into April. The 0.95 starting baseline reflects that on Jan 1,
the snowpack at 12,000' is typically at or near maximum cold content for the water year.

---
Session 18 — v21 (CCC pre-season extension + two bug fixes)
Motivation: The 2025 Front Range winter had significant above-40°F warm spells in January and February. The v20 archive window (Mar 1 → yesterday) missed all of that pre-season column depletion, causing CornCast to overestimate cold reserve heading into spring. Extending the archive start to Jan 1 captures warm-winter depletion.
Archive start extended Jan 1 → Mar 1: seasonStart in fetchWeatherData changed from year + '-03-01' to year + '-01-01'. hasPreSeason flag detects when seasonDays[0] falls in Jan or Feb (getMonth() < 2). When true, CCC initializes at 0.95 (deep-winter baseline) instead of the month-specific SEASONAL_BASELINE.
Bug 1 fixed — snow bonus missing ageFactor: if (d.snowfall > 1.0) ccc += 0.04 had no age weighting. A January storm counted identically to a March storm, adding unweighted +0.28 for 7 winter storms that completely overwhelmed the correctly age-weighted -0.10 warm-day depletion. Fixed: ccc += 0.04 * ageFactor. Diagnosis: warm winters were producing higher CCC than the Mar 1 model — backwards.
Bug 2 fixed — cold rebuild running in pre-season: The colCold term (tmax < 25°F × 0.003 × ageFactor) ran unconditionally on every Jan/Feb day. Cold January days are normal winter — they don't rebuild anything beyond the 0.95 starting baseline. The cold rebuild coefficient dominates over the heat depletion coefficient across 60 days of sub-25°F weather. Fixed: colCold and snow bonus gated behind if (d.date >= d.date.slice(0,4) + '-03-01'). Pre-season days apply only heat depletion and warm-night penetration terms.
Net physics effect: For a warm pre-season (multiple days above 40°F in Jan/Feb), Jan 1 model now correctly produces slightly lower CCC than the Mar 1 baseline. For a normal or cold pre-season, the difference is minimal. The correction compounds meaningfully for April and May targets where pre-season history has more weight.

----
### Session [N] — v22 (Chart: Panning + Temp Lines + Snowfall Bars)

Snowpack Condition Trajectory chart enhanced in three ways:

**Pannable canvas:** Canvas now renders at full season width using 7px/day fixed density (~950px for a full season). Wrapped in an `overflow-x:auto;-webkit-overflow-scrolling:touch` scroll container. Chart auto-scrolls on render so target/today is visible at 65% from the left edge, leaving earlier history accessible by panning left.

**Tmax/Tmin temperature lines:** Each trajectory point now carries `tmin`, `tmax`, and `snowfall` fields (added to `ripenessTrajectory.push`). Tmax rendered as warm red-orange (rgba 255,100,55), Tmin as cool blue-gray (rgba 140,205,245), both at 1.4px width and ~60% opacity to stay visually subordinate to the pack health lines. Solid for archive, dashed for forecast — same convention as existing lines. Mapped to a fixed left Y axis (-10°F–65°F) with reference lines at 0°F, 32°F (bold, labeled), and 50°F. Left padding increased from 14 to 32px to accommodate °F labels.

**Snowfall bars:** Per-day snowfall rendered as vertical bars in the bottom 16px strip of the chart area. Archive bars at 75% opacity, forecast at 40%. Bars ≥4" get a ❄ glyph above them. Scaled so 10" = full strip height. Visually separates snowfall events from the continuous pack health and temperature lines above.

Canvas height increased from 200px to 240px to accommodate the snow bar strip without crowding the data area.

Legend updated: Surface freeze · Col. cold · Hi°F · Lo°F · Snow · Fcst. Caption updated to note panning.

---


**v22b — Pre-season (Jan/Feb) chart extension:**
`ripenessTrajectory` now prepended with `preCCCDays` (Jan 1 → Feb 28). Each pre-season 
point carries `isPreSeason:true`, `surfaceRipeness:null`, and a running CCC value computed 
incrementally. `marchFirstIdx` added to the `wd` return object so the chart knows where 
pre-season ends. Chart changes: pre-season region shaded with a dim overlay and a rotated 
"Pre-season" label; vertical dashed "Mar 1" marker at the boundary; SFQ (orange) line 
starts at marchFirstIdx, skipping nulls; CCR (blue), Tmax, Tmin, and snowfall bars render 
across the full Jan 1 → target range. Auto-scroll focus unchanged — still centers on target/today.
## Architecture Decisions (Standing)
Bug fix — CCC trajectory seam at Mar 1: Season trajectory loop was passing only 
allDays.slice(0,j) into computeColumnColdContent, losing all pre-season history at 
the Mar 1 boundary. CCC reset to SEASONAL_BASELINE[3]=0.85 on Mar 1 regardless of 
Jan/Feb history, creating a visible step-discontinuity. Fixed: sDays now 
preCCCDays.concat(allDays.slice(0,j)), matching the logic used for the final target CCC.

-----

| Decision | Rationale |
|---|---|
| Physics-first, not lookup tables | Every input change ripples through the full chain; results are always internally consistent |
| Single HTML file | Deployable as a file share or GitHub Pages with no build pipeline; auditable by a single developer |
| SWE as nudge, not gate | A cold clear night at 75% median SWE still scores 80+; SWE informs direction, does not dominate |
| Terrain factor is explicitly subjective | Hand-coded from route descriptions and field observation; documented as known limitation |
| Score normalized to S. Arapaho SE | Range benchmark corn objective -- clean SE couloir, reliable coverage, consistent solar exposure |
| ERA5 archive + 16-day forecast | ERA5 is finalized, higher accuracy for the past; forecast capped at 15 days within free tier |
| Score < 20 suppresses departure times | A 12-minute theoretical window is not actionable on a mountain approach |
| High = good framing throughout UI | Chart, strip label, and driver bars all display pack health as high = better; internal model values inverted at display time only |

---

## Open Questions / Future Work

- Dust-on-snow modifier: Dust lowers albedo, advances onset 20-45 min, compresses window. CAIC publishes dust data; API integration is feasible.
- Multi-aspect routing: Chain corn windows by aspect segment for descents crossing multiple exposures.
- Column coefficient validation: Calibration pass against Niwot Ridge snow temperature sensor data (niwot.colorado.edu LTER).
- Ensemble forecast uncertainty: Replace linear shading proxy (1.5%/day) with true ECMWF ensemble spread.
- Dynamic season start: March 1 is arbitrary; late-season snowpack built in February would benefit from earlier start.
- Terrain star rubric duplication: Stars now appear in Step 7 driver row AND the conditions card metric row above. Consider removing from conditions card.
