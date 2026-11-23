# Moomoo × Qoder AI Trading Agent Challenge — submission

Two-agent system for the 2026 Moomoo × Qoder paper-trading competition:
a **read-only Scout** that produces signals, and a **Guard** that is the only
component allowed to place an order and that refuses anything it cannot verify.

Paper trading only. US equities and ETFs only. No options, no leveraged
products, no overnight holds. Submission deadline 2026-11-30 23:59 SGT;
trading window closes 2026-11-28.

## Architecture

```
                 +-----------------------------+
  schedule  ------->          SCOUT            |   read-only: market data,
  (SGT)            |  regime -> range -> signal  |   quotes, news, positions
                   +--------------+--------------+
                                  | signal JSON (status, levels, timestamps)
                                  v
                   +-----------------------------+
                   |           GUARD             |   sole order path
                   |  G0..G14 rule gate          |
                   |  submit -> verify -> stop   |
                   |  flatten on schedule        |
                   +--------------+--------------+
                                  |
                    +-------------+-------------+
                    v                           v
              moomoo MCP                  journal/*.md + evidence/
              (paper account)             (decision + rejection log)
```

Design claims, each testable:

- The component with opinions (Scout) has no order tools.
- The component with order tools (Guard) has no opinions; it can only reject.
- Every refusal is logged with the rule ID that fired, so rule enforcement is
  demonstrated rather than asserted.
- "Submitted" is never reported as "filled" — the order record is polled, and
  unfilled orders are journaled like fills.

## Repo map

| Path | Purpose |
|---|---|
| `RULES.md` | Enforceable constraints (R1–R17) with source and enforcement point |
| `PLAYBOOK.md` | Strategy spec: thesis, universe, signal, entry, exit, sizing, expectancy, calendar |
| `config/limits.yaml` | Single source of truth for every number the Guard checks |
| `agents/scout.md` | Scout role card — deploy as a Qoder Cloud Agent |
| `agents/guard.md` | Guard role card — deploy as a Qoder Cloud Agent |
| `journal/` | One file per session: decisions, rejections, fills, post-mortems |
| `evidence/` | Screenshots, schedule IDs, agent configs (no secrets) |

## Setup

1. Qoder → Extension → Connectors: confirm `moomoo OpenAPI` and
   `Qoder Cloud Agents` both show Ready.
2. Authorise moomoo MCP scopes: **Market Data** and **Watchlists** checked;
   **Accounts & Orders** and **Trade Execution** left unchecked at the app level.
   The Guard's own connection holds the execution scope.
3. Create two Cloud Agents in Forward mode from `agents/scout.md` and
   `agents/guard.md`, with the schedules listed in each card (SGT).
4. Add the competition timeline guardrails (holiday close, early-close flatten).
5. Run the smoke test: Scout produces signals, Guard receives a deliberately
   oversized ticket and rejects it with the rule ID. File the rejection in
   `evidence/`.

## Scoring map

| Criterion | Weight | Where it is answered |
|---|---|---|
| Risk-adjusted return (Sharpe) | 25% | `PLAYBOOK.md §6–7` sizing and expectancy gates; forced flatten keeps drawdown bounded |
| Strategy and risk design | 30% | `RULES.md` + `PLAYBOOK.md` + the rejection log in `journal/` |
| Qoder and QCA usage | 30% | Two-role QCA deployment, scheduled jobs, connector config, dashboard |
| Reusability and documentation | 15% | This README, role cards, `config/limits.yaml`, journal template |

## Status

- [x] Rules, playbook, agent cards, config, journal template scaffolded (2026-11-24)
- [ ] Connector verification and account audit recorded in `journal/2026-11-24.md`
- [ ] Both agents deployed and scheduled
- [ ] Smoke-test rejection filed in `evidence/`
- [ ] First live paper session journaled (2026-11-24 / 11-25)
- [ ] Expectancy review after 20 closed trades
- [ ] Dashboard published (equity, drawdown, exposure, orders used, rule compliance)
- [ ] Submission packaged 2026-11-29

## Disclaimer

Educational simulation. Nothing here is investment advice. All trading happens
in the competition's paper account. No credentials, tokens, or account numbers
belong in this repository.
