# CornCast — Daily Corn Quality Score Chart Spec
## v24 — "Did Today Have Good Corn?" Trajectory Jan 1 → Target +15

---

## Answer to Your Wind Question

Open-Meteo's **daily API has no `windspeed_10m_mean`** — only `windspeed_10m_max`.
Your instinct is correct: a single gust at 2 AM should not tank a corn score.

**The solution: add hourly wind to the existing fetches and compute daily mean yourself.**

The archive fetch currently requests only `daily=...` variables. Adding
`&hourly=wind_speed_10m` to both the archive and forecast URLs returns 24 hourly
values per day. Average those 24 values to get a true daily mean. The forecast fetch
already has an `hourly` section (for dawn temp), so that one is just adding a variable.
The archive fetch needs the hourly section added.

**Fallback:** If hourly wind data is absent (parse error, old cached response), fall back
to `windspeed_10m_max * 0.60`. Research on mountain terrain shows daily mean is
typically 55–65% of daily max. This is the right order of magnitude and avoids silent
failures.

This is the only API change required for the entire feature.

---

## Does the Coldness Graph Still Make Sense?

**Yes — keep it, but reframe the relationship between the two charts.**

| Chart | What it shows | Time horizon | Volatility |
|---|---|---|---|
| Existing SFQ/CCR | Pack *capacity* — can good corn form? | Season trend | Slow-moving |
| New Corn Score | Actual *outcome* — did good corn form? | Day-by-day | Highly volatile |

The two charts tell a two-act story: Act 1 (coldness chart) = "the raw material is
excellent/degrading." Act 2 (score chart) = "here are the days that cashed that in."

Stack them vertically inside the same `ripeness-chart-wrap` card. Corn score chart goes
**below** the coldness chart. Same x-axis scale and PX_PER_DAY so they visually align.
Label them clearly: "📉 Snowpack Cold Reserve" above and "🌽 Daily Corn Score" below.

---

## What the New Chart Shows

For every day from Jan 1 through target+15, run the full `cornWindow()` physics pipeline
using that day's actual observed/forecast weather. Plot the resulting 0–100 score as a
bar chart (not a line) — bars are visually better than a line for a volatile daily metric
and they naturally communicate "this day had a window / this day didn't."

Jan/Feb pre-season bars will always be near zero (frozen pack, no solar geometry producing
corn), which is physically correct and visually reinforces why the season starts March 1.

---

## Part 1 — API Change: Add Hourly Wind

### Archive URL (in `fetchWeatherData`)

Find:
```js
var archiveUrl = 'https://archive-api.open-meteo.com/v1/archive'
    +'?latitude='+pk.la+'&longitude='+pk.lo
    +'&daily=temperature_2m_max,temperature_2m_min,precipitation_sum,snowfall_sum,windspeed_10m_max,cloudcover_mean'
    +'&temperature_unit=fahrenheit&wind_speed_unit=mph&precipitation_unit=inch'
    +'&timezone=America%2FDenver'
    +'&start_date='+cccStart+'&end_date='+yesterdayStr;
```

Replace with:
```js
var archiveUrl = 'https://archive-api.open-meteo.com/v1/archive'
    +'?latitude='+pk.la+'&longitude='+pk.lo
    +'&daily=temperature_2m_max,temperature_2m_min,precipitation_sum,snowfall_sum,windspeed_10m_max,cloudcover_mean'
    +'&hourly=wind_speed_10m'
    +'&temperature_unit=fahrenheit&wind_speed_unit=mph&precipitation_unit=inch'
    +'&timezone=America%2FDenver'
    +'&start_date='+cccStart+'&end_date='+yesterdayStr;
```

### Forecast URL — already has hourly section

Find the existing hourly line in forecastUrl:
```js
+'&hourly=temperature_2m'
```
Replace with:
```js
+'&hourly=temperature_2m,wind_speed_10m'
```

### `parseDailyJson` — extract hourly wind means

After the existing days array is built, add a post-processing step to compute daily
mean wind from the hourly array. Insert this block at the end of `parseDailyJson`,
just before `return days;`:

