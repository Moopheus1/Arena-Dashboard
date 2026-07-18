# Arena Dashboard — Setup

Two things to do: (1) get this repo live on GitHub Pages, (2) set up the refresh in Cowork.

## 1. Publish the repo

```bash
# from this folder
git init
git add .
git commit -m "Arena dashboard: initial snapshot"
gh repo create Arena-Dashboard --public --source=. --push
# or manually: create a new empty repo on github.com, then:
# git remote add origin https://github.com/<you>/Arena-Dashboard.git
# git branch -M main
# git push -u origin main
```

Then: repo Settings → Pages → Source: `main` branch, `/ (root)` folder → Save.
Your dashboard will be live at `https://<you>.github.io/Arena-Dashboard/` within a minute or two.

## 2. Set up the refresh (Cowork scheduled tasks)

Open Claude Desktop → Cowork tab → Scheduled → New task.

US market hours (9:30am–4:00pm ET) span midnight in Singapore time, so a single
hourly cron can't cover the whole session — it needs two tasks:

- **Block 1**: `0 22,23 * * 1-5` — 22:00 and 23:00 SGT, Mon–Fri
- **Block 2**: `0 0,1,2,3 * * 2-6` — 00:00–03:00 SGT, Tue–Sat (covers the tail end
  of the same US trading session, which lands on the next SGT calendar day)

Each task uses the same prompt (see below), pointed at the same repo. Together they
give roughly hourly coverage, clipping the first ~30 min after open and the last
~hour before close.

### Task prompt to paste (use for both tasks):

```
You are refreshing the data.json file in the GitHub repo <you>/Arena-Dashboard with
the latest Rallies AI Arena leaderboard data. This is an unattended run — follow every
step exactly, do not ask for confirmation, do not touch any file except data.json.

STEP 1 — Fetch live data.
Call the MCP tool that returns Rallies AI Arena model performance with
recent_decisions_per_model=5, limit=20, sort_by="all_time". This returns all Arena
models with their 5 most recent decisions each.

STEP 2 — Transform to the exact target schema.
Top-level: as_of (ISO8601 with trailing Z), source ("rallies.ai/arena"), models (array).
Each model: id, name, provider, llm_model (strip any "openrouter/<provider>/" prefix —
keep only the text after the last "/"), url, account_value, initial_capital,
total_return_pct, week_return_pct, month_return_pct, daily_return_pct, sharpe_ratio,
win_rate_pct, max_drawdown_pct, number_of_trades, cash_balance, last_trade_date
(ISO8601 Z), open_positions (array of symbol/quantity/entry_price/current_price/
market_value/unrealized_pnl/entry_date, all ISO8601 Z), recent_actions (array of the
5 most recent decisions: action/symbol/quantity/date/thesis, most recent first), and
news (array built by taking the top 2 holdings by market_value, fetching up to 3 recent
news articles per ticker, and including symbol/title/url/published_at for each,
sorted by published_at descending).

STEP 3 — Update the GitHub repo: clone (or fresh temp clone), overwrite ONLY data.json,
commit as "Arena refresh <timestamp>", push to main. Never touch index.html or
README.md. Never print or log the auth token.
```

## Notes

- The positions table shows Invested (quantity × entry price — what was actually
  put into that position) and Alloc (current market value as a % of total account value).
  Both are computed client-side from existing data.json fields, no schema change needed.
- Every ticker symbol (positions table, recent-actions feed, news headlines) links
  out to `rallies.ai/research/<ticker>` for deeper research. Links stop click
  propagation so they don't also toggle the row open/closed.
- Positions table has a TOTAL row (bold, top border) summing Invested, Alloc %,
  and P&L across a model's open positions, plus a reconciliation line below
  the table: current market value of positions + cash, checked against the
  account_value the API reports (flags a mismatch if they differ by >$1).
  Total invested (cost basis) is shown for reference but will NOT equal account
  value or the reconciled figure — cost basis is entry-price based, the other
  two are current-price based. Don't expect those two numbers to tie out.
- Light/dark theme toggle (top right) is a pure front-end preference, stored in
  the visitor's own browser localStorage — no server or data.json involvement.
- The dashboard's client-side JS re-fetches `data.json` every 5 minutes, so once the
  scheduled task pushes a new commit, the open page updates without a manual reload.
  Expanded model rows stay open across that re-fetch (tracked by model id), so you can
  leave several rows open and scroll through them without losing your place.
- Each expanded row shows its 5 most recent actions (buy/sell/hold) with thesis text,
  and up to 6 recent headlines for its top 2 holdings by market value, as a rough proxy
  for the news context around that model's positions. This is NOT a record of what the
  model actually read — Arena doesn't expose that — it's just recent coverage of the
  same tickers.
- Ticker-matched news can be noisy (e.g. small-caps sharing a ticker root with unrelated
  companies). The automation doesn't semantically filter results, so occasional
  irrelevant headlines are a known limitation of the underlying news API, not a bug.
- If Rallies changes their MCP tool's response shape, the task will likely error rather
  than silently write bad data — check the Scheduled tab run history if the dashboard
  looks stale.
