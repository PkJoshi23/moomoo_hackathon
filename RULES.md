# RULES — enforceable constraints

Every rule here is checked by the Guard agent before an order leaves the system
and again when positions are reconciled. A rule is only "real" if a rejected
ticket exists in `journal/` proving it fired. Numbers come from
`config/limits.yaml`; sources are the Moomoo × Qoder paper-trading T&C and the
organiser briefing deck.

Status: **v1, effective 2026-11-24**. Changes require a dated entry in
`journal/` and a matching `config/limits.yaml` edit in the same commit.

## Hard limits (block the order)

| ID | Rule | Source | Enforced in |
|----|------|--------|-------------|
| R1 | Only US-listed equities and ETFs. No crypto, futures, FX, or non-US listings. | T&C | Guard: asset-type check on symbol metadata |
| R2 | Traded price ≥ USD 1.00 and market cap ≥ USD 500M. | T&C | Guard: quote + metadata pre-check |
| R3 | No single stock or ETF may exceed 40% of account equity, measured on entry notional plus existing exposure. Target 35–38%. | T&C / deck | Guard: exposure math before sizing is accepted |
| R4 | At least 3 distinct tickers are active in the strategy universe each session. | Deck | Scout: universe build; Guard rejects a universe with < 3 symbols |
| R5 | Maximum 200 executed orders per day, and never more than 6 for this strategy. | T&C / playbook | Guard: order counter read from order history at session start |
| R6 | Maximum 3 concurrent positions and 1 new position per ticker per session. | Playbook | Guard: position query before submit |
| R7 | Extended-hours orders are limit orders only. | T&C / deck | Guard: order-type whitelist by session |
| R8 | Options are out of scope for v1. If enabled later: close ETF/stock option positions before the expiration-day close, and never rely on auto-exercise. | T&C / deck | Guard: rejects any options ticket while `allowed_asset_types` excludes them |
| R9 | No Reset Card, no account reset, no attempt to reverse or dispute fills to recover losses. | T&C | Human commitment, logged in journal |
| R10 | Flat by 04:45 SGT every session. Flat by 02:00 SGT on 2026-11-27 (early close) and no positions into the weekend or the 2026-11-26 holiday. | Deck / calendar | Guard: scheduled flatten job; journal reconciliation |
| R11 | Never route execution through a live account. Paper/competition account only. | T&C | Guard: account-number allowlist |

## Process rules (block or degrade to HOLD)

- **R12 — Evidence or HOLD.** If the quote, the position snapshot, or the order
  history cannot be read, the Guard returns HOLD. Missing evidence is never
  resolved by inference.
- **R13 — Submitted is not filled.** Every submission is followed by a query of
  the order record. A ticket with no confirmed fill is reported as unfilled,
  not as a position.
- **R14 — One execution path.** Only the Guard agent holds order tools. The
  Scout is read-only. Any other component that can place orders is a defect.
- **R15 — Drawdown halt.** At 3% intraday drawdown on equity, or when trailing
  20-trade expectancy is below 0R, trading stops for the session and the human
  is notified. Resumption requires a human decision recorded in `journal/`.
- **R16 — Secrets stay out.** No PAT, OAuth token, account number, or cookie in
  the repo, in prompts, in logs, or in dashboard client code. `.gitignore`
  already blocks the common paths; do not defeat it.
- **R17 — Session awareness.** Sizing, order type, and flatten timers are chosen
  from the current SGT session flag (pre-market / regular / after-hours), never
  from wall-clock assumptions inside agent reasoning.

## Open interpretation (verify, do not assume)

R4 ("at least 3 distinct tickers") has two plausible readings: three symbols in
the traded universe, or three symbols actually held. The Guard enforces the
stricter of the two — three distinct tickers in the universe and at least three
distinct tickers across open positions whenever ≥ 3 positions are open. If the
organiser confirms the looser reading, update this section and the config
together.

## What nothing else enforces

The moomoo MCP exposes no competition-rule tool and no leaderboard tool. The
broker will not stop a 41% position or a 201st order on your behalf. R1–R11
exist only because this repo implements them.
