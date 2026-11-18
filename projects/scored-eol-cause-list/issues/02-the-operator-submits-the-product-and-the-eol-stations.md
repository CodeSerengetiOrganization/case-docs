# The operator submits the product and the EOL stations

The same API does both steps. This issue is step 1. The API accepts the spike and stores it. Step 2 is "The agent returns a ranked cause list." That issue is where this API calls the model.

Do these moves in order. They stay in this issue. Each move can be its own pull request.

1. The API accepts the product name. The product name is the only key. This issue accepts a mock service return for the product code, the working orders, and the failed EOL test results. The store rules in the API design still apply. The search copies at most 100 failed EOL test results on those working orders. A later pass does not remove a failed EOL test result. When more than 100 exist, it copies the most recent 100. When the search finds no failed part, the API returns the product name and `no failed part found under ` plus that name, and stores nothing. Design: [02-api-store-the-spike.md](../design/02-api-store-the-spike.md). Contract: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md).
2. The UI shows a hardcoded product dropdown. The operator selects one product name and clicks Report spike. The operator does not pick the EOL stations. Design: [02-ui-submit-the-spike.md](../design/02-ui-submit-the-spike.md). Contract: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md).

Close this issue when a submit from the UI is stored.

The operator selects the product name from a dropdown. The screen holds that product name.

The names in that dropdown are hardcoded for this issue.

The screen does not take an EOL station choice from the operator. A backend service searches for the stations.

The operator clicks Report spike. The screen sends the selected product name to `POST /apis/spikes`. The product name is the only key.

When the search finds no failed part, the screen shows `no failed part found under ` plus the product name.

After the API stores a new spike, the screen shows that the spike is reported and stored. When the API returns a repeated spike, the screen shows that this spike was already reported.

When the API returns an error, the screen shows that error message.

Add another check only after the design file states the next behavior. Do not invent the check first.

The design files and the contract file have a skeleton. A long write-up stays in the design file. A design file is not a second issue. Finish the contract before you build the API or the UI. After the contract is filled, those two builds can run in parallel.

This issue can start with "Learn the model call."

The stored spike waits for the agent. Tech note: [01-the-stored-spike-waits.md](../tech-notes/01-the-stored-spike-waits.md).

## Acceptance criteria

Each move maps to one group. An API pull request maps the API checks. A UI pull request maps the UI checks. Before merge, the developer maps each check in that pull request to a test.

### API

Design: [02-api-store-the-spike.md](../design/02-api-store-the-spike.md). Contract: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md).

Add each API check here after the design file states the behavior.

1. **Do:** The caller posts a product name `Ford-APCU-HasFailedRecord-001` to `POST /apis/spikes`. The service returns failed manufacturing results. No spike for that failed-result id set exists yet.
   **Result:** The API returns HTTP 201. The body matches the contract success shape. `repeated` is false. CASE stores one `spike` row and one `spike_failed_result` row per returned manufacturing result.
2. **Do:** The caller posts a product name `Ford-APCU-AllPass-001` to `POST /apis/spikes`. The service returns no failed manufacturing result.
   **Result:** The API returns HTTP 200. The body carries `product_name` and `message`. The message is `no failed part found under ` plus the product name. CASE stores nothing. `spike` has no new row. `spike_failed_result` has no new row.
3. **Do:** The store path receives a search result with failed EOL test results and a null `product_code`. This null `product_code` is for the test only. It is not a production mock path. No spike for that failed-result id set exists yet.
   **Result:** The store write fails. CASE stores nothing. `spike` has no new row. `spike_failed_result` has no new row. `spike_product_lock` has no new row for that product name.
4. **Do:** The caller posts to `POST /apis/spikes` with no product name, or with a blank product name.
   **Result:** The API returns HTTP 400. CASE stores nothing. `spike` has no new row. `spike_failed_result` has no new row.
5. **Do:** The caller posts a product name to `POST /apis/spikes`. More than 100 failed EOL test results exist for that product. The data-retrieving layer returns the search result.
   **Result:** The data-retrieving layer returns at most 100 failed manufacturing results, the most recent by result `created_at`. The list CASE uses before it writes has at most 100. The response body has at most 100 failed manufacturing results. `spike_failed_result` holds at most 100 rows for that spike.
6. **Do:** The caller posts a product name to `POST /apis/spikes`. A test-only mock repository returns more than 100 failed manufacturing results. This is not the production mock path.
   **Result:** The service layer trims the list to 100, the most recent by result `created_at`. The list CASE uses before it writes has 100. The response body has 100 failed manufacturing results. `spike_failed_result` holds 100 rows for that spike.
7. **Do:** A spike already exists for product name `Ford-APCU-HasFailedRecord-001`. The caller posts that same product name to `POST /apis/spikes`. The failed manufacturing-result id set in the new report is identical to the set on that existing spike.
   **Result:** CASE returns HTTP 200 and that existing spike with `repeated` set to true. CASE stores nothing. `spike` has no new row. `spike_failed_result` has no new row.
