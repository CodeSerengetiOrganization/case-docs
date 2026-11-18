# Temporary note — API AC checks vs unit/IT map (issue 02)

**Type:** Temporary tracking note  
**Project:** C.A.S.E. — Scored EOL cause list  
**Date:** 2026-11-16 (updated 2026-11-17)  
**For:** Jonathan + Sam + Elon (discuss; revise before PR AC→test map)  
**Delete when:** API store PR ships with an agreed AC→test map in the PR description.

Issue API checks: [projects/scored-eol-cause-list/issues/02-the-operator-submits-the-product-and-the-eol-stations.md](../projects/scored-eol-cause-list/issues/02-the-operator-submits-the-product-and-the-eol-stations.md)  
Design: [projects/scored-eol-cause-list/design/02-api-store-the-spike.md](../projects/scored-eol-cause-list/design/02-api-store-the-spike.md)  
Code today: `case/services/case-api/src/test/java/com/case_api/…`

UI checks stay out of this note. See [05-temporary-note-ui-gaps-for-issue-02.md](05-temporary-note-ui-gaps-for-issue-02.md).

## Map

| Check                           | What it covers                                           | Unit / controller                                                                                    | IT                                                                                                                                      |
| ------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| 1 Happy store 201               | new spike, `repeated=false`, rows written                | `SpikeServiceTest.reportSpike_whenHasFailed_stores…`; `SpikeControllerTest.…_returns201`             | `SpikeServiceIT.reportSpike_happyPath…`                                                                                                 |
| 2 No failed part                | 200 + message, no store                                  | `SpikeServiceTest.…_whenNoFailedPart…`; `SpikeControllerTest.…_returns200`                           | — (unit/controller only)                                                                                                                |
| 3 Store TX rollback             | null `product_code` knife → write fails → nothing stored | —                                                                                                    | `SpikeServiceIT.…_whenProductCodeNull_rollsBackWholeStore` (asserts `DataIntegrityViolationException`; lock + business rows stay empty) |
| 4 Blank product name            | HTTP 400, no store                                       | `SpikeControllerTest.…_whenValidationFails_returns400`                                               | — (unit/controller only)                                                                                                                |
| 5 Retriever ≤100                | search layer cap                                         | `MockSpikeDataRetrieverTest.search_whenManyFailedProduct…`                                           | — (unit only)                                                                                                                           |
| 6 Service trim >100             | trim before store                                        | `SpikeServiceTest.…_whenRetrieverReturnsMoreThan100…`                                                | — (unit only)                                                                                                                           |
| 7 Repeat same id set            | 200, `repeated=true`, no store                           | `SpikeServiceTest.…_whenIdSetMatchesExisting…`; `SpikeControllerTest.…_whenRepeatedSpike_returns200` | — (unit/controller; check 9 covers live no-double)                                                                                      |
| 8 New manufacturing-result id   | 201, new spike                                           | `SpikeServiceTest.…_whenNewManufacturingResultId…`                                                   | — (unit only)                                                                                                                           |
| 9 Concurrent same id set        | one new + one repeat                                     | lock helpers: `…_acquiresProductNameLock…`, `…_whenLockInsertIgnores…`                               | `SpikeServiceIT.…_twoConcurrentSameIdSet…`                                                                                              |
| 10 Missing equipment            | store null + log, no refuse                              | `SpikeServiceTest.…_whenNestMissing…` (log)                                                          | `SpikeServiceIT.…_whenNestMissing_persistsNullNestNumber`                                                                               |
| 11 Abnormal `failed_test_items` | flag + log + description skip                            | `SpikeServiceTest.…_whenFailedTestItemsAbnormal…` (log / description)                                | `SpikeServiceIT.…_whenFailedTestItemsIsArray_persistsAndSetsAbnormal`                                                                   |
| 12 Same ids, different JSON     | still repeat                                             | `SpikeServiceTest.…_whenSameIdsButDifferentFailedTestItems…`                                         | — (unit only)                                                                                                                           |

## Decisions (Elon + Jonathan)

1. **Check 3** — closed. Issue Do/Result match the null `product_code` IT. One persist-failure knife. Assert `DataIntegrityViolationException` (not bare `RuntimeException`). IT passed.
2. **Check 2 / 4** — no live-DB IT. Those paths never open the store write. Unit/controller are enough.
3. **Check 7** — no separate live-DB IT for “tables stay empty.” Unit/controller cover the repeat return. Check 9 IT already proves one spike and no double on real MySQL for the same id set.
4. **Live DB IT stays for** happy path, rollback, race, and DB-accept gaps (checks 1, 3, 9, 10, 11).
5. **Check 5 / 6** — unit-only. No MySQL IT for “≤100 / exactly 100 rows.”
6. **Check 1 IT** — acceptable. IT hits the service and asserts DB rows. Controller unit covers HTTP 201.
7. **Check 8 / 12** — unit + controller only. No extra IT.
8. **Check 10 / 11** — two thin live-DB ITs done and passed. Log / description / flag stay in unit tests.

## Remaining gaps (still open)

1. Put this map in the API PR description as the ship bar.
