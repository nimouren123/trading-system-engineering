# 01 — The exchange is the source of truth

## The version that seemed reasonable

The first version kept position state in a local JSON file. On startup it loaded that file, and during the loop it updated it after every fill. The exchange was queried only when the bot needed a price.

This worked for weeks, because most of the time the local file and the exchange agree. The failure mode appears only when they don't — and by then the bot has already made a decision based on the wrong one.

## How it failed

Two defects combined.

**First, the polling function was dead code.** A reconciliation loop existed to query every open order and update local records. It was implemented, imported, and never called — the call site had been removed during a refactor and nothing failed loudly. Local order records therefore stayed at whatever status they were written with, forever.

**Second, the recovery path inferred ownership from the local record.** On startup, the bot looked at the exchange position. If it had a matching local entry order marked `filled`, it adopted the position. Otherwise it classified the position as `FOREIGN` and refused to touch it.

Combined: the local order record said `pending` (because nothing ever updated it), the exchange said the position was filled. The bot concluded the position was not its own, and refused to act.

What happened next was the dangerous part. Refusing to adopt the position also meant refusing to place a protective stop. The position stayed open, at 50x, with no stop order on the exchange. It was not "idle" — it was unhedged. The only thing that closed it was the user noticing and closing it manually.

Neither defect is exotic. Each is individually survivable. The system failed because they composed.

> Detection note: the dead function was found independently by two reviewers using different lenses in the same round. That convergence is what moved it from "suspicious" to "must fix before next deploy."

## The rule

After that incident, the rule became:

> **The exchange is the only source of truth. Local state is a cache. On startup, local position and order state are discarded and rebuilt from exchange queries.**

Not "synced". Discarded and rebuilt. The distinction matters: a sync can be skipped, deferred, or silently fail. A rebuild cannot — if the rebuild throws, the bot does not start.

## What "source of truth" actually means in code

### 1. Recovery starts from queries, not files

```python
def sync_from_exchange(self, exchange, monitor, capital):
    """
    Exchange → local, one direction only.

    1. exchange has position, local doesn't    → rebuild from exchange (crash recovery)
    2. exchange has no position, local does    → stop was hit, record the exit
    3. both have position, sizes disagree      → trust the exchange, rebuild
    4. neither has position                    → nothing to do
    5. order partially filled                  → query the exchange to confirm
    """
```

The local JSON is still written — it's used for diagnostics and for restart history. But no decision is ever made from it.

### 2. A position must be provably yours before you adopt it

Adopting any position on the account is wrong: the user may have opened one manually, and the bot would then place stops against it, or close it as part of an exit signal. The bot needs evidence.

Naive implementation — "does a filled entry order exist that matches?" — has a hole. Positions get built in layers: an entry, then additions. If the check only matches the entry order, the total position (entry + additions) will exceed the entry size, the match fails, and the bot classifies its own position as foreign.

The fix is to accumulate all same-side additions ordered after the entry:

```python
def _is_bot_position(self, ex_pos) -> bool:
    ex_ct = ex_pos.get("qty_contracts", 0)
    expected_side = "sell" if ex_pos.get("posSide", "long") == "short" else "buy"

    with self._lock:                       # snapshot under lock — avoid
        orders_snapshot = list(self.orders.items())   # concurrent iteration

    for ord_id, rec in orders_snapshot:
        if rec.action != "entry" or rec.status != "filled":
            continue
        if rec.side != expected_side:
            continue

        # total position = entry fill + every same-side addition filled after it
        total_ct = rec.filled_size
        for _oid, _rec in orders_snapshot:
            if (_rec.action == "pyramid"
                    and _rec.status == "filled"
                    and _rec.side == expected_side
                    and (_rec.create_time or 0) >= (rec.create_time or 0)):
                total_ct += _rec.filled_size

        # 15% tolerance covers partial fills and rounding
        if abs(total_ct - ex_ct) <= max(1, ex_ct * 0.15):
            return True
    return False
```

The tolerance is deliberate. Exact matching fails on partial fills and exchange rounding, and a false negative here means refusing to protect a real position.

### 3. Only adopt what carries your own marker

Orders discovered on the exchange but not in the local table are filtered by client order ID prefix:

```python
if not cl_id or not cl_id.startswith("v6"):
    log.debug(f"Skipping foreign order {ord_id} ({side} {sz}ct {status})")
    continue
```

The prefix is set when the order is created (see [02](02-idempotency-and-recovery.md)). Anything without it was placed by a human, and the bot leaves it alone — including not placing stops against it.

## The cost of this rule

Treating local state as disposable means every restart costs a round of API calls, and recovery must handle every inconsistent state it might find. Three of the eight incidents in [03](03-failure-modes.md) are consequences of recovery logic, not of trading logic.

That is the trade. A cache that can be wrong in a way you cannot detect is more expensive than a cache you throw away.

## Choosing between "protect against it" and "detect it"

One question from this work that generalizes beyond trading:

When the process cannot tell whether an action succeeded, do you make the action safe to repeat, or do you make the failure detectable and stop?

For orders, the answer was *make it repeatable* — a client order ID means a duplicate request is harmless (see [02](02-idempotency-and-recovery.md)).

For position ownership, the answer was *detect and stop* — an unrecognized position is never adopted, and the bot halts rather than guess.

The rule of thumb used here: if a wrong guess causes an unbounded loss, stop. If a wrong guess is bounded and recoverable, make it repeatable.
