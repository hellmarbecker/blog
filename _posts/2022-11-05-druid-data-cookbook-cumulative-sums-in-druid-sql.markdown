---
layout: post
title:  "Druid Data Cookbook: Cumulative Sums in Druid SQL"
categories: blog apache druid imply sql tutorial
---
![Druid Cookbook](/assets/2021-12-21-elf.jpg)

An often asked feature that is currently not present in Druid SQL are [window functions](https://www.sqltutorial.org/sql-window-functions/). With these, you can create aggregations on levels other than what you are `GROUP`ing by, and you can relate rows within the same table to one another without resorting to a costly [cartesian join](https://en.wikipedia.org/wiki/Join_(SQL)#Cross_join). Until these capabilities are added, let's look at a few use cases and find out how to model them in Druid SQL!

Many thanks to [Gian Merlino](https://www.linkedin.com/in/gianmerlino/) and [John Kowtko](https://www.linkedin.com/in/jkowtko/) for inspiring the solutions I am summarizing in this blog.

The dataset for today's tutorial looks like this:

```csv
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

First, I want a report that shows, for each customer, how their cumulative revenue builds up over time, day by day. As a preparation, let's aggregate our data to the level of granularity that we will need in the report:

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
GROUP BY 1, 2
ORDER BY 1, 2
```

This way, I have built the equivalent of a `SUM() OVER (ROWS UNBOUNDED PRECEDING AND CURRENT ROW)` aggregation.

## Daily Differences

I want a report that shows me the difference in spending to the previous day, per customer. The pattern is going to be similar to the previous one. Because I have, for each customer, a date that has no predecessor, I have to use a `LEFT JOIN` clause though:

```sql
WITH cte AS (
  SELECT DATE_TRUNC('DAY', "__time") AS date_day, customer, SUM(revenue) AS rev_daily
  FROM cume_data
  GROUP BY 1, 2
)
SELECT
  t2.date_day,
  t2.customer,
  t2.rev_daily - t1.rev_daily AS delta
FROM cte t2 LEFT JOIN cte t1 
  ON t1.customer = t2.customer
  AND t2.date_day = t1.date_day + INTERVAL '1' DAY
```

For each customer, I join each day's data row to the previous day's row and compute the difference. I have just emulated a `LAG()` function!

## Daily Contribution per Customer

Finally, I want to know the relative contribution of each customer to each day's revenue. This is where, in many SQL dialects, you would use a window function like `SUM() OVER(PARTITION BY customer)`. We can emulate this by using [grouping sets](https://druid.apache.org/docs/latest/querying/sql.html#group-by). Specifically, I am going to use `GROUP BY ROLLUP()`, which treats the dimension list that I pass as a hierarchy.

Here's the base query:

```sql
SELECT
  CASE WHEN "__time" IS NULL THEN NULL ELSE __time END AS date_day, 
  customer,
  SUM(revenue) AS rev_daily,
  GROUPING(CASE WHEN "__time" IS NULL THEN NULL ELSE __time END, customer) AS rollup_bits
FROM
  cume_data
GROUP BY ROLLUP (1, 2)
```

Here's the result:

date_day|customer|rev_daily|rollup_bits
---|---|---|---
2022-01-01T00:00:00.000Z|alice|10.5|0
2022-01-01T00:00:00.000Z|bob|11.5|0
2022-01-02T00:00:00.000Z|alice|12.5|0
2022-01-02T00:00:00.000Z|bob|27.5|0
2022-01-03T00:00:00.000Z|alice|14.5|0
2022-01-03T00:00:00.000Z|bob|15.5|0
2022-01-01T00:00:00.000Z|_null_|22|1
2022-01-02T00:00:00.000Z|_null_|40|1
2022-01-03T00:00:00.000Z|_null_|30|1
_null_|_null_|92|3

Three things are worth noticing:

- The `ROLLUP` aggregation creates a grouping hierarchy. We group by all columns in the list, then by all but the last, then by all but the last two, and so forth up to the grand total. Each dimension that is left out gets a _NULL_ in the respective place. (You can get all combinations with `CUBE`, or an explicit list with `GROUPING SETS`.)
- In order to know which level a result row is aggregated to, you can use [the special `GROUPING()` aggregator](https://druid.apache.org/docs/latest/querying/aggregations.html#grouping-aggregator). The result of `GROUPING()` is a bitmask that has a `1` bit for every level that has been rolled up - the most fine grained result rows give a `0` value, the grand total is all ones. We will pick the rollup per day on one side of the join (this one will have a `1`), and the detail rows (they have a `0`) on the other side.
- The timestamp in Druid cannot be _NULL_. If you try to use it as part of a grouping sets aggregation, Druid complains about not being able to convert a `TIMESTAMP(3) NOT NULL` to a `TIMESTAMP(3)`. I work around this with a somewhat unwieldy `CASE` expression that can (in theory) yield a _NULL_ value.

With this, I am ready to assemble the final query:

```sql
WITH cte AS (
  SELECT
    CASE WHEN "__time" IS NULL THEN NULL ELSE __time END AS date_day, 
    customer,
    SUM(revenue) AS rev_daily,
    GROUPING(CASE WHEN "__time" IS NULL THEN NULL ELSE __time END, customer) AS rollup_bits
  FROM
    cume_data
  GROUP BY ROLLUP (1, 2)
)
SELECT
  t1.date_day,
  t2.customer,
  100.0 * t2.rev_daily / t1.rev_daily AS contribution_percent
FROM cte t1 INNER JOIN cte t2 ON t1.date_day = t2.date_day
WHERE t1.rollup_bits = 1
AND t2.rollup_bits = 0
```

Try it out!

## Conclusion

While window aggregation functions are not explicitly available in Druid at the time of this writing, there are numerous ways to get to the desired results.

- Many queries that use window aggregation functions can be modeled in Druid using self joins.
- Make sure to push down aggregations and filters in order to keep the result sets manageable.
- Grouping set aggregations can be helpful when you need multiple aggregation levels within the same query.


---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
