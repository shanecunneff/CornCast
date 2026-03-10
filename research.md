# CornCast -- Research & Literature References

Sources that directly inform model physics, coefficients, or design decisions. Organized by the pipeline step each source most directly supports.

Current version: v18. Metric terminology note: "surface ripeness" and "column cold content" are internal model variable names. In the UI these are displayed inverted as "surface freeze quality" (1 - surfaceRipeness) and "column cold reserve" (columnColdContent directly) so that high always means better skiing conditions.

---

## Snowpack Thermodynamics -- Surface and Column

**Fierz, C. (2011).** Temperature profile of snowpack and heat wave attenuation. In Encyclopedia of Snow, Ice and Glaciers, Springer.
- Thermal attenuation of surface temperature signals with depth in a seasonal snowpack
- Column layer warms approximately 4.5x slower than the surface -- basis for the asymmetric surface/column timescales in the two-layer model
- Used in: Two-Layer model, column cold content coefficient rationale

**Male, D.H. & Granger, R.J. (1981).** Snow surface energy exchange. Water Resources Research, 17(3).
- Energy balance at the snow surface; longwave radiative exchange between snow surface and atmosphere
- Basis for the clear-sky freeze bonus: clear nights lose heat to space via longwave emission, cloudy nights receive it back
- Used in: Step 1 (Overnight Freeze Quality), Step 4 (Melt Onset)