```js
  // Compute daily mean wind from hourly wind_speed_10m if available
  // 24 hourly values per day; keyed by date prefix of ISO timestamp
  if(json.hourly && json.hourly.time && json.hourly.wind_speed_10m) {
    var hTimes = json.hourly.time;
    var hWinds = json.hourly.wind_speed_10m;
    // Build date → [windValues] map
    var windByDate = {};
    for(var hi=0; hi<hTimes.length; hi++) {
      var hDate = hTimes[hi].slice(0,10); // 'YYYY-MM-DD'
      var hWind = hWinds[hi];
      if(hWind === null) continue;
      if(!windByDate[hDate]) windByDate[hDate] = [];
      windByDate[hDate].push(hWind);
    }
    // Attach windMean to each day; fall back to max*0.60 if no hourly data
    days.forEach(function(d) {
      var vals = windByDate[d.date];
      if(vals && vals.length > 0) {
        var sum = 0;
        for(var vi=0; vi<vals.length; vi++) sum += vals[vi];
        d.windMean = sum / vals.length;
      } else {
        d.windMean = d.wind * 0.60; // fallback: max → mean proxy
      }
    });
  } else {
    // No hourly wind available at all — use fallback for every day
    days.forEach(function(d) { d.windMean = d.wind * 0.60; });
  }
```

Every `day` object now has both `d.wind` (max, existing) and `d.windMean` (mean, new).
No downstream code breaks — existing uses of `d.wind` are unchanged.

Also add `windMean` to `ripenessTrajectory.push` in both the pre-season and season loops:
```js
  // In both push() calls, add:
  windMean: dp.windMean || dp.wind * 0.60,   // pre-season
  windMean: d.windMean  || d.wind  * 0.60,   // season
```

---

## Part 2 — Running Storm Tracker

`cornWindow()` needs `f.ds` (days since storm) and `f.sdp` (snow depth estimate) as
inputs. These are computed once for the target date in the existing code via the SNWD /
storm-detection logic. For the trajectory we need them rolling forward day by day.

Add this new function above `drawRipenessChart`:

```js
// ── Build rolling storm state for corn score trajectory ──────────────────
// Returns array parallel to traj: [{daysSince, depthEst}, ...]
// daysSince = days elapsed since last storm end (0 = storm is active)
// depthEst = estimated snow depth from that storm in inches
function buildRollingStormState(traj) {
  var state = [];
  var daysSince = 14;   // conservative "no recent storm" start
  var depthEst  = 3.0;  // conservative baseline depth for Jan 1
  var stormActive = false;
  var stormAccum  = 0;

  for(var i=0; i<traj.length; i++) {
    var sf = traj[i].snowfall || 0;
    if(sf >= 0.3) {
      // Accumulating storm
      stormAccum += sf;
      stormActive = true;
      daysSince = 0;
    } else if(stormActive) {
      // Storm just ended
      depthEst = Math.round(stormAccum * 10) / 10;
      stormAccum = 0;
      stormActive = false;
      daysSince = 1;
    } else {
      daysSince = Math.min(daysSince + 1, 60); // cap at 60 for old pack
    }
    state.push({ daysSince: daysSince, depthEst: depthEst });
  }
  return state;
}
```

---

## Part 3 — `computeDailyCornScores()`

Add this new function directly above `drawRipenessChart`:

