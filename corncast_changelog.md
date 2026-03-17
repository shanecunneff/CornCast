# CornCast — Physics Changelog

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
