# Momentum Stocks Screener

A two-part bot that scans **~2,000 US common stocks** (ETFs and ADRs excluded) for the **8-EMA Pullback** swing setup, filtered down to **market leaders** outperforming the S&P 500: stocks in an established uptrend that have just pulled back to the 8-day EMA on declining volume, with a bullish reversal candle signaling continuation — *and* showing positive relative strength.

The leadership filters and quality checks are distilled from **16 trader methodologies** (Minervini, Qullamaggie, O'Neil, Stan Weinstein, Mike Webster, Patrick Walker, Richard Moglen, Stockbee, Darvas, and others) that all share the same common DNA: trade leaders, not laggards; risk tight, profit wide.

Live dashboard: **https://minteck1975.github.io/swing-bot/**

## The strategy in 4 steps

### 1. Trend (the prerequisite)
- Higher highs and higher lows in the recent structure
- Price > 20-EMA > 50-EMA (**stacked**)
- 50-EMA sloping up
- Weekly chart confirms — price above the weekly 20-EMA, weekly slope up

### 2. Market leadership (the filter)
- **Relative Strength vs SPY** — outperforming over 3M, 6M, and ideally 12M (Minervini, O'Neil, Qullamaggie all insist on this)
- **Within 25% of 52-week high** — no falling knives
- **Healthy ADR** — Average Daily Range ≥ 2.5% over last 20 bars; a slow-moving stock won't deliver swing-trade returns

### 3. Pullback (the entry zone)
- Price has pulled back to within ~1 ATR of the 8-EMA
- **Volume declining** during the pullback vs the prior impulse move (the key tell — institutions taking a breath, not exiting)
- Pullback is shallow-to-moderate (didn't break the 20-EMA badly)
- Sweet spot: 2-7 bars since the impulse peak

### 4. Trigger + Quality checks (when to actually click buy)
A bullish reversal candle on the most recent bar (or yesterday):
- **Bullish engulfing** (strongest)
- **Hammer / pin bar**
- **Inside-bar breakout**

Plus two quality filters that reject setups that look good on shape but fail on substance:
- **Close-in-Range** (Stockbee) — the trigger bar must close in the upper part of its own daily range. A "perfect" engulfing that closes weak in its range = sellers absorbed the demand; not a real reversal.
- **ADR risk-distance** (Qullamaggie V2) — the entry-to-stop distance must be ≤ 2.0× the stock's daily ADR in dollars. If you have to risk more than 2 daily ranges to give the trade room, the stock is already extended from its base.

### Risk management
- **Stop** = min(trigger bar low, recent swing low) − 0.25 ATR, with a 0.75 ATR floor (so you're never stopped on a wick)
- **Scale out** in quarters:
  - ¼ at T1 = prior swing high
  - ¼ at T2 = 2R from entry
  - Runner trails behind the 8-EMA

## Tiers

| Tier | Score | Means |
|---|---|---|
| **A_SETUP** | ≥ 75 + trend + pullback + signal + risk ≤ 2.0× ADR | Full alignment, tight risk, ready to trade |
| **B_SETUP** | 60–74 + trend + pullback | Components present, may be slightly extended or signal pending |
| **WATCH** | 45–59 + trend | Trend OK but needs more pullback or no trigger yet |
| **PASS** | < 45 | Doesn't meet criteria |
| **ILLIQUID** | — | Filtered out for < $20M average daily dollar volume |

The dashboard **only displays A_SETUPs** — all other tiers are filtered out in the UI. The scoring tiers exist in `results.json` so you can inspect them, but the focused workflow is: open dashboard, see today's A_SETUPs, dig in.

## Files

| File | What it does |
|---|---|
| `swing_screener.py` | Scans the universe, writes `results.json` |
| `dashboard.html` | Self-contained dashboard with embedded demo data |
| `results.json` | Output (auto-loaded by the dashboard) |
| `icon.svg` | Elliott Wave app icon |
| `.github/workflows/postmarket-scan.yml` | Runs the scan post-market on weekdays and deploys to GitHub Pages |

## Run it locally

```bash
pip install -r requirements.txt

# Scan ~2,000 US common stocks (~10-12 min)
python swing_screener.py
```

Then open `dashboard.html` (double-click, or serve via `python -m http.server` and visit `localhost:8000/dashboard.html`).

The screener fetches the current US common-stocks universe from NASDAQ Trader's daily public listings, so the universe stays in sync as stocks come and go. If unreachable, it falls back to a bundled snapshot of S&P 500.

## Dashboard features

### Stat strip (6 boxes, all scoped to A_SETUPs only)
- **A Setups** — count of A-tier setups today
- **Strong Leaders** — A_SETUPs with RS3M ≥ 20 (the highest-conviction subset)
- **Top Stock** — single highest-scoring ticker, e.g. "GE · 96"
- **Trigger Today** — A_SETUPs with a fresh trigger candle today (not yesterday)
- **At 8-EMA** — A_SETUPs within 0.5 ATR of the 8-EMA (the perfect entry zone)
- **Avg Score** — average score across all A_SETUPs

### Filter buttons
- **Signal** — Any / Trigger Today / Has Signal
- **EMA distance** — Any / At 8-EMA (±0.5 ATR) / Near 8-EMA (±1.0 ATR)
- **Market Leaders** — Any RS / Leaders (RS3M ≥ 10) / Strong (RS3M ≥ 20)
- **Sector dropdown** — slice by Tech / Fin / Hlth / Cons / Ind / Engy / Util / RE / Mat / Comm
- **Search box** — ticker, name, sector

### Per-row expansion
Click any row to expand into a 2/3-width TradingView chart + 1/3-width side panel showing:
- **Trade plan** — Entry, Stop, T1 (swing high), T2 (2R), R-multiples
- **Signal breakdown** — every reason that scored or de-scored
- **Leadership vs SPY** — RS 3M/6M/12M, off-52w-high, ADR
- **Setup quality** — close-in-range %, risk vs daily ADR

## Customizing the universe

```python
from swing_screener import run, get_universe, NASDAQ_100, SP500_STATIC

# Default — ~2,000 US common stocks (ETFs/ADRs excluded)
run()

# Use S&P 500 only (smaller, faster scan)
run(universe_mode="sp500")

# Use NASDAQ-100
run(universe_mode="nasdaq100")

# Custom watchlist
run(tickers=["AAPL", "NVDA", "TSLA", "META"])

# Tighten liquidity filter — only mega-caps
run(min_dollar_vol=100_000_000)
```

## Strategy notes & tunable knobs

The two main dials are in `score_setup()`:

- **`risk_vs_adr` cutoff for A_SETUP** — currently **2.0×** daily ADR. Lower (1.5–1.8) for stricter A's; higher (2.2–2.5) for more.
- **Score cutoff for A_SETUP** — currently **75/100**. Bump to 80 for stricter.
- **Close-in-Range good-close bar** — currently **60%**. Lower this if too few setups pass; raise it if you want only the strongest closes.

Other key thresholds:
- **Distance to 8-EMA is measured in ATR units** to normalize across stocks. ±0.5 ATR = "right at the 8-EMA"; ±1.0 = "near"; ±1.5 = "approaching".
- **Minimum stop distance is 0.75 ATR** even if the trigger bar's low is closer, to avoid stop-hunt wicks.
- **Liquidity floor** — $20M average daily dollar volume.

**Not financial advice.** Backtest before risking real capital. Tuning happens in `score_setup()` weights and the tier thresholds.

## Automated daily scan via GitHub Actions

The repo ships with a workflow at `.github/workflows/postmarket-scan.yml` that:

1. **Scans the universe at 5:00pm ET** (1 hour after the 4pm close) Mon–Fri
2. **Commits the fresh `results.json`** back to your repo so history is preserved
3. **Auto-deploys the dashboard to GitHub Pages** so visiting `https://YOUR-USERNAME.github.io/REPO-NAME/` shows the latest scan every evening

### One-time setup

1. Create a public GitHub repo and push these files:
   ```bash
   git init && git add . && git commit -m "initial commit"
   gh repo create swing-bot --public --source=. --push
   ```
2. **Settings → Actions → General → Workflow permissions** → select **Read and write permissions**
3. **Settings → Pages → Build and deployment → Source** → select **GitHub Actions** (NOT "Deploy from a branch")
4. **Actions → Post-market swing scan → Run workflow** to trigger the first deploy
5. After ~1 minute, visit `https://YOUR-USERNAME.github.io/REPO-NAME/`

### Scheduling notes

GitHub Actions cron runs in UTC and doesn't understand US daylight saving. The workflow registers two crons (21:00 UTC and 22:00 UTC) so that one always fires at 5:00pm ET year-round, then the Python timing check exits early on the cron that's wrong for the current DST regime. You get exactly one scheduled run per market afternoon.

### Why 5pm ET (post-market) instead of pre-market

- Daily candle is fully formed — yesterday's bullish engulfing is real, not a half-bar that could still reverse
- Yahoo Finance has had hours to ingest closing prices, so the data is reliable
- You see tomorrow's setups the night before, with time to plan position sizing and write orders into your broker

### Caveats

- **GitHub Actions cron is "best effort"**, not millisecond-precise. Scheduled jobs are routinely delayed 5-20 minutes during peak load; occasionally skipped entirely.
- **Yahoo Finance rate limits.** A 2,000-ticker scan with multi-timeframe fetches is ~8,000 requests. yfinance occasionally returns empty/throttled responses; the screener catches these per-ticker and continues. On a bad day you'll see more `NO_DATA` rows.
- **Public repos have unlimited Actions minutes**; private repos get 2,000/month free — way more than this needs.
- **Free-tier GitHub Pages requires a public repo.** Your `results.json` becomes publicly readable, but it's just stock screener output.
- **Holidays** are hardcoded for 2026-2027 in `is_us_market_day()`. Extend the dict when 2028 approaches.

## Known limitations

- **Chart EMA lengths** — the embedded TradingView chart uses TradingView's free embed widget, which has API limitations: the EMA20 and EMA50 lines render in the same color, and the actual length parameter behavior is undocumented (may display at default length 9 in some browsers). For guaranteed EMA20 + EMA50 in distinct colors, click "Open in TradingView" — the link pre-loads the correct studies in the full TradingView page.
- **Universe completeness** — NASDAQ Trader's listings cover NASDAQ + NYSE + AMEX common stocks, which is the practical universe for swing trading. OTC and pink-sheet stocks are excluded by design.

## Credits / methodology sources

Strategy logic synthesizes ideas from:
- **Mark Minervini** — VCP (volatility contraction pattern) + Trend Template
- **Kristjan Kullamägi (Qullamaggie)** — Breakout V2: ADR-based risk distance, swing-low stops, MA-based trailing
- **William O'Neil / CAN SLIM** — Relative Strength rankings, leading stocks from leading industries
- **Stan Weinstein** — Stage 2 advance, 30-week MA
- **Stockbee (Pradeep Bonde)** — Bullish Combo / LTB / Swing screens, Close-in-Range quality filter
- **Nicolas Darvas** — box theory, top quintile of 52-week range
- **Mike Webster, Patrick Walker, Richard Moglen** — multi-timeframe RS confirmation
- **Other public materials** — Investors Business Daily, ChartLog, various Stocktwits and YouTube traders documenting their continuation-pattern playbooks
