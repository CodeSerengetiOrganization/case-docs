# API stores the spike

Issue: [02-the-operator-submits-the-product-and-the-eol-stations.md](../issues/02-the-operator-submits-the-product-and-the-eol-stations.md)

Tech note: [01-the-stored-spike-waits.md](../tech-notes/01-the-stored-spike-waits.md)

Tech note: [02-mysql-in-docker-for-store-tests.md](../tech-notes/02-mysql-in-docker-for-store-tests.md)

Tech note: [03-spike-failed-result-id-is-the-mes-result-id.md](../tech-notes/03-spike-failed-result-id-is-the-mes-result-id.md)

Contract: [02-contract-store-the-spike.md](02-contract-store-the-spike.md)

## Problem

The operator reports an EOL spike. The API accepts the product name, finds the facts in the contract, and stores the spike in the tables in the contract. This call does not call the model.

## Decisions

1. The request carries the product name. The product name is the only search key. The API validates the request. When the product name is missing or blank, the API returns HTTP 400. The API stores nothing.
2. The service layer returns the search result. This issue accepts a mock for that return. The API then stores the spike in `spike` and `spike_failed_result`, and uses `spike_product_lock` for the product-name lock.
3. `spike` holds the product facts. `spike_failed_result` holds the copied failed EOL test results. The spike id is how a later call loads this spike. The product name is not the row id. `spike` stores `description`. CASE writes that sentence when it stores the spike. The operator does not type it. The sentence gives the number of copied rows, then the test-item keys. CASE takes those keys only from a `failed_test_items` that is an object of pairs. The same key appears once. An abnormal field adds no keys. `spike_failed_result` stores `station_id`, `rack_id`, `fixture_id`, and `nest_number` as columns. The response returns those four fields.
4. A success response returns the spike, its copied failed results, and `repeated`. `repeated` is not a table column. A new store returns HTTP 201 and sets `repeated` to false. A repeat returns HTTP 200 and sets `repeated` to true. The screen uses that flag to tell the operator.
5. When the search finds no failed part, the API returns HTTP 200 with `message` and `product_name`. The message is `no failed part found under ` plus the product name. The API stores nothing. CASE does not write `spike`. CASE does not write `spike_failed_result`.
6. This call does not read the lookup set. This call does not call the model.
7. CASE writes `spike` and `spike_failed_result` in one transaction. If either write fails, CASE stores nothing. The product-name lock for that call lives in the same transaction.
8. `spike` stores `created_at`. That is the time CASE stored the report. It does not decide a repeat. Before CASE persists, it compares the new failed manufacturing-result id set to spikes already stored for that product name. When that set is identical to the set on one existing spike, CASE treats the report as a repeat. CASE returns that existing spike with HTTP 200 and `repeated` set to true and stores nothing. A different `failed_test_items` value on those same ids is still a repeat. CASE does not store a second spike for that MES shape change. One new manufacturing-result id makes a new spike with HTTP 201 and `repeated` set to false. The check and the write run in the same transaction. CASE locks by product name through `spike_product_lock`, not by locking every `spike` row.
9. CASE builds this API in Spring Boot under the CASE app repo path `services/`. That folder holds backend services. This store API lives in `services/case-api`. The package map has three layers: controller, service, and repository. The controller accepts `POST /apis/spikes`. The service runs validation, the search mock, the product-name lock, the repeat check, the 100 trim, the description, `failed_test_items_abnormal`, and the transaction. The repository reads and writes `spike`, `spike_failed_result`, and `spike_product_lock`. The data-retrieving layer sits in the service layer. For this issue it is a mock that returns product code, working orders, and failed EOL test results.
10. CASE limits failed EOL test results to 100 in two layers. The data-retrieving layer queries at most the most recent 100 by result `created_at`. The service layer trims to the most recent 100 when the repository returns more than 100. CASE then writes `description`, writes the tables, and builds the response from that trimmed list. The store path does not hold a longer failed-result list for that call. The production mock for this issue returns at most 100. The check that proves the service trim uses another mock repository that returns more than 100. That mock is for the test only.
11. Table keys live in the contract. `spike.id` is the primary key. `spike_failed_result` uses primary key `spike_id` plus `id`. `spike_failed_result.spike_id` is a foreign key to `spike.id`. When CASE deletes a `spike` row, the database deletes the matching `spike_failed_result` rows. The copied manufacturing-result id is not a foreign key to live MES. The equipment ids are not foreign keys to the equipment master. `spike_product_lock.product_name` is the primary key. That table has no foreign key to `spike`.
12. CASE uses MySQL `8.0.21` for this store. That version matches A-MES. Hand diagnose uses Compose at `docker/mysql/docker-compose.yml`. Store tests use Testcontainers with the same MySQL version. Tech note: [02-mysql-in-docker-for-store-tests.md](../tech-notes/02-mysql-in-docker-for-store-tests.md).
13. CASE uses Flyway to create and migrate the `spike`, `spike_failed_result`, and `spike_product_lock` tables. Migration scripts live in `services/case-api/src/main/resources/db/migration`. The table shape follows the contract. This design does not freeze every SQL column type yet. `spike.working_orders` is one JSON array column. It holds the working-order ids found when CASE stored the spike. It is a snapshot. A later MES change does not rewrite that column. `spike_failed_result.failed_test_items` is one JSON column. A present value keeps the JSON type MES sent: object, array, or primitive. A missing field is stored as null. `failed_test_items_abnormal` reads that JSON type.
14. The response field and the column name are `working_orders`. Do not use `active_working_orders`. At search time the service may find orders that are active then. The stored list does not claim those orders stay active later.
15. `spike_product_lock` is the concurrency pin. It is not a business table for the operator. One row per product name. Columns: `product_name` (primary key) and `created_at` (when CASE first inserted that lock row). In the store transaction CASE inserts the product-name row if it is missing, then runs `SELECT … FOR UPDATE` on that row, then runs the repeat check and the write. A second call for the same product name waits on that same row, including the first report when no `spike` row exists yet. If two first callers both insert the same `product_name`, one wins. The other continues. CASE uses MySQL `INSERT IGNORE` for that insert so a primary-key clash inserts nothing and does not throw. Catching `DataIntegrityViolationException` / MySQL `1062` inside the store transaction marks the transaction rollback-only and fails the request on commit. After the insert, both callers run `SELECT … FOR UPDATE` on that row. Any other database error fails the request. CASE rolls back.
16. CASE does not block Report spike when MES failed-result fields are missing. Example: an EOL fail with no nest number still stores. CASE does not invent the missing value. CASE writes a log line that names the product name, the manufacturing-result id, the station id, and the missing fields. That log is for the MES team. CASE and MES share those gaps so both teams improve faster. The hard refuse for incomplete equipment facts belongs on the MES write of `ManufacturingResult`, not on this CASE store call.
17. `failed_test_items` is one JSON column on `spike_failed_result`. On one failed record the value is pairs. The key is the test item. The value is the test value. When the field is missing, or the whole field is null, an array, or a primitive, that field is abnormal. A primitive is a string, a number, or a boolean. CASE still stores the failed record and reports the spike. CASE copies a present abnormal value as MES sent it. When the field is missing, CASE stores null. `description` skips the test items from that record. CASE logs the abnormal. The log names the product name, the manufacturing-result id, and that `failed_test_items` is missing or not pairs. The MES gap stays in the log. An empty object has no pairs and is not abnormal. A value under a test-item key can be nested. That field is still pairs. CASE stores one boolean `failed_test_items_abnormal` on `spike`. The value is `true` when any copied `failed_test_items` is missing, null, an array, or a primitive. The value is `false` when every copied field is an object of pairs. CASE does not store this flag on each failed-result row. One abnormal row sets `failed_test_items_abnormal` to `true`. The screen reads `failed_test_items_abnormal`. When the value is `true`, MES engineers get involved. The later agent reads this stored boolean. This call does not call the model.

