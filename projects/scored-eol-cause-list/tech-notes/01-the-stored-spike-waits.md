# Tech note — The stored spike waits

**Type:** Tech note
**Project:** CASE — Scored EOL cause list
**Date:** 2026-11-05
**Decided by:** Jonathan, with Vector Elon

Report spike stores the spike. That action does not call the model.

The stored spike waits. Hours later, the agent issue loads that same record and calls the model. The operator does not submit the product again. The backend does not search again.

The product name is the search key for this spike. The API takes that name, finds the facts in the contract, and stores the spike in two tables.

The product name is not the row id. The same product can be reported again later. Each report gets its own id. Issue 04 loads the spike by that id.

The call reads two inputs.

1. The stored spike. The field list is in [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md).
2. The lookup set: the other solved-case reports and the maintenance logs.

The lookup set is a separate store. The API does not copy that set onto the spike row.

CASE stores the spike in two tables. `spike` holds an id, the product name, the product code, `working_orders`, `description`, and `created_at`. `working_orders` is a snapshot of the working-order ids found when CASE stored the spike. `created_at` is the time CASE stored the report. It does not decide a repeat. A second report for the same product is a repeat when every manufacturing-result id in it is already on one spike for that product. CASE returns that spike and stores nothing. One new manufacturing-result id makes a new spike. CASE writes `description` when it stores the spike. The operator does not type it. The sentence gives the number of copied rows, then the distinct failed test items. The second table is `spike_failed_result`. CASE writes both tables in one transaction. If either write fails, CASE stores nothing. That table holds the copied failed manufacturing results. `spike_id` points at the spike id. CASE copies at most 100 failed EOL test results when it stores the spike. A result from any other station is not copied. A later pass on that part does not remove the failed EOL test result. When more than 100 failed EOL test results exist, CASE copies the most recent 100. The copy does not point at the live MES row. The table stores `station_id`, `rack_id`, `fixture_id`, and `nest_number` as columns. The response returns those four fields. This project accepts only an EOL tester id, so the rack id, the fixture id, and the nest number are required. The ranked cause list stays off these tables. The true cause stays off these tables. Humidity stays off these tables until a walk finds a humidity reading. The resolved column stays off these tables. Resolved means the field engineers confirmed the cause, took the fix, and watched manufacturing until no failed part remained. That fact comes after the score. A second model call stops when a model call already exists for that spike id. Issue 04 owns that fact.

Issue 02 stores the spike. Issue 03 prepares the lookup set. Issue 04 loads the stored spike, reads the lookup set, and calls the model. Issue 05 scores that call.
