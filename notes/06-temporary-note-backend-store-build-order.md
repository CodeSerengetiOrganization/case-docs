# Temporary note — Backend store build order for issue 02

**Type:** Temporary tracking note  
**Project:** C.A.S.E. — Scored EOL cause list  
**Date:** 2026-11-14  
**For:** Jonathan + Sam  
**Decided by:** Jonathan, with Vector Elon  
**Delete when:** The API PR for issue 02 maps API checks 1–12 to tests and the store path ships.

Design: [projects/scored-eol-cause-list/design/02-api-store-the-spike.md](../projects/scored-eol-cause-list/design/02-api-store-the-spike.md)  
Contract: [projects/scored-eol-cause-list/design/02-contract-store-the-spike.md](../projects/scored-eol-cause-list/design/02-contract-store-the-spike.md)  
Issue API checks: [projects/scored-eol-cause-list/issues/02-the-operator-submits-the-product-and-the-eol-stations.md](../projects/scored-eol-cause-list/issues/02-the-operator-submits-the-product-and-the-eol-stations.md)  
Tech note: [projects/scored-eol-cause-list/tech-notes/02-mysql-in-docker-for-store-tests.md](../projects/scored-eol-cause-list/tech-notes/02-mysql-in-docker-for-store-tests.md)

Code target: `case/services/case-api` (empty at the start of this note).

## Rule

Ship one vertical slice. Stop when that slice's API checks are green. Do not invent the real MES product-code or working-order lookup. The open choices stay parked. The mock returns those facts.

Follow store-call steps 1–7 in the API design. Sam must not invent another order.

## Build order

1. **Scaffold `services/case-api`.**  
   Spring Boot, Flyway, Testcontainers MySQL `8.0.21`. Prove the app starts against that MySQL. Optional Compose MySQL for hand diagnose stays at `case/docker/mysql/docker-compose.yml`.

2. **Migrate the three tables.**  
   Create `spike`, `spike_failed_result`, and `spike_product_lock` from the contract. Freeze `working_orders` as one JSON array column. Freeze `failed_test_items` as one JSON column. Freeze `failed_test_items_abnormal` as a boolean on `spike`.

3. **Controller and request validation.**  
   `POST /apis/spikes`. Accept the product name. When the product name is missing or blank, return HTTP 400 and store nothing. Prove API check 4.

4. **Search mock.**  
   Return product code, working orders, and failed EOL results. When the search finds no failed part, return HTTP 200 with `product_name` and `message`. Store nothing. Prove API check 2. Prove the 100 trim (API checks 5 and 6).

5. **Write path in one transaction.**  
   Write `spike` and `spike_failed_result` together. Write `description`. Store missing equipment fields as null and log the gap. Store abnormal `failed_test_items` as MES sent it (or null when missing). Skip those test items in `description`. Log the abnormal. Set `failed_test_items_abnormal`. Return HTTP 201 with `repeated` set to false. Prove API checks 1, 10, and 11.

6. **Repeat check.**  
   Compare the failed manufacturing-result id set only. Matching ids return the existing spike with HTTP 200 and `repeated` set to true. Store nothing. A different `failed_test_items` value on those same ids is still a repeat. One new manufacturing-result id makes a new spike. Prove API checks 7, 8, and 12.

7. **Product-name lock.**  
   Insert the lock row if missing. Recover only on duplicate key. Run `SELECT … FOR UPDATE`. Prove API check 9.

8. **Close the slice.**  
   Wire the happy path end to end. In the PR description, map each API check 1–12 to a test.

## Status

| Slice | Owner | Done |
|---|---|---|
| 1 Scaffold + Testcontainers | Sam | scaffold only (Boot + Flyway + packages); Testcontainers still open |
| 2 Flyway tables | Sam | V1 draft — review |
| 3 Validation (check 4) | Sam | draft — review |
| 4 Search mock (checks 2, 5, 6) | Sam | draft — review |
| 5 Write path (checks 1, 10, 11) | Sam | draft — review (repos mocked; Testcontainers still open) |
| 6 Repeat by id set (checks 7, 8, 12) | Sam | draft — review (repos mocked; Testcontainers still open) |
| 7 Product-name lock (check 9) | Sam | draft — review (unit path green; concurrent wait still needs Testcontainers) |
| 8 PR AC → test map | | |
