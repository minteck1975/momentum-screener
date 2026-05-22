# 8-EMA Pullback Swing Screener

A two-part bot that screens US stocks for the **8-EMA Pullback** swing setup: stocks in an established uptrend that have just pulled back to the 8-day EMA on declining volume, with a bullish reversal candle signaling continuation.

## The strategy in 4 steps

### 1. Trend (the prerequisite)
- Higher highs and higher lows in the recent structure
- Price > 20-EMA > 50-EMA (**stacked**)
- 50-EMA sloping up
- Weekly chart confirms — price above the weekly 20-EMA, weekly slope up

### 2. Pullback (the entry zone)
- Price has pulled back to within ~1 ATR of the 8-EMA
- **Volume declining** during the pullback vs the prior impulse move (the key tell)
- Pullback is shallow-to-moderate (didn't break the 20-EMA badly)
- Sweet spot: 2-7 bars since the impulse peak

### 3. Trigger (when to actually click buy)
A bullish reversal candle on the most recent bar (or yesterday):
- **Bullish engulfing** (strongest)
- **Hammer / pin bar**
- **Inside-bar breakout**

Per the source material: ideally enter in the last hour of the trading day once the reversal is confirmed.

### 4. Risk management
- **Stop** below the trigger bar low / recent swing low (whichever lower)
- **Position size** so risk = 1-2% of account on the trade (dashboard calculates this for you)
- **Scale out** in quarters:
  - ¼ at T1 = prior swing high
  - ¼ at T2 = 2R from entry
  - Runner trails behind the 8-EMA

## Tiers

| Tier | Score | Means |
|---|---|---|
| **A_SETUP** | ≥ 75 | Full alignment: trend + pullback + trigger candle, all in place |
| **B_SETUP** | 60–74 | Trend + pullback present, trigger may be soft/pending |
| **WATCH** | 45–59 | Trend OK but needs more pullback or no trigger yet |
| **PASS** | < 45 | Doesn't meet criteria |
| **ILLIQUID** | — | Filtered out for < $20M average daily dollar volume |

## Files

| File | What it does |
|---|---|
| `swing_screener.py` | Scans the universe, writes `results.json` |
| `dashboard.html` | Self-contained dashboard — open in any browser |
| `results.json` | Output (auto-loaded by the dashboard) |

## Run it

```bash
pip install yfinance pandas numpy lxml

# Scan the full S&P 500 (~5-7 min)
python swing_screener.py
```

Then open `dashboard.html` (double-click, or serve via `python -m http.server` and visit localhost:8000/dashboard.html).

The screener fetches the current S&P 500 constituent list from Wikipedia at runtime, so the universe stays in sync as the index changes. If Wikipedia is unreachable, it falls back to a bundled 500-ticker snapshot. (`lxml` is needed for the Wikipedia HTML parsing.)

When `dashboard.html` is opened directly via `file://`, click **Load results.json** in the header to load your fresh scan; otherwise it shows the bundled demo data so the UI is functional on first open.

## Dashboard features

- **Tier filter** defaults to "Actionable" (A + B + WATCH) so you skip past PASS rows by default
- **Sector dropdown** to slice the 500-stock universe by Tech / Fin / Hlth / Cons / Ind / Engy / Util / RE / Mat / Comm
- **Signal filter** — show only setups with a trigger candle today
- **8-EMA distance filter** — at (±0.5 ATR) or near (±1.0 ATR)
- **Position sizer** built into the toolbar — punch in your account size and risk %; every expanded row shows shares to buy, capital to deploy, and dollars at risk
- **Click any row** for full signal breakdown, multi-timeframe stats, and a scaled-out trade plan
- **Showing X of Y** counter tells you how aggressively your filters are cutting

## Customizing the universe

By default, the screener fetches the live S&P 500 from Wikipedia. You can also pass any custom ticker list:

```python
from swing_screener import run, position_size, get_sp500_universe, NASDAQ_100, SP500_STATIC

# Default — fetches live S&P 500
run()

# Use NASDAQ-100 instead
run(tickers=NASDAQ_100)

# Use static fallback (no Wikipedia call)
run(use_wikipedia=False)

# Custom watchlist
run(tickers=["AAPL", "NVDA", "TSLA", "META"])

# Position sizing helper
ps = position_size(account_size=50_000, risk_pct=1.0, entry=180.50, stop=176.20)
print(ps)  # {'shares': 116, 'position_value': 20938, 'risk_dollars': 499, ...}
```

To tighten the liquidity filter (e.g. only mega-liquid names), bump `min_dollar_vol`:

```python
run(min_dollar_vol=100_000_000)  # only stocks with >$100M avg daily dollar volume
```

## Strategy notes

- **The volume-decline filter is the most discriminating signal.** A pullback on heavy volume often means real distribution (institutions selling), not a healthy pause. The screener penalizes setups with vol_decline_ratio > 1.0x and rewards those < 0.7x of the impulse volume.
- **Distance to 8-EMA is measured in ATR units** to normalize across stocks. ±0.5 ATR = "right at the 8-EMA"; ±1.0 = "near"; ±1.5 = "approaching".
- The minimum stop distance is **0.75 ATR** even if the trigger bar's low is closer, to avoid stop-hunt wicks.
- **Not financial advice.** Backtest before risking real capital. Strategy tuning happens in `score_setup()` weights and the tier thresholds.

## Automate it on GitHub (free, runs daily before market open)

The repo ships with a workflow at `.github/workflows/premarket-scan.yml` that scans the S&P 500 at **8:30am ET (1 hour before market open)** Mon–Fri, commits the fresh `results.json` back to your repo, and uploads it as a workflow artifact.

**Setup (5 minutes):**

1. Create a new GitHub repo and push these files:
   ```bash
   git init
   git add .
   git commit -m "initial commit"
   gh repo create swing-screener --public --source=. --push
   # or: create the repo via the web UI, then git push
   ```
2. In your repo on GitHub, go to **Settings → Actions → General**.
   - Under "Workflow permissions", select **Read and write permissions** and save.  
     This lets the scheduled job commit the daily `results.json` back to the repo.
3. (Optional but recommended) Enable GitHub Pages:
   - **Settings → Pages → Source = main branch / root**.
   - After ~30 seconds, visit `https://YOUR-USERNAME.github.io/REPO-NAME/dashboard.html` — the dashboard auto-loads the latest `results.json` from the repo every time you refresh.
4. To smoke-test before tomorrow's pre-market run:
   - Go to **Actions → Pre-market swing scan → Run workflow**. It'll bail out with "outside pre-market window" unless you're actually in the 8am-9am ET window, since the script defends against double-firing across DST. To force a test, comment out the timing check in the workflow YAML.

**How the scheduling works:**

GitHub Actions cron is in UTC and doesn't understand US daylight saving. The workflow registers two crons (13:30 UTC and 14:30 UTC) so that at least one fires at 8:30am ET year-round, then the Python timing check exits early on the one that's wrong for the current DST regime. So you get exactly one run per market morning.

**Caveats to know:**

- **GitHub Actions cron is "best effort"**, not millisecond-precise. Scheduled jobs are routinely delayed 5-20 minutes during peak load; occasionally a job is skipped entirely. For real money, don't assume "8:30 sharp."
- **Yahoo Finance rate limits.** A 500-ticker scan with 4 fetches each (daily + 1H, plus indirect calls) is ~2000 requests. yfinance occasionally returns empty/throttled responses; the screener catches these per-ticker and continues, but on a bad day you'll see a higher than usual count of `NO_DATA` rows. If you start seeing this consistently, drop concurrency from 12 → 6 in `screen()` or switch to a paid feed.
- **Public repos have unlimited Actions minutes; private repos get 2000/month free** — way more than this needs (each run is ~5 min).
- **Holidays** are hardcoded for 2026-2027 in `is_us_market_day()`. Extend the dict when 2028 approaches.
