---
layout: post
title:  "Druid SQL: BETWEEN considered harmful"
categories: blog apache druid sql
twitter:
  image: /assets/6408417/650adbdd-4711-48e1-bf22-8834095e7d68
---

<img src="/assets/6408417/650adbdd-4711-48e1-bf22-8834095e7d68" width="50%">

When querying data in Druid (or another analytical database), your query will in almost all cases include a filter on the primary timestamp. And this timestamp filter will usually take the form of an interval.

The easiest way to describe such an interval seems to be the SQL `BETWEEN` operator.

Advice from a [grug brained developer](https://grugbrain.dev/): **Don't do that.**

Here's why.

## A harmless data sample

Imagine you have a table like this:

|__time                  |val|
|------------------------|---|
|2023-01-01T01:00:00.000Z|1  |
|2023-01-02T00:00:00.000Z|1  |
|2023-01-02T06:00:00.000Z|1  |
|2023-01-03T00:00:00.000Z|1  |
|2023-01-03T01:00:00.000Z|1  |
|2023-01-04T00:00:00.000Z|1  |
|2023-01-04T07:00:00.000Z|1  |

You can populate such a table in Druid using SQL ingestion like so:

```sql
REPLACE INTO "sample" OVERWRITE ALL
WITH "ext" AS (
  SELECT *
  FROM TABLE(
    EXTERN(
      '{"type":"inline","data":"datetime,val\n2023-01-01 01:00:00,1\n2023-01-02 00:00:00,1\n2023-01-02 06:00:00,1\n2023-01-03 00:00:00,1\n2023-01-03 01:00:00,1\n2023-01-04 00:00:00,1\n2023-01-04 07:00:00,1"}',
      '{"type":"csv","findColumnsFromHeader":true}'
    )
  ) EXTEND ("datetime" VARCHAR, "val" BIGINT)
)
SELECT
  TIME_PARSE(TRIM("datetime")) AS "__time",
  "val"
FROM "ext"
PARTITIONED BY DAY
```

You want to list all rows for 2nd and 3rd January. You write:

```sql
SELECT * FROM "sample"
WHERE __time BETWEEN TIMESTAMP'2023-01-02' AND TIMESTAMP'2023-01-03'
```

And here's the result:

|__time                  |val|
|------------------------|---|
|2023-01-02T00:00:00.000Z|1  |
|2023-01-02T06:00:00.000Z|1  |
|2023-01-03T00:00:00.000Z|1  |

You notice that all the rows for 2nd January are in the result, but only one row for 3rd January. What happened?

## The solution

We are being hit by two entirely documented features here, which together create a minor footgun.

1. The `BETWEEN` operator creates a closed interval, that is it includes both the left and right boundary value. This would by itself not be a problem, were it not for the second feature.
2. The literal `TIMESTAMP'2023-01-03'` does _not_ mean "the entire day of 3rd January", as one might naïvely think. It is equivalent to "3rd January, 00:00".

What we have done is: we have created a query that includes the entire 2nd January but only the data for 00:00 on 3rd January!

You could fix this by writing something like `TIMESTAMP'2023-01-03 23:59:59'` for the right interval boundary. But does this really catch every last bit of the data for that day? What if you have fractional timestamps? Is your precision milliseconds? or even microseconds?

This is why I argue that the proper way to model such time filter conditions is to use a right-open interval, which includes the left boundary value _but not_ the right boundary value. If you do that, you have to set the right boundary to the _next_ day (4th January), in order to still catch all of 3rd January in your filter:

```sql
SELECT * FROM "sample"
WHERE __time >= TIMESTAMP'2023-01-02' AND __time < TIMESTAMP'2023-01-04'
```

This query returns the correct result:

|__time                  |val|
|------------------------|---|
|2023-01-02T00:00:00.000Z|1  |
|2023-01-02T06:00:00.000Z|1  |
|2023-01-03T00:00:00.000Z|1  |
|2023-01-03T01:00:00.000Z|1  |

This way of filtering is also in line with the treatment of time intervals almost everywhere in Druid. Segment time chunks, for instance, are defined in terms of right open intervals, too.

## Learnings

- Don't use the `BETWEEN` operator in SQL. Especially not for time intervals. Because the operator creates an inclusive (closed) interval, the result may not be what you expect.
- Use a `WHERE` clause with simple comparison operators instead, to create a right open interval.

----
 <p class="attribution">"<a target="_blank" rel="noopener noreferrer" href="https://www.newgrounds.com/art/view/platinumfusi0n/grug">Pizza</a>" by <a target="_blank" rel="noopener noreferrer" href="https://platinumfusi0n.newgrounds.com/">PlatinumFusi0n</a> is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by/3.0/">CC BY 3.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>. </p> 