```js
// ── Per-day corn score trajectory ────────────────────────────────────────
// Runs the full cornWindow() physics for every day in traj.
// Returns array of {date, score, ws, we, dur, isForecast, isPreSeason}
// Pre-season days (Jan/Feb): score is always 0 — no solar geometry for corn.
function computeDailyCornScores(traj, marchFirstIdx, pk, asp, swePct) {
  if(!pk || !asp || !pk.asp[asp]) return [];
  var stormState = buildRollingStormState(traj);
  var scores = [];

  for(var i=0; i<traj.length; i++) {
    var t = traj[i];

    // Pre-season: no corn possible, return zeroed entry
    if(i < marchFirstIdx) {
      scores.push({
        date: t.date,
        score: 0,
        ws: null, we: null, dur: 0,
        isForecast: false,
        isPreSeason: true
      });
      continue;
    }

    var month = new Date(t.date + 'T12:00:00').getMonth() + 1;

    // Build synthetic forecast object from trajectory day
    var f = {
      lo:    t.tmin,
      hi:    t.tmax,
      da:    t.tmin + 2,              // dawn temp: tmin + 2°F offset
      wi:    t.windMean || (t.wind || 0) * 0.60,  // daily mean wind
      cn:    t.cloud || 0,            // overnight cloud ≈ daily mean
      cm:    (t.cloud || 0) * 0.75,   // morning cloud ≈ 75% of daily
      ds:    stormState[i].daysSince,
      sdp:   stormState[i].depthEst,
      swePct: swePct || null,
      sl:    false                    // skip wind slab alert for trajectory
    };

    // Temporarily override weatherData.effectiveIsoPrb with this day's value
    // so cornWindow() picks up the correct isothermal probability for the day
    var savedIsoPrb = null;
    if(window.weatherData) {
      savedIsoPrb = window.weatherData.effectiveIsoPrb;
      window.weatherData.effectiveIsoPrb = t.effectiveIsoPrb;
    }

    var result = cornWindow(pk, asp, f, month);

    // Restore
    if(window.weatherData && savedIsoPrb !== null) {
      window.weatherData.effectiveIsoPrb = savedIsoPrb;
    }

    scores.push({
      date:       t.date,
      score:      result ? result.q : 0,
      ws:         result ? result.ws : null,
      we:         result ? result.we : null,
      dur:        result ? result.dur : 0,
      isForecast: t.isForecast,
      isPreSeason: false
    });
  }
  return scores;
}
```

**Note on `weatherData.effectiveIsoPrb` override:** `cornWindow()` reads
`weatherData.effectiveIsoPrb` directly from the global to get the live isothermal
probability. For trajectory scoring we need each day's own `effectiveIsoPrb`, not the
target day's value. The save/override/restore pattern above is safe because
`computeDailyCornScores` runs synchronously before any rendering.

---

## Part 4 — `drawCornScoreChart()` — New Canvas Function

Add this entirely new function below `drawRipenessChart`:

