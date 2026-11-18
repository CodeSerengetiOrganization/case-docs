# Tech debt — `created_at` DATETIME(3) vs `Instant` / UTC

**Type:** Tech debt (open)  
**Project:** CASE — Scored EOL cause list  
**Date:** 2026-11-17  
**Logged by:** Kai (from Copilot PR review + Jonathan / Elon / Kai discuss)  
**Status:** Open — not a merge blocker for issue 02 store  
**Close when:** Contract or deploy notes state the UTC rule, and app/DB session config matches that rule (optional column-type change only if the team chooses it).

Source finding: [09-copilot-review-on-issue2-backend-code-change.md](../../../notes/09-copilot-review-on-issue2-backend-code-change.md)  
Contract: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md)  
Migration: `case/services/case-api/src/main/resources/db/migration/V1__create_spike_tables.sql`  
Code: `SpikeEntity` / `SpikeFailedResultEntity` / `SpikeProductLockEntity` use `Instant`; store path uses `Instant.now()`.

## Debt

`created_at` columns are `DATETIME(3)`. That type does not store a time zone.

Java entities and the API use `java.time.Instant` (a UTC timeline point). JDBC / Hibernate convert using the JVM or Hibernate JDBC time zone. If that zone is not fixed to UTC, write and read can shift across laptops, CI, and plant servers.

Issue 02 does not set `hibernate.jdbc.time_zone` (or an equivalent connection UTC rule) in `application.properties`.

## Why it is debt, not a store bug today

- Repeat identity uses the manufacturing-result **id set**, not `created_at`.
- Trim / “most recent” only needs a consistent clock inside one environment.
- Unit and IT fixtures use fixed `Instant` values with `Z`.

## Decision still required (pay the debt)

Axiom (Elon / Kai): treat stored spike and lock times as **UTC**.

| Step | Action |
|------|--------|
| 1 | Document: all CASE `created_at` values are UTC. |
| 2 | Config: set `spring.jpa.properties.hibernate.jdbc.time_zone=UTC` (and align MySQL session / deploy if needed). |
| 3 | Optional later: move columns to `TIMESTAMP(3)` if the team wants stronger UTC semantics in MySQL — not required to close issue 02. |

## Suggested trigger to pay

Pay this when any of these land:

- Deploy to a host whose default zone is not UTC.
- Operators or audits compare CASE times to plant-local clocks and see shifts.
- A second service reads the same `DATETIME(3)` without the same JDBC zone rule.

## Out of scope for this debt file

- `product_name` collation — see [01-product-name-mysql-collation.md](01-product-name-mysql-collation.md).
- MES `ManufacturingResult.created_at` SoT (MES owns that clock; CASE copies the value).

## Links

- Issue 02: [02-the-operator-submits-the-product-and-the-eol-stations.md](../issues/02-the-operator-submits-the-product-and-the-eol-stations.md)
- MySQL for store tests: [02-mysql-in-docker-for-store-tests.md](../tech-notes/02-mysql-in-docker-for-store-tests.md)
