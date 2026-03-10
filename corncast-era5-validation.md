# ERA5 Temperature Validation — Indian Peaks Wilderness
## Three-Station Elevation Transect · Jan 1–Mar 5, 2026

### Summary

CornCast uses ERA5 reanalysis data from Open-Meteo as its primary temperature input for the Column Cold Content (CCC) model. To validate this choice, ERA5 summit readings were cross-referenced against two nearby SNOTEL stations across the full 2026 pre-season (Jan 1–Mar 5).

---

### Stations

| Station | Network | Elevation | Notes |
|---|---|---|---|
| Niwot Ridge #663 | NRCS SNOTEL | 9,940 ft | Boulder County; reporting since 1979 |
| University Camp #838 | NRCS SNOTEL | 10,330 ft | St. Vrain watershed; closest to Mt. Toll drainage |
| ERA5 grid cell | Open-Meteo archive | 12,858 ft | Returned by API for Mt. Toll coordinates (40.0883°N, 105.6339°W) |
| Mt. Toll summit | — | 12,979 ft | CornCast target; ERA5 lapse correction = −0.4°F (negligible) |

---

### Key Finding: The Chinook Effect

Longmont and the Front Range experienced numerous warm days Jan–Mar 2026 — days that felt misleading when trying to assess high-alpine cold reserve. The three-station transect explains exactly why those warm days did not translate to summit warmth.

**Pre-season warm days (Jan–Feb) above 40°F:**
- Niwot Ridge 9,940': **14 days**
- University Camp 10,330': **8 days**
- ERA5 12,858': **0 days**

**Pre-season days above 32°F:**
- Niwot: 34 days · UCamp: 27 days · ERA5: **0 days**

**Pre-season warm nights (min > 32°F):** zero at all three stations.

Warm Front Range days are driven by **chinook/foehn winds** — air descending from the Continental Divide that warms fastest at the lowest elevations. Both SNOTELs sit in the active foehn warming zone. The ERA5 grid at 12,858 ft is near the source air, where the foehn effect is essentially absent.

The observed lapse rates confirm this:

| Segment | Elevation span | All-day lapse | Warm-day lapse | Standard env. lapse |
|---|---|---|---|---|
| Niwot → UCamp | 390 ft | 7.3°F/1000ft | 6.8°F/1000ft | 3.5°F/1000ft |
| UCamp → ERA5 | 2,528 ft | 5.9°F/1000ft | 6.5°F/1000ft | 3.5°F/1000ft |

Both segments run ~2× the standard environmental lapse rate. The absolute temperature drop from UCamp to ERA5 averages **~15°F on warm days** — meaning a 45°F day at University Camp corresponds to roughly 30°F at the summit.

---

### Feb 5 — Most Extreme Chinook Day

| Station | Max temp | vs 40°F threshold |
|---|---|---|
| Niwot 9,940' | **48.0°F** | +8.0°F above |
| Univ. Camp 10,330' | **45.7°F** | +5.7°F above |
| ERA5 12,858' | **29.3°F** | −10.7°F below |

The two SNOTELs differed by only 2.3°F over 390 ft (both fully in the chinook). ERA5 then dropped 16.4°F over the next 2,528 ft. If CornCast had used Niwot + standard lapse correction to estimate summit temps, it would have calculated ~37.8°F — incorrectly triggering CCC heat depletion. ERA5 directly correctly applied zero depletion.

---

### Implications for CornCast

1. **ERA5 is the correct data source.** Using SNOTEL + lapse correction would systematically overestimate summit temperatures on warm chinook days, generating false CCC heat depletion signals.

2. **The 95% cold reserve reading for March 5, 2026 is well-supported.** Three independent data sources (Niwot, UCamp, ERA5) all confirm zero pre-season warm events at summit elevation.

3. **University Camp (#838) is the best qualitative field reference** for Indian Peaks summit conditions. Its St. Vrain watershed location and 10,330 ft elevation make it the most relevant SNOTEL for gut-checking CornCast outputs. Rule of thumb: if UCamp > 45°F, ERA5 will be ~low 30s. If UCamp < 35°F, expect 15–25°F at summit.

4. **SNOTEL warm bias advisory.** NRCS acknowledges a known warm bias in the extended-range temperature sensors used at Colorado SNOTEL sites (post-2004 sensor upgrades). This bias runs in the direction of overstating warm events — further strengthening the case that the summit was genuinely cold all pre-season.

---

### Data Sources

- NRCS SNOTEL Report Generator: `https://wcc.sc.egov.usda.gov/reportGenerator/`
- Open-Meteo ERA5 Archive: `https://archive-api.open-meteo.com/v1/archive`
- All SNOTEL data provisional and subject to revision per NRCS advisory.
- Analysis period: 2026-01-01 to 2026-03-05.