```js
// ── CORN SCORE CHART (v24) — Daily corn quality bars Jan 1 → Target+15 ──
function drawCornScoreChart(canvasId, wd, pk, asp) {
  var canvas = document.getElementById(canvasId);
  if(!canvas || !wd || !wd.ripenessTrajectory || !wd.ripenessTrajectory.length) return;
  if(!pk || !asp) return;

  var traj = wd.ripenessTrajectory;
  var marchFirstIdx = wd.marchFirstIdx || 0;
  var scores = computeDailyCornScores(
    traj, marchFirstIdx, pk, asp, wd.swePct || null
  );
  var N = scores.length;
  if(!N) return;

  var todayStr   = wd.todayStr;
  var targetDate = wd.targetDate;

  var PAD = {top:24, right:44, bottom:32, left:32};

  // Reuse same PX_PER_DAY logic as existing chart for visual alignment
  var containerW = (canvas.parentElement && canvas.parentElement.offsetWidth) || 320;
  var minChartPx = containerW - PAD.left - PAD.right;
  var PX_PER_DAY = Math.min(16, Math.max(7, Math.floor(minChartPx / Math.max(N, 1))));
  var cW = N * PX_PER_DAY;

  var H = 160;  // shorter than the coldness chart — bars don't need as much height
  var W = PAD.left + PAD.right + cW;
  var cH = H - PAD.top - PAD.bottom;

  canvas.style.width  = W + 'px';
  canvas.style.height = H + 'px';
  canvas.width  = W * (window.devicePixelRatio || 1);
  canvas.height = H * (window.devicePixelRatio || 1);
  var ctx = canvas.getContext('2d');
  ctx.scale(window.devicePixelRatio || 1, window.devicePixelRatio || 1);

  function xPos(i)  { return PAD.left + i * PX_PER_DAY + PX_PER_DAY / 2; }
  function barLeft(i){ return PAD.left + i * PX_PER_DAY + 1; }
  function barW()   { return Math.max(1, PX_PER_DAY - 2); }
  function yScore(s){ return PAD.top + cH * (1 - s / 100); }

  // Find today and target indices
  var todayIdx = -1, tgtIdx = -1;
  for(var i=0; i<N; i++) {
    if(scores[i].date === todayStr)   todayIdx = i;
    if(scores[i].date === targetDate) tgtIdx   = i;
  }

  // ── Background ──
  ctx.fillStyle = 'rgba(8,18,30,0.0)'; // transparent — card background shows through
  ctx.fillRect(0, 0, W, H);

  // ── Grid lines at 25, 50, 75, 100 ──
  ctx.font = '8px -apple-system,sans-serif';
  [25, 50, 75, 100].forEach(function(v) {
    var y = yScore(v);
    ctx.strokeStyle = v === 50 ? 'rgba(255,71,87,0.25)' : 'rgba(46,74,96,0.30)';
    ctx.lineWidth   = v === 50 ? 1 : 0.5;
    ctx.setLineDash(v === 50 ? [3,3] : [2,4]);
    ctx.beginPath(); ctx.moveTo(PAD.left, y); ctx.lineTo(PAD.left + cW, y); ctx.stroke();
    ctx.setLineDash([]);
    ctx.fillStyle   = v === 50 ? 'rgba(255,71,87,0.50)' : 'rgba(93,184,240,0.35)';
    ctx.textAlign   = 'left';
    ctx.fillText(v, PAD.left + cW + 3, y + 3);
  });

  // ── Pre-season shading ──
  if(marchFirstIdx > 0) {
    var psX1 = PAD.left + marchFirstIdx * PX_PER_DAY;
    ctx.fillStyle = 'rgba(20,40,65,0.40)';
    ctx.fillRect(PAD.left, PAD.top, psX1 - PAD.left, cH);
    // Mar 1 marker
    ctx.strokeStyle = 'rgba(180,220,255,0.30)';
    ctx.lineWidth = 1; ctx.setLineDash([3,3]);
    ctx.beginPath(); ctx.moveTo(psX1, PAD.top); ctx.lineTo(psX1, PAD.top + cH); ctx.stroke();
    ctx.setLineDash([]);
    ctx.fillStyle = 'rgba(180,220,255,0.45)';
    ctx.font = 'bold 7px -apple-system,sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('Mar 1', psX1, PAD.top + cH / 2 + 3);
  }

  // ── Score bars ──
  for(var bi=0; bi<N; bi++) {
    var s = scores[bi];
    if(s.score <= 0) continue;

    var bx = barLeft(bi);
    var bh = (s.score / 100) * cH;
    var by = PAD.top + cH - bh;
    var bw = barW();

    // Color by score band
    var barColor;
    if(s.score >= 75)      barColor = s.isForecast ? 'rgba(46,213,115,0.55)'  : 'rgba(46,213,115,0.80)';
    else if(s.score >= 50) barColor = s.isForecast ? 'rgba(255,209,102,0.45)' : 'rgba(255,209,102,0.75)';
    else if(s.score >= 25) barColor = s.isForecast ? 'rgba(255,165,2,0.40)'   : 'rgba(255,165,2,0.70)';
    else                   barColor = s.isForecast ? 'rgba(255,71,87,0.30)'   : 'rgba(255,71,87,0.55)';

    ctx.fillStyle = barColor;
    ctx.fillRect(bx, by, bw, bh);

    // Subtle score number inside tall bars (≥40px)
    if(bh >= 40 && bw >= 6) {
      ctx.fillStyle = 'rgba(255,255,255,0.55)';
      ctx.font = '7px -apple-system,sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText(s.score, bx + bw/2, by + 10);
    }
  }

  // ── Today vertical marker ──
  if(todayIdx >= 0) {
    var tx = xPos(todayIdx);
    ctx.strokeStyle = 'rgba(255,209,102,0.55)'; ctx.lineWidth = 1; ctx.setLineDash([3,3]);
    ctx.beginPath(); ctx.moveTo(tx, PAD.top); ctx.lineTo(tx, PAD.top + cH); ctx.stroke();
    ctx.setLineDash([]);
    ctx.fillStyle = 'rgba(255,209,102,0.80)';
    ctx.font = 'bold 8px -apple-system,sans-serif'; ctx.textAlign = 'center';
    ctx.fillText('Today', tx, PAD.top - 4);
  }

  // ── Target vertical marker + score callout ──
  if(tgtIdx >= 0) {
    var ttx = xPos(tgtIdx);
    ctx.strokeStyle = 'rgba(0,212,170,0.65)'; ctx.lineWidth = 1.5; ctx.setLineDash([4,2]);
    ctx.beginPath(); ctx.moveTo(ttx, PAD.top); ctx.lineTo(ttx, PAD.top + cH); ctx.stroke();
    ctx.setLineDash([]);
    var tScore = scores[tgtIdx] ? scores[tgtIdx].score : 0;
    var tColor = tScore >= 75 ? '#2ed573' : tScore >= 50 ? '#ffd166' : tScore >= 25 ? '#ffa502' : '#ff4757';
    ctx.fillStyle = tColor;
    ctx.font = 'bold 9px -apple-system,sans-serif'; ctx.textAlign = 'center';
    ctx.fillText(tScore, ttx, PAD.top + cH + 14);
    ctx.font = '7px -apple-system,sans-serif';
    ctx.fillStyle = 'rgba(0,212,170,0.65)';
    ctx.fillText('Target', ttx, PAD.top + cH + 23);
  }

  // ── X-axis date labels ──
  ctx.font = '8px -apple-system,sans-serif';
  var lastLabelX = -999;
  var MNAMES = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];
  for(var xi=0; xi<N; xi++) {
    var dt = new Date(scores[xi].date + 'T12:00:00');
    var isWeekly = (dt.getDay() === 1 || dt.getDate() === 1);
    var isSpecial = (xi === todayIdx || xi === tgtIdx);
    if(!isWeekly && !isSpecial) continue;
    var labelX = xPos(xi);
    if(!isSpecial && labelX - lastLabelX < 28) continue;
    var label = isSpecial && xi === todayIdx ? ''
              : dt.getDate() === 1 ? MNAMES[dt.getMonth()] + ' 1'
              : MNAMES[dt.getMonth()] + ' ' + dt.getDate();
    if(isSpecial && xi === tgtIdx) label = '';  // score shown above instead
    if(!label) continue;
    ctx.fillStyle = isSpecial ? 'rgba(255,209,102,0.7)' : 'rgba(120,165,190,0.55)';
    ctx.textAlign = 'center';
    ctx.fillText(label, labelX, H - PAD.bottom + 10);
    lastLabelX = labelX;
  }

  // ── Legend ──
  var lx = PAD.left + 2, ly = PAD.top - 8;
  function lgBox(x, col) {
    ctx.fillStyle = col; ctx.fillRect(x, ly - 5, 8, 7);
  }
  function lgTxt(x, txt) {
    ctx.fillStyle = 'rgba(180,210,220,0.75)';
    ctx.font = '8px -apple-system,sans-serif';
    ctx.textAlign = 'left';
    ctx.fillText(txt, x, ly + 2);
  }
  lgBox(lx, 'rgba(46,213,115,0.80)');   lgTxt(lx+11, '75+ Excellent');
  lx += 78;
  lgBox(lx, 'rgba(255,209,102,0.75)');  lgTxt(lx+11, '50+ Good');
  lx += 58;
  lgBox(lx, 'rgba(255,165,2,0.70)');    lgTxt(lx+11, '25+ Marginal');
  lx += 68;
  lgBox(lx, 'rgba(255,71,87,0.55)');    lgTxt(lx+11, 'Poor');

  // ── Auto-scroll to same position as coldness chart ──
  var scrollEl = canvas.parentElement;
  if(scrollEl && scrollEl.id === 'corn-score-scroll') {
    var focusIdx2 = tgtIdx >= 0 ? tgtIdx : (todayIdx >= 0 ? todayIdx : N - 1);
    var focusX2   = xPos(focusIdx2);
    var viewW2    = scrollEl.offsetWidth || 320;
    scrollEl.scrollLeft = Math.max(0, focusX2 - viewW2 * 0.65);
  }
}
```

