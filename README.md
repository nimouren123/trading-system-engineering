# Trading System Engineering

Notes from building and running an automated ETH/USDT perpetual futures system on real capital since June 2026.

This repository is **not** about the strategy. It's about the part that decides whether you lose money for reasons you didn't intend: keeping a stateful system correct while it is exposed to a live exchange, process crashes, network failures, and your own past fixes.

The system has been running on a single VPS for ~5 months, through three architecture rewrites. It has been reviewed in 12 rounds, producing ~38 fixes, 3 of them critical.

---

## The three problems worth writing about

Most of the work was not writing features. It was answering three questions over and over:

**1. What do you trust when your memory and reality disagree?**
[`docs/01-source-of-truth.md`](docs/01-why-exchange-is-source-of-truth.md) — why the exchange is the only source of truth, and how a "smarter" local cache turned into a 50x unhedged position.

**2. How do you make an action safe to repeat?**
[`docs/02-idempotency-and-recovery.md`](docs/02-idempotency-and-recovery.md) — client order IDs, an order state machine, and what recovery actually means when the process dies at the worst possible moment.

**3. What breaks that you didn't design for?**
[`docs/03-failure-modes.md`](docs/03-failure-modes.md) — eight real incidents, each with symptom, root cause, fix, and how the fix was verified.

Two more documents:

- [`docs/04-architecture-iterations.md`](docs/04-architecture-iterations.md) — three rewrites, and why each one happened
- [`docs/05-review-methodology.md`](docs/05-review-methodology.md) — how 12 rounds of review worked, and why round 8 still found critical bugs after round 4 declared convergence

---

## Structure

```
v6/
├── main.py           entry point, mode selection
├── engine.py         main loop: fetch → decide → execute
├── exchange.py       OKX REST client, retries, failure counters
├── order_manager.py  idempotency, order state machine, fill polling
├── position.py       position lifecycle, PnL accounting
├── risk.py           drawdown brake, cooldown, emergency stop
├── state.py          persistence, startup reconciliation
├── strategy.py       signal generation
├── config.py         parameters
└── constants.py      enums, dataclasses
```

The loop:

```
        ┌──────────────┐
        │  fetch bars  │
        └──────┬───────┘
               v
        ┌──────────────┐      crash anywhere here?
        │   strategy   │      → restart rebuilds from
        └──────┬───────┘        the exchange, not from disk
               v
        ┌──────────────┐
        │  risk gate   │──── blocked ──> wait
        └──────┬───────┘
               v
        ┌──────────────┐
        │ order manager│  clOrdId → place → poll → state machine
        └──────┬───────┘
               v
        ┌──────────────┐
        │ reconcile    │  exchange position vs local view
        └──────────────┘
```

---

## Design rules that came out of this

1. **The exchange is the source of truth.** Local state is a cache. On startup, local position and order records are discarded and rebuilt from exchange queries.
2. **Every order carries a client-generated ID** persisted before the request is sent. A retry after a timeout is matched, not duplicated.
3. **No `sleep()` as a substitute for confirmation.** Orders move through an explicit state machine; nothing proceeds until the current state is terminal.
4. **The bot must be able to tell its own position from a manually-opened one.** Without a marker, recovery logic will adopt whatever is on the account.
5. **A fix is a bug until proven otherwise.** Round 7 found that a method added in round 5 was silently shadowed by an older property of the same name — the fix itself broke a code path.
6. **Stop reviewing when a full pass produces nothing new**, not when a pass feels thorough.

---

## Scope and disclaimer

- The trading logic, parameter values, and account details are not included. This repository documents **engineering decisions and failure modes**, not a strategy.
- Nothing here is investment advice.
- Code snippets are excerpts, lightly edited for readability; they are not runnable as-is.

## License

MIT
