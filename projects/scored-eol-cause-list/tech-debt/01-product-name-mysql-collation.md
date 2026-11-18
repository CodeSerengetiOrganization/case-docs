# Tech debt — `product_name` MySQL collation vs identity

**Type:** Tech debt (open)  
**Project:** CASE — Scored EOL cause list  
**Date:** 2026-11-17  
**Logged by:** Kai (from Copilot PR review + Jonathan / Elon / Kai discuss)  
**Status:** Open — not a merge blocker for issue 02 store  
**Close when:** Contract states the compare rule, and schema/collation (or an explicit “accept CI default”) matches that rule.

Source finding: [09-copilot-review-on-issue2-backend-code-change.md](../../../notes/09-copilot-review-on-issue2-backend-code-change.md)  
Contract: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md)  
Migration: `case/services/case-api/src/main/resources/db/migration/V1__create_spike_tables.sql`  
Tables: `spike.product_name`, `spike_product_lock.product_name`

## Debt

`product_name` is the lock primary key and the spike repeat-lookup key. V1 does not set a column or database collation.

MySQL 8 often defaults to a case- and accent-insensitive collation (for example `utf8mb4_0900_ai_ci`). Then `Ford-APCU` and `ford-apcu` can share one lock row and one repeat-lookup bucket.

The Java side passes the request string through as written. The UI for issue 02 posts exact dropdown values. Issue 02 acceptance criteria do not define case-sensitive vs case-insensitive identity.

## Why it is debt, not a store bug today

- Operator selects a fixed product name from the dropdown.
- Mocks and checks use fixed exact strings.
- No check requires two case variants to be different products.

## Decision still required (pay the debt)

Pick **one** axiom and make schema + contract agree:

| Option | Rule | Typical schema move |
|--------|------|---------------------|
| A | Exact identity (bytes / case-sensitive) | Column (or DB) collation such as `utf8mb4_bin` or a case-sensitive utf8mb4 collation; document in contract |
| B | Folded identity (case/accent insensitive) | Keep CI collation; document in contract that variants are the same product |

Do not leave “Java exact, MySQL folded” without a written rule.

## Suggested trigger to pay

Pay this when any of these land:

- Product name is free text, or comes from an external catalog that can vary case.
- A second client posts product names outside the hardcoded dropdown.
- An AC or contract states exact vs folded identity.

## Out of scope for this debt file

- `DATETIME(3)` vs `Instant` timezone — see [02-created-at-datetime-vs-instant-utc.md](02-created-at-datetime-vs-instant-utc.md).
- Repeat-check performance as spike history grows.

## Links

- Lock pin: [05-lock-pin-vs-business-write.md](../tech-notes/05-lock-pin-vs-business-write.md)
- Issue 02: [02-the-operator-submits-the-product-and-the-eol-stations.md](../issues/02-the-operator-submits-the-product-and-the-eol-stations.md)
