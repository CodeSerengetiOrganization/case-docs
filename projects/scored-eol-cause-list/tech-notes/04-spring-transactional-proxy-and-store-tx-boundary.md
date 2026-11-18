# Tech note — Spring `@Transactional` proxy and the store TX boundary

**Type:** Tech note  
**Project:** CASE — Scored EOL cause list  
**Date:** 2026-11-16  
**Decided by:** Jonathan, with Vector Elon

Code under review: `case/services/case-api/src/main/java/com/case_api/service/SpikeService.java`  
Design: [02-api-store-the-spike.md](../design/02-api-store-the-spike.md) (lock, repeat check, and write in one transaction)

## Decision

1. The store transaction must cover only **lock + repeat check + write**. It must not wrap the MES search mock or the empty “no failed part” return.
2. Spring starts that transaction only when a caller hits a **proxy** method that carries `@Transactional`.
3. CASE must not rely on `@Transactional` on a same-class helper method that `reportSpike` calls through `this`. That call skips the proxy.
4. The thin fix is a **separate Spring bean** for the write path (or a `TransactionTemplate` around that block). Annotation on `storeOrReturnRepeat` inside `SpikeService` alone is not enough.

## How `@Transactional` works

At context startup Spring builds the real service instance, then wraps it in a **proxy** when a transactional advisor applies. Controllers and other beans receive the proxy.

On each call:

1. The caller invokes the proxy.
2. If that method is `@Transactional`, the interceptor opens a transaction and binds a database connection.
3. The interceptor calls the real method.
4. On normal return the interceptor commits. If an unchecked exception leaves the transactional method, the interceptor rolls back.

The annotation is advice on the proxy entry. It is not a magic flag on every internal call.

```text
Controller → SpikeService proxy → TransactionInterceptor → real SpikeService method
```

## Best practice for the TX boundary

Define the boundary from the data law, not from the public API shape.

| In the transaction | Out of the transaction |
|---|---|
| Insert / recover `spike_product_lock` | MES search (`SpikeDataRetriever`) |
| `SELECT … FOR UPDATE` on that lock row | Trim of the in-memory failed list |
| Repeat check by manufacturing-result id set | `NoFailedPartResponse` when the list is empty |
| Insert `spike` and `spike_failed_result` | |

Axiom: one connection, one atomic unit for lock + repeat + write. A second concurrent caller for the same product name waits on the same lock row. Search can stay outside that unit.

## When the annotation does not work

The annotation does nothing when the call never enters the proxy.

Common miss — **self-invocation**:

```text
proxy.reportSpike(...)
  → real.reportSpike(...)
       → this.storeOrReturnRepeat(...)   // same instance, no proxy
```

Facts that do **not** fix self-invocation (common false comforts):

| False comfort | What people mean | Why it still fails |
|---|---|---|
| The same-class helper method is package-visible or public (not `private`) | “Spring can proxy non-private methods, so the annotation must run.” | Proxy rules decide which methods *can* be advised when a call **enters the proxy**. `reportSpike` still calls `this.storeOrReturnRepeat(...)`. That call never enters the proxy. Visibility does not rewrite `this.` into a proxy call. |
| The same-class helper method does not call itself | “This is not recursion, so it is not self-invocation.” | “Does not call itself” only means `storeOrReturnRepeat` is not recursive. Self-invocation here means **another method on the same bean** (`reportSpike`) calls the same-class helper method on the **same instance**. Different words. Same miss. |
| The same-class helper method also has `@Transactional` | “The annotation sits on the write method, so a TX must start there.” | The annotation is advice on a **proxy entry**. No proxy entry → interceptor never runs → no TX. Putting the annotation on the same-class helper method does not change the call path. |

Only a call that enters a Spring bean proxy again starts that transaction. Visibility rules matter for proxies in some setups. They do not turn `this.storeOrReturnRepeat(...)` into a proxy call.

## Why the “smallest scope on `storeOrReturnRepeat`” answer was not 100% correct

The **scope** was right: the transaction belongs on the write path, not on all of `reportSpike`.

The **mechanism** was wrong: “not private” and “does not call itself” do not make `@Transactional` on `storeOrReturnRepeat` run when `reportSpike` calls it on the same object. Spring never sees that call. The interceptor never starts. So annotation-only on the same-class helper method is not a complete answer.

## Why the code in the tree is wrong

Jonathan’s intended boundary was already the thin one: `@Transactional` on `storeOrReturnRepeat` only. `reportSpike` was not meant to own a database transaction.

