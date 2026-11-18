# Tech note — `spike_failed_result.id` is the MES result id

**Type:** Tech note  
**Project:** CASE — Scored EOL cause list  
**Date:** 2026-11-14  
**Decided by:** Jonathan, with Vector Elon

Contract: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md)  
Migration: `case/services/case-api/src/main/resources/db/migration/V1__create_spike_tables.sql`

## Decision

1. `spike.id` is the only auto-increment id on this store path. The database assigns it when CASE stores a new spike.
2. `spike_failed_result.id` is the manufacturing-result id from MES. CASE copies that value. The column is not auto-increment.
3. The primary key of `spike_failed_result` is the pair `(spike_id, id)`. One manufacturing-result id appears once under one spike.
4. The repeat check compares the set of those copied manufacturing-result ids. It does not invent a CASE-side row number.

## Why a re-test still works

A re-test can keep the same `serial_number`. MES still assigns a new manufacturing-result id for that second test. CASE copies the new id. The two rows stay distinct.

If MES reused the same manufacturing-result id for a second test, that is a MES bug. CASE does not add a second auto-increment id to hide that bug.

## What the team must not do

| Wrong | Right |
|---|---|
| Make `spike_failed_result.id` `AUTO_INCREMENT` | Keep the copied MES result id |
| Treat `serial_number` as the event key | Treat the manufacturing-result id as the event key |
| Drop the MES id and invent a local surrogate only | If a local surrogate is ever needed later, keep the MES result id as its own column |

## Related

- Repeat rule: the manufacturing-result id set. Design: [02-api-store-the-spike.md](../design/02-api-store-the-spike.md) decision 8.
- Stored spike waits: [01-the-stored-spike-waits.md](01-the-stored-spike-waits.md).
