# TILT — System Blueprint & Handoff Doc

*Living document. Paste this into any new session to bring the assistant up to speed instantly. Update it as the system changes.*

---

## What TILT is
A read-only risk & discipline tool for on-chain perp traders on Hyperliquid. It watches a trader's public on-chain activity, detects self-destructive behaviour, alerts to Slack, and shows a live dashboard + trading journal. No keys, no custody, no order placement — just public data in, insight + alerts out.

Everything runs **inside one Supabase project** (Postgres + pg_cron + http extension + an edge function). No separate servers.

- Supabase project ref: `mdyeujlmiqiztzfnlogc` (region ap-south-1)
- Hyperliquid mainnet API: `https://api.hyperliquid.xyz/info`
- Hyperliquid testnet API: `https://api.hyperliquid-testnet.xyz/info`

## Data flow (the whole thing in one line)
`pg_cron (every ~4s) -> tilt_poll_fast() -> pull fills+positions from Hyperliquid -> store -> run detectors -> send new alerts to Slack`. The dashboard reads the same tables via an edge-function JSON API.

---

## Tables
- **tilt_traders** — who we watch + per-trader config. Key columns: `wallet` (internal key, PK), `hl_address` (real on-chain address used for API calls), `api_base` (mainnet/testnet URL), `network`, `capital_usd`, and thresholds: `risk_pct_per_trade` (0.02), `daily_loss_pct` (0.03), `max_trades_day` (50), `revenge_window_min` (30), `fallback_vol` (0.06), `alert_cooldown_min` (30).
- **tilt_fills** — every trade (fill). PK `(wallet, tid)`.
- **tilt_account_snapshots** — point-in-time equity + open positions.
- **tilt_alerts** — alerts fired. Deduped by `alert_key`; `slack_sent` tracks delivery.
- **tilt_token_vol** — daily volatility per coin (from 30 daily candles).
- **tilt_config** — key/value store (holds the Slack webhook).

## Views
- **tilt_fill_risk** — each entry with volatility-adjusted $ risk.
- **tilt_journal_overall / _by_coin / _by_hour / _by_dow** — journal analytics.
- **tilt_journal_tilt_windows** — calm vs. within-30-min-of-a-loss performance (the headline insight).
- **tilt_daily_pnl** — daily realized PnL.
- **tilt_desk** — one row per trader for the desk view.
- **tilt_trades** — realized trades (fills with closed_pnl <> 0).

## Functions
- **tilt_poll(wallet, snapshot=true)** — one full check: ingest fills, snapshot, detect, send alerts.
- **tilt_poll_fast()** — loops ~50s polling every 4s (near-real-time). Scheduled every minute.
- **tilt_detect(wallet)** — event detectors: revenge, daily_bleed, overtrading.
- **tilt_detect_positions(wallet, clearinghouse_json)** — position-based over-risk (one alert per open position, not per fill).
- **tilt_refresh_vol()** — refresh per-coin volatility (nightly).
- **tilt_send_slack(msg)** — post to the Slack webhook.
- **tilt_health()** — pings Slack if the poller fails/stalls.

## Detectors (all tunable in tilt_traders)
1. **Over-risk** — `position size x coin daily volatility`; alert if > `risk_pct_per_trade` of equity ('stop' if > 5%). Dynamic per token (a thin alt trips smaller than BTC).
2. **Revenge** — a sized-up entry (> 1.5x your normal $ risk) within `revenge_window_min` of a realized loss.
3. **Daily bleed** — realized day PnL crosses `-daily_loss_pct` of equity -> stop for the day.
4. **Overtrading** — more than `max_trades_day` trades in a day.

Spam control: a per-(detector, coin) **cooldown** (`alert_cooldown_min`) collapses bursts to one ping.

## Scheduled jobs (pg_cron)
- `tilt_poll` -> `tilt_poll_fast()` every minute (internally ~4s loop).
- `tilt_vol` -> `tilt_refresh_vol()` nightly.
- `tilt_health` -> `tilt_health()` every 15 min.

## Dashboard
- Edge function `dashboard` = a **JSON data API** (Supabase sandboxes HTML, so it can't serve a page directly).
  - URL: `https://mdyeujlmiqiztzfnlogc.supabase.co/functions/v1/dashboard?key=grull-tilt-7f3a9c2e&wallet=<wallet>`
  - Access key is `grull-tilt-7f3a9c2e` (in the edge function source; change it to rotate).
- Viewer = `TILT-dashboard.html` — open locally or host on a static host; reads from the API, has a mainnet/testnet wallet switcher, auto-refreshes every 5s.

## Wallets currently registered
- `0x1bec5E965dcE874D895d40B41d6f5BDF910689A6` — mainnet (real history, ~$22k).
- `0x1B00BEf27c0b0aD74E6E7479F9d741d7b1905128` — testnet (faucet ~$1k).

---

## How to do common things

**Add a trader (desk colleague):**
```sql
insert into tilt_traders(wallet, label, hl_address, capital_usd)
values ('0xTHEIR_ADDRESS', 'Raj', '0xTHEIR_ADDRESS', 50000);
```
They're auto-polled and appear on the dashboard next cycle.

**Change a threshold (e.g. tighter daily limit):**
```sql
update tilt_traders set daily_loss_pct = 0.02 where wallet = '0x...';
```

**Test a change safely:** make it, then watch the testnet wallet — trade on app.hyperliquid-testnet.xyz and confirm alerts behave before it matters on mainnet.

**Rotate the dashboard key:** edit `ACCESS_KEY` in the `dashboard` edge function and redeploy.

## Known limitations / backlog
- Polling is ~4s, not tick-level. True sub-second needs an always-on WebSocket worker on a separate host.
- Slack alerts all go to one webhook/channel. Per-trader routing is a future add.
- Over-risk uses 30-day daily volatility; could add intraday/ATR.
- Dashboard auth is a URL key, not a login.
- Journal "by hour" is in UTC.
