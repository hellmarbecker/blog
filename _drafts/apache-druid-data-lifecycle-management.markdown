---
layout: post
title:  "Apache Druid: Data Lifecycle Management"
categories: blog druid tutorial
---

![Rollup in Druid](/assets/2023-01-22-01-rollup.png)

One feature that contributes greatly to the performance of Druid is [rollup](https://druid.apache.org/docs/latest/ingestion/rollup.html). Rolling up converts a detail table into an aggregate table that contains

- a _truncated timestamp_: the granularity of that timestamp is known as the _query granularity_
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

### Data Generation

Let's generate ourselves some data. I want to create a data set with 6 months' worth of data, 10 records per day, and each record has a random user ID because I want to count unique users as well.

This little Python script does the job:

```python
import random
from datetime import date, timedelta

MAXUSERS = 1000
ROWSPERDAY = 10

def daterange(start_date, end_date):
    for n in range(int((end_date - start_date).days)):
        yield start_date + timedelta(n)

def main():

    start_date = date(2022, 1, 1)
    end_date = date(2022, 7, 1)
    for single_date in daterange(start_date, end_date):
        this_day = single_date.strftime("%Y-%m-%d")
        for i in range(ROWSPERDAY):
            print(this_day + ",u" + "{:04d}".format(random.randrange(MAXUSERS)))


if __name__ == "__main__":
    main()
```

Run the script and save the output to a file `data.csv`.

### Data Ingestion

Let's ingest the data. Here is the ingestion spec:

```json
{
  "type": "index_parallel",
  "spec": {
    "dataSchema": {
      "dataSource": "user_data",
      "timestampSpec": {
        "column": "ts",
        "format": "auto"
      },
      "dimensionsSpec": {
        "dimensions": [
          "dummy1",
          "dummy2"
        ]
      },
      "metricsSpec": [
        {
          "type": "count",
          "name": "__count"
        },
        {
          "type": "thetaSketch",
          "name": "theta_user",
          "fieldName": "user",
          "size": 16384
        },
        {
          "type": "HLLSketchBuild",
          "name": "hll_user",
          "fieldName": "user",
          "lgK": 12,
          "tgtHllType": "HLL_4"
        }
      ],
      "granularitySpec": {
        "type": "uniform",
        "segmentGranularity": "DAY",
        "queryGranularity": "DAY",
        "rollup": true
      },
      "transformSpec": {
        "transforms": [
          {
            "type": "expression",
            "name": "dummy1",
            "expression": "'A'"
          },
          {
            "type": "expression",
            "name": "dummy2",
            "expression": "'B'"
          }
        ]
      }
    },
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "local",
        "baseDir": "/Users/hellmarbecker/meetup-talks/data-lifecycle",
        "filter": "data.csv"
      },
      "inputFormat": {
        "type": "csv",
        "columns": [
          "ts",
          "user"
        ],
        "findColumnsFromHeader": false
      },
      "appendToExisting": false,
      "dropExisting": false
    },
    "tuningConfig": {
      "type": "index_parallel",
      "maxRowsPerSegment": 5000000,
      "maxRowsInMemory": 1000000,
      "partitionsSpec": {
        "type": "range",
        "maxRowsPerSegment": 5000000,
        "partitionDimensions": [
          "dummy1",
          "dummy2"
        ]
      },
      "forceGuaranteedRollup": true
    }
  }
}
```

A few notes:

- I am rolling up by day and also creating daily segments.
- I created two dummy dimensions using [transforms](/2022/02/09/druid-data-cookbook-ingestion-transforms/). This is so I can enforce [range partitioning](/2022/01/25/partitioning-in-druid-part-3-multi-dimension-range-partitioning/) and make sure it is preserved during my lifecycle management operations.

You can submit the ingestion task using the API call above, or by pasting into the Druid console wizard.

### Querying the Data

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

Note this query down, we will repeat it after every step.

### Reference Result

![Query in Druid Workbench](/assets/2023-01-22-02-query.jpg)

Once the data has been ingested, run the above query. Here's my result - your unique user counts will likely be different:

date|numRowsCompacted|numRowsOrig|uniqueUsersTheta
---|---|---|---
2022-01-01T00:00:00.000Z|31|310|266
2022-02-01T00:00:00.000Z|28|280|244
2022-03-01T00:00:00.000Z|31|310|258
2022-04-01T00:00:00.000Z|30|300|263
2022-05-01T00:00:00.000Z|31|310|266
2022-06-01T00:00:00.000Z|30|300|257
_null_|181|1810|838

We have 1810 events, aggregated into 181 database rows. Each month's event coount is 10x the number of days in the month.

## Compaction of Older Data

Let's first implement a simple compaction strategy. All data except the latest month should be rolled up monthly. Here's the compaction spec for this:

```json
{
  "type": "compact",
  "dataSource": "user_data",
  "ioConfig": {
    "type": "compact",
    "inputSpec": {
      "type": "interval",
      "interval": "2022-01-01/2022-06-01"
    }
  },
  "granularitySpec": {
    "segmentGranularity": "month",
    "queryGranularity": "month"
  },
  "tuningConfig": {
    "type": "index_parallel",
    "maxRowsPerSegment": 5000000,
    "maxRowsInMemory": 1000000,
    "partitionsSpec": {
      "type": "range",
      "maxRowsPerSegment": 5000000,
      "partitionDimensions": [
        "dummy1",
        "dummy2"
      ]
    },
    "forceGuaranteedRollup": true
  }
}
```

We chose the segment granularity and query granularity both to be a month; the `tuningConfig` section has been copied from the ingestion spec.

<mark>If you do not specify the `tuningConfig -> partitionsSpec`, you will end up with dynamic partitioning, which is usually not what you want.</mark>

Submit the compaction task and wait for it to finish. Then run the same query again. Here is my result:










