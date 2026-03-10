# CornCast — Snowpack Trajectory Chart Enhancement Spec
## All 8 Ideas — Implementation Reference for v23

---

## Overview

All changes live entirely inside `drawRipenessChart()` unless noted.
`drawRipenessChart` receives `wd` (weather data object) and `month`.
The corn window values `w.ws` (opens, decimal hours) and `w.we` (closes) must be
passed into the function — see **Prerequisite** below for the one signature change needed.

---

## Prerequisite — Pass corn window into drawRipenessChart

Currently called as:
```js
drawRipenessChart('ripeness-canvas', weatherData, month);
```

Change the call site (in `renderResult`, inside the `requestAnimationFrame` block) to:
```js
drawRipenessChart('ripeness-canvas', weatherData, month, w);
```

Change the function signature from:
```js
function drawRipenessChart(canvasId, wd, month) {
```
To:
```js
function drawRipenessChart(canvasId, wd, month, cornWin) {
```

`cornWin` is the `w` object from `cornWindow()`. It has `.ws` (opens, decimal hour),
`.we` (closes, decimal hour), `.q` (score 0–100). It may be null — guard everywhere with
`if(cornWin && cornWin.ws !== null)`.

---

## Enhancement 1 — Corn Window Quality Band on Target Date
**Priority: HIGH**

### What it does
On the target date x-position, draws a vertical green band spanning the full chart height,
width proportional to window duration, color-coded by score. Tells the skier at a glance
when corn opens/closes relative to the entire seasonal trajectory.

### Where to insert
After the Target date vertical line + dot block, before the X-axis labels block.

### Exact code to add
```js
  // ── Corn window band on target date ──
  if(cornWin && cornWin.ws !== null && tgtIdx >= 0) {
    // Convert decimal hours to fraction of day for x-positioning within the target day column
    var wsHr = cornWin.ws;  // e.g. 8.5 = 8:30 AM
    var weHr = cornWin.we;  // e.g. 10.5 = 10:30 AM
    var daySpan = 18;        // chart represents roughly 6AM-midnight = 18hr visible day
    var startFrac = Math.max(0, (wsHr - 6) / daySpan);
    var endFrac   = Math.min(1, (weHr - 6) / daySpan);

    // x positions: the target column is PX_PER_DAY wide centered on xPos(tgtIdx)
    var colLeft  = xPos(tgtIdx) - PX_PER_DAY / 2;
    var bandX0   = colLeft + startFrac * PX_PER_DAY;
    var bandX1   = colLeft + endFrac   * PX_PER_DAY;
    var bandW    = Math.max(2, bandX1 - bandX0);

    // Color by score
    var bandCol = cornWin.q >= 75 ? 'rgba(46,213,115,0.18)'
                : cornWin.q >= 50 ? 'rgba(255,209,102,0.15)'
                : cornWin.q >= 25 ? 'rgba(255,165,2,0.13)'
                :                   'rgba(255,71,87,0.12)';
    var borderCol = cornWin.q >= 75 ? 'rgba(46,213,115,0.55)'
                  : cornWin.q >= 50 ? 'rgba(255,209,102,0.45)'
                  : cornWin.q >= 25 ? 'rgba(255,165,2,0.40)'
                  :                   'rgba(255,71,87,0.40)';

    ctx.fillStyle = bandCol;
    ctx.fillRect(bandX0, PAD.top, bandW, dataBot - PAD.top);

    // Left edge (opens) — solid border
    ctx.strokeStyle = borderCol;
    ctx.lineWidth = 1.5;
    ctx.setLineDash([]);
    ctx.beginPath();
    ctx.moveTo(bandX0, PAD.top);
    ctx.lineTo(bandX0, dataBot);
    ctx.stroke();

    // Right edge (closes) — dashed border
    ctx.setLineDash([3,2]);
    ctx.beginPath();
    ctx.moveTo(bandX1, PAD.top);
    ctx.lineTo(bandX1, dataBot);
    ctx.stroke();
    ctx.setLineDash([]);

    // Label above: "🌽 Opens 9:30" and "Closes 11:00"
    ctx.font = 'bold 7px -apple-system,sans-serif';
    ctx.textAlign = 'center';
    ctx.fillStyle = borderCol;
    var midBand = (bandX0 + bandX1) / 2;
    // Only show text if band is wide enough
    if(bandW > 18) {
      var wsTxt = cornWin.ws % 1 === 0
        ? Math.floor(cornWin.ws) + ':00'
        : Math.floor(cornWin.ws) + ':' + String(Math.round((cornWin.ws % 1)*60)).padStart(2,'0');
      ctx.fillText('🌽 ' + wsTxt, midBand, PAD.top + 8);
    }
  }
```

