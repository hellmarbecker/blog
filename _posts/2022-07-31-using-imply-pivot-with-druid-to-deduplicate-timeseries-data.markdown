---
layout: post
title:  "Using Imply Pivot with Druid to Deduplicate Timeseries Data"
categories: blog druid imply analytics tutorial
---

![Solar farm banner](/assets/2022-07-31-00-640px-Mount_Majura_solar_farm_and_Majura_Parkway.jpg)

Imagine you are running a solar or wind power plant. You are measuring the power output, among other parameters, on a regular basis, but the measurements are not evenly distributed along the time axis. Sometimes you get several measurements for the same unit of time, sometimes there is only one. How do you measure the average power output without giving undue weight to those time periods that had more measurements?

Consider a very simplified example:

```csv
ts,var,val
2022-07-31T10:01:00,p,31
2022-07-31T10:02:00,p,33
2022-07-31T10:18:00,p,48
```

There are two measurements of power output between 10:00 and 10:15, but only one between 10:15 and 10:30. Ingest those data into Druid with a datasource name of `power_data`. Do not use rollup. Here is the ingestion spec:

```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "inline",
        "data": "ts,var,val\n2022-07-31T10:01:00,p,31\n2022-07-31T10:02:00,p,33\n2022-07-31T10:18:00,p,48\n"
      },
      "inputFormat": {
        "type": "csv",
        "findColumnsFromHeader": true
      }
    },
    "tuningConfig": {
      "type": "index_parallel",
      "partitionsSpec": {
        "type": "dynamic"
      }
    },
    "dataSchema": {
      "dataSource": "power_data",
      "timestampSpec": {
        "column": "ts",
        "format": "iso"
      },
      "dimensionsSpec": {
        "dimensions": [
          "var",
          {
            "type": "long",
            "name": "val"
          }
        ]
      },
      "granularitySpec": {
        "queryGranularity": "none",
        "rollup": false,
        "segmentGranularity": "day"
      }
    }
  }
}
```

Because the average power during the first 15 minutes has been 32 watts, and during the second 15 minutes, 48 watts, you would want to show the average power output as **40 watts**. Keep that in mind.

## Using an Average Measure in Pivot

From the Druid datasource, create a cube in Imply Pivot. Make sure it is a SQL cube, and prepopulate the dimensions and measures:

![Create cube from datasource](/assets/2022-07-31-01-define-cube.jpg)

By default, for each numeric field Pivot creates a `SUM` measure. Change this to `Average` for the `Val` measure, and also change the name to `Avg Val`:

![Define Average measure](/assets/2022-07-31-02-define-naive-avg.jpg)

Let's show this measure in the Cube view. Select the measure:

![Select Average measure](/assets/2022-07-31-03-select-avg-measure.jpg)

and view the result.

![Show Average result is 37](/assets/2022-07-31-04-avg-result.jpg)

The result is `(31 + 33 + 48) / 3 = 37`, which is the correct average but not the answer to the question in this case. What you would like to do is:

1. Take the average of all measurements for each 15 minute interval
2. Then take the average of the results from step 1.

This is actually possible with Imply Pivot.

## Nested Aggregation Measures

Pivot offers multi step aggregations in what is called [_nested aggregations_](https://docs.imply.io/latest/measures/#nested-aggregation-measures). Here, you can define an expression to split the data first into buckets in which to aggregate (the _inner aggregation_), then an _outer aggregation_ runs over the results of the inner aggregation.

In this case, the split expression should yield 15 minute buckets, and both aggregations are averages.

Add a new measure to your cube:

![Add measure](/assets/2022-07-31-05-add-measure.jpg)

And define the new measure with a custom SQL expression:

![Define 15m Average Measure](/assets/2022-07-31-06-define-nested-measure.jpg)

Here is the SQL expression for copying:

```sql
PIVOT_NESTED_AGG(
    TIME_FLOOR(t.__time, 'PT15M'),
    AVG(t.val) AS "V",
    AVG(t."V"))
```

Display the new `15m Avg Val` measure instead of the naïve average. This time the result matches the requirement: 

![Display 15m Average Measure](/assets/2022-07-31-07-15m-result.jpg)

If you enable [query monitoring](https://docs.imply.io/latest/monitor-queries/), you can see that Pivot generates a nested query with two levels of `GROUP BY`:

```sql
SELECT
AVG(t."V") AS __VALUE__
FROM (SELECT
AVG(t.val) AS "V"
FROM "power_data" AS t
WHERE (TIMESTAMP '2022-07-30 10:19:00'<=(t."__time") AND (t."__time")<TIMESTAMP '2022-07-31 10:19:00')
GROUP BY TIME_FLOOR(t.__time, 'PT15M')) AS t
GROUP BY ()
```

The [Pivot documentation](https://docs.imply.io/latest/measures/#nested-aggregation-measures) has more examples of nested aggregations.

## Learnings

- Pivot offers multi step aggregations.
- This is valuable for timeseries data where measures are not evenly distributed in time.
- Use the `TIME_FLOOR` function to define intermediate buckets.

---

 <p class="attribution">"<a target="_blank" rel="noopener noreferrer" href="https://commons.wikimedia.org/w/index.php?curid=64634325">File:Mount Majura solar farm and Majura Parkway.jpg</a>" by <a target="_blank" rel="noopener noreferrer" href="https://commons.wikimedia.org/wiki/User:Grahamec">Grahamec</a> is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-sa/4.0/?ref=openverse">CC BY-SA 4.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>. </p> 
