# Tech note — DB lock: same process vs two instances

**Type:** Tech note  
**Project:** CASE — Scored EOL cause list  
**Date:** 2026-11-17  
**Decided by:** Jonathan, with Vector Elon and Kai  
**Source:** Copilot review note [09-copilot-review-on-issue2-backend-code-change.md](../../../notes/09-copilot-review-on-issue2-backend-code-change.md)

Contract lock: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md)  
Lock pin detail: [05-lock-pin-vs-business-write.md](05-lock-pin-vs-business-write.md)  
IT: `SpikeServiceIT.reportSpike_twoConcurrentSameIdSet_oneNewOneRepeated`

## Decision

1. The product-name pin is a **MySQL row lock** (`INSERT IGNORE` then `SELECT … FOR UPDATE` on `spike_product_lock`). It is not a Java `synchronized` block.
2. Two threads in **one** `case-api` process and two **separate** `case-api` instances use the **same** lock mechanism: they wait on that database row.
3. The concurrent IT in one process is enough proof for issue 02. A second-process / multi-pod IT is optional later, not a design fix.

## Same process (what the IT covers)

One Spring Boot JVM. Two threads call report at the same time. They share one process and one connection pool. Serialization still comes from MySQL holding `FOR UPDATE` on the lock row until the store transaction commits or rolls back.

## Two instances (what Copilot asked about)

Two running apps (two pods, two machines). Each has its own JVM and pool. Both still talk to the same MySQL. Instance A holds the row lock; instance B blocks on that row until A finishes the transaction.

The lock design does not change. Only the deploy shape changes.

## What the team must not do

- Do not add an in-process-only lock and call it the concurrency pin.
- Do not treat “IT is single-process” as proof that the DB lock is wrong for multi-instance.
- Do not block the store PR on a cross-instance IT unless deploy acceptance explicitly requires that demo.

## When a cross-instance test is worth it

Add it only if the team must show two live replicas against one MySQL for a ship or ops bar. That test proves topology. It does not invent a new lock rule.
