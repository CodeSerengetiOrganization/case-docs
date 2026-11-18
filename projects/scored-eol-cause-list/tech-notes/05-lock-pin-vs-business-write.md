# Tech note — Lock pin vs business write

**Type:** Tech note  
**Project:** CASE — Scored EOL cause list  
**Date:** 2026-11-16  
**Decided by:** Jonathan, with Vector Elon

## Decision

1. A **lock pin** may not exist yet. Creating that pin can race. `INSERT IGNORE` (or equivalent “ensure row”) is OK for that pin write.
2. A **business write** (`spike`, Depanel split / COMPLETE) must not use `INSERT IGNORE`. Clash or refuse must follow plant rules, not silent skip.
3. Depanel locks an existing `panel_registration` row. CASE uses `spike_product_lock` because the product name is not guaranteed to exist as a pin.

## CASE product-name lock

`spike_product_lock` is only a concurrency pin. The first report for a product name may find no row.

Two threads can race to insert the same `product_name`. Both outcomes are success: insert the row, or the row is already there. Then both run `SELECT … FOR UPDATE` on that row.

## Integration test failure — root cause and fix

IT: `SpikeServiceIT.reportSpike_twoConcurrentSameIdSet_oneNewOneRepeated`.

Both threads call `reportSpike` for the same product name. Search stays outside the store TX. Each thread then enters `SpikeStoreWriter.storeOrReturnRepeat` and Spring opens one store TX per thread.

### Old path — insert and catch

1. Thread A inserts `spike_product_lock`. The insert succeeds.
2. Thread B inserts the same primary key. MySQL returns `1062`. Hibernate raises `DataIntegrityViolationException`. Spring marks Thread B’s store TX **rollback-only**.
3. <span style="color: blue">**Root cause of the IT failure (not a spike-store business step):** Thread B catches the exception and clears the persistence context. Java continues. The rollback-only flag stays. Catching the exception does not clear that Spring flag. That is why commit later throws `UnexpectedRollbackException`.</span>
4. Thread B may run `SELECT … FOR UPDATE` and finish the Java path. On commit, Spring sees rollback-only and throws `UnexpectedRollbackException`. The store TX rolls back.

### New path — `INSERT IGNORE`

1. Thread A runs `INSERT IGNORE`. MySQL inserts the row. No exception. Thread A’s store TX stays clean.
2. Thread B runs `INSERT IGNORE` for the same key. MySQL inserts nothing. No exception. Thread B’s store TX stays clean.
3. Both threads run `SELECT … FOR UPDATE` on that lock row. MySQL makes one wait while the other holds the row lock.
4. The first thread to own the lock does the repeat check and the write, then commits. The second thread gets the lock, sees the same id set, returns the repeat, and commits.

### Why the fix works

| Event | Store TX state |
|---|---|
| SQL error that Hibernate surfaces as an exception | Marked rollback-only, even if Java catches it |
| `INSERT IGNORE` that inserts zero rows | Normal success. No exception. No rollback-only |

`INSERT IGNORE` prevents the exception that set the flag. It does not clear a flag that already exists. The concurrent IT passes because the losing insert never marks the store TX.

### Why `EntityManager` went away

On the old path, `EntityManager` existed only for `entityManager.clear()` after a failed `saveAndFlush`.

The duplicate-key insert left a failed insert in the persistence context. The next query could try to flush that failed insert again. `clear()` wiped that dirty state so `SELECT … FOR UPDATE` could run.

`INSERT IGNORE` does not throw on the clash. There is no failed insert stuck in the persistence context. So there is nothing to clear, and `SpikeStoreWriter` does not need `EntityManager` anymore.

## Why not `INSERT IGNORE` on `spike` or Depanel

| Write | “Row already there” | Tool |
|---|---|---|
| Lock / pin | Expected. Continue with `FOR UPDATE`. | `INSERT IGNORE` OK |
| Business (`spike`, depanel split) | Must repeat, refuse, or fail closed | Normal insert |

CASE serializes on the product-name lock, then runs the repeat check, then writes `spike`. Silent ignore would hide a failed business insert or skip the repeat path.

Depanel serializes on an existing panel join row, then inserts split / COMPLETE. A second writer must refuse or get `ALREADY_COMPLETE`, not quietly skip inserts.

## If the Depanel panel row is missing

`SELECT … FOR UPDATE` on a missing `panel_registration` locks nothing. Two writers do not wait on each other.

The gate then finds no join and returns **ORPHAN**. `DepanelSplitWriter.store` inserts no split rows.

Depanel does not invent a pin. Panel Registration must have created the panel row first. That is the opposite of CASE’s product-name lock.

## Related

- CASE store TX and `INSERT IGNORE`: [04-spring-transactional-proxy-and-store-tx-boundary.md](04-spring-transactional-proxy-and-store-tx-boundary.md)
- CASE MySQL / IntelliJ Docker: [02-mysql-in-docker-for-store-tests.md](02-mysql-in-docker-for-store-tests.md)
- A-MES Depanel joins panel lock: [28-tech-note-depanel-split-transaction-joins-panel-lock.md](../../../../a-mes-docs/requirements/tech-notes/28-tech-note-depanel-split-transaction-joins-panel-lock.md)
- A-MES panel `FOR UPDATE` lifetime: [23-tech-note-concurrency-idempotency-transactional-station-complete.md](../../../../a-mes-docs/requirements/tech-notes/23-tech-note-concurrency-idempotency-transactional-station-complete.md)
