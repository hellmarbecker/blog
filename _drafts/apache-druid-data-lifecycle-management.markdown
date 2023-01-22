---
layout: post
title:  "Apache Druid: Data Lifecycle Management"
categories: blog druid tutorial
---

![Rollup in Druid](/assets/2023-01-22-01-rollup.png)

One feature that contributes greatly to the performance of Druid is [rollup](https://druid.apache.org/docs/latest/ingestion/rollup.html). Rolling up converts a detail table into an aggregate table that contains

- a _truncated_ timestamp: the granularity of that timestamp is known as the _query granularity_
- a number of _dimensions_, usually string fields that the data is grouped by
- a number of _metrics_. These are usually aggregations of numeric fields, but in the case of Druid you could also find data sketches as metrics, such as theta sketches that allow you to get approximate distinct counts out of aggregated data.

Rollup can significantly reduce the amount of storage needed for a dataset and thence the amount of data that needs to be scanned for a query.

## Rollup and Data Lifecycle Management

### Staggered Rollups

Many of my customers would like to take advantage of the space savings promised by rollup. Their requirement is often like this:

- For a limited period of time, such as the current and last month, keep the data at a relatively detailed level, such as daily of hourly.
- Older data should be aggregated on a monthly level.

Sometimes the requirement is a bit more complex, or staggered:

- Keep daily data for the current month.
- Roll up to weekly data for the last 3 months
- and to monthly data for anything that is older than 3 months.

It is this more complex scenario where some traps for the unwary are lurking, which I want to explore today. This tutorial works with the [Druid 25.0 quickstart](https://druid.apache.org/docs/latest/tutorials/index.html), and I am running Druid on my laptop - for other configurations you would have to adapt the API endpoint accordingly.

### How Staggered Rollups Are Done in Druid

Rollups as part of data lifecycle management are executed using [compaction tasks](https://druid.apache.org/docs/latest/data-management/compaction.html) in Druid. You can picture a compaction task as a reindexing task where most configuration options are taken 1:1 from the original datasource, and the datasource is reindexed onto itself. If you need only one kind of compaction, you can run it as a so called autocompaction job in the background. But the staggered scenario that we are discussing means, you need to submit those compaction jobs explicitly, via API calls, to the Druid Overlord. A compaction spec is a JSON file, much like an ingestion spec but simpler.

You can submit a compaction task using a little shell script like this:

```bash
#!/bin/bash
curl -XPOST -H "Content-Type: application/json" http://localhost:8888/druid/indexer/v1/task -d@$1
```

## The Data


## Submitting Tasks


Let's first craft a query to verify our data before and after each step. We are going to look at

- the number of rows in our datasource, as an indicator whether the rollup worked
- the sum of the total count of original rows
- the number of unique users,

broken down by month, and as a grand total.

```sql
SELECT
  CASE WHEN FLOOR(__time TO MONTH) IS NULL THEN NULL ELSE FLOOR(__time TO MONTH) END AS "date",
  COUNT(*) AS numRowsCompacted,
  SUM(__count) AS numRowsOrig,
  THETA_SKETCH_ESTIMATE(DS_THETA(theta_user)) AS uniqueUsersTheta
FROM "user_data"
GROUP BY ROLLUP(1)
```

The reason for the somewhat awkward incantation around the date expression is explained [here](/2022/11/05/druid-data-cookbook-cumulative-sums-in-druid-sql).