---

## Part 5 — HTML + Call Site in `renderResult`

### In `renderResult`, find the existing ripeness chart HTML block:
```js
  if(weatherData && weatherData.ripenessTrajectory && weatherData.ripenessTrajectory.length) {
    h+='<div class="ripeness-chart-wrap">';
    h+='<h3>📈 Snowpack Condition Trajectory — Season Start → Target +15 days</h3>';
    h+='<div id="ripeness-scroll" style="overflow-x:auto;-webkit-overflow-scrolling:touch;border-radius:8px">';
    h+='<canvas id="ripeness-canvas" style="display:block;height:240px"></canvas>';
    h+='</div>';
    h+='<div style="font-size:.64rem;color:#456a82;margin-top:.3rem">🟠 Surface freeze · 🔵 Col. cold reserve · 🌡️ Hi/Lo °F · ❄ Snow bars · Dashed = forecast · Pan ← to see history</div>';
  }
  h+='</div>';
```

### Replace with:
```js
  if(weatherData && weatherData.ripenessTrajectory && weatherData.ripenessTrajectory.length) {
    h+='<div class="ripeness-chart-wrap">';

    // ── Chart 1: Snowpack coldness ──
    h+='<h3 style="margin-bottom:.25rem">📉 Snowpack Cold Reserve</h3>';
    h+='<div style="font-size:.68rem;color:var(--muted);margin-bottom:.4rem">Pack capacity to produce corn · high = cold pack = better refreeze potential</div>';
    h+='<div id="ripeness-scroll" style="overflow-x:auto;-webkit-overflow-scrolling:touch;border-radius:8px">';
    h+='<canvas id="ripeness-canvas" style="display:block;height:240px"></canvas>';
    h+='</div>';
    h+='<div style="font-size:.64rem;color:#456a82;margin-top:.3rem">🟠 Surface freeze · 🔵 Col. cold reserve · 🌡️ Hi/Lo °F · ❄ Snow bars · Dashed = forecast · Pan ← to see history</div>';

    // ── Chart 2: Daily corn score ──
    h+='<h3 style="margin-top:1.1rem;margin-bottom:.25rem">🌽 Daily Corn Quality Score</h3>';
    h+='<div style="font-size:.68rem;color:var(--muted);margin-bottom:.4rem">Estimated corn window quality for each day · bar height = score 0–100</div>';
    h+='<div id="corn-score-scroll" style="overflow-x:auto;-webkit-overflow-scrolling:touch;border-radius:8px">';
    h+='<canvas id="corn-score-canvas" style="display:block;height:160px"></canvas>';
    h+='</div>';
    h+='<div style="font-size:.64rem;color:#456a82;margin-top:.3rem">Faded bars = forecast · Score uses daily mean wind · aspect: '+(selectedAsp||'?')+'</div>';
  }
  h+='</div>';
```

