# Contract — store the spike

Issue: [02-the-operator-submits-the-product-and-the-eol-stations.md](../issues/02-the-operator-submits-the-product-and-the-eol-stations.md)

Tech note: [01-the-stored-spike-waits.md](../tech-notes/01-the-stored-spike-waits.md)

Tech note: [03-spike-failed-result-id-is-the-mes-result-id.md](../tech-notes/03-spike-failed-result-id-is-the-mes-result-id.md)

Words: [GLOSSARY.md](../../../GLOSSARY.md)

This file is the constraint for the API move and the UI move. A pull request that breaks this contract is rejected. This file is not a second issue.

## Contract

| Part | Contents |
|---|---|
| Endpoint | `POST /apis/spikes` |
| Request | Product name |
| Stored tables | Three tables. `spike`, `spike_failed_result`, and `spike_product_lock`. The field lists below. |
| Success response | The spike, its copied failed results, and `repeated`. A new store returns HTTP 201 with `repeated` set to false. A repeat returns HTTP 200 with `repeated` set to true. |
| No failed part response | When the search finds no failed part, the API returns HTTP 200. The response carries `product_name` and `message`. The message is `no failed part found under ` plus the product name. The API stores nothing. CASE does not write `spike`. CASE does not write `spike_failed_result`. |
| Validation response | When the product name is missing or blank, the API returns HTTP 400. The API stores nothing. |

CASE stores the spike in two business tables and one lock table. The first table is `spike`. That table holds the product facts. The second table is `spike_failed_result`. That table holds one row for each manufacturing result. A second test of the same serial is a second row. The third table is `spike_product_lock`. That table holds one row per product name. It is the concurrency pin. It is not a business record for the operator. CASE writes `spike` and `spike_failed_result` in one transaction. If either write fails, CASE stores nothing. The product-name lock lives in the same transaction. Before CASE persists, it compares the new failed manufacturing-result id set to spikes already stored for that product name. When that set is identical to the set on one existing spike, CASE treats the report as a repeat. CASE returns that existing spike with HTTP 200 and `repeated` set to true and stores nothing. A different `failed_test_items` value on those same ids is still a repeat. CASE does not store a second spike for that MES shape change. One new manufacturing-result id makes a new spike with HTTP 201 and `repeated` set to false. The check and the write run in the same transaction. `created_at` on `spike` is the time CASE stored the report. It does not decide the repeat. `repeated` is a response field. It is not a table column.

`spike` holds these fields.

| Field | What it tracks |
|---|---|
| Id | The database assigns this auto-increment id. The id has no business meaning. A later call loads this spike by this id. |
| Product name | The name the operator selects. This name is the search key. It is not the row id. The same product can be reported again. A repeat returns the existing spike. A new manufacturing-result id gets a new spike id. |
| Product code | The code the API finds from the product name. |
| working_orders | The working-order ids CASE found for this product when it stored the spike. This is a snapshot. A later change in MES does not rewrite this list. CASE stores the list as one JSON array column on `spike`. |
| description | The list sentence for this spike. CASE writes it when it stores the spike. The operator does not type it. The sentence gives the number of copied rows, then the test-item keys. CASE takes those keys only from a `failed_test_items` that is an object of pairs. The same key appears once. An abnormal field adds no keys. The number is at most 100. |
| failed_test_items_abnormal | One boolean on this spike. `true` when any copied `failed_test_items` is missing, null, an array, or a primitive. `false` when every copied `failed_test_items` is an object of pairs. CASE stores this on `spike`. The screen reads this field. MES engineers get involved when the value is `true`. |
| created_at | The time CASE stored this report. The list uses it to show which report is newer. It does not decide a repeat. |

The success response also carries `repeated`. That field is not stored on `spike`.

| Field    | What it tracks                                                                                                                 |
| -------- | ------------------------------------------------------------------------------------------------------------------------------ |
| repeated | True when CASE returned an existing spike with HTTP 200 and stored nothing. False when CASE stolred a new spike with HTTP 201. |

One station setup holds these fields. The MES project builds this setup on `ManufacturingResult`. This ticket uses the setup. This ticket does not build it.

| Field | What it tracks |
|---|---|
| Station id | The station that ran the test. This project stores an EOL tester id. A result from any other station is not copied. |
| Rack id | The rack on that station for this setup. Required, because the station is an EOL tester. |
| Fixture id | The fixture that held the DUT for this setup. Required, because the station is an EOL tester. |
| Nest number | The nest that held the DUT for this setup. Required, because the station is an EOL tester. |