**Sam** added `@Transactional` on `reportSpike` later (training draft / IT shortcut) so a proxy entry would open a TX for lock + write. That addition is what widened the scope. Jonathan already rejects that width: search and the empty “no failed part” path must stay outside the store TX.

Shape in the tree after Sam’s change (simplified):

```java
@Transactional   // Sam added — not Jonathan’s intended boundary
public Object reportSpike(SpikeRequest request) {
	// search + empty return + write path
	return storeOrReturnRepeat(productName, search);
}

@Transactional   // Jonathan’s intended place — same-class call still skips proxy
SpikeResponse storeOrReturnRepeat(...) { ... }

@Transactional
void acquireProductNameLock(...) { ... }
```

Problems:

1. **Sam’s outer annotation widens the TX.** `@Transactional` on `reportSpike` starts a database transaction for search and for the empty path. Those steps do not need a connection or a lock. Delete that outer annotation; do not keep it as the way to “make the proxy work.”
2. **Nested annotations on the same class do not open a new proxy entry.** `storeOrReturnRepeat` and `acquireProductNameLock` run as `this.` calls. Their annotations do not start a transaction by themselves. They only join if an outer proxy TX already exists (today: Sam’s annotation on `reportSpike`).
3. **Removing Sam’s outer annotation without a proxy fix leaves no TX.** If the team deletes `@Transactional` from `reportSpike` and leaves it only on same-class `storeOrReturnRepeat`, the write path runs with **no** transaction. Lock, repeat check, and write then fail the design rule (same connection, `FOR UPDATE`, atomic store).

So: the tree “works” only because Sam wrapped the whole public method. That is the wrong boundary. Jonathan’s thin placement was right on scope and still needs a real proxy call (separate bean or `TransactionTemplate`) to run.

## Fixed shape — separate bean (proxy call)

Keep orchestration on `SpikeService` with no class-level store TX. Move lock + repeat + write to a second `@Service`. Injection gives a proxy. The call from `SpikeService` starts the transaction at the right edge.

```java
@Service
public class SpikeService {

	private final SpikeDataRetriever spikeDataRetriever;
	private final SpikeStoreWriter spikeStoreWriter;

	public Object reportSpike(SpikeRequest request) {
		String productName = request.getProductName();
		SpikeSearchResult search = spikeDataRetriever.search(productName);
		trimToMostRecent(search.failedManufacturingResults(), MAX_FAILED_RESULTS);

		if (search.failedManufacturingResults().isEmpty()) {
			return new NoFailedPartResponse(
					productName, "no failed part found under " + productName);
		}

		return spikeStoreWriter.storeOrReturnRepeat(productName, search);
	}
}
```

```java
@Service
public class SpikeStoreWriter {

	@Transactional
	public SpikeResponse storeOrReturnRepeat(String productName, SpikeSearchResult search) {
		acquireProductNameLock(productName);
		SpikeResponse repeated = findRepeatByIdSet(productName, search.failedManufacturingResults());
		if (repeated != null) {
			return repeated;
		}
		return storeNewSpike(productName, search);
	}

	// acquireProductNameLock, findRepeatByIdSet, storeNewSpike stay here (same TX)
}
```

Call graph after the fix:

```text
Controller → SpikeService (no store TX)
                → SpikeStoreWriter proxy → storeOrReturnRepeat (TX starts)
                     → lock + repeat check + write (same TX)
```

Alternative with the same boundary and no extra type: `TransactionTemplate.execute(...)` around the write block inside `reportSpike`. Prefer the separate bean when the write path owns repositories and the product-name lock.

## Incorrect idea — annotate `storeNewSpike` so it “joins”

Jonathan’s idea after the separate-bean fix:

> Put `@Transactional` on `storeNewSpike`, because we want the new transaction to join the existing transaction created by its caller `storeOrReturnRepeat`.

**What was right:** lock, repeat check, and write must share one transaction.

**What was wrong:** a second `@Transactional` on `storeNewSpike` is not how that share works.

`storeOrReturnRepeat` calls `this.storeNewSpike(...)`. That call never enters the `SpikeStoreWriter` proxy. Spring does not see the annotation on `storeNewSpike`. The method does not start a join. The annotation does nothing.

**Correct idea:** put `@Transactional` only on `storeOrReturnRepeat`, the proxy entry. Leave `acquireProductNameLock`, `findRepeatByIdSet`, and `storeNewSpike` without the annotation. Those helpers already run on the same thread after the interceptor opened the transaction. They use that connection. No nested annotation is required.