8. **Do:** A spike already exists for a product name with one failed manufacturing-result id set. The caller posts that same product name to `POST /apis/spikes`. The new report includes at least one manufacturing-result id that is not on that existing spike.
   **Result:** The API returns HTTP 201 and a new spike with `repeated` set to false. CASE stores a new `spike` row. CASE does not treat the call as a repeat of the existing spike.
9. **Do:** Two callers post the same product name to `POST /apis/spikes` at the same time. Both reports carry the same failed manufacturing-result id set. No spike for that set exists yet. The first call holds the product-name lock before it commits. The second call runs while that lock is held.
   **Result:** The second call waits until the first call ends. CASE stores one spike for that id set. One response is HTTP 201 with `repeated` set to false. The other response is HTTP 200 with that same spike and `repeated` set to true. `spike` has one new row for that set. `spike_failed_result` is not doubled for that set.
10. **Do:** The caller posts a product name to `POST /apis/spikes`. The service returns a failed EOL manufacturing result that is missing an expected equipment field for that station type (example: nest number missing on an EOL tester). No spike for that failed-result id set exists yet.
   **Result:** The API returns HTTP 201 with `repeated` set to false. CASE stores the `spike` and the `spike_failed_result` row. The missing equipment field is stored as null. CASE does not invent a value. CASE writes a log line that names the product name, the manufacturing-result id, the station id, and the missing fields. CASE does not refuse the call for that gap.
11. **Do:** The caller posts a product name to `POST /apis/spikes`. The service returns a failed EOL manufacturing result whose `failed_test_items` is missing, or the whole field is null, an array, or a primitive. No spike for that failed-result id set exists yet.
   **Result:** The API returns HTTP 201 with `repeated` set to false. CASE stores the `spike` and the `spike_failed_result` row. CASE reports the spike. A present abnormal value is stored as MES sent it. A missing field is stored as null. `description` takes no test items from that row. `spike.failed_test_items_abnormal` is `true`. CASE writes a log line that names the product name, the manufacturing-result id, and that `failed_test_items` is missing or not pairs.
12. **Do:** A spike already exists for a product name. The caller posts that same product name to `POST /apis/spikes`. The failed manufacturing-result id set matches that spike. `failed_test_items` on at least one of those ids differs from the stored JSON value.
   **Result:** CASE returns HTTP 200 and that existing spike with `repeated` set to true. CASE stores nothing. `spike` has no new row. `spike_failed_result` has no new row.

### UI

Design: [02-ui-submit-the-spike.md](../design/02-ui-submit-the-spike.md). Contract: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md).

1. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The API returns `repeated` set to false.
   **Result:** The screen shows that the spike is reported and stored.
2. **Do:** The operator does not select a product name.
   **Result:** The Report spike button is gray. The operator cannot click the button.
3. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The API returns HTTP 200 with `product_name` and `message`. The message is `no failed part found under ` plus the product name.
   **Result:** The screen shows that message. The screen does not treat this call as an error.
4. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The API returns an error (for example HTTP 400 or HTTP 500).
   **Result:** The screen shows the error message returned from the backend.
5. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The API stores the spike and returns failed results.
   **Result:** The screen shows a table of the copied failed results. The columns are `product_name`, `serial_number`, `failed_test_item`, `op_code`, `station_id`, `rack_id`, `fixture_id`, `nest_number`, `operator_id`, `shift_code`, and `created_at`. The `failed_test_item` column shows the keys of `failed_test_items`. The screen reads the four flat equipment fields from the response. The screen does not invent a nested `station_setup` object.
6. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The API returns `repeated` set to true.
   **Result:** The screen shows that this spike was already reported. The operator does not need to report it again.
7. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The API returns HTTP success with a spike `id` and without a `repeated` field (or with a `repeated` value that is not strictly true or false).
   **Result:** The screen shows `The report response is incomplete. Please contact the support team.` The screen does not show that the spike is reported and stored. The screen does not show that this spike was already reported. The screen does not show the failed-results table.
8. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The POST to `POST /apis/spikes` is still in flight.
   **Result:** The Report spike button is disabled while the request is in flight. After the request settles, the button is enabled again when a product name is still selected.
9. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The API returns HTTP success (`status` under 400) with an empty body, or with a body that is not valid JSON (so the screen cannot read `id` or `message`).
   **Result:** The screen shows `The report response is incomplete (missing id, missing message). Please contact the support team.` The screen does not show an empty info line. The screen does not show that the spike is reported and stored. The screen does not show that this spike was already reported. The screen does not show the failed-results table.
10. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The API returns HTTP 400 with an empty JSON body `{}` (no `message` field).
   **Result:** The screen shows `The report failed.` The screen does not show that the spike is reported and stored.
11. **Do:** The operator selects a product name from the dropdown. The operator clicks Report spike. The API returns a stored spike with `failed_test_items_abnormal` set to `true`.
   **Result:** The screen shows that the spike is reported and stored. The screen shows `Test data is abnormal. A later cause list can be wrong. MES engineers need to get involved.` The screen does not mark each failed-result row with its own status.