## Example rows

These rows show the stored shape. The response returns the same four fields. The values are not plant data.

`spike`

| id | product_name | product_code | working_orders | description | failed_test_items_abnormal | created_at |
|---|---|---|---|---|---|---|---|
| 1 | Ford APCU-CD4 | Ford APCU-01 | ["WO-DEMO-001", "WO-DEMO-002"] | 1 unit failed on EOL test, failed test items: switch-open,switch-close. | false | 2026-11-06T12:05:00Z |

`spike_failed_result`

| Column | Example |
|---|---|
| spike_id | 1 |
| id | 1 |
| serial_number | APCU-CD4-PCBA-001 |
| product_type | Ford-APCU |
| mo_id | WO-DEMO-001 |
| op_code | COLD_EOL |
| station_id | AEL-01-EOL-01 |
| rack_id | RACK-APCU-G1-01 |
| fixture_id | FIXTURE-APCU-G2-01 |
| nest_number | 1 |
| overall_result | FAIL |
| cycle_time_seconds | 30 |
| sw_version | sw-1.0 |
| hw_revision | hw-2.0 |
| failed_test_items | { "switch-open": "0010", "switch-close": "0001" } |
| error_code | switch-error |
| operator_id | Alice |
| shift_code | day-shift |
| created_at | 2026-11-06T12:00:00Z |