```text
SpikeService → SpikeStoreWriter proxy → storeOrReturnRepeat  (@Transactional — TX starts)
                     → this.acquireProductNameLock(...)     (no annotation — same TX)
                     → this.findRepeatByIdSet(...)           (no annotation — same TX)
                     → this.storeNewSpike(...)               (no annotation — same TX)
```

## When a transaction joins

Spring’s default propagation is `REQUIRED`.

| Situation on this thread | What `REQUIRED` does |
|---|---|
| No transaction is open | Start one. Commit or roll back when this proxy method returns. |
| A transaction is already open | Join it. Do not commit when this proxy method returns. |

A **join** needs a second proxy entry while a transaction is already open on the thread.

```text
BeanA proxy → methodA (@Transactional)     // TX starts
               → beanB.methodB (@Transactional REQUIRED)  // joins TX
```

Both calls must hit injected bean proxies. Then Spring sees the open transaction and joins it. The inner method does not commit on return. The outer method owns the commit.

That is not the same as a same-class helper:

| Call | What happens |
|---|---|
| `beanB.methodB(...)` through an injected proxy, with a TX already open | Real join. Inner `@Transactional REQUIRED` participates. |
| `this.storeNewSpike(...)` on the same class | No proxy. Annotation is ignored. The code still uses the open TX because it already sits on the thread. That is ambient use of the outer TX, not a Spring join. |
| Inner method is `REQUIRES_NEW` | Spring suspends the outer TX and starts a separate TX. |

CASE does not need a join today. One proxy entry on `SpikeStoreWriter.storeOrReturnRepeat` starts the store TX. The helpers stay on `this` with no annotation.

A-MES does use a join. `DepanelSplitWriter.store` starts T1. Inside that method, `StationGateService.complete` is called on another bean. `complete` is `@Transactional REQUIRED`, so it joins T1. The panel lock stays until `store` commits. Tech note 28 owns that plant rule.

## Product-name lock insert — do not catch duplicate key in the store TX

Decision 15 says: insert the lock row if missing; if two callers race, one wins and the other continues with `SELECT … FOR UPDATE`.

Wrong implementation: `saveAndFlush` plus catch `DataIntegrityViolationException` / MySQL `1062` inside `storeOrReturnRepeat`. Hibernate still marks that Spring transaction **rollback-only**. Catching the exception does not clear the flag. Commit then throws `UnexpectedRollbackException`. The concurrent IT fails.

Correct implementation: MySQL `INSERT IGNORE INTO spike_product_lock …` through `SpikeProductLockRepository.insertIgnore`. A clash inserts nothing and does not throw. The store transaction stays clean. Then run `SELECT … FOR UPDATE` on that row.

## What the team must not do

| Wrong | Right |
|---|---|
| Put `@Transactional` on all of `reportSpike` so the proxy “just works” | Keep the TX on lock + repeat + write only |
| Trust `@Transactional` on a same-class helper method called via `this` | Call a proxied bean or use `TransactionTemplate` |
| Put `@Transactional` on `storeNewSpike` so it “joins” the caller TX | Keep `@Transactional` only on `storeOrReturnRepeat`; helpers join by running inside that TX |
| Treat “public / not private” as proof the annotation runs | Treat “did the call enter a proxy?” as the proof |
| Open a TX for the empty “no failed part” path | Return that response with no store TX |

## Related

- Product-name lock and same-TX repeat/write: [02-api-store-the-spike.md](../design/02-api-store-the-spike.md) decisions 8 and 15.
- Concurrent MySQL for that check: [02-mysql-in-docker-for-store-tests.md](02-mysql-in-docker-for-store-tests.md).
- Lock pin vs business write (`INSERT IGNORE` only on the pin): [05-lock-pin-vs-business-write.md](05-lock-pin-vs-business-write.md).
- A-MES — Depanel split joins the panel lock (`REQUIRED` join, separate writer bean): [28-tech-note-depanel-split-transaction-joins-panel-lock.md](../../../../a-mes-docs/requirements/tech-notes/28-tech-note-depanel-split-transaction-joins-panel-lock.md).
- A-MES — Spring `@Transactional` proxy, self-call, rollback interview traps: [24-tech-note-spring-transactional-proxy-interview.md](../../../../a-mes-docs/requirements/tech-notes/24-tech-note-spring-transactional-proxy-interview.md).
- A-MES — Concurrency and `@Transactional` on station COMPLETE: [23-tech-note-concurrency-idempotency-transactional-station-complete.md](../../../../a-mes-docs/requirements/tech-notes/23-tech-note-concurrency-idempotency-transactional-station-complete.md).
