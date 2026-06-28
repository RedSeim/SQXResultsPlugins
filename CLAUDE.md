# StrategyQuantX ResultsPlugins — AI Context Guide

This file gives AI assistants full context for building ResultsPlugins for StrategyQuantX (SQX).

---

## Architecture

- ResultsPlugins are **single-file HTML apps** (`index.html`) loaded in an `<iframe>` inside the SQX Results tab.
- Communication with SQX is exclusively via the **PostMessage API** (`window.parent.postMessage` / `window.addEventListener('message', ...)`).
- SQX automatically scans `user/extend/ResultsPlugins/` and registers each subfolder with an `index.html` as a new Results tab.
- After changing plugins, **restart SQX** or use the **reload icon** in the Results UI to pick up changes without a full restart.

### Plugin limits by license tier

| License | Custom plugins allowed |
|---------|----------------------|
| **Starter** | 0 (not available) |
| **Pro** | 3 |
| **Ultimate** | Unlimited |

If the limit is exceeded, the plugin list is truncated and any additional plugins are ignored and won’t be registered or shown in the UI.

---

## Folder Structure

```
user/extend/ResultsPlugins/
  └── MyPluginName/
      ├── index.html          ← Required entry point
      ├── vue.global.prod.js  ← Optional: copy from an existing plugin if using Vue 3
      └── locales/            ← Optional: i18n JSON files
          ├── en.json
          └── de.json
```

---

## Tech Stack

- **Vanilla JS** or **Vue 3** (`vue.global.prod.js` included locally — no CDN, no npm, no bundler).
- Everything in one `index.html` file unless you add local assets.
- CSS custom properties for theming. No Bootstrap, no Tailwind (unless you inline it).
- Canvas (`<canvas>`) for custom charts; no chart library is pre-bundled.

---

## Lifecycle / Initialization Pattern

```javascript
// On DOMContentLoaded (vanilla) or onMounted (Vue)
document.addEventListener('DOMContentLoaded', () => {
  // 1. Apply default theme from system preference
  const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
  document.documentElement.setAttribute('data-theme', prefersDark ? 'dark' : 'light');

  // 2. Register the PostMessage listener
  window.addEventListener('message', handleMessage);

  // 3. Optional: ResizeObserver for canvas-based charts
  // const ro = new ResizeObserver(() => drawChart());
  // ro.observe(document.getElementById('myCanvas'));
});
```

---

## PostMessage API

### Incoming messages (SQX → plugin)

#### `STRATEGY_DATA`
Fired when a strategy is selected or the tab becomes active.

```javascript
// event.data.data structure:
{
  projectName: string,
  databankName: string,
  strategyName: string,
  resultKey: string,          // Usually 'Portfolio'
  isAlgoWizard: boolean,
  isAlgoWizardCloud: boolean,
  backtestID: string,         // AlgoWizard only
  nodeID: string              // AlgoWizardCloud only
}
```

**Always validate before making requests:**
```javascript
function isValidStrategy(d) {
  return d?.projectName?.trim() && d?.databankName?.trim() && d?.strategyName?.trim();
}
```

---

#### `SET_THEME`
Fired on plugin load and when the user changes the skin.

```javascript
// event.data.theme === 'dark' | 'light'
document.documentElement.setAttribute('data-theme', event.data.theme);
// For canvas charts: redraw after theme change
```

---

#### `SET_LANGUAGE`
Fired when the user changes the application language.

```javascript
// event.data.language or event.data.data
// value is a string locale code e.g. 'en', 'de', 'cz', 'es', 'ru', 'pl', 'pt', 'fr', 'it', 'id', 'zh-CN', 'zh-TW'
```

---

#### `STATS_RESPONSE`
Response to `GET_STATS`.

```javascript
// event.data.data.stats — object with the fields listed in the Stats Fields section below
const stats = event.data.data.stats;
```

---

#### `ORDERS_RESPONSE`
Response to `GET_ORDERS`.

