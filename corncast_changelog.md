# CornCast — Physics Changelog

### v37b — Add missing descent routes (2026-03-19)

Added FRSK-verified descent polylines to 7 asp entries that were
previously rendering without route lines on the map:

Mt. Audubon:
- N face: Coney Couloirs (drops toward Coney Lakes)
- E face: East Cirque (replaces 2-pt Autobon with cleaner line)
- S face: South Face
- W face: Crooked Couloir

Paiute Peak:
- E face: Little Blue Couloir
- S face: Curvaceous Couloir
- W face: South Slope

All coordinates sourced from FRSK CalTopo GeoJSON (caltopo.com/m/12M99),
confirmed as FRSK descent-pattern lines (dashed-circle style).

### v37a — Peak coordinate fixes, approach lines removed (2026-03-19)

Fixed incorrect summit coordinates for 6 peaks identified via
USGS Topo / GNIS verification:
- Hallett Peak: was 2.5mi too far east (40.2981/-105.6447 →
  40.3033/-105.6861)
- Terra Tomah: was 5.5mi too far south/east — peak removed from
  PK array entirely. Terra Tomah is a remote RMNP peak accessible
  only from Trail Ridge Road west side with no practical spring
  corn approach for Front Range users.
- Pagoda Mtn: minor 0.4mi correction (40.2431 → 40.2492)
- Mt. Meeker: minor 0.2mi correction (40.2456 → 40.2486)
- Mt. Audubon: longitude correction (-105.6256 → -105.6246)
- Red Deer Mtn: latitude correction from wrong drainage
  (40.1120 → 40.1379)

Route coordinate top-points updated to match corrected summit
locations for all fixed peaks.

Approach polylines removed from map rendering — approach route
data (app field) preserved in PK array for future implementation
when accurate trail GPS data is available. Approach legend entry
removed.

### v37 — Ski descent polylines + approach routes (2026-03-19)

Added ski line polylines to the Map tab. After calculating conditions,
each aspect's descent route is drawn as a colored polyline matching the
corn score color (green/amber/red). Before any calculation, lines show
in neutral blue. Tapping a line opens the peak popup.

Added approach route polylines as thin amber dashed lines showing the
skin track from trailhead to ski terrain for 12 peaks. Approach lines
are always neutral — they do not change color based on conditions.

Added `sl` (slope angle in degrees) to all ski faces in the PK array.
Used by the physics model in v38 (slope geometry correction session).

Added Plan tab auto-population: selecting a peak pre-fills the approach
distance from trailhead data. Selecting an aspect pre-fills elevation
gain as ski zone elevation minus trailhead elevation. A hint line shows
the trailhead name. Both values remain editable.

Route coordinates sourced from FRSK CalTopo master map (caltopo.com/m/12M99),
converted from GeoJSON [lon,lat] to PK array [lat,lon] format. RMNP
routes (Hallett, Longs) are topo estimates pending TNT4 verification.

### v36a — Result tab UX improvements (2026-03-18 MST)
Result tab now scrolls to top on every load.
Section reorder: Hourly Conditions timeline moved immediately
below departure times — the most actionable information now
appears before the score detail and driver narrative.
Score duplicate removed from departure row — Opens, Closes,
Duration remain. Score displayed once in Conditions card at
larger size (2.2rem).
Corn score chart: height 160px → 200px. Bar labels now appear
above short bars instead of being hidden. Score labels shown for
all bars ≥20 score and ≥5px wide. Red 50-line replaced with
neutral white (red implied a threshold warning, which is wrong).
All gridline opacity reduced. Chart legend font size increased
with teal aspect label.
Ripeness chart: orange forecast uncertainty shading and dashed
border lines removed — dashed CCR/SFQ lines already distinguish
forecast from archive clearly. Right Y-axis gridline opacity
reduced from 0.40 to 0.20. Legend expanded to two lines with
clearer metric labels.

### v36 — Ensemble forecast confidence bands (2026-03-18 MST)
**New: GFS ensemble fetch** added as a 4th parallel request in
fetchWeatherData, using Open-Meteo ensemble API with gfs_seamless
(31 members). Fails silently — if unavailable, app runs exactly
as before with no confidence bands visible.
**ensembleBands computed in processWeatherData:** For each
forecast date, collects temperature_2m_min/max values across all
GFS ensemble members, sorts them, and computes lapse-corrected
P10/P50/P90 percentiles. Passed through as wd.ensembleBands.
**Confidence bands in drawCornScoreChart:** For forecast bars
days 5–15, runs cornWindow with P10 and P90 overnight low inputs
to compute a score range. Draws a thin semi-transparent white
band extending above and below the main bar proportional to
ensemble spread. First 4 forecast days are high-confidence and
show no band. Band opacity scales with ensemble spread width.
Chart legend updated to note "White range = ensemble uncertainty
(days 5–15)."

### v35d — hotfix: remove freezinglevel_height from forecastUrl (2026-03-18 MST)
freezinglevel_height is not reliably supported by Open-Meteo's
models=best_match endpoint. Including it in forecastUrl daily=
caused the entire forecast fetch to return an HTTP error, which
was caught as null. This emptied forecastDays, leaving allDays
with archive data only (through yesterday). Any selected date
from today onward was not found in allDays, causing the strip
to appear frozen and the beyondWindow warning to fire even for
dates within the 16-day window.
Removed from forecastUrl. The freezeLevelFt gate in cornWindow
receives null and is safely bypassed. The v35c beyondWindow
detection and isFcst simplification remain correct and intact.