---

## Enhancement 2 — Freeze Threshold Crossing Dots on Tmin
**Priority: HIGH**

### What it does
Scans each trajectory point. When `tmin` crosses 32°F (transitions from above to below,
or below to above), draws a colored dot on the 32°F horizontal reference line at that
x-position. Below = good refreeze (teal dot), above = no refreeze (red dot).

### Where to insert
After the Tmin `drawLine` calls, before the Today vertical line block.

### Exact code to add
```js
  // ── Freeze threshold crossing dots on Tmin line ──
  for(var fc=1; fc<N; fc++) {
    var prevMin = traj[fc-1].tmin;
    var currMin = traj[fc].tmin;
    if(prevMin === null || currMin === null) continue;
    var crossedBelow = prevMin >= 32 && currMin < 32;  // good: started freezing
    var crossedAbove = prevMin < 32  && currMin >= 32; // bad: stopped freezing
    if(!crossedBelow && !crossedAbove) continue;

    var dotX = xPos(fc);
    var dotY = yT(32); // on the 32°F reference line
    var dotColor = crossedBelow ? '#00d4aa' : '#ff4757';
    var dotLabel = crossedBelow ? '❄' : '💧';

    ctx.beginPath();
    ctx.arc(dotX, dotY, 3.5, 0, Math.PI * 2);
    ctx.fillStyle = dotColor;
    ctx.fill();
    ctx.strokeStyle = 'rgba(0,0,0,0.4)';
    ctx.lineWidth = 0.5;
    ctx.stroke();

    // Tiny emoji label above dot
    ctx.font = '8px -apple-system,sans-serif';
    ctx.textAlign = 'center';
    ctx.fillStyle = dotColor;
    ctx.fillText(dotLabel, dotX, dotY - 6);
  }
```

---

## Enhancement 3 — Storm Event Labels on Snowfall Bars
**Priority: HIGH**