```javascript
// event.data.data.orders — array of order objects (see Order Fields section below)
const orders = event.data.data.orders;
```

---

#### `LAST_SETTINGS_XML_RESPONSE`
Response to `GET_LAST_SETTINGS_XML`.

```javascript
// event.data.data.lastSettingsXml — XML string with backtest configuration
// Sections: Data (symbol, timeframe, date range), Money Management, Trading Options, etc.
const xml = event.data.data.lastSettingsXml;
const doc = new DOMParser().parseFromString(xml, 'text/xml');
```

---

#### `SOURCE_CODE_RESPONSE`
Response to `GET_SOURCE_CODE`.

```javascript
const d = event.data.data;
// d.success === 'ok' on success
// d.code       — generated source code string
// d.warning    — optional warning message
// d.learnMoreURL — optional documentation URL
```

---

#### `SYMBOL_INFO_RESPONSE`
Response to `GET_SYMBOL_INFO`.

```javascript
const symbolObj = event.data.data;  // symbol object or null
```

---

### Outgoing messages (plugin → SQX)

> **Note:** `projectName`, `databankName`, and `strategyName` are **automatically merged** by SQX from the current strategy context. You do not need to include them in params.

#### `GET_STATS`

```javascript
window.parent.postMessage({
  type: 'GET_STATS',
  params: {
    resultKey: 'Portfolio',
    direction: '0',      // '0'=Both, '1'=Long, '-1'=Short
    sampleType: '127',   // '127'=Full, '10'=In-Sample, '20'=Out-of-Sample
    plType: '10'         // '10'=Money, '20'=Percent, '30'=Pips
  }
}, '*');
```

#### `GET_ORDERS`

```javascript
window.parent.postMessage({
  type: 'GET_ORDERS',
  params: {
    resultKey: 'Portfolio',
    direction: '0',
    sampleType: '127'
  }
}, '*');
```

#### `GET_LAST_SETTINGS_XML`

```javascript
window.parent.postMessage({ type: 'GET_LAST_SETTINGS_XML', params: {} }, '*');
```

#### `GET_SOURCE_CODE`

```javascript
// By format shorthand:
window.parent.postMessage({
  type: 'GET_SOURCE_CODE',
  params: { format: 'xml' }  // 'xml' | 'pseudo' | 'mq4' | 'mq5' | 'el' | 'pla' | 'java'
}, '*');

// Or by full type name:
window.parent.postMessage({
  type: 'GET_SOURCE_CODE',
  params: { type: 'MetaTrader 4 (*.mq4)' }
}, '*');
```

#### `GET_SYMBOL_INFO`

```javascript
window.parent.postMessage({
  type: 'GET_SYMBOL_INFO',
  symbol: 'EURUSD_M1_dukas'
}, '*');
```

---

## CSS Theming Pattern

```css
:root {
  --bg-body: #ffffff;
  --bg-surface: #f8f8f8;
  --text-main: #333333;
  --text-muted: #777777;
  --border-color: #dddddd;
  --primary: #337ab7;
  --primary-hover: #286090;
  --success: #5cb85c;
  --danger: #d9534f;
}

[data-theme="dark"] {
  --bg-body: #1e1e1e;
  --bg-surface: #2d2d2d;
  --text-main: #e0e0e0;
  --text-muted: #aaaaaa;
  --border-color: #444444;
  --primary: #337ab7;
  --primary-hover: #4090d0;
  --success: #5cb85c;
  --danger: #d9534f;
}

body {
  background-color: var(--bg-body);
  color: var(--text-main);
  font-family: "Open Sans", Helvetica, Arial, sans-serif;
  font-size: 14px;
  margin: 0;
  padding: 20px;
  height: 100vh;
  overflow: hidden; /* IMPORTANT: fit into iframe, no outer scroll */
}
```

Apply theme:
```javascript
document.documentElement.setAttribute('data-theme', theme); // 'dark' | 'light'
```

---

