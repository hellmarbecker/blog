---
layout: post
title:  "Druid Sneak Peek: Timeseries Interpolation"
categories: blog apache druid imply iot sql tutorial
---
![Druid Cookbook](/assets/2023-04-08-01-hotandcold.jpg)

Today I am going to look at another new Druid feature.

This is currently only available in [Imply Enterprise](https://imply.io/download-imply/) which ships with all the featured discussed today, and comes with a free 30 day trial license. I sure hope it will come to open source Druid too.

In this tutorial, you will 

- ingest a data sample and
- run a query to fill in missing values at regular time intervals, using a simple linear interpolation scheme.

Why is it cool? It uses 

-  the new `UNNEST` function, which takes a collection and joins it laterally against the main table
-  the new `DATE_EXPAND` function, which takes a start and end date and a step interval, and created an array of timestamps, spaced by the step interval, between the start and end points
-  [window functions](/2023/03/26/druid-26-sneak-peek-window-functions/), in our case the `LEAD` function to retrieve values from the succeeding row.

## The data sample

Today's data set is a simple time series of temperature measurements, which have been conducted every 6 hours:

```csv
date_start,temperature
2023-04-07T00:00:00Z,5
2023-04-07T06:00:00Z,8
2023-04-07T12:00:00Z,14
2023-04-07T18:00:00Z,12
2023-04-08T00:00:00Z,3
2023-04-08T06:00:00Z,6
2023-04-08T12:00:00Z,11
2023-04-08T18:00:00Z,5
```

We would like to fill the gaps, interpolating temperature values for each hour between the measurements.

Let's ingest the data into Druid.

The ingestion spec is straightforward:

```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "inline",
        "data": "date_start,temperature\n2023-04-07T00:00:00Z,5\n2023-04-07T06:00:00Z,8\n2023-04-07T12:00:00Z,14\n2023-04-07T18:00:00Z,12\n2023-04-08T00:00:00Z,3\n2023-04-08T06:00:00Z,6\n2023-04-08T12:00:00Z,11\n2023-04-08T18:00:00Z,5"
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
      "dataSource": "iot_data",
      "timestampSpec": {
        "column": "date_start",
        "format": "iso"
      },
      "granularitySpec": {
        "queryGranularity": "none",
        "rollup": false,
        "segmentGranularity": "month"
      },
      "dimensionsSpec": {
        "dimensions": [
          {
            "type": "double",
            "name": "temperature"
          }
        ]
      }
    }
  }
}
```

## Query the data

Both the window functions and the `UNNEST` function are currently hidden behind context flags. Use the following query context:

```json
{
  "windowsAreForClosers": true,
  "enableUnnest": true
}
```

With that, here is the query:

```sql
WITH cte AS (
  SELECT 
    __time AS thisTime, 
    temperature,
    LEAD(__time,  1) OVER (ORDER BY __time) nextTime,
    LEAD(temperature,  1) OVER (ORDER BY __time) nextTemperature
  FROM "iot_data"
  GROUP BY 1,2
)
SELECT
  thisTime, nextTime, timeByHour, temperature, nextTemperature,
  CASE (TIMESTAMP_TO_MILLIS(nextTime) - TIMESTAMP_TO_MILLIS(thisTime)) 
    WHEN 0 THEN temperature
    ELSE ((TIMESTAMP_TO_MILLIS(nextTime) - TIMESTAMP_TO_MILLIS(timeByHour)) * temperature 
            + (TIMESTAMP_TO_MILLIS(timeByHour) - TIMESTAMP_TO_MILLIS(thisTime)) * nextTemperature) 
          / (TIMESTAMP_TO_MILLIS(nextTime) - TIMESTAMP_TO_MILLIS(thisTime))
    END interpTemp
FROM cte, UNNEST(DATE_EXPAND(TIMESTAMP_TO_MILLIS(thisTime), TIMESTAMP_TO_MILLIS(NVL(nextTime, thisTime)), 'PT1H')) AS t(timeByHour)
WHERE timeByHour <> nextTime
```

It uses the common table expression technique that already came in handy last time.

Here is the result:

thisTime|nextTime|timeByHour|temperature|nextTemperature|interpTemp
---|---|---|---|---|---
2023-04-07T00:00:00.000Z|2023-04-07T06:00:00.000Z|2023-04-07T00:00:00.000Z|5|8|5
2023-04-07T00:00:00.000Z|2023-04-07T06:00:00.000Z|2023-04-07T01:00:00.000Z|5|8|5.5
2023-04-07T00:00:00.000Z|2023-04-07T06:00:00.000Z|2023-04-07T02:00:00.000Z|5|8|6
2023-04-07T00:00:00.000Z|2023-04-07T06:00:00.000Z|2023-04-07T03:00:00.000Z|5|8|6.5
2023-04-07T00:00:00.000Z|2023-04-07T06:00:00.000Z|2023-04-07T04:00:00.000Z|5|8|7
2023-04-07T00:00:00.000Z|2023-04-07T06:00:00.000Z|2023-04-07T05:00:00.000Z|5|8|7.5
2023-04-07T06:00:00.000Z|2023-04-07T12:00:00.000Z|2023-04-07T06:00:00.000Z|8|14|8
2023-04-07T06:00:00.000Z|2023-04-07T12:00:00.000Z|2023-04-07T07:00:00.000Z|8|14|9
2023-04-07T06:00:00.000Z|2023-04-07T12:00:00.000Z|2023-04-07T08:00:00.000Z|8|14|10
2023-04-07T06:00:00.000Z|2023-04-07T12:00:00.000Z|2023-04-07T09:00:00.000Z|8|14|11
2023-04-07T06:00:00.000Z|2023-04-07T12:00:00.000Z|2023-04-07T10:00:00.000Z|8|14|12
2023-04-07T06:00:00.000Z|2023-04-07T12:00:00.000Z|2023-04-07T11:00:00.000Z|8|14|13
2023-04-07T12:00:00.000Z|2023-04-07T18:00:00.000Z|2023-04-07T12:00:00.000Z|14|12|14
2023-04-07T12:00:00.000Z|2023-04-07T18:00:00.000Z|2023-04-07T13:00:00.000Z|14|12|13.666666666666666
2023-04-07T12:00:00.000Z|2023-04-07T18:00:00.000Z|2023-04-07T14:00:00.000Z|14|12|13.333333333333334
2023-04-07T12:00:00.000Z|2023-04-07T18:00:00.000Z|2023-04-07T15:00:00.000Z|14|12|13
2023-04-07T12:00:00.000Z|2023-04-07T18:00:00.000Z|2023-04-07T16:00:00.000Z|14|12|12.666666666666666
2023-04-07T12:00:00.000Z|2023-04-07T18:00:00.000Z|2023-04-07T17:00:00.000Z|14|12|12.333333333333334
2023-04-07T18:00:00.000Z|2023-04-08T00:00:00.000Z|2023-04-07T18:00:00.000Z|12|3|12
2023-04-07T18:00:00.000Z|2023-04-08T00:00:00.000Z|2023-04-07T19:00:00.000Z|12|3|10.5
2023-04-07T18:00:00.000Z|2023-04-08T00:00:00.000Z|2023-04-07T20:00:00.000Z|12|3|9
2023-04-07T18:00:00.000Z|2023-04-08T00:00:00.000Z|2023-04-07T21:00:00.000Z|12|3|7.5
2023-04-07T18:00:00.000Z|2023-04-08T00:00:00.000Z|2023-04-07T22:00:00.000Z|12|3|6
2023-04-07T18:00:00.000Z|2023-04-08T00:00:00.000Z|2023-04-07T23:00:00.000Z|12|3|4.5
2023-04-08T00:00:00.000Z|2023-04-08T06:00:00.000Z|2023-04-08T00:00:00.000Z|3|6|3
2023-04-08T00:00:00.000Z|2023-04-08T06:00:00.000Z|2023-04-08T01:00:00.000Z|3|6|3.5
2023-04-08T00:00:00.000Z|2023-04-08T06:00:00.000Z|2023-04-08T02:00:00.000Z|3|6|4
2023-04-08T00:00:00.000Z|2023-04-08T06:00:00.000Z|2023-04-08T03:00:00.000Z|3|6|4.5
2023-04-08T00:00:00.000Z|2023-04-08T06:00:00.000Z|2023-04-08T04:00:00.000Z|3|6|5
2023-04-08T00:00:00.000Z|2023-04-08T06:00:00.000Z|2023-04-08T05:00:00.000Z|3|6|5.5
2023-04-08T06:00:00.000Z|2023-04-08T12:00:00.000Z|2023-04-08T06:00:00.000Z|6|11|6
2023-04-08T06:00:00.000Z|2023-04-08T12:00:00.000Z|2023-04-08T07:00:00.000Z|6|11|6.833333333333333
2023-04-08T06:00:00.000Z|2023-04-08T12:00:00.000Z|2023-04-08T08:00:00.000Z|6|11|7.666666666666667
2023-04-08T06:00:00.000Z|2023-04-08T12:00:00.000Z|2023-04-08T09:00:00.000Z|6|11|8.5
2023-04-08T06:00:00.000Z|2023-04-08T12:00:00.000Z|2023-04-08T10:00:00.000Z|6|11|9.333333333333334
2023-04-08T06:00:00.000Z|2023-04-08T12:00:00.000Z|2023-04-08T11:00:00.000Z|6|11|10.166666666666666
2023-04-08T12:00:00.000Z|2023-04-08T18:00:00.000Z|2023-04-08T12:00:00.000Z|11|5|11
2023-04-08T12:00:00.000Z|2023-04-08T18:00:00.000Z|2023-04-08T13:00:00.000Z|11|5|10
2023-04-08T12:00:00.000Z|2023-04-08T18:00:00.000Z|2023-04-08T14:00:00.000Z|11|5|9
2023-04-08T12:00:00.000Z|2023-04-08T18:00:00.000Z|2023-04-08T15:00:00.000Z|11|5|8
2023-04-08T12:00:00.000Z|2023-04-08T18:00:00.000Z|2023-04-08T16:00:00.000Z|11|5|7
2023-04-08T12:00:00.000Z|2023-04-08T18:00:00.000Z|2023-04-08T17:00:00.000Z|11|5|6
2023-04-08T18:00:00.000Z|null|2023-04-08T18:00:00.000Z|5|null|5

As you can see in the last columns, the values have been neatly interpolated.

## Side Quests

It is worth looking at some details of the query. Some of these are just common, others are due to quirks in the Druid query engine.

### The date expansion

```sql
DATE_EXPAND(TIMESTAMP_TO_MILLIS(thisTime), TIMESTAMP_TO_MILLIS(NVL(nextTime, thisTime)), 'PT1H')
```

The general syntax would be `DATE_EXPAND(from, to, interval)`. But since we are using `LEAD()` to get the `to` value, the last row will have _null_ in that place. Unfortunately, `DATE_EXPAND` doesn't handle that situation well and the query fails. That's why in the case of a _null_ value, I use the row time instead, generating only one row.

```sql
WHERE timeByHour <> nextTime
```

`DATE_EXPAND` considers its time interval as left and right inclusive. This means that the end values will be duplicated with the start values of the next interval. The `WHERE` clause filters out the duplicates.

### The interpolation

```sql
  CASE (TIMESTAMP_TO_MILLIS(nextTime) - TIMESTAMP_TO_MILLIS(thisTime)) 
    WHEN 0 THEN temperature ...
```

The general formula for linear interpolation has to divide by the total time range. If this is 0, just use the one value that is provided. This comes from the corner case treatment above.

## Conclusion

Druid's timeseries capabilities are ever expanding.

- With `DATE_EXPAND` and `UNNEST`, it is possible to generate even spaced time series.
- Using window functions and standard interpolation algorithms, this can be used to fill in missing values.
- Currently this is only available in Imply's release.

---

 <p class="attribution">"<a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/53575715@N02/6620214217">Hot & Cold</a>" by <a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/53575715@N02">astronomy_blog</a> is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/?ref=openverse">CC BY-NC-SA 2.0
  <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/>
  <img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/>
  <img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/>
  <img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>. </p> 
