# Note — How we run C.A.S.E.

**Type:** Project method  
**Project:** C.A.S.E. (Candidate Actions from Station Evidence), Customer A  
**Date:** 2026-11-05  
**Decided by:** Jonathan, with Vector Elon  

This note is the way we manage the project. The first proof is a scored cause list for an EOL spike.

## The two parts

Palantir tells us where the truth comes from. The person who feels the miss drives the next build.

Tesla tells us what to delete. Any process that does not move the proof goes away.

C.A.S.E. is small enough to run both. We do not copy either company's org chart.

## Rules

1. One outcome. The agent returns a ranked cause list for an EOL spike, and we can score it.
2. One owner. Jonathan owns the UI, the API, the model call, the lookup files, and the score. We do not split a UI track from an agent track.
3. The field sets the work. A miss on a held-out solved case creates the next issue. Later, a miss on the physical part does the same. A framework does not create issues.
4. The score is the status. An issue closes when a rescore shows the change. A column that says "in progress" proves nothing.
5. Delete the change that does not move the score.

## Words

Use **project** and **issue** in `case-docs` and in the GitHub Project.

A task appears only when one person cannot finish the issue in one change.

A ticket means any card in the tool. It is not a level in this method.

We do not use the bank ladder (epic, feature, story). We do not use stage-gates.

## The board

```text
Project: Scored EOL cause list
  metric: the held-out true cause sits at the top of the list
  issue: The operator submits the product and the EOL stations
  issue: The agent returns a ranked cause list
  issue: The score shows whether the true cause is at the top
```

The first project covers five things.

- The UI takes a product name and the EOL stations that show the unexpected fails.
- The API stores that spike and calls the agent.
- We rent a large language model.
- We give the agent other solved-case reports and maintenance logs as lookup material.
- We score the ranked list.

The score needs a held-out answer. A solved case states the true cause. We keep that cause out of the prompt. We give the agent the other cases and the logs. We then see whether the true cause sits at the top of the list.

We can mock the path that finds the stations. The mock must still pass the product, the stations, and the fail counts. If a solved case turns on a failed test item, the UI needs that item too, or the score is unfair.

## Later projects

The fail-window project stays closed until this score is real. The window statistic stays open inside that later project. That project lets the workflow find the stations from the product, instead of the operator naming the stations.

The live physical check waits with it. This first project uses the solved cases as the check.

## What we do not copy

We do not copy a forward-deployed department, a hack week, a factory cell, or a program increment.

We need the miss, the next change, and the next score.
