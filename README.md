# TD385 — orderflow imbalance, tested honestly

**Paper only. No broker code exists. No promised returns.**

TD385 tests the strategy in M. Pecchiari's thesis *Orderflow Imbalance and High Frequency
Trading* (LUISS, 2020) on market data from Yahoo Finance: sign each bar's volume into
buying and selling pressure, then buy when strong buying pressure meets heavy volume. It
recomputes the thesis's statistics, backtests its strategies, two from its literature
review and two of its own after realistic costs, and paper-trades all of them forward on
live bars across 192 large US stocks and ETFs.

The thesis reports +4.12 % over 496 trades with zero costs. That is 0.83 bp a trade,
less than the cost of crossing the spread. "No edge" is therefore the expected and a
perfectly legitimate result. Details: `docs/12_ORDERFLOW_PLAN.md` §2.

## Status

| Stage | Content | Status |
|---|---|---|
| O0 | Foundation: data, orderflow, study, strategies, engine, paper trader | ✅ built 2026-10-11, tested on synthetic bars |
| O1 | ≥ 20 sessions of Yahoo 1 m bars cached | — needs `td385 fetch` on a machine that can reach Yahoo |
| O2 | Thesis Ch. 3 statistics on Yahoo bars | — |
| O3 | Backtest gate | — |
| O4 | ≥ 4 weeks of forward paper trading | — start with `td385 paper` |
| O5 | Real money | **not built; the owner's decision only** |
| Meme | pump.fun memecoin bot, paper only (below) | ✅ built 2026-10-17, tested on a synthetic market; needs `td385 meme run` on a machine that can reach PumpPortal |

## Quickstart

```bash
uv sync                                  # Python 3.11+; creates .venv from uv.lock
git config core.hooksPath scripts/hooks  # secret scan on every commit
uv run pytest
uv run python scripts/security_check.py

uv run td385 fetch                       # 1-minute bars for 192 large US names into data/bars/
uv run td385 study                       # the thesis's Ch. 3 tables, recomputed
uv run td385 backtest                    # every strategy at 1x/2x/4x cost
uv run td385 paper                       # forward paper trading; opens the monitor window
uv run td385 monitor                     # (re)open the dashboard yourself
uv run td385 paper-report                # the dashboard once, saved to data/reports/
uv run td385 paper --replay              # no-network check: same path over cached bars
```

## Watching it

`td385 paper` logs every fill and closed trade as a coloured line (buys green, sells
magenta, winning trades green, losing ones red, errors red) and writes the same lines to
`data/paper/paper.log`. It also opens **a second window**, `td385 monitor`, a dashboard that
redraws every 2 seconds:

- **IS IT SUCCESSFUL?** YES / NO / NOT YET, judged by the pre-declared gate (≥ 200 trades,
  ≥ 4 forward weeks, positive net at 2× cost, HAC t > 2, DSR ≥ 0.95, beats a matched
  benchmark, ≥ 60 % of weeks up), with each check marked ok / x / ?;
- per strategy: trades, wins/losses, win rate with a 95 % interval, net bp per trade at
  1×/2×/4× cost, P&L in USD, profit factor, max drawdown, best/worst trade, average hold;
- open positions marked to the last close, the process heartbeat (LIVE / STOPPED),
  session open or closed, fetch errors and Yahoo's bar delay;
- the last fills and trades.

Closing the monitor does not stop trading; Ctrl-C in the trading window does, and a restart
resumes. On a machine with no desktop, run `td385 monitor` in a second terminal.
`--replay` drives the real trader through cached bars on a simulated clock, into its own
directory under `data/paper-replay/`: it shows the pipeline works, and is not evidence.

Yahoo serves 1-minute bars for about 30 days only. Run `td385 fetch` at least weekly, or
keep `td385 paper` running, so the local cache keeps growing.

## Many stocks at once

By default TD385 watches **192 large US names**: the S&P 100, the Nasdaq-100 and 20 broad
and sector ETFs (`us-large`).

```bash
uv run td385 fetch                    # 1m bars for all 192, 8 at a time (waits out rate limits)
uv run td385 screen                   # keep the liquid, well-covered ones (never by returns)
uv run td385 paper --quiet --telegram # trades what the screen kept (else all 192)
uv run td385 backtest                 # every strategy on every cached symbol, pooled + per symbol
```

`--universe` takes `us-large`, `sp100`, `ndx100`, `etfs`, `megacaps`, `index` (SPY, QQQ, ES=F),
`screened`, `cached`, a text file of tickers (anything that is not a valid ticker is rejected),
or several joined by commas (`--universe sp100,etfs`). Every symbol gets its own book per
strategy and they all trade side by side; one poll logs one summary line (symbols refreshed,
new bars, fills, trades, P&L, open positions, seconds taken).

Yahoo publishes no rate limit, so the paper trader allows itself 120 requests a minute
(`--max-requests`): symbols holding a position are refreshed every minute, the others take
turns, and if Yahoo says "too many requests" the budget halves and grows back. A poll of all
192 symbols × 6 strategies takes about 17 s of CPU here. The screen looks at liquidity only,
never at returns: picking stocks by how well they traded is selection bias.

## Telegram

The bot answers only chats you allow, and it cannot place orders (there is no order code).

1. In Telegram, create a bot with @BotFather and copy its token.
2. `uv run td385 telegram setup` and paste the token (hidden; stored owner-only in
   `~/.config/td385/telegram.json`, `%APPDATA%\td385\telegram.json` on Windows, never in the repo).
3. Message your bot. It replies with your chat id; run `uv run td385 telegram allow <id>`.
4. `uv run td385 paper ... --telegram` (or `uv run td385 telegram run` beside it).

