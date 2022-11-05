---
layout: post
title:  "Druid Data Cookbook: Cumulative Sums in Druid SQL"
categories: blog apache druid imply sql tutorial
---

An often asked feature that is currently not present in Druid SQL are [window functions](https://www.sqltutorial.org/sql-window-functions/). With these, you can create aggregations on levels other than what you are `GROUP`ing by, and you can relate rows within the same table to one another without resorting to a costly [cartesian join](https://en.wikipedia.org/wiki/Join_(SQL)#Cross_join). Until these capabilities are added, let's look at a few use cases and find out how to model them in Druid SQL!

Many thanks to [Gian Merlino](https://www.linkedin.com/in/gianmerlino/) and [John Kowtko](https://www.linkedin.com/in/jkowtko/) for the solutions that I am summarizing in this blog!

The dataset for today's tutorial looks like this:

```
ts,customer,revenue
2022-01-01,alice,10.50
2022-01-01,bob,11.50
2022-01-02,alice,12.50
2022-01-02,bob,13.50
2022-01-02,bob,14.00
2022-01-03,alice,14.50
2022-01-03,bob,15.50
```

Ingest it using the Druid console, using standard batch ingestion with a Paste source. I name the resulting Druid datasource `cume_data`.

## Cumulative Sums

First, I want a report that shows me for each customer, how their cumulative revenue builds up over time, day by day. As a preparation, let's aggregate our data to the level of granularity that we will need in the report:

```sql
SELECT DATE_TRUNC('DAY', "__time") AS date_day, customer, SUM(revenue) AS rev_daily
FROM cume_data
GROUP BY 1, 2
```

This is a common pattern with Druid: try to push groupings and filters down to the individual subqueries such that the final join can work on relatively small result sets.

Now, for each customer and date, join the _current_ date against all dates up to and including that date, aggregating over current date and customer. Since the date condition is not a simple equals condition, it has to go into the `WHERE` clause: 

```sql
WITH cte AS (
  SELECT DATE_TRUNC('DAY', "__time") AS date_day, customer, SUM(revenue) AS rev_daily
  FROM cume_data
  GROUP BY 1, 2
)
SELECT
  cte.date_day,
  cte.customer,
  SUM(t2.rev_daily)
FROM cte INNER JOIN cte t2 ON cte.customer = t2.customer
WHERE t2.date_day <= cte.date_day
GROUP BY cte.date_day, cte.customer
```

