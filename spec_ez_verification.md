# CornCast — Spec: Ski Zone Elevation (`ez`) Verification & Physics Correction

**Status:** Pre-coding research task  
**Version target:** v22 (physics correction) — pending completion of this spec  
**Dependencies:** Must complete Phase 1 and Phase 2 before any code changes  

---

## Background & Motivation

Every aspect in the CornCast peaks database has an `ez` field representing the ski zone elevation — where the corn actually forms and is skied, as opposed to the summit. For example, S. Arapaho Peak has a summit at 13,397' but an `ez` of 12,600', because the SE couloir ski line lives at that lower elevation band.

The problem is twofold:

**1. The `ez` values are unverified estimates.** They were hand-coded with no documented methodology or source. Some may be reasonably accurate; others may be off by several hundred feet. There is currently no way to know without checking each one.

**2. The physics pipeline uses summit elevation (`pk.el`), not ski zone elevation (`ad.ez`), for temperature lapse correction.** All weather data is lapse-corrected to the summit, then only partially walked back via a small elevation bonus in the freeze quality step. The model is computing refreeze quality, melt onset, and column cold content as if conditions exist at the very top of the mountain. The corn doesn't form there — it forms on the ski line.

For a peak like Longs Peak (summit 14,259', NE Trough ski zone ~13,400'), this is a ~3°F systematic error in the temperature inputs that affects every downstream calculation. On optimal corn nights — clear, calm, cold — this is exactly when the error matters most.

The fix is straightforward once the data is clean: lapse-correct temperatures to `ad.ez` per-aspect inside `cornWindow` rather than to `pk.el` at fetch time. But correcting the physics to use `ez` consistently is only meaningful if the `ez` values themselves are accurate first.

---

## Phase 1 — `ez` Verification Table (Research, No Coding)

### Goal
Replace all 24 peaks' hand-estimated `ez` values with midpoint elevations derived from actual route data.

### Method
For each skiable aspect on each peak, determine:
- **Route base elevation** — where the sustained skiable snow line begins (bottom of the ski line)
- **Route top elevation** — where the ski line reaches (usually near-summit or top of couloir)
- **Derived midpoint** — `(base + top) / 2` — this becomes the new `ez` candidate

### Primary Source — frontrangeskimo.com
Every Indian Peaks route page on frontrangeskimo.com includes:
- Explicit approach elevation ranges in the route descriptions (e.g. "tarn at 11,440'", "bench at 12,280'")
- A linked Caltopo map with routes drawn on topo for visual confirmation

The Apache Peak page demonstrated the pattern clearly:
- E face (Apache Couloir): base tarn 11,440', summit 13,441' → midpoint ~12,440' (current `ez`: 12,700')
- NE face (Queens Way): bench at 12,280' suggests midpoint lower than current `ez` of 12,800'

### Deliverable — Comparison Table

| Peak | Aspect | Current `ez` | Route Base | Route Top | Derived Midpoint | Source | Flag |
|------|--------|-------------|------------|-----------|-----------------|--------|------|
| Apache Peak | E | 12,700 | | | | | |
| Apache Peak | NE | 12,800 | | | | | |
| ... | | | | | | | |

**Flag column values:**
- ✅ Confirmed — frontrangeskimo data clear, Caltopo consistent
- 🗺️ Map check needed — description vague, requires Caltopo visual review
- 👤 Field knowledge — no frontrangeskimo coverage, use personal knowledge or manual Caltopo

### Labor Split
- **Claude (~80%):** Fetch all Indian Peaks route pages from frontrangeskimo.com, extract elevation data, populate draft table
- **Manual review (~20%):** Aspects with no frontrangeskimo coverage, ambiguous Caltopo readings, final sign-off on all values
- **Session type:** Dedicated research session — no code, output is the completed table only

### Peaks Requiring Special Attention
The following peaks in CornCast have aspects that may not have frontrangeskimo coverage and will require Caltopo or field knowledge:
- N. Arapaho Peak (N, NW, W, SW aspects — less-traveled faces)
- Navajo Peak (exposed aspects)
- Any RMNP peaks (Terra Tomah, Longs Peak, Mt. Ida) — frontrangeskimo coverage thinner in RMNP

---

## Phase 2 — Data Entry (Mechanical, No Coding)

Once the comparison table is reviewed and signed off:

1. Open `index.html` in GitHub editor
2. For each aspect with a changed `ez` value, find the relevant line in the peaks database
3. Update the `ez` number
4. Commit with message: "v22-data: verified ez values from frontrangeskimo route data"

This is a find-and-replace pass through the data, not a logic change. Low risk. Can be done in one GitHub editing session.

---

## Phase 3 — Physics Correction (Coding Session)

**Target function:** `cornWindow(pk, asp, f, month)`

**Current behavior:** Temperature inputs lapse-corrected to `pk.el` (summit elevation) at fetch time. A small `elevB` bonus partially compensates but is not equivalent.

**Required change:** Inside `cornWindow`, after retrieving `ad` (the aspect data object), apply a secondary lapse correction from summit elevation to ski zone elevation before using temperature inputs in the freeze quality and melt delay calculations.

```
// Current (incorrect):
var elevB = (ad.ez - 11500) / 1800 * 0.18;  // partial bonus only

// Target behavior:
// Temperatures already lapse-corrected to pk.el at fetch time
// Apply delta correction from summit to ski zone elevation inside cornWindow
var skiZoneTemp = elevCorrectTemp(summitTemp, ad.ez, pk.el);
```

**Affected calculations inside `cornWindow`:**
- `rawFQ` — freeze quality (uses overnight low)
- `meltDelay` — corn opens calculation (uses dawn temp and daytime high)
- The `elevB` bonus term — should be removed or replaced by the explicit correction

**Physics motivation:** The corn forms on the ski line, not the summit. A 14,259' summit with a 13,400' ski zone is ~3°F warmer at the ski line on a standard lapse rate. That difference shifts freeze quality, melt delay, and window duration in ways that compound across the full pipeline.

**Scope:** Moderate — touches `cornWindow` and temperature handling within it. Does not affect the weather fetch architecture or the two-layer model calculations, which correctly use `pk.el` for the column cold content history (the column spans the full snowpack depth, not just the ski zone).

**Changelog entry:** Document the `ez` source methodology, the physics motivation, and the specific formula change. Update the research doc to note frontrangeskimo.com as a primary source for `ez` values.

---

## Future Extension — Terrain Visualization

The verified `ez` values and ski line elevation data from this work also lay the groundwork for a future terrain visualization feature:

**Near-term (achievable within single-file architecture):**
- Ski line routes as GeoJSON polylines on the Leaflet map, colored by corn window score
- Route data sourced from frontrangeskimo Caltopo maps (GPX export → GeoJSON)
- Leaflet handles polyline overlay natively

**Longer-term (requires architectural reconsideration):**
- Per-pixel terrain coloring by aspect and corn score
- Requires DEM data, per-pixel aspect calculation, canvas rendering
- Likely pushes beyond single-file constraint — warrants separate planning when ready

---

## Session Sequence

| Session | Type | Input | Output |
|---------|------|-------|--------|
| 1 | Research | frontrangeskimo route pages | Draft `ez` comparison table |
| 2 | Review | Draft table | Signed-off table with flags resolved |
| 3 | Data entry | Signed-off table | Updated `ez` values committed to GitHub |
| 4 | Coding | Verified `ez` data | Physics correction in `cornWindow`, v22 |
