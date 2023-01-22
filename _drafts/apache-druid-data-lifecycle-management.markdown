---
layout: post
title:  "Apache Druid: Data Lifecycle Management"
categories: blog druid tutorial
---

One feature of Druid that contributes greatly to its performance is [rollup](https://druid.apache.org/docs/latest/ingestion/rollup.html). This converts a detail table

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