### v35c — hotfix: strip tracking + isFcst simplification (2026-03-17 MST)
Two fixes to renderWeatherStrip:
**isFcst simplification:** Replaced the marchFirstIdx offset
calculation introduced in v35b with direct use of d.isForecast,
which is already set correctly on each allDays entry by
processWeatherData. No index math needed.
**Strip tracks selected date:** When the selected date is beyond
the 16-day forecast window (today+15), targetIdx was silently
falling back to days.length-1 for every out-of-range date,
making the strip appear frozen regardless of which future date
was selected. Fixed by detecting the out-of-range case, anchoring
the strip on today's position, and showing an amber note in the
label bar: "Forecast data available through [date]."

### v35b — hotfix: ripenessTrajectory index offset in weather strip (2026-03-17 MST)
renderWeatherStrip was indexing wd.ripenessTrajectory with ti,
where ti is an index into allDays (March 1 = index 0). But
ripenessTrajectory starts January 1 and includes marchFirstIdx
pre-season entries before March 1. This caused isFcst to always
read from a pre-season archive entry, meaning forecast tiles were
never rendered as forecast tiles. Fixed by offsetting the
trajectory index by wd.marchFirstIdx.

### v35a — hotfix: remove freezinglevel_height from archive URL (2026-03-17 MST)
ERA5 archive API does not support freezinglevel_height. Adding it
caused the archive fetch to fail entirely, breaking CCC, surface
ripeness, and all weather data. Removed from archiveUrl daily=
parameter. Forecast URL retains it — freezeLevelFt is now populated
for forecast dates only (today → today+15). Past target dates fall
back to null gracefully. Gate logic and driver narrative unchanged.

### v35 — Freezing level gate (2026-03-17 MST)
**freezinglevel_height added to both archive and forecast daily=
parameters.** Open-Meteo returns the elevation (meters, converted
to feet) at which temperature crosses 0°C overnight.
**Freezing level gate in cornWindow:** if freezinglevel_height
shows the freezing level was above ski zone elevation + 300ft,
effFQ is reduced to 15% of its computed value. This eliminates
false positives on warm nights where lapse correction produces a
technically sub-freezing summit temperature but the freeze never
physically reached the face. The gate is a correction only —
it cannot increase scores.
**Driver narrative:** Step 1 now shows a warning when the
freezing level was above ski zone elevation, or a confirmation
note when it was well below. Users can see directly whether the
overnight freeze was confirmed to have reached the terrain.
No RAW_MAX recalibration needed — this change only reduces
scores in edge cases.

### v33b — Physics calibration: packReadiness architecture (2026-03-17)
Fundamental physics rework grounded in field literature
(frontrangeskimo.com, OpenSnow, CU snow hydrology, Colbeck 1987/1997):

**New: computePackReadiness()** — composite score (0–1) of grain
maturity (ftCyclesTotal/8), cycling reliability (isoPrb bell curve
peaking at 0.35), and CCC threshold model (functional range 0.35–0.65).
Captures the physical reality that April–May isothermal cycling packs
produce more reliable, higher-quality corn than cold March packs or
collapsing June packs on identical overnight conditions.

**New: ftCyclesTotal** — cumulative freeze-thaw cycles from March 1.
Grain metamorphism is irreversible: a May pack that cycled 15 times
since March 1 has mature polycrystals regardless of recent cold snaps.
Replaces 14-day ftCycles in grain maturity computation.

**ccFreezeBonus capped at CCC=0.65.** High cold reserve signals a
pre-corn winter pack state, not better corn quality.

**Melt delay: cap 4.5h → 2.0h**, dth/6 (was /8), minDenom 1.0 (was 0.8).
Corrects unrealistic 11 AM openings on very cold SE faces.

**isoPrb melt factor activates only above 0.50.** Active corn cycling
packs (isoPrb 0.25–0.45) are no longer penalized at melt onset.

**Afternoon high threshold: 32°F → 46°F** (ski zone), coefficient
0.065 → 0.085. OpenSnow: "30s–low 40s°F highs are optimal for corn."

**packReadiness replaces packWinFactor in duration.** Season-appropriate
pack cycling now drives window duration rather than raw CCC.

**FQ bell curve** peaks at effFQ=0.55 (teens–20°F = optimal).
Step function (30/22/12/5) removed.

**crustQ bell curve** peaks at 0.42 (consolidated polycrystal structure).
Linear 0–10 removed. Fresh angular storm snow no longer scores highest.

**Graduated short-window penalty:** <0.35h ×0.20, <0.60h ×0.55,
<0.90h ×0.80. Replaces binary 0.75h cliff. Literature: 1–2h is
normal good corn.

**packReadiness rawScore multiplier:** April/May scores 10–13% above
March/June on identical overnight inputs. Season ordering from pure
physics: April ≈ May > March > June.

**Effect:** May good days (15°F, 40" consolidated) score ~60–66 instead
of 15–20. April prime scores 80–88. March excellent ~75–82. June modest
correctly. RAW_MAX unchanged at 71.8.
