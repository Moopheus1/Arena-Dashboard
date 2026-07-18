# Arena Dashboard — Setup

Two things to do: (1) get this repo live on GitHub Pages, (2) set up the hourly refresh in Cowork.

## 1. Publish the repo

```bash
# from this folder
git init
git add .
git commit -m "Arena dashboard: initial snapshot"
gh repo create arena-dashboard --public --source=. --push
# or manually: create a new empty repo on github.com, then:
# git remote add origin https://github.com/<you>/arena-dashboard.git
# git branch -M main
# git push -u origin main
```

Then: repo Settings → Pages → Source: `main` branch, `/ (root)` folder → Save.
Your dashboard will be live at `https://<you>.github.io/arena-dashboard/` within a minute or two.

## 2. Set up the hourly refresh (Cowork scheduled task)

Open Claude Desktop → Cowork tab → Scheduled → New task → paste the prompt below → set frequency to **Hourly**, restricted to US market hours (9:30pm–4:00am SGT).

> Note: Cowork's built-in cadence options don't include a "market hours only" filter, so the task will fire every hour around the clock unless you use `/schedule` in chat with a specific cron-style time window, or just let it run hourly and accept a few off-hours updates (the data won't change materially outside market hours anyway, since Arena only trades during the session).

### Task prompt to paste:

```
Call Rallies:get_arena_model_performance (recent_decisions_per_model: 1) to get the
latest Arena leaderboard data. Transform the result into the exact JSON schema used
in data.json in the arena-dashboard GitHub repo (https://github.com/<you>/arena-dashboard):
top-level as_of (ISO8601), source, and a models array where each model has: id, name,
provider, llm_model, url, account_value, initial_capital, total_return_pct,
week_return_pct, month_return_pct, daily_return_pct, sharpe_ratio, win_rate_pct,
max_drawdown_pct, number_of_trades, cash_balance, last_trade_date, open_positions
(array of symbol/quantity/entry_price/current_price/market_value/unrealized_pnl/entry_date),
and last_action (action/symbol/quantity/date/thesis, using the single most recent
decision entry). Then update data.json in the repo with this new data (clone if needed,
overwrite data.json only, commit with message "Arena refresh <timestamp>", push to main).
Do not modify index.html or README.md.
```

## Notes

- The dashboard's client-side JS re-fetches `data.json` every 5 minutes, so once the
  scheduled task pushes a new commit, the open page updates without a manual reload.
- If Rallies changes their MCP tool's response shape, the task will likely error rather
  than silently write bad data — check the Scheduled tab run history if the dashboard
  looks stale.
- True 10-minute refresh isn't possible with Cowork's scheduler (hourly floor). If you
  later get Claude Code Desktop set up with a local `/loop` on your own machine, that
  supports 1-minute intervals — different mechanism, worth revisiting if 10-minute
  cadence really matters to you.
