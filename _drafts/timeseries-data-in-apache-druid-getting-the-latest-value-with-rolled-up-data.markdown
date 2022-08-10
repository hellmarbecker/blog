---
layout: post
title:  "Timeseries Data in Apache Druid: Getting the Latest Value with Rolled Up Data"
categories: blog druid imply timeseries analytics tutorial
---

![Stock ticker image]()

Imagine you are analyzing time based data and your most frequent queries are about retrieving the latest values for each time interval. One example would be stock ticker data where you would most likely want to know the closing prices for each trading day. IoT related use cases are another scenario where this would be useful.

This is easy if you use detail data. After all, Druid comes with [`LATEST`](https://docs.imply.io/latest/druid/querying/sql-functions/#latest) and [`EARLIEST`](https://docs.imply.io/latest/druid/querying/sql-functions/#earliest) aggregators that give you exactly what you asked for. But you cannot use rollup because that would limit the granularity, and it would lead to incorrect results with imperfect rollup such as in realtime ingestion. That is why you cannot use those aggregators during rollup.

Or can you?

## Using `LATEST` with Rollup

It turns out that you actually _can_ use a `LATEST` (or `EARLIEST`) aggregator during rollup. This behavior is currently implemented only for the `STRING` flavor of the aggregator: if you want to aggregate over a numeric field, you will have to use a cast. Hopefully this will change in a future release of Druid.

Here is a minimalistic data sample with stock prices.

```csv
timestamp,symbol,price
2022-08-10 10:01:00,AAAA,25.90
2022-08-10 10:11:00,AAAA,26.90
2022-08-10 11:55:00,AAAA,28.10
```

Ingest those data with hourly rollup. The `LATEST` aggregator is not available in the ingestion wizard, so you will have to edit the ingestion spec manually. It should look like this:

```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "inline",
        "data": "timestamp,symbol,price\n2022-08-10 10:01:00,AAAA,25.90\n2022-08-10 10:11:00,AAAA,26.90\n2022-08-10 11:55:00,AAAA,28.10"
      },
      "inputFormat": {
        "type": "csv",
        "findColumnsFromHeader": true
      }
    },
    "tuningConfig": {
      "type": "index_parallel",
      "partitionsSpec": {
        "type": "hashed"
      },
      "forceGuaranteedRollup": true
    },
    "dataSchema": {
      "dataSource": "ticker_data",
      "timestampSpec": {
        "column": "timestamp",
        "format": "auto"
      },
      "dimensionsSpec": {
        "dimensions": [
          "symbol"
        ]
      },
      "granularitySpec": {
        "queryGranularity": "hour",
        "rollup": true,
        "segmentGranularity": "hour"
      },
      "metricsSpec": [
        {
          "name": "count",
          "type": "count"
        },
        {
          "name": "last_price",
          "type": "stringLast",
          "fieldName": "price"
        }
      ]
    }
  }
}
```

What have we got now?  A glance at the _Segments_ view shows that there are two (hourly) segments with one row each:

![Segments view with hourly segments](/assets/2022-08-10-01-segments.jpg)

Let's query the data:

![Scan all data](/assets/2022-08-10-02-complex-latest.jpg)

This is interesting. The `last_price` column is not a single value but a tuple of timestamp and value. Druid stored an intermediate result for us. Getting the numeric value needs another aggregation step by means of a `GROUP BY` query:

```sql
SELECT TIME_FLOOR(__time, 'PT1H'), symbol, CAST(LATEST(last_price, 1024) AS DOUBLE)
FROM ticker_data
GROUP BY 1, 2
```

![Aggregate query](/assets/2022-08-10-03-latest-latest.jpg)

## Adding More Data

What happens if more data is added? These will end up in extra segments until a compaction is executed. Let's add a small data set.

```csv
timestamp,symbol,price
2022-08-10 10:50:00,AAAA,23.90
2022-08-10 11:20:00,AAAA,22.10
```

Note how the first new row is later than existing rows for that hour. The second new row isn't.

To add these data, we have to make a few changes to the ingestion spec:

![New ingestion spec](/assets/2022-08-10-04-dyn.jpg)

- Set `appendToExisting` to `true` in `ioConfig`
- The partition strategy has to be set to `dynamic`
- Unset `forceGuaranteedRollup`

We now have four segments. The two previous segments have shard type `hashed`, which is the default; the new segments have type `numbered` which corresponds to dynamic partitioning.

![New segment list](/assets/2022-08-10-05-segments2.jpg)

Run the same aggregation quety as before again.

![Aggregate query again](/assets/2022-08-10-06-latest-latest2.jpg)

The result is as expected. Druid has correctly picked up the latest value for each hourly interval.

## Learnings

- Druid can use `LATEST` or `EARLIEST` aggregations during ingestion.
- Druid stores these aggregations in an intermediate representation that allows for adding more data later. 
- In the current version, this requires working with `SRTING` fields.
- For numeric fields, you can work around this limitation using a `CAST` expression.
