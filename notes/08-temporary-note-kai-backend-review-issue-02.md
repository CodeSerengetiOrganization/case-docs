# Temporary note — Kai backend review (issue 02 store)

**Type:** Temporary tracking note  
**Project:** C.A.S.E. — Scored EOL cause list  
**Date:** 2026-11-17  
**For:** Jonathan (fix one by one) + Kai (re-check after fixes)  
**Branch / commit reviewed:** `case` `feature/02-store-the-spike` @ `a7e3b28`  
**Delete when:** API store PR merges and open items below are closed or deferred in writing.

Issue: [projects/scored-eol-cause-list/issues/02-the-operator-submits-the-product-and-the-eol-stations.md](../projects/scored-eol-cause-list/issues/02-the-operator-submits-the-product-and-the-eol-stations.md)  
AC→test map: [07-temporary-note-api-ac-to-test-map-issue-02.md](07-temporary-note-api-ac-to-test-map-issue-02.md)  
Code: `case/services/case-api/`, `case/docker/mysql/`

UI is out of this note.

## Verdict

Directionally solid for issue 02 store. TX split, lock pin, Flyway, and the AC→test split look production-minded.  
**Code/CI review items:** #1 / #3 / #4 / #6 closed. **#2 / #5** wait on Jonathan at PR submit.

## What looks right (no action)

- `SpikeService` keeps search + trim outside the store TX; `SpikeStoreWriter.storeOrReturnRepeat` is the only `@Transactional` proxy entry.
- Lock path uses `INSERT IGNORE` then `FOR UPDATE` (no catch-1062 → rollback-only).
- Repeat is id-set equality; different `failed_test_items` still repeats.
- Schema + entities match the contract shape (flat equipment fields, copied MES id as PK part).
- Unit + IT coverage matches the closed map in note 07 (live DB: checks 1, 3, 9, 10, 11).

## Open items — fix or decide one by one

| # | Severity | Item | Status |
|---|----------|------|--------|
| 1 | Must confirm / fix | **CI for ITs.** `.github/workflows/case-api-ci.yml` — Docker + `./mvnw clean verify` on PRs/pushes to `develop` (path-filtered). Testcontainers owns MySQL. No GHCR (no Dockerfile). Kai accepted. | closed |
| 2 | Must fix (process) | **PR ship bar.** Paste the AC→test map from note 07 into the API PR description before merge. Jonathan does this manually at PR time. | Jonathan owns (manual) |
| 3 | Must confirm | **Store failure HTTP shape.** Unexpected store/DB failures → bare HTTP 500; details in backend logs only; UI shows a fixed general line (no exception leak). Rich 500 body / correlation id deferred. Agreed Jonathan + Kai + Elon. | closed |
| 4 | Should fix | **`reportSpike` returns `Object`.** Sam: sealed `ReportSpikeResult` (`NoFailedPart` / `Stored`); controller exhaustive `switch`; dead 500 removed. | closed |
| 5 | Should fix (process) | **Commit / PR wording.** Commit says “not ye reviewed.” Do not leave that as the final PR title/body. Jonathan cleans this when submitting the PR. | Jonathan owns (manual) |
| 6 | Question | **Failsafe without Docker.** Hard-fail (no skip) on CI and local `verify`. Kai accepted. | closed |

## Suggested fix order

1. Decide #6 (skip vs hard-fail), then close #1 (add CI or document the local `mvn verify` bar).
2. Close #3 (accept generic 500 for this ticket, or add error body).
3. Optional cleanup #4.
4. Before open/merge PR: #2 map in description + #5 clean title/body.
5. Ask Kai to re-check closed rows.

## Re-check log

| Date | What changed | Kai |
|------|--------------|-----|
| 2026-11-17 | Initial review logged | — |
| 2026-11-17 | Sam drafted `case-api-ci.yml` (verify + Testcontainers; no image push) | #1 / #6 accepted closed |
| 2026-11-17 | #2 PR ship bar → Jonathan owns manually at PR time | — |
| 2026-11-17 | #3 accept bare 500 + UI general message; no rich 500 contract this ticket | closed |
| 2026-11-17 | #5 PR title/body wording → Jonathan owns at PR submit | — |
| 2026-11-17 | #4 sealed `ReportSpikeResult` (Sam); tests passed | #4 accepted closed |
