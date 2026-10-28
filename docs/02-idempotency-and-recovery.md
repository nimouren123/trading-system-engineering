# 02 — Idempotency, state machines, and recovery

An order request has three possible outcomes and the client can only observe two of them:

1. It failed. (Observable)
2. It succeeded. (Observable)
3. It may or may not have succeeded — the request was sent, the response never arrived. (Not observable)

Case 3 is the one that costs money. A retry either duplicates the order or, if you retry blindly while the first one is live, races against it.

Everything in this document exists to make case 3 harmless.

---

## Client order IDs

Every order is created with a client-generated ID before the request is sent:

```python
def make_client_ord_id(action: str) -> str:
    """v6{action}{ts_ms}{uuid8}"""
    return f"v6{action}{int(time.time() * 1000)}{uuid.uuid4().hex[:8]}"
```

Three properties matter:

- **Uniqueness** — timestamp plus random suffix; collision is not a scenario worth designing for.
- **Self-identification** — the `v6` prefix is how recovery tells bot orders from human orders on the same account.
- **Queryability** — the ID is written to local storage *before* the request is sent, so a crashed process can ask the exchange "did you ever see this ID?"

That last point is the whole mechanism. The ID is not a convenience for logging; it is the recovery key.

---

## The order state machine

An order is never "just placed". It moves through explicit states:

```
CREATED
   │  (request sent)
   v
PENDING ──> PARTIALLY_FILLED ──> FILLED
   │                │
   │                └─────────> CANCELED
   v
FAILED                  UNKNOWN  ← response lost / query failed
                           │
                           └──> (reconciled by querying the exchange)
```

```python
class OrderStatus(Enum):
    CREATED = "created"
    PENDING = "pending"
    PARTIALLY_FILLED = "partially_filled"
    FILLED = "filled"
    CANCELED = "canceled"
    FAILED = "failed"
    UNKNOWN = "unknown"
```

The rule that matters: **nothing downstream proceeds until the current state is terminal.** In particular, there is no `sleep(N)` followed by an assumption that the order filled. Sleep-based confirmation is the most common way a trading script silently diverges from reality — it looks like confirmation and is not.

`UNKNOWN` is a first-class state, not an error path. An order whose status query fails is marked `UNKNOWN` and left in the active set; it is retried on the next loop rather than being assumed dead or alive.

---

## Single-flight guard

Only one order may be in flight at a time:

```python
def _check_idempotent(self):
    if self._pending and self._pending.is_active:
        return (f"IDEMPOTENCY BLOCK: order '{self._pending.cl_ord_id}' "
                f"({self._pending.action}) still {self._pending.state.value}")
    return None
```

Every entry and exit path calls this first. If a previous order has not reached a terminal state, the new request is refused rather than queued — a queue would eventually execute stale intentions at current prices.

---

## Polling must not be starved by its own failures

The status poller iterates over all non-terminal orders. The obvious implementation lets an exception on one order kill the loop:

```python
for ord_id, rec in self.state.orders.items():
    status_info = self.exchange.get_order_status(ord_id)   # ← raises on one bad order
    ...                                                     #   and every later order
                                                            #   never converges
```

This is how a single malformed field on one order becomes a system-wide stall. The fix is per-order containment:

```python
for ord_id, rec in list(self.state.orders.items()):
    if rec.status in ("filled", "canceled", "failed"):
        continue
    try:
        status_info = self.exchange.get_order_status(ord_id)
        if status_info is None:
            rec.status = "unknown"
            continue
        ...
        # the exchange returns "" rather than 0 for live orders
        _fsz = status_info.get("accFillSz", 0)
        rec.filled_size = float(_fsz) if _fsz not in (None, "", 0) else 0.0
    except (ValueError, TypeError, KeyError, RuntimeError) as e:
        log.warning(f"Order {ord_id} convergence error: {e}")
        continue
```

Two details in there were both real incidents:

- **`float("")` raises `ValueError`.** The exchange returns an empty string — not `0`, not `null` — for fill fields on live orders. An unguarded cast kills the convergence loop, which is the one loop that keeps local records consistent with reality.
- **One bad order must not starve the others.** `continue` inside the `except` keeps the remaining orders converging.

The same empty-string assumption appeared twice, in two different modules, five review rounds apart. Once a data-shape quirk is found, it is worth grepping for every other place the same field is parsed.

---

## Recovery: what "restart" actually has to do

On startup the process may find any of these:

| Local view | Exchange | Correct action |
|---|---|---|
| no position | no position | start clean |
| no position | **position exists** | rebuild — the process died after entry |
| position | no position | stop was hit while down — record the exit, do not re-enter |
| position | position, sizes differ | trust the exchange, rebuild |

The dangerous one is the second row. The naive rebuild reads equity and sets it as the capital baseline — and equity includes unrealized PnL. When the position later closes, the realized PnL is added again. Capital is double-counted, and every risk calculation downstream (drawdown brake, position sizing) is wrong from then on.

The fix:

```python
# rebuild capital from equity, but exclude unrealized PnL —
# it will be realized at close and would otherwise be counted twice
capital = equity - unrealized_pnl
```

This is a good example of why recovery is not "reload the file and continue". Recovery is a reconstruction problem: you are inferring what happened while you were dead, from evidence that has side effects.

---

## What this does not solve

- **The response can arrive after the retry.** Client IDs make the duplicate harmless at the exchange, but the local order table still has to reconcile two records pointing at one order. Handled, but it's why `orders` is keyed by exchange order ID, not by client ID.
- **Recovery has a time window.** If the process is down long enough, the state it finds may no longer be actionable — an exit signal that fired an hour ago should not be executed now. Recovery checks the clock before acting; see the incident in [03](03-failure-modes.md#6-recovery-re-entering-a-stale-market).
- **Recovery logic is where fixes go to hide.** Two of the three critical defects in this project were introduced by recovery code, not trading code. That is not a coincidence — recovery runs rarely, on inconsistent state, and is the least-tested path in the system.
