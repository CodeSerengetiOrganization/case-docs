# CASE — Scored EOL cause list

**Metric:** The held-out true cause sits at the top of the list.

The GitHub Project holds the status. These files hold the issue, the design, and the tech notes. Equipment words are in [GLOSSARY.md](../../GLOSSARY.md).

Finish issue 01 before you open issue 04. Finish issue 03 before you open issue 04.

| Issue | File |
|---|---|
| Learn the model call | [01-learn-the-model-call.md](issues/01-learn-the-model-call.md) |
| The operator submits the product and the EOL stations | [02-the-operator-submits-the-product-and-the-eol-stations.md](issues/02-the-operator-submits-the-product-and-the-eol-stations.md) |
| Prepare the lookup data | [03-prepare-the-lookup-data.md](issues/03-prepare-the-lookup-data.md) |
| The agent returns a ranked cause list | [04-the-agent-returns-a-ranked-cause-list.md](issues/04-the-agent-returns-a-ranked-cause-list.md) |
| The score shows whether the true cause is at the top | [05-the-score-shows-whether-the-true-cause-is-at-the-top.md](issues/05-the-score-shows-whether-the-true-cause-is-at-the-top.md) |

Design files live in `design/`. Tech notes live in `tech-notes/`. Open follow-ups live in [`tech-debt/`](tech-debt/README.md).

| Tech note | File |
|---|---|
| The stored spike waits | [01-the-stored-spike-waits.md](tech-notes/01-the-stored-spike-waits.md) |
| MySQL in Docker for store tests | [02-mysql-in-docker-for-store-tests.md](tech-notes/02-mysql-in-docker-for-store-tests.md) |
| DB lock: same process vs two instances | [06-db-lock-same-process-vs-two-instances.md](tech-notes/06-db-lock-same-process-vs-two-instances.md) |

| Tech debt | File |
|---|---|
| `product_name` MySQL collation | [01-product-name-mysql-collation.md](tech-debt/01-product-name-mysql-collation.md) |
| `created_at` DATETIME vs Instant / UTC | [02-created-at-datetime-vs-instant-utc.md](tech-debt/02-created-at-datetime-vs-instant-utc.md) |
| Repeat check scales with spike history | [03-repeat-check-scales-with-spike-history.md](tech-debt/03-repeat-check-scales-with-spike-history.md) |