`spike_failed_result` holds these fields. The MES project builds `ManufacturingResult`. CASE copies each failed EOL test result into `spike_failed_result` when it stores the spike. A result from any other station is not copied. A part that fails once at any EOL tester is included. A later pass on that part does not remove the failed EOL test result. CASE limits failed EOL test results to 100 in two layers. The data-retrieving layer queries at most the most recent 100 by result `created_at`. The service layer trims to the most recent 100 when the repository returns more than 100. CASE then copies that list. The store path does not hold a longer failed-result list for that call. The copy does not point at the live MES row. A later change in MES does not change this copy. The table stores `station_id`, `rack_id`, `fixture_id`, and `nest_number` as columns. The response returns those four fields. MES does not use `controller_id`. `description` is the list sentence on `spike`. CASE writes it when it stores the spike. The operator does not type it. The sentence gives the number of copied rows for that `spike_id`, then the test-item keys. CASE takes those keys only from a `failed_test_items` that is an object of pairs. The same key appears once. An abnormal field adds no keys. The number is at most 100.

| Field | What it tracks |
|---|---|
| spike_id | The spike that owns this copy. This key points at the spike id. |
| id | The id of the manufacturing result. CASE copies this id. The manufacturing-result id alone is not the key of this table. `spike_id` and this id together identify the copied row. |
| serial_number | The serial of the failed part. |
| product_type | The product type on that result. |
| mo_id | The manufacturing order id. |
| op_code | The process path op for this failed result. Examples: `COLD_EOL`, `HOT_EOL`, `AMBIENT_EOL`. |
| station_id | The tester that ran the test. The response returns this field. |
| rack_id | The rack on that tester for this test. The response returns this field. |
| fixture_id | The fixture that held the DUT for this test. The response returns this field. |
| nest_number | The nest that held the DUT for this test. The response returns this field. |
| overall_result | The pass or fail result. |
| cycle_time_seconds | The cycle time in seconds. |
| sw_version | The software version. |
| hw_revision | The hardware revision. |
| failed_test_items | One JSON column on `spike_failed_result`. A present value keeps the JSON type MES sent: object, array, or primitive. A missing field is stored as null. The failed test items for this one DUT. Each entry is one pair. The key is the test item. The value is the test value. A passing item stays out. The same test item can appear on more than one failed result. When the field is missing, or the whole field is null, an array, or a primitive, the field is abnormal. CASE still stores the row and reports the spike. CASE copies a present abnormal value as MES sent it. When the field is missing, CASE stores null. That record adds no keys to `description`. CASE logs the abnormal. |
| error_code | The error code. |
| operator_id | The operator. |
| shift_code | The shift. |
| created_at | The time of the result. |

`spike_product_lock` holds these fields. One row per product name. Many spikes for the same product share this one lock row.

| Field | What it tracks |
|---|---|
| product_name | The product name the operator posts. This is the primary key. |
| created_at | The time CASE first inserted this lock row. |

## Keys and locks

| Table | Key | Rule |
|---|---|---|
| `spike` | Primary key | `id`. The database assigns this auto-increment id. |
| `spike` | Lookup | `product_name`. CASE finds existing spikes by this name for the repeat check. |
| `spike_failed_result` | Primary key | `spike_id` and `id` together. One manufacturing-result id appears once under one spike. |
| `spike_failed_result` | Foreign key | `spike_id` points at `spike.id`. The database enforces this foreign key. When CASE deletes a `spike` row, the database deletes the `spike_failed_result` rows that point at that spike. |
| `spike_product_lock` | Primary key | `product_name`. One lock row per product name. No foreign key to `spike`. |

`spike_failed_result.id` is a copied manufacturing-result id. It is not a foreign key to the live MES `ManufacturingResult` row. `station_id`, `rack_id`, `fixture_id`, and `nest_number` are copied values. They are not foreign keys to the equipment master.

For the repeat check and the write, CASE uses `spike_product_lock` as the pin. In the same transaction CASE inserts the `product_name` row if it is missing, then runs `SELECT … FOR UPDATE` on that row, then runs the repeat check and the write. A second call for the same product name waits on that same row. This works when no `spike` row exists yet. If two first callers both insert the same `product_name`, one wins. The other hits the primary-key clash. That caller does not fail the request. It runs `SELECT … FOR UPDATE` on the row the winner wrote, then continues. The code recovers only on a duplicate-key error for that insert. In MySQL that is error `1062`. In Spring that is usually `DuplicateKeyException`, or a `DataIntegrityViolationException` whose cause is that MySQL error. Any other database error fails the request. CASE rolls back. CASE does not catch a bare exception and treat it as a second insert. CASE does not lock by selecting every `spike` row for that product name.

The station type still has an expected equipment shape. An EOL tester is expected to carry rack id, fixture id, and nest number. A later UV Inspection Machine is expected to carry the station id only. CASE uses that shape to spot gaps. CASE does not refuse the spike when a field is missing. CASE stores the copy with nulls where MES left gaps. CASE writes a log line for the MES team. The hard refuse for incomplete equipment facts belongs on the MES write of `ManufacturingResult`, not on the CASE report.

