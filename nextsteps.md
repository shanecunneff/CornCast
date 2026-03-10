1. Dust-on-snow modifier — highest impact for the target geography and season. CAIC publishes dust layer data, the physics are well understood (albedo reduction → earlier onset, compressed window), and it fills the single biggest real-world gap in the model right now. Complexity is moderate.
   
2. SNOTEL temperature cross-check on overnight low — this is a present accuracy problem disguised as future work. Niwot Ridge (#663) already returns TMIN data from the same fetch architecture you have. The auto-filled overnight low is most wrong on exactly the nights that matter most (clear, calm, inversion-prone). Complexity is low — it's an addition to existing fetch logic.
   
3. Approach time uncertainty — adding a simple pace multiplier input (or at minimum a visible caveat that the regression is one person's data) would make the departure time output more honest. Complexity is very low.

4. Using terrain shading from shae map
   
5. Incorproating DEM data in order to better evaluate terrain conditions.
  
Prerequisite 6a — ez verification table (research task, no coding)
Goal: replace hand-estimated ski zone elevations with midpoint elevations derived from actual route data for all 24 peaks.
Process: In a dedicated session, fetch all Indian Peaks route pages from frontrangeskimo.com and extract approach elevation ranges from route descriptions. Build a comparison table: Peak · Aspect · Current ez · Route base · Route top · Derived midpoint · Source. Cross-check against linked Caltopo maps for any aspects with vague descriptions. Flag any aspects with no frontrangeskimo coverage for manual Caltopo review.
Expected labor split: Claude builds the draft table from frontrangeskimo data (~80%). Manual review and sign-off on final values (~20%) — especially for faces not covered by frontrangeskimo and any ambiguous Caltopo map readings.
Deliverable: A verified ez value for every aspect in the peaks database, ready to paste into the code as a find-and-replace pass in a subsequent coding session.
Dependency: Must be completed before implementing item 6 (ski line elevation correction).

Prerequisite for item 6 — verify ez values against topo:
Current ski zone elevations (ez) in the peaks database are hand-estimated with no documented methodology. Before implementing the ski line elevation correction, all 24 peaks should have their ez values verified against Caltopo or USGS topo for each specific aspect. Method: find the top and bottom of the sustained skiable snow line on each aspect, take the midpoint elevation, use that as ez. Peaks with personal field knowledge can be verified from memory. Lesser-known faces require map work. This is a data quality task that can be done independently before any code changes — correcting the physics to use ez consistently is only meaningful if the ez values themselves are accurate.

6. Ski line elevation correction (use ez consistently throughout physics pipeline)
The ez field already exists on every aspect and represents the actual ski zone elevation, not the summit. Currently all weather data is lapse-corrected to pk.el (summit), then only partially adjusted. The fix is to lapse-correct temperatures to ad.ez per-aspect inside cornWindow rather than to pk.el at fetch time. For peaks like Longs (summit 14,259’ vs ski line ~13,400’) this is a ~3°F difference that affects freeze quality, melt onset, and column cold content. Data infrastructure already in place — moderate change to cornWindow and temperature inputs. Physics motivation: the corn forms on the ski line, not the summit.

7. Expanding this to an area larger than the front range 

8. is there a way to build a graph of snow quality incorporating SWe?

10. 
