# Trade
# turtle-bot — daily breakout strategy on BTC / ETH / SOL

A production-ready live trading harness built around the **validated breakout
strategy** in `strategy_breakout_final.py`. Paper trading mode by default.

> ⚠️ **DO NOT** switch to live mode until you've completed the full paper-trading
> protocol at the bottom of this file.

---

## Strategy (locked — see `strategy_breakout_final.py` and the validation report)

| Element       | Setting                                                    |
|---------------|------------------------------------------------------------|
| Entry         | close > 20-day prior close max **AND** close > SMA_200     |
| Volume filter | >1.2× 20-bar avg on **BTC/ETH only**; SOL no filter        |
| Stop          | signal-bar low − 1.0 × ATR_14                              |
| Trail         | 15-day low of closes                                       |
| Trail activates | after price moves +2.0 × ATR_14 above entry              |
| Per-trade risk| 1.0% of equity                                             |
| Total risk cap| 1.75% across all open positions (strict — V4 winner)       |
| Max positions | 2 concurrent (BTC > ETH > SOL alphabetical priority)       |
| Universe      | BTCUSDT, ETHUSDT, SOLUSDT — daily candles                  |
| Schedule      | one cycle per day at 00:05 UTC (5 min after candle close)  |

Validated 2021–2026 performance: **PF 4.26**, Sharpe 0.93, MaxDD −6.81%,
expectancy +$148/trade, final equity +45.6% on $10k start. P(profit) 99.6%
under bootstrap MC.

---

## Setup

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Copy the env template, fill in real credentials
cp .env.template .env
# edit .env — at minimum set TELEGRAM_BOT_TOKEN/CHAT_ID if you want alerts;
# Binance keys are only required for LIVE mode

# 3. Initialise the database and verify connectivity
python bot.py --setup

# 4. Verify signal logic matches historical trades
python bot.py --backfill
#    Compare any "FIRED" lines to entries in final_trades.csv

# 5. Run a single daily cycle right now (sanity-check before scheduling)
python bot.py --once

# 6. Start the daily scheduler (runs forever, daily at 00:05 UTC)
python bot.py
```

---

## File map

```
turtle-bot/
├── strategy_breakout_final.py   ← validated reference backtest (do not modify)
├── config.py                    ← locked strategy parameters + paths
├── data.py                      ← Binance fetch w/ cache + retry + ccxt order helpers
├── indicators.py                ← ATR, SMA, EMA, etc.
├── signal_engine.py             ← signal evaluation, trail logic
│                                  (named to avoid shadowing stdlib `signal`)
├── risk.py                      ← strict-cap sizing + circuit breakers
├── state.py                     ← SQLite persistence (trades, account, signals)
├── executor.py                  ← Executor(mode) — paper + live order placement
├── alerts.py                    ← Telegram notifications
├── bot.py                       ← main loop (APScheduler daily cron)
├── monitor.py                   ← read-only status CLI
├── requirements.txt
├── .env.template
└── README.md
```

`state.db` (SQLite), `bot.log`, and `.live_cache/*.csv` are created on first run.

---

## Daily flow (what the bot actually does at 00:05 UTC)

1. Fetch latest daily candles (uses CSV cache; refreshes last 5 bars from API).
2. Compute mark-to-market equity from open positions, update peak.
3. **Circuit breakers** — at ≥12% drawdown, halt; at ≥8%, alert.
4. **Manage open positions**: check hard-stop (intra-bar low) → check trail
   activation → ratchet trail → check close-based trail exit.
5. **Evaluate signals** on the just-closed bar for all pairs that have no
   open position.
6. **Place entries** subject to MAX_POSITIONS and TOTAL_RISK_CAP.
   Second position is sized smaller (per V4 strict cap).
7. Record account snapshot. Send daily summary via Telegram.

Every signal evaluation — fired or not — is logged to the `signals` table so
you can monitor strategy health without waiting for trades.

---

## Monitor CLI

```bash
python monitor.py --status      # account equity, open positions, drawdown
python monitor.py --trades      # last 20 trades + running PF/WR/expectancy
python monitor.py --signals     # last 30 signal evaluations + reasons
python monitor.py --health      # data freshness, last run, connectivity
python monitor.py --switch-live # PAPER → LIVE (requires CONFIRM LIVE TRADING)
python monitor.py --switch-paper# LIVE → PAPER (no confirmation; safer way)
```

---

## Paper trading protocol — before you ever touch real money

### Week 1 — Verification
- Run `python bot.py --backfill`. Cross-check every "FIRED" line against
  `final_trades.csv`. **Any mismatch is a bug to fix before paper trading.**
- Run `python monitor.py --health` and confirm Telegram alerts arrive.

### Weeks 2–10 — Paper trading observation
- Let the bot run daily; `monitor.py --signals` every few days.
- Watch for: signals firing at expected cadence (~6/year), daily summaries
  arriving at 00:05 UTC, no crashes or DB errors in `bot.log`, trail-stop
  updates appearing for active trades.

### 10-trade review
Once 10 paper trades have closed:
- Compute PF, expectancy, WR from `monitor.py --trades`.
- **Acceptable range**: PF > 2.0 · Exp > $50 · no single loss > 2× expected stop.
  (Half of backtest values, allowing for small-sample variance.)
- Out of range → halt, audit signals, find the divergence.

### Go-live decision (after 20+ paper trades)
All of:
- PF > 2.5 in paper · Exp > $80 · MaxDD < 10% · 60+ days no crashes
- You understand every trade the bot took

Then:
```bash
python monitor.py --switch-live
```
- Start with **20%** of capital. Keep 80% in reserve.
- Scale up only after 10 live trades match paper expectations.

---

## Safety rules baked in

- **12% drawdown circuit breaker.** All trading halts when equity falls 12%
  below peak. Open positions are NOT auto-liquidated — they remain so manual
  review can choose to keep, scale out, or close.
- **Strict total-risk cap (1.75%).** A second simultaneous position is sized
  smaller so total at-risk capital can never exceed 1.75% — addresses the
  85.7% correlation finding from `strategy_breakout_v3.py`.
- **Hard stop tracked in DB**, not just on the exchange. Even if an exchange
  stop order fails to place in LIVE mode, the daily loop will close out the
  trade on the next cycle if hard_stop was breached intra-bar.
- **Mode persistence**. Switching to LIVE writes a row in the `account` table
  with the new mode. The bot reads the latest mode at the start of every
  daily cycle.

---

## Validation chain (proof of edge)

The whole strategy was built by an explicit chain of evidence — every parameter
justified by a specific audit or sweep, no parameter selected from intuition:

| Stage                          | Output                                |
|--------------------------------|---------------------------------------|
| hypothesis_audit.py            | Mean reversion dead; breakout has edge|
| breakout_audit.py              | Bull regime + vol filter both help    |
| strategy_breakout_v1.py        | Real edge confirmed (PF 3.5–4.7)      |
| strategy_breakout_v2.py        | 1.2× volume threshold beats 1.5×      |
| strategy_breakout_v3.py        | 2.0×ATR activation; flagged correlation|
| strategy_breakout_v4.py        | Total risk cap 1.75% wins Sharpe/MDD  |
| strategy_breakout_final.py     | 7/8 must-pass; substantive PASS       |

If live results diverge from backtest, walk back through these files to find
which assumption broke.