## Stats Fields (STATS_RESPONSE)

All fields on `event.data.data.stats`:

| Field | Description |
|-------|-------------|
| `NetProfit` | Net profit in currency |
| `NetProfitInPips` | Net profit in pips |
| `NumberOfTrades` | Total trades |
| `NumberOfProfits` | Winning trades |
| `NumberOfLosses` | Losing trades |
| `NumberOfCanceled` | Canceled trades |
| `GrossProfit` | Total gross profit |
| `GrossLoss` | Total gross loss |
| `AvgTrade` | Average profit per trade |
| `AvgAbsTrade` | Average absolute profit per trade |
| `AvgWin` | Average winning trade |
| `AvgLoss` | Average losing trade |
| `MaxProfit` | Best single trade |
| `MaxLoss` | Worst single trade |
| `WinningPct` | Win rate (0–100) |
| `WinLossRatio` | Win/Loss ratio |
| `ProfitFactor` | Gross profit / Gross loss |
| `PayoutRatio` | Payout ratio |
| `Drawdown` | Max drawdown (currency) |
| `PctDrawdown` | Max drawdown (%) |
| `PipsDrawdown` | Max drawdown (pips) |
| `SharpeRatio` | Sharpe ratio |
| `CalmarRatio` | Calmar ratio |
| `SQN` | System Quality Number |
| `SQNScore` | SQN score (0.0–1.0, multiply by 100 for %) |
| `Stability` | Equity curve straightness/steepness |
| `Symmetry` | Long vs Short profit symmetry |
| `ReturnDDRatio` | Return / Drawdown |
| `ReturnOpenDDRatio` | Return / Open Drawdown |
| `AAR_DD_Ratio` | Avg Annual Return / Drawdown |
| `RExpectancy` | Risk-adjusted expectancy |
| `RExpectancyScore` | Risk-adjusted expectancy score |
| `Expectancy` | Expected value per trade |
| `StandardDev` | Standard deviation |
| `AHPR` | Average Holding Period Return |
| `ZScore` | Z-Score |
| `ZProbability` | Z-Score probability |
| `Exposure` | Market exposure % |
| `ExposurePosition` | Position exposure |
| `MaxConsecWins` | Max consecutive wins |
| `MaxConsecLoss` | Max consecutive losses |
| `AvgConsecWin` | Avg consecutive wins |
| `AvgConsecLoss` | Avg consecutive losses |
| `AvgBarsWin` | Avg bars in winning trades |
| `AvgBarsLoss` | Avg bars in losing trades |
| `AvgBarsTrade` | Avg bars per trade |
| `AvgTradesPerDay` | Avg trades/day |
| `AvgTradesPerMonth` | Avg trades/month |
| `AvgTradesPerYear` | Avg trades/year |
| `AvgProfitByDay` | Avg profit/day |
| `AvgProfitByMonth` | Avg profit/month |
| `AvgProfitByYear` | Avg profit/year |
| `AvgPctProfitByYear` | Avg % profit/year |
| `TotalTradingDays` | Total trading days |
| `TotalTradingMonths` | Total trading months |
| `TotalTradingYears` | Total trading years |
| `ProfitableMonths` | Number of profitable months |
| `DegreesOfFreedom` | Degrees of freedom |
| `StagnationPeriod` | Stagnation period (bars) |
| `StagnationPeriodPct` | Stagnation period (%) |
| `StagnationFrom` | Stagnation start (bar index) |
| `StagnationTo` | Stagnation end (bar index) |
| `MaxNewHighDurationPeriod` | Max new-high duration (bars) |
| `MaxNewHighDurationPct` | Max new-high duration (%) |
| `MaxNewHighDurationFrom` | Max new-high start (bar index) |
| `MaxNewHighDurationTo` | Max new-high end (bar index) |
| `Commission` | Total commission |
| `SlippageInMoney` | Total slippage (currency) |
| `CommSwapInMoney` | Total commission + swap (currency) |

---