Send `/start`: every reply carries a button dashboard (📊 Status, 🎯 Gate, 🧠 Strategies,
💼 Positions, 🧾 Trades, 🌐 Symbols, 🩺 Health, ⏸️ Pause / ▶️ Resume, 🔕 Mute). A button
redraws the same message in place, so one message works as a live dashboard; 🔄 marks the
view on screen and redraws it. Verdicts are coloured: 🟢 YES, 🔴 NO, 🟡 NOT YET, and each gate
check is ✅ / ❌ / ❔. The same views are commands too: `/status` (is it successful? P&L),
`/gate`, `/strategies`, `/positions`, `/trades [n]`, `/symbols`, `/health`, `/pause` and
`/resume` (stop/allow new entries; exits still run), `/notify on|off`; Telegram's `/` menu
lists them. Closed trades (✅ win / ❌ loss), errors (🚨) and feed stalls (🧊) are pushed as
they happen, within about a second of the trader writing them. Replies are instant: a
background thread keeps the statistics ready and recomputes them only when the trader writes
something new, and the bot reuses one connection to Telegram instead of opening one per message.

## Memecoin bot (pump.fun, paper only)

A second track, at the owner's request: a bot that watches every new pump.fun coin on
Solana, filters out the trash, buys breakouts with a fixed target and stop, and logs every
transaction to Telegram. **It paper-trades only**: fills are simulated on the live market,
and there is no code that can sign or send a transaction. The evidence says to expect no
edge after costs (98.6 % of pump.fun tokens are rugs or pump-and-dumps; fees are about
4.5 % per round trip before price impact). The paper record is there to answer that question
cheaply. Full design: `docs/13_MEMECOIN_PLAN.md`.

```bash
uv run td385 meme setup     # keys, all optional, typed hidden, stored outside the repo
uv run td385 meme sim       # offline demo on a synthetic market (never evidence)
uv run td385 meme run       # paper trading on live data + the Telegram bot; Ctrl-C stops
uv run td385 meme status    # which keys are set (never their values), last status line
uv run td385 meme report    # results per settings version, and a test of each hunch
```

**Data.** New coins come free from PumpPortal's websocket. Trades need either a PumpPortal
API key (0.01 SOL per 10,000 trades, metered against a daily budget you set; default 0.3 SOL)
or a free Solana RPC key (Helius, QuickNode), from which the bot decodes pump.fun's own trade
events. **A PumpPortal API key can also trade the wallet linked to it**: create a separate
PumpPortal wallet holding only a small float for data fees. The bot sends the key only to the
data stream (`tests/test_meme_paper_only.py`).

**Rules built in.** Every trade has a take profit (default +50 %) and a stop (−20 %) that
cannot be switched off, plus a 15-minute time stop. No coin gets more than 10 % of equity
(default 5 %); at most 5 open, 40 % exposure, and a −20 % daily stop. Copycats of an earlier
name or ticker, bundled launches, concentrated holders, devs with dead tokens or a brand-new
wallet, and devs who start selling are rejected or exited. Names that match a trending or
celebrity narrative (Trump, Elon, …, editable) score higher. Entry is a breakout above the
last minute's swing high on a volume surge, with buyers dominating.

**Telegram.** The bot runs in its own process, so a slow Telegram never slows trading. It
pushes every buy, sell (P&L, reason, hold time), failed fill, halt and settings change. Its
button dashboard covers 📊 Status, 💼 Positions (❌ closes one), 🧾 Trades, 👀 Watchlist,
🛡️ Filters (with a preset button: strict / normal / loose), ⚙️ Settings (every setting with
➖ ➕), 🗞️ Narratives, ⏸️ Pause / ▶️ Resume and 🧯 Close all. It also takes commands:
`/set NAME VALUE`, `/narrative add|del WORD`, `/close SYMBOL`. It uses the TD385 bot's token
and allowed chats unless `td385 meme setup` gives it its own; don't run both bots on one token
at the same time.

**Is it working?** `td385 meme report` scores each settings version separately (changing a
setting starts a new trial) with a 95 % interval on the win rate. It also checks the owner's
hunches about volatility, holders and attention against the trades. Before any real money,
the gate in `docs/13` §7 must pass: ≥ 200 trades over ≥ 4 weeks, positive after fees in ≥ 3
of 4 weeks, surviving doubled fees, DSR ≥ 0.95. Paper fills can't see same-block competition,
so paper results are an upper bound.

## Strategies

| Name | Bars | Rule | Source |
|---|---|---|---|
| `thesis_fixed` | 1 m | OF and volume > trailing 85th pct → buy next open, sell that bar's close | thesis §3.4 |
| `thesis_trailing` | 1 m | same entry; hold while OF > 0 | thesis §3.4 |
| `chordia_daily` | 5 m | yesterday's imbalance > 0 → buy the open, sell the close | Chordia & Subrahmanyam (2004), thesis §2.4 |
| `huang_sign` | 5 m | buy when OF > 0, sell when it turns negative | Huang, Su & Liu (2014), thesis §2.4 |
| `thesis_adj` | 1 m | the thesis rule with its known defects fixed: buying *share* > p85 and volume > p85 *for that minute of the day*; hold while buying lasts, at most 10 bars; only stocks that move enough to cover costs | declared 2026-10-14, `docs/12` §3 |
| `knn_of` | 5 m | k-nearest-neighbours vote (k = 25) on 7 orderflow features; buy when ≥ 60 % of similar past bars beat the round trip, hold one bar; refitted each session on the 5 before it | not in the thesis |

## History

The Solana DEX track (D0–D2) and the equity platform before it found no edge or never
reached a gate. Both are archived: `archive/dex/`, `docs/archive/dex/`,
`docs/archive/equities/`. The experiment ledger and its trial count carry over.
