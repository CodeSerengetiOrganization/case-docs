# UI submits the spike

Issue: [02-the-operator-submits-the-product-and-the-eol-stations.md](../issues/02-the-operator-submits-the-product-and-the-eol-stations.md)

Contract: [02-contract-store-the-spike.md](02-contract-store-the-spike.md)

## Problem

The operator reports an EOL spike. The screen sends the product name. The product name is the only key. The API stores the spike. This screen does not call the model.

## Decisions

1. The operator selects the product name from a dropdown. The names in that dropdown are hardcoded for this issue.
2. The operator does not pick the EOL stations. A backend service searches for the stations. This file does not decide the search rule.
3. The screen sends the product name to the API. The product name is the only key. The screen does not store the spike itself.
4. The screen does not call the model. The stored spike waits for a later call. Tech note: [01-the-stored-spike-waits.md](../tech-notes/01-the-stored-spike-waits.md).
5. The operator clicks a button labeled Report spike. That click sends the selected product name to the API.
6. When the operator does not select a product name, the Report spike button is gray. The operator cannot click the button.
7. After the API returns a success body with `repeated` set to false, the screen shows that the spike is reported and stored.
8. After the API returns a success body with `repeated` set to true, the screen shows that this spike was already reported. The operator does not need to report it again.
9. When the API returns HTTP 200 with `product_name` and `message` for no failed part, the screen shows that message. That call is not an error path.
10. When the API returns an error (`status` 400 or higher), the screen shows the backend `message` when it is present. When the error body has no `message` (empty body, invalid JSON, or JSON without `message`), the screen shows `The report failed.`
11. After the API returns the spike, the screen also shows a table of the copied failed results. The columns are `product_name`, `serial_number`, `failed_test_item`, `op_code`, `station_id`, `rack_id`, `fixture_id`, `nest_number`, `operator_id`, `shift_code`, and `created_at`. The `failed_test_item` column shows the keys of `failed_test_items`. The screen uses the four flat equipment fields from the response. The screen does not invent a nested `station_setup` object.
12. When the API returns a body with a spike `id` and `repeated` is not strictly `true` or `false` (missing or any other value), the screen treats the response as incomplete. The screen shows `The report response is incomplete. Please contact the support team.` The screen does not show the stored success line. The screen does not show the already-reported line. The screen clears the failed-result rows. The screen does not show the table. The screen does not invent `repeated` as false or true.
13. While the Report spike request is in flight, the Report spike button is disabled. The button enables again when the request settles (success, info, or error).
14. When the API returns HTTP success (`status` under 400) and the body is not usable — no spike `id` and no non-empty `message` — the screen treats the response as incomplete. This includes an empty body and a body that fails JSON parse. The screen shows an error that names each missing part, then asks the operator to contact support. Example when both are missing: `The report response is incomplete (missing id, missing message). Please contact the support team.` The screen does not show an empty info line. The screen does not show the stored success line. The screen does not show the already-reported line. The screen does not show the table. Decision 9 still applies when `message` is non-empty and `id` is absent (no failed part).
15. The screen reads `failed_test_items_abnormal` on the spike. The screen does not classify each failed result. When `failed_test_items_abnormal` is `true`, the screen shows `Test data is abnormal. A later cause list can be wrong. MES engineers need to get involved.` When `failed_test_items_abnormal` is `false`, the screen does not show that note. The `failed_test_item` cell shows keys only from a `failed_test_items` object of pairs. The screen does not invent a test item from an array index or from a primitive. The screen still shows the stored spike.

## Out of scope

The API store. The lookup data. The model call. The score.

## Open choices

Write a choice here before you add a check to the issue.