## Order Fields (ORDERS_RESPONSE)

Each object in `event.data.data.orders`:

| Field | Description |
|-------|-------------|
| `Ticket` | Order ID |
| `Symbol` | Trading symbol |
| `ResultName` | Setup/result name |
| `NumberOfTrade` | Sequence number |
| `OpenTime` | Open timestamp |
| `CloseTime` | Close timestamp |
| `OriginalOpenTime` | Original open time (pending orders) |
| `Type` | Order type (byte — see constants below) |
| `CloseType` | Close reason (byte — see constants below) |
| `SampleType` | 127=Full, 10=In-Sample, 20=Out-of-Sample |
| `OpenPrice` | Entry price |
| `ClosePrice` | Exit price |
| `OriginalOpenPrice` | Original price (pending orders) |
| `SLLevel` | Stop Loss price |
| `PTLevel` | Profit Target price |
| `Size` | Lot size |
| `BarsInTrade` | Bars open |
| `TimeInTrade` | Duration (seconds) |
| `ProfitLoss` | P&L (currency) |
| `ProfitLossPct` | P&L (%) |
| `ProfitLossPips` | P&L (pips) |
| `Drawdown` | Drawdown at close (currency) |
| `PctDrawdown` | Drawdown at close (%) |
| `MAE` | Max Adverse Excursion (currency) |
| `MAEPips` | Max Adverse Excursion (pips) |
| `MFE` | Max Favorable Excursion (currency) |
| `MFEPips` | Max Favorable Excursion (pips) |
| `Balance` | Account balance after close |
| `CommSwap` | Commission + swap |
| `SlippageInMoney` | Slippage (currency) |
| `Comment` | Order comment |

> **Note:** Missing/N/A values are `null` or `-99999999` (constant `NOT_DEFINED`).

### Order Type Constants

```
0=Any  1=Buy  2=Sell  3=BuyLimit  4=SellLimit  5=BuyStop  6=SellStop
7=BuyStopLimit  8=SellStopLimit  9=Deposit  10=Withdrawal  11=Balance
100=BuyToCoverStop  101=BuyToCoverLimit  102=SellToCoverStop  103=SellToCoverLimit
```

### Order CloseType Constants

```
1=Manual  2=SL  3=PT  4=EndTest  5=EndOfDay  6=Expired  7=Reversed
8=Deleted  9=Replaced  11=OCA  12=Commission  13=EndOfDay(Time)
14=EndOfFriday  16=EndOfFriday(Time)  17=EndOfRange  18=ControlOrder
19=ExitAfterXBars  20=MoveSL2BE  21=TrailingStop  22=ExitSignal
55=EndOfDay(NextMarketOpen)  60=Delisted
```

---