`spike.id` and `spike_failed_result.spike_id` are the same id. `spike_failed_result.id` is the copied manufacturing-result id. The two columns together identify that row.

## Out of scope

The lookup set. The model call. The ranked cause list. The true cause. The score. The resolved column. Resolved means the field engineers confirmed the cause, took the fix, and watched manufacturing until no failed part remained. That fact comes after the score. The UI controls. The request and response shape. That shape lives in the contract file. Refusing a spike because MES left equipment fields empty. That refuse belongs in MES. CASE only logs the gap.

## Store call order

The store call runs these steps in this order. Sam must not invent another order.

1. Validate the request. When the product name is missing or blank, return HTTP 400. Store nothing.
2. Run the search through the data-retrieving mock. Find the product code, the working orders, and the failed EOL test results. Apply the search rules below. Trim to at most 100 when needed.
3. When the search finds no failed part, return HTTP 200 with `message` and `product_name`. Store nothing. Do not take the product-name lock.
4. Take `spike_product_lock` for that product name. Insert the lock row if it is missing. On duplicate key, recover as in decision 15. Run `SELECT … FOR UPDATE` on that row.
5. Run the repeat check against spikes already stored for that product name.
6. When the failed manufacturing-result id set is identical to one existing spike, return HTTP 200 with that spike and `repeated` set to true. Store nothing new. A different `failed_test_items` value on those same ids stays on this step.
7. When the set is new, write `spike` and `spike_failed_result` in the same transaction. Write `description`. When a copied failed result is missing expected equipment fields for that station type, store the row with nulls and write a log line for the MES team. When `failed_test_items` on a copied row is missing, null, an array, or a primitive, still store the spike. Store a present abnormal value as MES sent it. When the field is missing, store null. Skip that row's test items in `description`. Log the abnormal as in decision 17. Set `spike.failed_test_items_abnormal` to `true` when any copied field is abnormal. Set it to `false` when every copied field is an object of pairs. Return HTTP 201 with `repeated` set to false.

The search runs before the lock. The lock runs before the repeat check and the write.

## Search rule

These rules apply inside store-call step 2.

1. The API uses the product name as the key and finds the product code.
2. The API uses the product code as the key and finds the working orders to search. At search time those orders are the active set the service returns. CASE then stores that list as `working_orders` when it writes the spike.
3. The API searches failed EOL test results on every working order in that list. A result from any other station stays out of this search. The data-retrieving layer queries at most the most recent 100 by result `created_at`.
4. A part that fails once at any EOL tester is included. A later pass on that part does not remove the failed EOL test result.
5. The service layer trims the failed-result list to at most the most recent 100 by result `created_at` when the repository returns more than 100. CASE then copies that list into `spike_failed_result` when it writes. The store path does not hold a longer failed-result list for that call.
6. When the search finds no failed part, the store call returns at step 3 above. The API returns HTTP 200 with `message` and `product_name`. The API stores nothing. CASE does not write `spike`. CASE does not write `spike_failed_result`.

## Service return

The service layer returns the product code, the working orders, and the failed EOL test results. This issue accepts a mock for that return. The real MES tables and the real MES APIs stay parked.

The mock stands in for the data-retrieving layer. The mock does not skip the store rules. The API still copies failed EOL test results only. A later pass does not remove a failed EOL test result. The production mock returns at most 100 failed results for this issue. The service layer still trims to 100 when a repository return has more than 100. To prove that trim, the test uses another mock repository that returns more than 100. That mock is for the test only. When the return has no failed part, the API stores nothing.

## Open choices

Write a choice here before you add a check to the issue.

- Where the service finds the product code. Design this when the MES source exists.
- Where the service finds the working orders. Design this when the MES source exists.

## Closed choices

- Where the service finds the failed EOL test results. For this issue, the service returns mock data. The real MES source stays parked.
- When `failed_test_items` is missing or not pairs. CASE reports the spike, copies a present abnormal value as MES sent it, stores null when the field is missing, skips that record's test items in `description`, and logs the abnormal. `spike.failed_test_items_abnormal` is `true` when any copied field is abnormal. Decision 17.