**Jennings, K.S., Molotch, N.P., & Trujillo, E. (2018).** Spatial variation of snow water equivalent and snowmelt timing across a mixed-elevation watershed. The Cryosphere, 12.
- Column cold content delays snowmelt onset by up to 5.7 hours at Niwot Ridge (11,500') compared to a fully isothermal pack
- Key empirical basis for the column layer's 40% weighting in the combined ripeness score
- Used in: Two-Layer model modifier

**DeWalle, D.R. & Rango, A. (2008).** Principles of Snow Hydrology. Cambridge University Press. Chapter 4.
- Cold content definition: total energy required to bring the snowpack to isothermal (0 C) from its current temperature state
- Seasonal accumulation and release dynamics
- Used in: Two-Layer model, column cold content definition

**Sturm, M. & Holmgren, J. (1998).** Thermal conductivity and heat transfer in natural snow. Journal of Geophysical Research.
- Thermal conductivity of natural snowpacks varies with snow type, density, and temperature gradient
- Supports the model's assumption that column layer responds on a longer timescale than the surface
- Used in: Two-Layer model, column coefficient rationale

**Pomeroy, J.W. & Brun, E. (2001).** Physical properties of snow. In Snow Ecology, Cambridge University Press.
- Comprehensive treatment of snow physical properties: density, thermal conductivity, grain size and shape, liquid water content
- Basis for understanding surface layer response to diurnal forcing
- Used in: Step 1 (Overnight Freeze Quality)

---

## Grain Metamorphism & Crust Quality

**Colbeck, S.C. (1987).** Review of metamorphism and classification of seasonal snow cover crystals. IAHS Publication 162.
- Equilibrium metamorphism: sharp angular crystals -> large rounded melt-freeze polycrystals over weeks to months of repeated freeze-thaw cycling
- Rounded grains have less contact area -> weaker bonds when refrozen -> thinner, more fragile crust from the same overnight temperature
- Basis for monthly grain rounding penalty (0% March -> 22% June)
- Used in: Seasonal Maturity modifier, Step 5 (Snow Surface)

**Colbeck, S.C. (1979).** Water movement through heterogeneous snow. Journal of Colloid and Interface Science.
- Ice-to-ice bond strength in melt-freeze snow; bonding mechanism at grain contact points
- Used in: Step 5 (Snow Surface)

**Colbeck, S.C. (1986).** Statistics of coarsening in water-saturated granular materials. Acta Metallurgica, 34.
- Grain coarsening kinetics in wet snow; rate of grain size increase during melt-freeze cycling
- Used in: Step 5 (Snow Surface)

**Colbeck, S.C. (1997).** A review of sintering in seasonal snow. CRREL Report 97-10.
- Sintering and grain equilibrium in the consolidated spring pack; why aged-pack grain size eventually stabilizes
- Basis for the consolidated pack floor: crustQ = 0.35 x (depth/8) x seasonal_scale
- Used in: Step 5 (Snow Surface)

**Fierz, C., Armstrong, R.L., Durand, Y., et al. (2009).** International Classification for Seasonal Snow on the Ground. UNESCO-IHP Technical Documents in Hydrology No. 83.
- Standard classification of melt-freeze crust grain types (MFcr)
- Used in: Step 5 (Snow Surface)

**Armstrong, R.L. & Brun, E. (eds.) (2008).** Snow and Climate. Cambridge University Press.
- Seasonal snowpack evolution; relationships between grain type, metamorphism rate, and thermal properties
- Used in: Seasonal Maturity modifier

---

## Solar Geometry & Slope Irradiance

**Iqbal, M. (1983).** An Introduction to Solar Radiation. Academic Press.
- Solar geometry at 40N latitude; direct solar irradiance on tilted surfaces as a function of slope angle, aspect, and solar declination
- Basis for all aspect radiation multipliers and their monthly variation (March through June)
- Used in: Step 2 (Solar Radiation by Aspect), Step 3 (Sun-Hit Time)

---

## Snowmelt Energy Balance

**Kustas, W.P. & Rango, A. (1994).** A simple energy budget algorithm for the snowmelt runoff model. Water Resources Research, 30(5).
- Simplified energy budget for snowmelt onset; partitioning of available radiation into sensible heat, latent heat, and melt energy
- Basis for the melt delay formula: meltDelay = (freezeQuality x temperature deficit at dawn) / (radiation x warm-up rate)
- Used in: Step 4 (Melt Onset / Corn Opens)

---

## Seasonal Snowpack Trends -- Colorado Front Range

**Clow, D.W., Nanus, L., Verdin, K.L., & Schmidt, J. (2012).** Evaluation of SNODAS snow depth and SWE estimates for the Colorado Rocky Mountains. Journal of Geophysical Research: Atmospheres.
- Snowpack trends and variability at elevation in the southern Rocky Mountains
- Supports the graduated isothermal probability schedule (5% March -> 70% June) for the 11,000-13,500' elevation band
- Used in: Seasonal Maturity modifier

---

## Field Sources & Route Information

**Roach, G. (2001).** Colorado's Indian Peaks Wilderness. Fulcrum Publishing.
- Route descriptions and approach information for Indian Peaks peaks and aspects
- Primary source for terrain factor ratings (Snow Retention, Aspect Purity, Wind Shelter) and per-face shade delay estimates

**frontrangeskimo.com** -- Trip reports and aspect-specific observation notes for Indian Peaks and RMNP objectives.
- Supplementary source for terrain factor ratings; contemporary field observations

---

## Peak Coordinate Sources

**TopoZone.com / USGS GNIS** -- USGS Geographic Names Information System; primary coordinate source for all peaks (v10 verification).

**Peakbagger.com** -- Cross-reference; community-verified summit coordinates.

**PeakVisor.com** -- Cross-reference; 3D terrain visualization confirms which coordinate lands on the actual summit.

**climber.org** -- USGS NAD27 peak database; used for peaks requiring WGS84 conversion.

---

## Weather Data Infrastructure

**Open-Meteo (open-meteo.com)**
- ERA5 archive endpoint (archive-api.open-meteo.com/v1/archive): ECMWF ERA5-Land reanalysis, finalized ~1-2 day latency, ~9km grid
- Forecast endpoint (api.open-meteo.com/v1/forecast): GFS/ECMWF best-match, up to 16-day horizon, free tier
- Variables used: temperature_2m_max, temperature_2m_min, precipitation_sum, snowfall_sum, windspeed_10m_max, cloudcover_mean, hourly temperature_2m (for 6 AM dawn temp)
- Deep link format: open-meteo.com/en/docs#latitude=LAT&longitude=LON&daily=...&temperature_unit=fahrenheit&wind_speed_unit=mph&timezone=America/Denver

**USDA Natural Resources Conservation Service -- SNOTEL**
- Station network measuring snow water equivalent, snow depth, air temperature, and precipitation
- CornCast uses SWE as percentage of 1991-2020 median for the date
- Stations: Niwot Ridge #663 (11,480'), University Camp #838 (10,300'), Sawtooth #1251 (10,360'), Wild Basin #1042 (9,410'), Bear Lake #322 (9,450'), Joe Wright #551 (10,020')
- Elevation weight formula: w = 1 / (1 + |station_elev - 12000| / 1000) -- centers weight on ski zone elevation

---

## Model Calibration Status

The following coefficients are physically motivated estimates rather than values derived from observational regression at the relevant elevation. A calibration pass against Niwot Ridge snow temperature sensor data (niwot.colorado.edu LTER) would improve confidence in the column layer coefficients.

| Coefficient | Value | Basis | Status |
|---|---|---|---|
| Column heat load per F/day above 40F | 0.004 | ~4.5x slower than surface (Fierz 2011) | Estimate |
| Column warm-night penetration per day above 32F | 0.015 | Latent heat penetration slow at depth | Estimate |
| Column cold rebuild per F/day below 40F | 0.003 | Symmetric with heat load, attenuated | Estimate |
| SWE deviation effect on column CCC | +/-0.05 at +/-50% | Conservative; station elevation mismatch limits precision | Estimate |
| Surface/column weighting | 0.60 / 0.40 | Surface governs refreeze; column governs collapse speed | Physically motivated, not validated |
| Monthly isothermal probability | 5/15/40/70% | Consistent with Clow et al. 2012 trends | Semi-quantitative |
| Grain rounding penalty | 0/6/14/22% | Consistent with Colbeck 1987 metamorphism rates | Semi-quantitative |
| Consolidated pack floor | 0.35 x depthFactor x seasonal_scale | Colbeck (1997) sintering plateau | Physically motivated |
| Melt delay cap | 4.5 hours | Conservative field observation upper bound | Field estimate |
| Score normalization constant (RAW_MAX) | 66.85 | S. Arapaho SE, March, benchmark conditions -> score 100 | Calibrated to benchmark |
