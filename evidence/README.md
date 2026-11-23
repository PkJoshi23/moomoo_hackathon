# Evidence

Proof that the system behaves as documented. Judges cannot verify a claim they
cannot see, so this folder carries the artifacts behind `README.md` and
`RULES.md`.

## Naming

`YYYY-MM-DD_<what>.<ext>` — for example
`2026-11-24_guard-rejects-45pct-ticket.png`,
`2026-11-24_schedule-list.png`,
`2026-11-25_fill-verification-qqq.png`.

## What belongs here

- Connector status screenshot (both connectors Ready).
- MCP scope configuration screenshot (which access boxes are checked).
- Both agent cards as deployed, with their schedules.
- The smoke-test rejection: a ticket that the Guard refused, with the rule ID.
- One fill with its order-record verification, showing submitted vs filled.
- One end-of-session flatten, showing zero open positions.
- Dashboard screenshot or URL once published.

## What must never be here

Tokens, PATs, OAuth callbacks, cookies, full account numbers, or any screenshot
that shows a credential field filled in. Blur or crop account identifiers to the
last four characters. If an artifact would leak a secret, do not save it —
describe it in `journal/` instead.
