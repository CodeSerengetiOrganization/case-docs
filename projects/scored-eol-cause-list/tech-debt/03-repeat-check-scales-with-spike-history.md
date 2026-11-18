# Tech debt — repeat check scales with spike history per product

**Type:** Tech debt (open)  
**Project:** CASE — Scored EOL cause list  
**Date:** 2026-11-17  
**Logged by:** Kai (from Copilot PR review + Jonathan / Elon / Kai discuss)  
**Status:** Open — not a merge blocker for issue 02 store  
**Close when:** Volume shows pain, or a cheaper repeat path ships with an AC and tests.

Source finding: [09-copilot-review-on-issue2-backend-code-change.md](../../../notes/09-copilot-review-on-issue2-backend-code-change.md)  
Contract: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md)  
Code: `SpikeStoreWriter.findRepeatByIdSet`  
Related: [05-lock-pin-vs-business-write.md](../tech-notes/05-lock-pin-vs-business-write.md)

## Debt

Indexes for today’s queries are enough:

- `idx_spike_product_name` supports `findByProductName`
- PK `(spike_id, id)` supports `findBySpikeId`

The cost is the **repeat algorithm**, not a missing index. For one product name the store path:

1. Loads every `spike` row for that name.
2. For each spike, loads that spike’s `spike_failed_result` ids.
3. Compares each id set to the incoming set until one matches.

That work grows with **spikes per product** × **failed rows per spike**, and it runs under the product-name lock.

## Why it is debt, not a store bug today

- Issue 02 volume is low (operator report, mock data, few spikes per name).
- The id-set rule is correct; the loop implements that rule.
- Copilot marks this as an optimization opportunity, not a schema defect.

## Decision still required (pay the debt)

When volume hurts, pick a thinner path that still compares id sets, for example:

| Direction | Idea |
|-----------|------|
| A | Store a fingerprint / sorted-id hash on `spike` and look that up first |
| B | Compare newest spikes first and stop early when product history is huge |
| C | Cap or archive old spikes per product if the product rule allows it |

Do not add unused indexes that do not change this loop.

## Suggested trigger to pay

Pay this when any of these land:

- One product name holds a large spike history and report latency or lock wait shows up.
- An AC requires a latency bound for repeat under deep history.
- Metrics show many `findBySpikeId` calls per report.

## Out of scope for this debt file

- `product_name` collation — [01-product-name-mysql-collation.md](01-product-name-mysql-collation.md)
- `created_at` UTC — [02-created-at-datetime-vs-instant-utc.md](02-created-at-datetime-vs-instant-utc.md)

## Links

- Issue 02: [02-the-operator-submits-the-product-and-the-eol-stations.md](../issues/02-the-operator-submits-the-product-and-the-eol-stations.md)