`failed_test_items` is one JSON column on `spike_failed_result`. On one failed record the value is pairs. The key is the test item. The value is the test value. When the field is missing, or the whole field is null, an array, or a primitive, that field is abnormal. A primitive is a string, a number, or a boolean. CASE still stores the failed record and reports the spike. CASE copies a present abnormal value as MES sent it. When the field is missing, CASE stores null. `description` skips the test items from that record. CASE logs the abnormal. The log names the product name, the manufacturing-result id, and that `failed_test_items` is missing or not pairs. The MES gap stays in the log. An empty object has no pairs and is not abnormal. A value under a test-item key can be nested. That field is still pairs. CASE stores one boolean `failed_test_items_abnormal` on `spike`. The value is `true` when any copied `failed_test_items` is missing, null, an array, or a primitive. The value is `false` when every copied field is an object of pairs. CASE does not store this flag on each failed-result row. The screen reads `failed_test_items_abnormal`. When the value is `true`, the screen shows `Test data is abnormal. A later cause list can be wrong. MES engineers need to get involved.` The `failed_test_item` cell shows keys only from an object of pairs.

This call does not read the lookup set. This call does not call the model.

## Example

The JSON keys are the API names for the fields above. The values show the shape. The values are not plant data.

The request body holds the product name. The product name is the only key. The operator does not send a failed count or a window size.

```json
{
  "product_name": "Ford APCU-CD4"
}
```

The success response returns the spike, the copied failed results, and `repeated`. A new store uses HTTP 201. A repeat uses HTTP 200. The example below is a new store. Each failed result holds `spike_id`, `station_id`, `rack_id`, `fixture_id`, and `nest_number`. Each entry in `working_orders` is the id of one working order. `failed_test_items` holds the failed test items for that one DUT. The key is the test item. The value is the test value. A passing item stays out. One test item has one test value. This contract does not freeze the test-item names. When the field is missing, or the whole field is null, an array, or a primitive, CASE still reports the spike. CASE copies a present abnormal value as MES sent it. When the field is missing, CASE stores null. CASE skips that record's test items in `description` and logs the abnormal. CASE stores `failed_test_items_abnormal` on the spike. Any abnormal field sets that boolean to `true`. The example below is `false`.

```json
{
  "id": 1,
  "product_name": "Ford APCU-CD4",
  "product_code": "Ford APCU-01",
  "working_orders": ["WO-DEMO-001", "WO-DEMO-002"],
  "description": "1 unit failed on EOL test, failed test items: switch-open,switch-close.",
  "failed_test_items_abnormal": false,
  "created_at": "2026-11-06T12:05:00Z",
  "repeated": false,
  "failed_manufacturing_results": [
    {
      "spike_id": 1,
      "id": 1,
      "serial_number": "APCU-CD4-PCBA-001",
      "product_type": "Ford-APCU",
      "mo_id": "WO-DEMO-001",
      "op_code": "COLD_EOL",
      "station_id": "AEL-01-EOL-01",
      "rack_id": "RACK-APCU-G1-01",
      "fixture_id": "FIXTURE-APCU-G2-01",
      "nest_number": 1,
      "overall_result": "FAIL",
      "cycle_time_seconds": 30,
      "sw_version": "sw-1.0",
      "hw_revision": "hw-2.0",
      "failed_test_items": {
        "switch-open": "0010",
        "switch-close": "0001"
      },
      "error_code": "switch-error",
      "operator_id": "Alice",
      "shift_code": "day-shift",
      "created_at": "2026-11-06T12:00:00Z"
    }
  ]
}
```

When the search finds no failed part, the API returns HTTP 200. The response carries the product name and the message. The API stores nothing. CASE does not write `spike`. CASE does not write `spike_failed_result`.

```json
{
  "product_name": "Ford APCU-CD4",
  "message": "no failed part found under Ford APCU-CD4"
}
```

This contract stores the ids. It does not define how a rack and a fixture attach. It does not define how many nests a fixture has. Those tables belong to the MES equipment master. Issue: [01-the-equipment-master-tracks-the-rack-the-fixture-and-the-nests.md](../../mes-equipment-master/issues/01-the-equipment-master-tracks-the-rack-the-fixture-and-the-nests.md).

The failed EOL test results come from the MES `ManufacturingResult` table. MES is still building that table. CASE limits failed EOL test results to 100 in two layers. The data-retrieving layer queries at most the most recent 100. The service layer trims to 100 when the repository returns more than 100. CASE then copies that list into `spike_failed_result`. A later pass on that part does not remove the failed EOL test result. The copy does not point at the live MES row. The table stores `station_id`, `rack_id`, `fixture_id`, and `nest_number` as columns. The response returns those four fields. Temporary note, to move to the MES project: [02-temporary-note-manufacturing-result-keeps-the-equipment-ids.md](../../../notes/02-temporary-note-manufacturing-result-keeps-the-equipment-ids.md).

## Out of scope

The equipment master. The rack and fixture assignment. The nest count on a fixture. Building the `ManufacturingResult` table. Refusing incomplete equipment fields on MES write. The lookup set. The model call. The score. The resolved column. Resolved means the field engineers confirmed the cause, took the fix, and watched manufacturing until no failed part remained. That fact comes after the score.