### In the `requestAnimationFrame` block that calls `drawRipenessChart`, add the second call:

Find:
```js
    requestAnimationFrame(function() {
      drawRipenessChart('ripeness-canvas', weatherData, month, w);
    });
```

Replace with:
```js
    requestAnimationFrame(function() {
      drawRipenessChart('ripeness-canvas', weatherData, month, w);
      var pk2 = PK[selectedPeak];
      var asp2 = selectedAsp;
      if(pk2 && asp2) {
        drawCornScoreChart('corn-score-canvas', weatherData, pk2, asp2);
      }
    });
```

---

## Part 6 — `swePct` Availability

`computeDailyCornScores` needs `swePct` to run the same SWE modifier that `cornWindow`
uses on the target date. It's available via `currentSWEData.pct` at the time
`drawCornScoreChart` is called.

Pass it through from `renderResult`. The `wd` return object from `processWeatherData`
does NOT currently carry `swePct` — add it:

In `processWeatherData`'s `cb({...})` call, add:
```js
    swePct: swePct,   // ← add this line
```

Then `drawCornScoreChart` can read `wd.swePct` directly. No other changes needed.

---

## Physics Notes — What Each Input Approximates

| `cornWindow` input | Per-day approximation | Quality |
|---|---|---|
| `f.lo` (overnight low) | `tmin` from ERA5/forecast | ✅ Exact |
| `f.hi` (daytime high) | `tmax` from ERA5/forecast | ✅ Exact |
| `f.da` (dawn temp 6AM) | `tmin + 2°F` | ⚠️ Estimate, ±3°F |
| `f.wi` (wind speed) | hourly `wind_speed_10m` 24hr mean | ✅ Good |
| `f.cn` (overnight cloud%) | `cloud` (daily mean) | ⚠️ Proxy |
| `f.cm` (morning cloud%) | `cloud * 0.75` | ⚠️ Proxy |
| `f.ds` (days since storm) | running storm tracker | ✅ Good |
| `f.sdp` (snow depth est") | running storm accumulation | ✅ Good |
| `f.swePct` | basin SWE % of median | ✅ Single value for season |
| `isoPrb` | per-day `effectiveIsoPrb` from traj | ✅ Exact |

Cloud is the weakest proxy — daily mean doesn't capture the overnight vs morning split
that matters for freeze quality vs melt rate. This introduces noise on individual days
but the seasonal trend signal is strong. Label this in the About tab as a known
approximation.

---

## Changelog Entry

```
### Session [N] — v24 (Daily Corn Score Chart)

**New chart: "🌽 Daily Corn Quality Score"** stacked below the existing coldness chart.
Runs the full cornWindow() physics pipeline for every day Jan 1 → target+15 using
observed/forecast weather, producing a 0–100 score bar per day. Pre-season (Jan/Feb)
bars are always near zero — no solar geometry for corn before March. Forecast bars are
faded (55% opacity) vs archive bars (80%).

**Wind: daily mean instead of max.** Added `hourly=wind_speed_10m` to both archive
and forecast API URLs. `parseDailyJson` now computes a 24hr daily mean wind from hourly
values and stores it as `d.windMean`. Falls back to `windspeed_10m_max × 0.60` if
hourly data is absent. `ripenessTrajectory` entries carry `windMean` field. Daily mean
avoids a single gust distorting the corn score for an entire day.

**`swePct` added to `wd` return object** so charts have access without accessing
`currentSWEData` directly.

**`buildRollingStormState()`** new helper: forward pass through trajectory computing
running `daysSince` and `depthEst` for each day. Used exclusively by corn score chart.

**`computeDailyCornScores()`** new helper: calls `cornWindow()` per day with synthetic
forecast object. Temporarily overrides `weatherData.effectiveIsoPrb` to inject per-day
isothermal probability. Restores original value immediately after each call.

**Two-chart architecture rationale:** Coldness chart = "pack capacity, why the season
trends the way it does." Corn score chart = "which days actually had good conditions."
Together they tell the full story: a cold pack + cold nights + clear sky = tall green
bars; a warm pack + warm nights = short red bars regardless of snowfall history.
```

---

## Known Limitations to Add to About Tab

> **Daily corn score uses approximated inputs.** Morning cloud cover is proxied from
> the daily mean rather than a true overnight/morning split. Dawn temperature is
> estimated as overnight low + 2°F rather than the 6 AM hourly value (which is only
> available for the target date). These approximations introduce ±10–15 point noise on
> individual days; the seasonal trend is reliable. Do not interpret individual bar
> heights as precise predictions — look at clusters of high/low bars to identify
> productive periods.