## Minimal Working Template (Vanilla JS)

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>My Plugin</title>
<style>
:root { --bg-body:#fff; --text-main:#333; --primary:#337ab7; }
[data-theme="dark"] { --bg-body:#1e1e1e; --text-main:#e0e0e0; }
body { margin:0; padding:20px; background:var(--bg-body); color:var(--text-main);
       font-family:Arial,sans-serif; font-size:14px; height:100vh; overflow:hidden; }
</style>
</head>
<body>
<div id="app">
  <h2 id="title">Select a strategy...</h2>
  <pre id="output"></pre>
</div>
<script>
let strategy = null;

window.addEventListener('message', function(e) {
  const { type, data, theme } = e.data;

  if (type === 'SET_THEME') {
    document.documentElement.setAttribute('data-theme', theme);
  }

  if (type === 'STRATEGY_DATA') {
    strategy = data;
    if (!strategy?.projectName) return;
    document.getElementById('title').textContent = strategy.strategyName;
    // Request stats
    window.parent.postMessage({
      type: 'GET_STATS',
      params: { resultKey: 'Portfolio', direction: '0', sampleType: '127', plType: '10' }
    }, '*');
  }

  if (type === 'STATS_RESPONSE') {
    const s = data.stats;
    document.getElementById('output').textContent =
      `Trades: ${s.NumberOfTrades}\nNet Profit: ${s.NetProfit}\nWin%: ${s.WinningPct}`;
  }
});
</script>
</body>
</html>
```

---

## Minimal Working Template (Vue 3)

```html
<!DOCTYPE html>
<html lang="en" data-theme="light">
<head>
<meta charset="UTF-8">
<title>My Vue Plugin</title>
<script src="vue.global.prod.js"></script>
<style>
:root { --bg-body:#fff; --text-main:#333; --primary:#337ab7; }
[data-theme="dark"] { --bg-body:#1e1e1e; --text-main:#e0e0e0; }
*, *::before, *::after { box-sizing:border-box; }
body { margin:0; padding:20px; background:var(--bg-body); color:var(--text-main);
       font-family:Arial,sans-serif; font-size:14px; height:100vh; overflow:hidden;
       display:flex; flex-direction:column; }
#app { display:flex; flex-direction:column; height:100%; }
</style>
</head>
<body>
<div id="app">
  <h2>{{ strategy?.strategyName || 'Select a strategy...' }}</h2>
  <p v-if="stats">Trades: {{ stats.NumberOfTrades }} | Net Profit: {{ stats.NetProfit }}</p>
</div>
<script>
const { createApp, ref, onMounted } = Vue;

createApp({
  setup() {
    const strategy = ref(null);
    const stats = ref(null);

    function handleMessage(e) {
      const { type, data, theme } = e.data;
      if (type === 'SET_THEME') {
        document.documentElement.setAttribute('data-theme', theme);
      }
      if (type === 'STRATEGY_DATA') {
        strategy.value = data;
        if (!data?.projectName?.trim()) return;
        window.parent.postMessage({
          type: 'GET_STATS',
          params: { resultKey: 'Portfolio', direction: '0', sampleType: '127', plType: '10' }
        }, '*');
      }
      if (type === 'STATS_RESPONSE') {
        stats.value = data.stats;
      }
    }

    onMounted(() => {
      window.addEventListener('message', handleMessage);
    });

    return { strategy, stats };
  }
}).mount('#app');
</script>
</body>
</html>
```

---

## Key Rules & Pitfalls

- **Always validate `STRATEGY_DATA`** before sending requests. The message can arrive with empty/null fields on initial load.
- **`body` must be `overflow:hidden`** — the plugin is in an iframe and should not add outer scrollbars.
- **Use `ResizeObserver`** (not `window.resize`) for canvas charts — the iframe can resize independently.
- **Use `nextTick()`** (Vue) before reading canvas/DOM dimensions after a reactive update.
- **Canvas charts need manual redraw** on `SET_THEME` — CSS variables don't affect canvas paint colors.
- **Don't include jQuery or heavy libraries** — SQX is offline; all assets must be local.
- **`vue.global.prod.js`** is Vue 3 IIFE build. Use `const { createApp, ref, reactive, computed, onMounted, nextTick, watch } = Vue;`.
- **Auto-merged params**: you never need to pass `projectName`/`databankName`/`strategyName` in request params — SQX adds them automatically.
- **Source code formats**: `xml`, `pseudo`, `mq4`, `mq5`, `el`, `pla`, `java`.
- **`plType` affects numeric values** in stats: `'10'`=Money (currency), `'20'`=Percent, `'30'`=Pips.
- **`sampleType`**: `'127'`=Full data, `'10'`=In-Sample, `'20'`=Out-of-Sample.
- **`direction`**: `'0'`=Both sides, `'1'`=Long only, `'-1'`=Short only.

---

## Tooltips & Explaining Metrics to Users

SQX users range from beginners to professional quants. Most stats fields (`SQN`, `RExpectancy`, `Stability`, `ZScore`, `Calmar`, etc.) mean nothing without context. **Every non-obvious metric shown in a plugin should have a tooltip** that answers three questions in one or two short sentences:

1. **What it is** — one-line definition in plain language
2. **How to read it** — what counts as good / ok / bad, with concrete numbers
3. **Why it matters** — what decision it helps the user make

### Implementation pattern (pure CSS, no libraries)

```html
<span class="metric" data-tip="Profit Factor: gross profit ÷ gross loss.
Good: >1.5 · OK: 1.2–1.5 · Weak: <1.2.
Tells you how much you win per $1 lost.">
  Profit Factor: 1.87
</span>
```

```css
.metric { position: relative; cursor: help; border-bottom: 1px dotted var(--text-muted); }
.metric:hover::after {
  content: attr(data-tip);
  position: absolute;
  left: 0; top: 100%;
  margin-top: 6px;
  white-space: pre-line;   /* keep line breaks from the attribute */
  background: var(--bg-surface-2, #2f3033);
  color: var(--text-main);
  border: 1px solid var(--border-color);
  padding: 8px 10px;
  border-radius: 6px;
  font-size: 12px;
  max-width: 280px;
  z-index: 100;
  box-shadow: 0 4px 12px rgba(0,0,0,0.25);
  pointer-events: none;
}
```

**Rules:**
- Use `white-space: pre-line` so `\n` in `data-tip` renders as real line breaks.
- Put tooltips on anything derived (scores, grades, composites) AND on any raw metric a non-trader wouldn't recognize on sight.
- Keep tooltips to ≤3 short lines. If more is needed, the metric is too complex for a tooltip — add a help panel instead.
- Use thresholds consistent with the rest of the plugin. If your Profit Factor "good" starts at 1.3, don't write 1.5 in the tooltip.
- Never quote copyrighted definitions — write your own plain-language version.

### Ready-to-use tooltip copy for common metrics

| Metric | Plain tooltip text |
|---|---|
| `ProfitFactor` | Gross profit ÷ gross loss. Good: >1.5 · OK: 1.2–1.5 · Weak: <1.2. How much you win per $1 lost. |
| `SQN` | System Quality Number (Van Tharp). >2.5 tradeable, >3.5 excellent, >5 exceptional. Rewards high expectancy + many trades. |
| `Stability` | How straight the equity curve is. 1.0 = perfect line, <0.7 = bumpy. Higher means returns are spread evenly, not from a few lucky trades. |
| `RExpectancy` | Average profit per trade measured in units of risk (R). 0.3R is solid, 0.5R+ is great. Tells you what you make per unit risked. |
| `ReturnDDRatio` | Net profit ÷ max drawdown. >3 is healthy, >5 is strong. How much reward you got per unit of pain. |
| `CalmarRatio` | Annual return ÷ max drawdown. >1 decent, >3 excellent. Same idea as Return/DD but annualized. |
| `SharpeRatio` | Risk-adjusted return. >1 acceptable, >2 good, >3 excellent. Penalizes volatility in returns. |
| `MaxDD` / `PctDrawdown` | Worst peak-to-valley loss. The realistic pain you must survive to trade this system live. |
| `Symmetry` | How balanced long and short sides are. 1.0 = perfectly symmetric, 0 = one-sided. One-sided strategies depend on a single market regime. |
| `ZScore` / `ZProbability` | Whether wins and losses cluster (streaky) or alternate randomly. Large |Z| with high probability means non-random streaks — investigate before trusting. |
| `Exposure` | % of time money is in the market. Low exposure + high return is efficient; high exposure means more market risk. |
| `MaxConsecLoss` | Longest losing streak observed. You must be psychologically and financially prepared for at least this many losses in a row. |
| `DegreesOfFreedom` | Roughly: trades minus parameters. Low values mean the strategy has too few trades to justify its complexity — overfitting risk. |

### Practical examples of user-friendly metric displays

**Example 1 — Raw stat with inline tooltip**

```html
<div class="stat">
  <span class="label" data-tip="Profit Factor: gross profit ÷ gross loss.
Good: >1.5 · OK: 1.2–1.5 · Weak: <1.2.
Higher = more won per $1 lost.">Profit Factor</span>
  <span class="value">1.87</span>
</div>
```

**Example 2 — Composite score with breakdown in tooltip**

```javascript
const tip =
  `Robustness: weighted average of 6 pillars.\n` +
  `Profitability ${pScores.profit} · Risk ${pScores.risk} · Consistency ${pScores.consistency}\n` +
  `85+ A · 70+ B · 55+ C · 40+ D`;
```
```html
<div class="big-score" :data-tip="tip">87</div>
```

**Example 3 — Threshold-aware tooltip that reuses your CONFIG**

Instead of hard-coding tooltip thresholds, build them from the same normalization points used for scoring, so the two can never drift apart:

```javascript
function tooltipFor(label, points, desc) {
  // points: [[x,score],...]  e.g. [[1.0,0],[1.3,50],[2.0,100]]
  const weak = points[0][0], ok = points[1][0], good = points[2][0];
  return `${label}\n${desc}\nWeak: <${weak} · OK: ${ok} · Good: ${good}+`;
}

const pfTip = tooltipFor(
  'Profit Factor',
  [[1.0, 0], [1.3, 50], [2.0, 100]],
  'Gross profit ÷ gross loss.'
);
```

**Example 4 — Pillar row with per-input tooltips (for scorecards)**

```html
<div class="pillar">
  <div class="pillar-name" :data-tip="'Profitability\nHow much this system earns per trade and per risk unit.\nInputs: ProfitFactor, RExpectancy.'">
    Profitability
  </div>
  <div class="bar-track"><div class="bar-fill" :style="{width: score+'%'}"></div></div>
  <span class="inp" :data-tip="pfTip">PF {{ pf }}</span>
  <span class="inp" :data-tip="rexpTip">RExp {{ rexp }}</span>
</div>
```

**Example 5 — Help icon pattern for long explanations**

For anything that won't fit in 3 lines, show a small `(?)` circle that opens a help panel instead of cramming the tooltip:

```html
<h3>Robustness Score
  <button class="help-btn" @click="showHelp = !showHelp">?</button>
</h3>
<div v-if="showHelp" class="help-panel">
  <p>This score combines 6 pillars that together describe whether a strategy
     is likely to survive live trading...</p>
</div>
```

### Accessibility & UX notes

- Add a `title` attribute as fallback for touch devices: `title="..." data-tip="..."`.
- Keep tooltip text in a single place (a `TOOLTIPS` object) so you can later translate it via `SET_LANGUAGE` without hunting through markup.
- Don't put tooltips on decorative elements — they become noise.
- Dark mode: make sure `--bg-surface-2` and `--border-color` give the tooltip enough contrast against both themes.

---

## Disclaimer (Required)

Every ResultsPlugin that displays stats, risk metrics, simulations, scores, grades, or any interpretation of trading performance **must include a disclaimer at the bottom** of the UI. This is already the convention across the existing plugins — `Prop analytics` and `Prop Monte Carlo` both render one via a `disclaimerHtml` locale key, and new plugins must match.

### Why
Plugins are read by traders evaluating real strategies, sometimes for live deployment on prop-firm accounts. Statistical scores, Monte Carlo simulations, robustness grades, and any "good/bad" interpretation can be mistaken for financial advice. The disclaimer makes it explicit that the plugin is an analysis tool based on historical data, not a recommendation.

### Standard disclaimer text (English)

Use this wording verbatim as the baseline — it matches what `Prop analytics` and `Prop Monte Carlo` already use, so plugins stay consistent:

> This analysis is provided strictly for informational and educational purposes within StrategyQuant X. It does not constitute financial, investment, or trading advice. All results, including backtests and simulations, are based on historical data and hypothetical performance, which may not reflect future market conditions. Past performance is not indicative of future results. Trading involves substantial risk, and you should conduct your own independent research and consult a licensed financial advisor before making any investment or trading decisions. **This tool utilizes statistical math based on historical sequences; actual market execution may vary significantly.**

If your plugin does something unusual (Monte Carlo, ML, optimization, discretionary suggestions), append one extra sentence naming the specific caveat — don't replace the baseline text.

### Standard CSS (copy as-is)

This is the exact `.disclaimer` style used by the existing plugins. Reuse it so visual consistency across the Results tab is preserved.

```css
.disclaimer {
  margin-top: 20px;
  padding-top: 15px;
  border-top: 1px solid var(--border-color);
  font-size: 11px;
  color: var(--text-muted);
  line-height: 1.5;
  text-align: justify;
  flex-shrink: 0;
}
```

Notes:
- `flex-shrink: 0` keeps the disclaimer visible when the plugin uses a flex column layout (which it should — see the Vue 3 template).
- `border-top` + muted color makes it read as footer content, not body content.
- Do **not** put the disclaimer inside a scrollable region — it must always be visible.

### Placement

The disclaimer must be the **last element inside `#app`**, after all content. In the standard flex-column layout:

```
┌─────────────────────────┐
│ header                  │  flex-shrink: 0
├─────────────────────────┤
│                         │
│ main content (scrolls)  │  flex: 1
│                         │
├─────────────────────────┤
│ disclaimer              │  flex-shrink: 0
└─────────────────────────┘
```

### i18n pattern (match existing plugins)

The existing plugins store the disclaimer in each locale file under the key `disclaimerHtml` and render it with `v-html` so `<strong>` / `<br>` tags work:

```json
// locales/en.json
{
  "disclaimerHtml": "This analysis is provided strictly for informational..."
}
```

```html
<!-- Vue 3 -->
<div class="disclaimer" v-html="tHtml('disclaimerHtml')"></div>
```

For plugins without i18n, inline it directly:

```html
<div class="disclaimer">
  This analysis is provided strictly for informational and educational purposes
  within StrategyQuant X. It does not constitute financial, investment, or trading
  advice. All results are based on historical data and hypothetical performance,
  which may not reflect future market conditions. Past performance is not indicative
  of future results. Trading involves substantial risk, and you should conduct your
  own independent research before making any investment or trading decisions.
  <strong>This tool utilizes statistical math based on historical sequences; actual
  market execution may vary significantly.</strong>
</div>
```

### Checklist when reviewing a plugin
- [ ] `.disclaimer` element exists and is the last child of `#app`
- [ ] Uses the standard CSS block above (border-top, 11px, muted, justified)
- [ ] Visible without scrolling in the default iframe size
- [ ] Text covers: informational/educational purpose, not advice, historical/hypothetical data, past performance caveat, substantial risk, consult advisor
- [ ] If plugin runs simulations or produces scores/grades, adds one extra sentence naming that specific limitation
- [ ] If the plugin is localized, `disclaimerHtml` key exists in every `locales/*.json` file

### Reference plugins
- `Prop analytics/index.html:410` — CSS definition
- `Prop analytics/index.html:909` — markup with `v-html="tHtml('disclaimerHtml')"`
- `Prop analytics/locales/en.json` → `disclaimerHtml` — reference wording
- `Prop Monte Carlo/index.html:392` + `:518` — same pattern

---

## Existing Plugin Examples

| Plugin | Tech | Features |
|--------|------|---------|
| `Prop analytics/` | Vue 3 | Stats dashboard, metric cards, CSS var theming, responsive layout |
| `Prop Monte Carlo/` | Vue 3 + Canvas | Simulation with user params, canvas chart, locale loading, orders processing |
| `CustomPlugin/` | Vanilla JS | Full API reference + live demo of all message types |
| `RobustnessScorecard/` | Vue 3 | Weighted 6-pillar robustness score (0–100) with letter grade, pillar bars, hover tooltips with plain-language metric explanations, multi-direction `GET_STATS` (both/long/short) for symmetry analysis, centralized `TOOLTIPS` object for easy i18n |