### What it does
Adds a text label (e.g. `6"`) above snowfall bars that meet or exceed a threshold (4").
Smaller events already have the ❄ glyph — this adds the inch value for significant storms.

### Where to change
Inside the existing snowfall bars loop. Find this block:
```js
    if(sf >= 4) {
      ctx.fillStyle = 'rgba(210,235,255,0.9)';
      ctx.font = '7px -apple-system,sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('❄', bx + PX_PER_DAY/2, by - 1);
    }
```

Replace with:
```js
    if(sf >= 4) {
      ctx.fillStyle = 'rgba(210,235,255,0.9)';
      ctx.font = 'bold 7px -apple-system,sans-serif';
      ctx.textAlign = 'center';
      var snowLabel = sf >= 10
        ? Math.round(sf) + '"'
        : sf.toFixed(sf < 1 ? 1 : 0) + '"';
      ctx.fillText(snowLabel, bx + PX_PER_DAY/2, by - 2);
    } else if(sf >= 2) {
      // Smaller but notable — just the glyph, smaller
      ctx.fillStyle = 'rgba(210,235,255,0.7)';
      ctx.font = '6px -apple-system,sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('❄', bx + PX_PER_DAY/2, by - 1);
    }
```

---

## Enhancement 4 — Tap/Click Tooltip
**Priority: MEDIUM**

### What it does
On tap (mobile) or click (desktop), shows a small popup card above the chart with:
date, Hi°F, Lo°F, snowfall, and the SFQ%/CCR% values for that day.
Dismisses on second tap anywhere.

### Where to add — HTML (in `renderResult`, the ripeness-chart-wrap block)
Change:
```js
h+='<div id="ripeness-scroll" style="overflow-x:auto;-webkit-overflow-scrolling:touch;border-radius:8px">';
h+='<canvas id="ripeness-canvas" style="display:block;height:240px"></canvas>';
h+='</div>';
```
To:
```js
h+='<div id="ripeness-scroll" style="overflow-x:auto;-webkit-overflow-scrolling:touch;border-radius:8px;position:relative">';
h+='<canvas id="ripeness-canvas" style="display:block;height:240px"></canvas>';
h+='<div id="ripeness-tooltip" style="display:none;position:absolute;top:4px;left:50%;transform:translateX(-50%);background:rgba(13,27,42,0.95);border:1px solid rgba(0,212,170,0.4);border-radius:8px;padding:.35rem .6rem;font-size:.68rem;color:var(--text);pointer-events:none;white-space:nowrap;z-index:10"></div>';
h+='</div>';
```

### Where to add — JS (at the end of `drawRipenessChart`, before the closing `}`)
```js
  // ── Tap tooltip ──
  var ttEl = document.getElementById('ripeness-tooltip');
  if(ttEl) {
    canvas.onclick = function(e) {
      var rect = canvas.getBoundingClientRect();
      var scaleX = canvas.width / (window.devicePixelRatio || 1) / rect.width;
      var clickX = (e.clientX - rect.left) * scaleX;
      // Find nearest day index
      var nearestIdx = Math.round((clickX - PAD.left - PX_PER_DAY/2) / PX_PER_DAY);
      nearestIdx = Math.max(0, Math.min(N-1, nearestIdx));
      var t = traj[nearestIdx];
      if(!t) return;

      if(ttEl.dataset.shown === t.date) {
        ttEl.style.display = 'none';
        ttEl.dataset.shown = '';
        return;
      }

      var MNAMES=['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];
      var dt = new Date(t.date + 'T12:00:00');
      var dateStr = MNAMES[dt.getMonth()] + ' ' + dt.getDate();
      var sfqPct = t.surfaceRipeness !== null ? Math.round((1 - t.surfaceRipeness)*100)+'%' : '—';
      var ccrPct = t.columnColdContent !== null ? Math.round(t.columnColdContent*100)+'%' : '—';
      var snowStr = (t.snowfall && t.snowfall >= 0.1) ? t.snowfall.toFixed(1)+'"' : 'none';
      var hiStr = t.tmax !== null ? Math.round(t.tmax)+'°' : '—';
      var loStr = t.tmin !== null ? Math.round(t.tmin)+'°' : '—';
      var fcstTag = t.isForecast ? ' · forecast' : ' · archive';

      ttEl.innerHTML = '<strong>' + dateStr + '</strong>' + fcstTag
        + ' &nbsp;|&nbsp; Hi ' + hiStr + ' Lo ' + loStr
        + ' &nbsp;|&nbsp; ❄ ' + snowStr
        + ' &nbsp;|&nbsp; Sfc ' + sfqPct + ' · Col ' + ccrPct;

      // Position tooltip horizontally near click, clamped to scroll container
      var scrollEl = canvas.parentElement;
      var scrollLeft = scrollEl ? scrollEl.scrollLeft : 0;
      var dotX = xPos(nearestIdx) / (window.devicePixelRatio || 1);
      var visibleLeft = scrollLeft;
      var pct = Math.max(10, Math.min(90, ((dotX - visibleLeft) / (scrollEl.offsetWidth||320)) * 100));
      ttEl.style.left = pct + '%';
      ttEl.style.display = 'block';
      ttEl.dataset.shown = t.date;
    };
  }
```

---

## Enhancement 5 — Optimal Zone Shading (75–100% band)
**Priority: MEDIUM**

### What it does
Draws a very subtle green horizontal band between `yH(0.75)` and `yH(1.0)` across the
full chart width. Helps users visually anchor what "excellent" pack health looks like
without reading the % labels.

### Where to insert
Right after the grid lines `forEach` block and after the pre-season shading block,
before the temperature reference lines. It must render before all data lines.

### Exact code to add
```js
  // ── Optimal zone highlight (75–100%) ──
  ctx.fillStyle = 'rgba(46,213,115,0.045)';
  ctx.fillRect(PAD.left, yH(1.0), cW, yH(0.75) - yH(1.0));
  // Tiny "optimal" label at right edge
  ctx.fillStyle = 'rgba(46,213,115,0.30)';
  ctx.font = '7px -apple-system,sans-serif';
  ctx.textAlign = 'right';
  ctx.fillText('optimal', PAD.left + cW - 2, yH(0.88) + 3);
```

---

## Enhancement 6 — Historical Average CCC Reference Line
**Priority: MEDIUM**

### What it does
Draws a faint horizontal dashed line showing the historical average column cold content
for the current month, so the user can see whether this year is running above or below
normal. Uses the `SEASONAL_BASELINE` values from the model (which represent typical
March/April/May averages).

### Where to insert
After the 50% marginal zone line, before the snowfall bars.

### Exact code to add
```js
  // ── Historical average CCC reference for current month ──
  var SEASONAL_BASELINE = {3:0.85, 4:0.60, 5:0.35, 6:0.15};
  var histAvg = SEASONAL_BASELINE[month] || 0.50;
  var histY = yH(histAvg);
  if(histY >= PAD.top && histY <= dataBot) {
    ctx.strokeStyle = 'rgba(93,184,240,0.20)';
    ctx.lineWidth = 1;
    ctx.setLineDash([6,4]);
    ctx.beginPath();
    ctx.moveTo(PAD.left, histY);
    ctx.lineTo(PAD.left + cW, histY);
    ctx.stroke();
    ctx.setLineDash([]);
    ctx.fillStyle = 'rgba(93,184,240,0.30)';
    ctx.font = '7px -apple-system,sans-serif';
    ctx.textAlign = 'right';
    ctx.fillText('avg', PAD.left + cW - 2, histY - 2);
  }
```

---

## Enhancement 7 — "Window Closing" Annotation at 50% CCR Crossing
**Priority: LOWER**

### What it does
Scans the season portion of the trajectory (marchFirstIdx onward) for the first day where
`columnColdContent` drops below 0.50. Marks that x-position with a small vertical tick
and label "Pack → marginal" above the marginal line. Gives urgency without requiring
the user to interpret the chart themselves.

### Where to insert
After the Today vertical line block, before the Target date vertical line block.

### Exact code to add
```js
  // ── "Pack going marginal" crossing annotation ──
  var marginalCrossIdx = -1;
  for(var mc=marchFirstIdx+1; mc<N; mc++) {
    if(traj[mc-1].columnColdContent >= 0.50 && traj[mc].columnColdContent < 0.50) {
      // Only annotate future crossings (at or after today)
      if(mc >= todayIdx) {
        marginalCrossIdx = mc;
        break;
      }
    }
  }
  if(marginalCrossIdx >= 0) {
    var mcX = xPos(marginalCrossIdx);
    ctx.strokeStyle = 'rgba(255,71,87,0.50)';
    ctx.lineWidth = 1.5;
    ctx.setLineDash([]);
    ctx.beginPath();
    ctx.moveTo(mcX, yH(0.50) - 6);
    ctx.lineTo(mcX, yH(0.50) + 6);
    ctx.stroke();
    ctx.fillStyle = 'rgba(255,71,87,0.65)';
    ctx.font = 'bold 7px -apple-system,sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('⚠ pack→marginal', mcX, yH(0.50) - 9);
  }
```

---

## Enhancement 8 — Pre-Season Label Polish
**Priority: LOWER**

### What it does
Replaces the hard-to-read rotated "Pre-season" text with a ❄️ icon centered in the
pre-season zone, and adds a short horizontal label above the chart (in the PAD.top area)
reading "← Jan/Feb" in muted text — readable on small phones without rotation.

### Where to change
Find the existing pre-season label block inside `if(marchFirstIdx > 0)`:
```js
    // "Pre-season" label
    ctx.fillStyle = 'rgba(120,160,190,0.45)';
    ctx.font = '8px -apple-system,sans-serif';
    ctx.textAlign = 'center';
    ctx.save();
    ctx.translate(psX0 + (psX1 - psX0) / 2, PAD.top + (dataBot - PAD.top) / 2);
    ctx.rotate(-Math.PI / 2);
    ctx.fillText('Pre-season', 0, 0);
    ctx.restore();
```

Replace with:
```js
    // Pre-season label — icon centered in zone, text in top padding
    var psMidX = psX0 + (psX1 - psX0) / 2;
    var psMidY = PAD.top + (dataBot - PAD.top) / 2;

    // Large icon in center of zone
    ctx.font = '13px -apple-system,sans-serif';
    ctx.textAlign = 'center';
    ctx.fillStyle = 'rgba(120,160,190,0.35)';
    ctx.fillText('❄️', psMidX, psMidY + 5);

    // Short label in the top padding above the chart area
    ctx.font = '7px -apple-system,sans-serif';
    ctx.fillStyle = 'rgba(120,160,190,0.45)';
    ctx.fillText('← Jan/Feb', psMidX, PAD.top - 10);
```

---

## Suggested Implementation Order

1. **Enhancement 8** — 5 min, pure visual polish, zero risk
2. **Enhancement 5** — 5 min, three lines of code
3. **Enhancement 6** — 5 min, five lines of code
4. **Enhancement 3** — 10 min, modifies existing loop
5. **Enhancement 2** — 15 min, new loop, straightforward
6. **Enhancement 7** — 15 min, new scan + draw
7. **Enhancement 1** — 20 min, needs signature change + band logic
8. **Enhancement 4** — 30 min, HTML + JS, most complex

---

## Changelog Entry for v23

```
### Session [N] — v23 (Chart Comprehension: 8 enhancements)

1. Corn window band on target date: green/amber/red vertical band at tgtIdx spanning 
   corn opens→closes. Width proportional to window duration within the day column. 
   Left edge solid (opens), right edge dashed (closes). Color by score band. 
   Requires cornWin passed as 4th arg to drawRipenessChart.

2. Freeze threshold crossing dots: scans traj for tmin crossing 32°F. Teal dot = 
   started freezing (upside), red dot = stopped freezing (risk). Plotted on the 32° 
   reference line with ❄/💧 glyph above.

3. Storm event inch labels: snowfall bars ≥4" now show inch value (e.g. "6\"") above 
   the bar in bold 7px. Bars 2–4" show ❄ glyph at 6px. Bars <2" unchanged.

4. Tap tooltip: canvas onclick shows floating card with date, Hi°F, Lo°F, snowfall, 
   SFQ%, CCR%. Second tap on same point dismisses. Positioned proportionally within 
   scroll container. Implemented via ripeness-tooltip div injected in renderResult.

5. Optimal zone shading: rgba(46,213,115,0.045) fill between yH(1.0) and yH(0.75). 
   Faint "optimal" label at right edge. Renders before all data lines.

6. Historical average CCR reference line: SEASONAL_BASELINE[month] drawn as faint 
   blue dashed horizontal with "avg" label. Only renders if within PAD.top→dataBot.

7. Pack-going-marginal annotation: scans for first future day where CCR crosses below 
   0.50. Draws small vertical tick on the marginal line + "⚠ pack→marginal" label. 
   Only annotates future crossings (mc >= todayIdx).

8. Pre-season label polish: rotated "Pre-season" text replaced with ❄️ icon centered 
   in zone + "← Jan/Feb" label in PAD.top area. More readable on small phones.
```
