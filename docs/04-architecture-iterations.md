# 04 — Three architectures, and why each one got replaced

The system was rewritten twice. Neither rewrite was planned as a rewrite; both started as "make this one change" and ended with the old structure unable to absorb it.

---

## v1 — One script (June 2026)

**Shape.** A single file, roughly 1,200 lines. Fetches bars, computes signals, places orders, manages positions, writes a log. Started with `nohup`, monitored by reading the log.

**Why it worked.** For one strategy on one symbol, a single file is genuinely fine. Everything is in scope, there is no import graph to reason about, and a change is visible in one place.

**What broke it.**

- **Every change touched everything.** Adjusting an exit rule meant editing a function that also contained order placement. There was no seam to test at.
- **State lived in globals.** Position, capital, and the last signal were module-level variables, mutated from several places. Reasoning about "what is the position right now" required reading the whole file.
- **Crash recovery was implicit.** Restarting meant re-reading whatever variables had been persisted, which was an arbitrary subset.

The final straw was a parameter change that required editing three separate places in the file, one of which was easy to miss. Missing it produced no error — just a wrong behavior weeks later.

**The lesson that carried forward:** the failure wasn't the file size. It was that *one concern had no boundary*, so changes to it could not be reasoned about locally.

---

## v2 — Layered modules (July 2026)

**Shape.** Split into nine modules by responsibility:

```
main.py          57      entry point, mode selection
engine.py      2,013     main loop: fetch → decide → execute
strategy.py    1,841     signal generation, pure functions
exchange.py      735     REST client, retries, failure counters
order_manager.py 536     idempotency, state machine, fill polling
state.py         447     persistence, startup reconciliation
config.py        340     parameters
risk.py          273     drawdown brake, cooldown, emergency stop
position.py      271     position lifecycle, PnL accounting
```

**How the boundaries were chosen.** Not by "layer" (data / logic / IO) but by **what changes together**. Price parsing and retry policy change together → `exchange.py`. Order lifecycle and idempotency change together → `order_manager.py`. Signals change constantly and never touch IO → `strategy.py` became pure functions with no side effects, which made it testable against historical frames.

The rule that made this work: **`strategy.py` imports nothing that can perform IO.** A strategy that cannot place an order cannot accidentally place one during a backtest.

**What it bought.** Every module could be exercised independently. The strategy could be replayed over five months of bars in seconds. The order manager could be driven by a stub exchange, which is how the eight failure modes in [03](03-failure-modes.md) were reproduced at all.

**What it cost — and this is the part that gets skipped.**

- **The seams became the bugs.** Two of the three critical defects lived *between* modules: a persistence contract (`state` assumed `order_manager` would flush; it didn't) and a recovery path that inferred ownership from records it did not own.
- **Review scope grew.** A change in `order_manager` could now be wrong because of an assumption in `state`, five files away. Reviewing a diff stopped being sufficient; reviewing a *data flow* became necessary.
- **The module split does not match the failure modes.** Failures are cross-cutting: a kill switch is `risk` × `state` × systemd. The boundary that makes code testable is not the boundary that makes failures understandable.

**The lesson:** splitting by responsibility reduces local complexity and increases coupling at the seams. Both effects are real; only the first is usually acknowledged.

---

## v3 — Multiple services (October 2026)

**Shape.** Several small processes under systemd, each with its own lifecycle:

- the main strategy process
- a guard process that watches for a specific adverse condition and intervenes independently
- a supervisor that restarts and re-syncs

**Why the single process stopped being enough.** Two reasons:

1. **Different restart semantics.** A guard that watches for a wick-and-reversal condition must keep running when the strategy process is mid-restart. Putting them in one process means either the guard dies with the strategy, or the strategy cannot restart independently.
2. **Failure isolation.** A bug in the strategy loop should not take down the thing that places protective orders.

**What it cost.** Every cross-process interaction became a consistency problem. Two processes can now disagree about the position, and only one of them writes state. The honest summary: **this architecture solved an availability problem and created a consistency problem.**

**Where it landed.** The consistency problem is managed rather than solved: exactly one process owns the position state; the others read it and can only act through it. Any second writer is a bug by definition.

---

## What three rewrites taught

| | v1 | v2 | v3 |
|---|---|---|---|
| Lines | ~1,200 | ~6,800 | ~6,800 + guards |
| Testability | none | per-module | per-process |
| Failure surface | everywhere | seams | seams + consistency |
| Recovery | implicit | explicit | distributed |
| Debug cost | read one file | trace a data flow | trace across processes |

**The pattern:** each architecture moves the difficulty rather than removing it.

- v1's problem was *no boundaries* → v2 added boundaries → boundaries created seams → seams created the critical bugs.
- v2's problem was *every failure cross-cuts* → v3 separated lifecycles → separation created consistency problems.

This is worth stating plainly because "we should refactor" is almost always true and almost never the whole story. The right question is **which failures you are trading for which**, not whether the current shape has problems.

---

## A note on what was *not* changed

Several known differences between backtest and live were documented and deliberately left alone, because the fix was more dangerous than the defect:

- **Bars-held across restarts** — the repair required reconciling a local ISO timestamp with UTC bar boundaries. The timezone trap was a larger risk than the lost bar count.
- **Cooldown anchor** — backtest anchors to the entry bar, live anchors to the actual exit. Live is stricter; forcing equivalence would loosen it.
- **Funding fees** — neither side models them; capital is reconciled from exchange equity, which absorbs them.

Documenting a known deviation is a decision, not an oversight. The failure mode of "fix everything" is that you ship a larger change you have not reviewed as carefully as the small one you were trying to correct.
