# orby-engine (Phase 1)

Deriv bot engine skeleton with the Even/Odd strategy. Zero dependencies (Node 22+).

## Run (demo account first!)
1. Register a **PAT-type** app at developers.deriv.com and create a PAT with the `trade` scope.
   Legacy app IDs do not work with the new API.
2. `cp .env.example .env` and fill in `DERIV_APP_ID`, `DERIV_PAT`, `DERIV_ACCOUNT_ID` (demo account).
3. `npm start` (Ctrl+C stops after the current trade settles).
4. `npm test` runs the unit tests (no network needed).

## Layout
- `src/auth.js`: PAT -> one-time authenticated WebSocket URL (OTP endpoint)
- `src/deriv-client.js`: one socket, req_id matching, subscriptions, ping, auto-reconnect with backoff
- `src/bot.js`: tick -> signal -> guardrails -> proposal -> buy -> settle -> repeat (one trade at a time)
- `src/guardrails.js`: stop-loss, take-profit and max stake are mandatory; plus max consecutive losses / trades
- `src/money/`: MoneyManager plug-ins (Fixed now; the other five in phase 2)
- `src/strategies/`: Strategy plug-ins (Even/Odd now)

## Safety defaults
- Real accounts are blocked unless `ALLOW_REAL=true`.
- A contract that doesn't settle in 60s stops the bot (unknown exposure).
- 3 consecutive API errors stop the bot.

## Not yet verified
Tested against a fake socket only, not the live Deriv API. Run on demo first and check the
field names in `src/bot.js` against the current docs at developers.deriv.com.

## Adding a strategy (phase 2+)
Implement `onTick(tick) -> null | { contract_type, duration, duration_unit, reason }`
and register it in `src/index.js`.

## Paper trading comparison (no account, no token, no money)
Runs 9 strategies side by side on live ticks from Deriv's public endpoint with virtual stakes.

    npm run paper        # live ticks; Ctrl+C stops and saves paper-results.json
    npm run paper:sim    # offline harness check on synthetic random ticks

Options (env): `SYMBOL` (default 1HZ100V), `MARGIN` (assumed payout margin, default 0.03),
`REPORT_EVERY` (seconds, default 60), `TICKS` (stop after N ticks), `OUT` (results file).

For each strategy it reports win rate, the breakeven win rate its payout requires, ROI, and a z-score.
Verdicts use a Bonferroni correction (the more strategies you test, the higher the bar), so the
"best of 9" cannot win on luck. Two coin-flip **control** rows are included; if a strategy can't
beat them clearly, it has no edge.

Reading the results:
- Needs thousands of trades. On a 1-second index, 5-tick strategies make ~700 trades/hour, 1-tick ones ~1800/hour.
- `POSSIBLE EDGE` only means "re-test on fresh data", never "go live".
- `MARGIN` is an assumption. Calibrate it from real `proposal` quotes (payout vs stake) for better realism.
- Tick gaps (disconnects) void open paper contracts instead of settling them unfairly.

Validated by tests: 9 strategies on 300k random ticks are all `NO EDGE` (ROI near minus the margin),
and a strategy that peeks at the future IS flagged, so the harness can detect an edge if one exists.

## Live bot default: EMA+RSI Rise/Fall, 2-minute contracts
`STRATEGY=ema-rsi-2m` (default) buys CALL/PUT with a 2-minute duration on `SYMBOL` (default 1HZ100V),
only on a fresh EMA 30/90 crossover confirmed by RSI (so about 15 trades/hour at most).
This was chosen because digit contracts can't run 2 minutes (ticks only), NOT because it is proven to win.
Paper-test it first with `npm run paper` (the "EMA+RSI cross 2min" row uses the same settings). It needs
roughly 3 days of live ticks to reach ~1000 trades, which is the minimum for a meaningful verdict.
Check on demo that your account allows 2-minute Rise/Fall on the chosen market; a rejected proposal
shows up as a trade error and 3 in a row stop the bot.
-
