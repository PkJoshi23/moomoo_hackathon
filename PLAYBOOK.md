# PLAYBOOK — ETF Opening-Range Breakout (baseline v1)

Owner: participant. Effective: 2026-11-24. Supersedes: none.
Parameters live in `config/limits.yaml`; constraints live in `RULES.md`.

## 1. Thesis

Liquid broad ETFs trend intraday from the cash open when the overnight move and
the first 15 minutes agree in direction. The edge is not the pattern; it is the
combination of (a) a small, fixed disaster stop, (b) a session that ends with a
forced flatten, and (c) position sizing that keeps a losing streak survivable.
Backing off on ambiguous openings is part of the strategy, not a failure of it.

Chosen because it is deterministic, expressible as a rule gate, and does not
depend on news interpretation — which is what lets the Scout stay read-only.

## 2. Universe

`QQQ`, `SPY`, `IWM` — three broad ETFs, all far above the USD 1 / USD 500M floor,
no single-name event risk, no borrow cost, no margin surprises.

Not traded in v1: single stocks (event risk inside an 8.5-day window),
leveraged/inverse ETFs (daily-reset decay on any multi-day hold), options
(minimum commission makes small-lot scalping structurally unprofitable).

## 3. Signal (Scout output, no orders)

Computed once per session at the open (22:30 SGT), refreshed at 22:45 SGT:

1. **Regime filter** — daily close versus 20-period EMA on the same ETF.
   Longs only when price > 20 EMA; shorts only when price < 20 EMA.
2. **Opening range** — high and low of the first 15 minutes of the regular
   session (22:30–22:45 SGT).
3. **Breakout trigger** — close of a 5-minute bar beyond the range edge, with
   that bar's volume ≥ 1.25× the average volume of the three range bars.
4. **Disqualifiers** — range width > 1.5× the 20-day average true range, or
   range width < 0.15% of price. Both mean the open told us nothing.

Output: a signal object per symbol with regime, range high/low, trigger price,
invalidation price, and a confidence field that is descriptive only — the Guard
does not trade on it.

## 4. Entry

- One entry per ticker per session, at most one position per ticker.
- Entry order: limit at the trigger price or better, valid for 5 minutes, then
  cancel. No market orders in extended hours (R7).
- Skip the entry if more than 30 minutes have elapsed since the trigger.
- No new entries after 03:30 SGT (one hour before the regular close).

## 5. Stop and exit

- Initial stop: the opposite side of the opening range, but never wider than the
  distance that puts 0.75% of equity at risk. If the range is too wide for that,
  size down; if the size falls below one whole share, skip the trade.
- Target: close 50% at 1R, move the stop on the remainder to breakeven, trail the
  remainder with a 0.5R stop.
- Time stop: everything closes at 04:45 SGT regardless of P&L (R10).
- No averaging down. No widening a stop. A re-entry after a stop-out requires a
  new opening-range break setup and counts against the daily trade cap.

## 6. Sizing

```
risk_usd   = equity * 0.0075
stop_dist  = |entry - stop|
qty        = floor(risk_usd / stop_dist)
notional   = qty * entry
qty        = min(qty, floor(equity * 0.35 / entry))   # exposure cap, R3
```

Reject the trade if `notional > 0.38 * equity`, if total heat would exceed 6% of
equity, or if it would create a 4th concurrent position.

## 7. Expectancy — the number that decides whether this survives

```
E[R] = (win_rate * avg_win_R) - (loss_rate * avg_loss_R)
```

Worked example, used as the design assumption (not a claim about results):
45% win rate, average win 1.6R, average loss 1.0R.

```
E[R] = (0.45 * 1.6) - (0.55 * 1.0) = 0.72 - 0.55 = +0.17R per trade
```

At 0.75% equity risk per trade, +0.17R is roughly +0.13% equity per trade before
fees. Fees matter: USD 0.99 platform + activity fee per round trip is roughly
0.01–0.02% of a 30% position, so six trades a day costs about 0.1% of equity in
friction. That is why the trade cap is 6, not 60.

Review gate: after every 20 closed trades, recompute E[R] and win rate from
`journal/`. If E[R] < 0, halt (R15) and record the post-mortem before resuming.

## 8. Session calendar (SGT)

| Session | SGT | Behaviour |
|---|---|---|
| Pre-market | 17:00–22:30 | Scout only. No entries. Build the watchlist, evaluate overnight gaps. |
| Regular | 22:30–05:00+1 | Opening range 22:30–22:45, entries to 03:30, flatten 04:45. |
| After-hours | 05:00–09:00+1 | Report generation only. |
| 2026-11-26 | closed | No trading. Audit logs, review expectancy, tune parameters. |
| 2026-11-27 | early close 14:00 ET | Flatten by 02:00 SGT. No new entries after 00:30 SGT. |
| 2026-11-28 | window closes | No trading. Package and submit. |

## 9. Failure modes

| Situation | Response |
|---|---|
| Quote stale or missing at trigger | HOLD, log the skip with the reason (R12) |
| Order submitted, no fill confirmation | Query order state; reprice at most 3 steps of USD 0.02; then cancel (R13) |
| Position snapshot unreadable | HOLD all new entries until readable |
| Fill differs materially from ticket | Journal the delta, verify stop is still inside risk budget, resize or exit |
| 3% intraday drawdown | Flatten, halt, notify human (R15) |
| Agent schedule missed a run | Human checks positions directly; never assume the flatten job ran |

## 10. What this playbook deliberately does not do

No options, no leveraged products, no overnight holds, no shorting single names,
no averaging down, no discretionary overrides after 22:45 SGT. Each exclusion is
a documented answer to a specific competition constraint, and each is defended in
`README.md` where the judges will look for it.
