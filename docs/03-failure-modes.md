# 03 — Eight failure modes

Each entry is a real defect that reached review, was fixed, and was verified. They are ordered roughly by severity.

The pattern worth noticing: **six of the eight are not trading bugs.** They are ordinary software defects — dead code, a shadowed method, a cache-invalidation mistake, an unguarded cast — that in this context happen to have a price attached.

---

## 1. The dead reconciliation loop — unhedged position *(critical)*

**Symptom.** A position sat open at 50x with no stop order on the exchange. The bot reported no errors and appeared idle.

**Root cause.** Two defects composed:

- A reconciliation function, written to poll every open order and update local records, was imported and never called. A refactor removed the call site; nothing failed, because a function that never runs never raises. Local order records stayed at `pending` permanently.
- The startup recovery path inferred position ownership from those local records: "is there a filled entry order matching this position?" With every record stuck at `pending`, the answer was always no — so the bot classified its own position as `FOREIGN` and refused to manage it.

Refusing to adopt the position also meant refusing to place a protective stop. The system was not idle; it was exposed.

**Fix.** Local state stopped being an input to ownership decisions (see [01](01-why-exchange-is-source-of-truth.md)). Reconciliation was wired into the main loop and the call site covered by a startup assertion.

**How it was found.** Two independent reviewers, using different lenses in the same round, converged on the same dead function. That convergence is what escalated it from "worth checking" to "blocks deploy".

**Takeaway.** Dead code is not neutral. A function that exists but never runs makes the code *look* like it handles a case it does not handle — which is worse than the handler being absent.

---

## 2. The fix that shadowed itself *(critical)*

**Symptom.** A re-entry path threw `TypeError` at runtime. It had passed review three rounds earlier.

**Root cause.** A round-5 fix added a method `is_emergency_stopped()` to the risk class. The class already had a property with the same name, defined earlier in the body. In Python, **the later definition wins** — so the old property silently overwrote the new method, and every call site that expected a callable got a `bool`.

The new tests exercised the method in isolation, where it worked. The failing path was the one that called it through the class — which no test did.

**Fix.** Renamed the method, and added a test that calls through the public surface rather than the definition.

**Takeaway.** A fix is a change to a namespace you do not fully control. Round 7 of this project found a defect introduced by round 5; round 9 found one introduced by round 8. **Every fix should be the first thing the next review looks at.**

---

## 3. The order table that never persisted *(high)*

**Symptom.** After a restart, the order table was empty even though orders had been placed and filled.

**Root cause.** Persistence used a dirty flag:

```python
def save_orders(self, force=False):
    if not self._dirty and not force:
        return
```

But the order records were mutated in place from several call sites — `rec.status = ...`, `rec.filled_size = ...` — without going through the setter that raised the flag. The flag stayed `False`, the write never happened, and the table was silently rebuilt empty on every restart.

**Fix.** Writes are now forced at the end of every mutation path, and the dirty flag is not relied on for records that are mutated externally.

**Takeaway.** A dirty flag is a contract between "anything that changes state" and "the thing that persists it". In a codebase with several mutation sites, that contract is easy to break silently, and breaking it produces no error — just missing data later.

---

## 4. `float("")` — one empty string stops everything

**Symptom.** Order records stopped converging. New orders stopped being tracked.

**Root cause.** The exchange returns an empty string — not `0`, not `null` — for fill fields on orders that are live but unfilled. `float("")` raises `ValueError`. The conversion was unguarded inside the status-polling loop, so one live order took down the poller for every order behind it.

**Fix.** Guarded casts, and per-order exception containment so one failure cannot starve the rest.

**Takeaway.** The same empty-string behavior appeared in a second module five rounds later. When an external API has a data-shape quirk, grep for every place that field is parsed — the first fix will not be the only one needed.

---

## 5. The kill switch that wouldn't stay dead

**Symptom.** The bot restarted every ~30 seconds, forever. Each start logged a kill-switch trigger and exited; systemd dutifully restarted it.

**Root cause.** A previous run had written a kill-switch file after repeated API failures. The file was never cleaned up. Every new process started, read the stale file, wrote "KILL SWITCH DETECTED — stopping", and exited. The restart policy turned a one-time failure into an infinite loop that also spammed the log.

**Fix.** Startup now clears stale kill-switch state before it is first read, and the restart policy is bounded.

**Takeaway.** A persistent failure signal outlives the failure. Anything written to disk to stop a process needs an explicit lifecycle — who clears it, when.

---

## 6. Recovery re-entering a stale market

**Symptom.** After a crash, the bot woke up and opened a position at a price far from the signal that triggered it — effectively buying into a move that had already happened.

**Root cause.** Recovery logic checked *whether* an exit/entry event had occurred, but not *when*. An event that fired 40 minutes ago was treated as equally actionable as one that fired 5 seconds ago.

**Fix.** Recovery checks the age of the triggering event against a window (30 minutes) before acting, and re-checks price before sending the order. Outside the window, the event is logged and dropped.

**Takeaway.** In an event-driven system, recovery needs a freshness rule, not just a state rule. "The signal fired" and "the signal is still valid" are different predicates, and only the second one is safe to act on.

---

## 7. The data window that made a module unreachable *(high)*

**Symptom.** A strategy module never produced a signal in live trading, while backtests showed it firing regularly.

**Root cause.** The module required index ≥ 336 in the bar series. The exchange's kline endpoint returns a maximum of 300 bars per request. The live feed requested 300 and stopped; the index the module required could never exist. The module was silently dead — no error, no warning, just no signals.

**Fix.** Paginated fetch to build a ≥ 360-bar window, plus a startup assertion that every module's data requirement is satisfiable by the configured window.

**Takeaway.** Backtest and live have different resource ceilings. A module that works on a full DataFrame can be unreachable on a paginated API. **Every `len(df) >= N` requirement is a production risk until you've checked the feed can deliver N.**

---

## 8. The stop that pointed the wrong way

**Symptom.** In a fallback path, a short position was closed immediately after opening.

**Root cause.** The fallback stop price was computed as a multiplier of entry price without branching on direction. That formula is correct for a long. For a short, the resulting "stop" was below the current price, i.e. already triggered — the stop fired on creation.

**Fix.** Stop direction is derived from position side, with an assertion that the stop is on the correct side of entry before placing.

**Takeaway.** Any price arithmetic that reads the same for longs and shorts is a suspect. A one-line assertion (`stop < entry` for long, `stop > entry` for short) would have caught it before the order was sent.

---

## Summary table

| # | Defect | Class | Caught by |
|---|---|---|---|
| 1 | dead reconciliation loop | composition of two bugs | two independent lenses |
| 2 | shadowed method | fix-introduced | review of round-5 changes |
| 3 | dirty-flag cache | persistence contract | full re-read |
| 4 | `float("")` | API data shape | chaos/edge-case lens |
| 5 | stale kill switch | lifecycle | deployment lens |
| 6 | stale event on recovery | missing freshness rule | chaos lens |
| 7 | unreachable data window | resource ceiling | environment lens |
| 8 | direction-agnostic stop | sign error | chaos lens |

None of these were found by running the system normally. All of them required constructing a specific state and then asking what the code does in it.
