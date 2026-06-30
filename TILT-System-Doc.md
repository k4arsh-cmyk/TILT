# TILT — System Blueprint & Handoff Doc

*Living document. Paste into any new session to bring the assistant up to speed instantly. Update as the system changes.*

Last updated: 2026-06-30 (migrated to Grull's own Supabase; multi-trader desk + per-trader Slack + settings UI).

## What TILT is
Read-only risk & discipline tool for on-chain perp traders on Hyperliquid. Watches each trader's public on-chain activity in near-real-time, alerts to Slack when they're about to blow up, and shows a desk-wide dashboard + per-trader journal. No keys, no custody, no order placement — public data in, insight + alerts out. Runs entirely inside one Supabase project (Postgres + pg_cron + http extension + an edge function). No separate servers.

## Accounts & endpoints
- **Supabase project (owner: arsh@grull.space):** ref `jbrwfbzgewrghsrughby`, region ap-northeast-1 (Tokyo). Dashboard: https://supabase.com/dashboard/project/jbrwfbzgewrghsrughby
- Hyperliquid mainnet API: `https://api.hyperliquid.xyz/info` · testnet: `https://api.hyperliquid-testnet.xyz/info`
- **Web app (GitHub Pages):** repo `k4arsh-cmyk/TILT`, live at https://k4arsh-cmyk.github.io/TILT/ (file: `index.html`)
- **Dashboard data API (edge function):** https://jbrwfbzgewrghsrughby.supabase.co/functions/v1/dashboard — access key `grull-tilt-7f3a9c2e`
- Slack: shared incoming webhook stored in `tilt_config`; optional per-trader webhook on `tilt_traders.slack_webhook`.
- (Old project under tarun@grull.space was deleted after migration.)

## Data flow
`pg_cron (every minute) -> tilt_poll_fast() loops ~50s polling every 4s -> for each trader: pull fills + positions from Hyperliquid -> store -> detect -> send new alerts to that trader's Slack`. The web app reads via the edge-function JSON API.

## Tables
- **tilt_traders** — who we watch + per-trader config. Cols: `wallet` (PK / internal key), `label`, `hl_address` (on-chain address used for API), `api_base`, `network` (mainnet/testnet), `capital_usd` (0 = auto-calibrate from live equity on first poll), `slack_webhook` (per-trader channel; null = shared), and thresholds: `risk_pct_per_trade` (0.02), `daily_loss_pct` (0.03), `max_trades_day` (50), `revenge_window_min` (30), `fallback_vol` (0.06), `alert_cooldown_min` (30).
- **tilt_fills** — every trade. PK `(wallet, tid)`.
- **tilt_account_snapshots** — equity + open positions over time.
- **tilt_alerts** — alerts; deduped by `alert_key`; `slack_sent` tracks delivery.
- **tilt_token_vol** — daily volatility per coin (30 daily candles).
- **tilt_config** — key/value (shared Slack webhook).

## Views
tilt_trades, tilt_fill_risk, tilt_journal_overall / _by_coin / _by_hour / _by_dow, tilt_journal_tilt_windows (calm vs after-loss), tilt_daily_pnl, tilt_desk (one row per trader for the desk board).

## Functions
- **tilt_poll(wallet, snapshot=true)** — full cycle for one trader: ingest fills, snapshot, auto-calibrate capital if 0, detect, send new alerts (rich Slack blocks) to the trader's webhook.
- **tilt_poll_fast()** — loops ~50s polling every 4s across all traders (near-real-time). Scheduled every minute.
- **tilt_detect(wallet)** — revenge (per-order, sized above risk budget, within window of a loss), daily_bleed, overtrading.
- **tilt_detect_positions(wallet, clearinghouse_json)** — over-risk on the live position (one alert per coin per day per severity).
- **tilt_refresh_vol()** — refresh per-coin volatility (nightly).
- **tilt_send_slack(msg)** / **tilt_send_slack_blocks(payload, hook)** — Slack senders.
- **tilt_health()** — pings Slack if the poller fails/stalls.

## Detectors (all tunable per trader)
1. **Over-risk** — `position size × coin daily volatility`; alert > `risk_pct_per_trade` of equity ('stop' if > 5%). Dynamic per token.
2. **Revenge** — a sized-up entry (above the risk budget) within `revenge_window_min` of a realized loss; evaluated per order.
3. **Daily bleed** — realized day PnL crosses `-daily_loss_pct` of equity → stop for the day.
4. **Overtrading** — more than `max_trades_day` trades in a day.
Spam control: per-(detector, coin, severity) cooldown collapses bursts; warn→stop escalation re-alerts.

## Scheduled jobs (pg_cron)
- `tilt_poll` → `tilt_poll_fast()` every minute (internal ~4s loop)
- `tilt_vol` → `tilt_refresh_vol()` nightly
- `tilt_health` → `tilt_health()` every 15 min

## Edge function `dashboard` (JSON API, verify_jwt off, key-gated)
- `GET ?key=…&view=desk` → all traders (tilt_desk)
- `GET ?key=…&wallet=X` → that trader's detail + `settings`
- `POST {action:"add_trader", wallet, label, network, capital, slack_webhook}`
- `POST {action:"update_trader", wallet, …any of: label, capital, slack_webhook, risk_pct, daily_loss_pct, max_trades_day, revenge_window_min, alert_cooldown_min}` (percent fields are entered as % and stored as fractions)
- `POST {action:"remove_trader", wallet}`

## Web app (index.html on GitHub Pages)
- **Desk view:** summary strip + risk-accented trader cards (auto-sorted by danger), "+ Add trader".
- **Trader detail:** KPIs, calm-vs-revenge insight, daily PnL chart, per-coin table, recent alerts.
- **⚙ Settings:** edit name, capital, per-trader Slack webhook, and all thresholds live.
- Access key entered once, stored in browser localStorage. Auto-refresh ~10s.

## Wallets currently registered
- `0x1bec5E965dcE874D895d40B41d6f5BDF910689A6` — me (mainnet, ~$22k, full history).
- `0x1B00BEf27c0b0aD74E6E7479F9d741d7b1905128` — me (testnet, faucet).

## Common tasks
- **Add / edit / remove a trader:** all from the web app (Add trader button / ⚙ Settings / Remove). New traders auto-onboard within a minute.
- **Per-trader private channel:** create the Slack channel → add an incoming webhook → paste it in Add Trader or Settings.
- **Change the shared webhook:** `update tilt_config set value='…' where key='slack_webhook';`
- **Backup:** `tilt_schema.sql` recreates the engine in a fresh project.

## Known limitations / backlog
- Polling ~4s, not tick-level (true sub-second needs an always-on WebSocket worker).
- Dashboard auth is a shared URL key, not per-user login.
- Journal "by hour" is UTC.
- Engine changes are applied live in Supabase (migration ledger + daily backups exist); commit `tilt_schema.sql` to GitHub at milestones for full version history.
