---
layout: post
title: "Druid Data Cookbook: Upserts in Druid SQL"
categories: blog druid sql imply adtech tutorial update crud
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

In [an earlier blog](/2023/03/07/selective-bulk-upserts-in-apache-druid/), I demonstrated a technique to combine existing and new data in Druid batch ingestion in a way that more or less emulates what is usually expressed in SQL as a `MERGE` or `UPSERT` statement. This technique involves a `combine` datasource and works only in JSON-based ingestion. Also, it works on bulk data where you replace an entire range of data based on a time interval and key range.

Today I am going to look at a similar, albeit more surgical construction, implementing what is usually expressed in SQL as a `MERGE` or `UPSERT` statement. I will be using [SQL based ingestion](https://druid.apache.org/docs/latest/multi-stage-query/) that is available in newer versions of Druid.

The `MERGE` statement, in a simplified way, works like this:

```sql
MERGE INTO druid_table
(SELECT * FROM external_table)
ON druid_table.keys = external_table.keys
WHEN MATCHED THEN UPDATE ...
WHEN NOT MATCHED THEN INSERT ...
```

So, you compare old _(druid_table)_ and new data _(external_table)_ with respect to a _matching condition_. This would entail a combination of timestamp and key fields, which in the above pseudocode is denoted by _keys_. There are three possible outcomes for any combination of _keys:_

1. If _keys_ exists only in _druid table_, leave these data untouched.
2. If _keys_ exists in both tables, replace the row(s) in _druid_table_ with those in _external_table_.
3. If _keys_ exists only in _external_table_, insert that data into _druid_table_.

But Druid SQL does not offer a `MERGE` statement, at least not at the time of this writing. Can we do this in SQL anyway? Stay tuned if you want to know!

This tutorial works with [the Druid 28 quickstart](https://druid.apache.org/docs/latest/tutorials/).

## Recap: the data

Let's use the same data as in [the bulk upsert blog](/2023/03/07/selective-bulk-upserts-in-apache-druid/):

```json
{"date": "2023-01-01T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 2770, "ads_revenue": 330.69}
{"date": "2023-01-01T00:00:00Z", "ad_network": "fakebook", "ads_impressions": 9646, "ads_revenue": 137.85}
{"date": "2023-01-01T00:00:00Z", "ad_network": "twottr", "ads_impressions": 1139, "ads_revenue": 493.73}
{"date": "2023-01-02T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 9066, "ads_revenue": 368.66}
{"date": "2023-01-02T00:00:00Z", "ad_network": "fakebook", "ads_impressions": 4426, "ads_revenue": 170.96}
{"date": "2023-01-02T00:00:00Z", "ad_network": "twottr", "ads_impressions": 9110, "ads_revenue": 452.2}
{"date": "2023-01-03T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 3275, "ads_revenue": 363.53}
{"date": "2023-01-03T00:00:00Z", "ad_network": "fakebook", "ads_impressions": 9494, "ads_revenue": 426.37}
{"date": "2023-01-03T00:00:00Z", "ad_network": "twottr", "ads_impressions": 4325, "ads_revenue": 107.44}
{"date": "2023-01-04T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 8816, "ads_revenue": 311.53}
{"date": "2023-01-04T00:00:00Z", "ad_network": "fakebook", "ads_impressions": 8955, "ads_revenue": 254.5}
{"date": "2023-01-04T00:00:00Z", "ad_network": "twottr", "ads_impressions": 6905, "ads_revenue": 211.74}
{"date": "2023-01-05T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 3075, "ads_revenue": 382.41}
{"date": "2023-01-05T00:00:00Z", "ad_network": "fakebook", "ads_impressions": 4870, "ads_revenue": 205.84}
{"date": "2023-01-05T00:00:00Z", "ad_network": "twottr", "ads_impressions": 1418, "ads_revenue": 282.21}
{"date": "2023-01-06T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 7413, "ads_revenue": 322.43}
{"date": "2023-01-06T00:00:00Z", "ad_network": "fakebook", "ads_impressions": 1251, "ads_revenue": 265.52}
{"date": "2023-01-06T00:00:00Z", "ad_network": "twottr", "ads_impressions": 8055, "ads_revenue": 394.56}
{"date": "2023-01-07T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 4279, "ads_revenue": 317.84}
{"date": "2023-01-07T00:00:00Z", "ad_network": "fakebook", "ads_impressions": 5848, "ads_revenue": 162.96}
{"date": "2023-01-07T00:00:00Z", "ad_network": "twottr", "ads_impressions": 9449, "ads_revenue": 379.21}
```

Save this file as `data1.json`. Also, save the "new data" bit:

```json
{"date": "2023-01-03T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 4521, "ads_revenue": 378.65}
{"date": "2023-01-04T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 4330, "ads_revenue": 464.02}
{"date": "2023-01-05T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 6088, "ads_revenue": 320.57}
{"date": "2023-01-06T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 3417, "ads_revenue": 162.77}
{"date": "2023-01-07T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 9762, "ads_revenue": 76.27}
{"date": "2023-01-08T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 1484, "ads_revenue": 188.17}
{"date": "2023-01-09T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 1845, "ads_revenue": 287.5}
```

as `data2.json`.

## Initial data ingestion

Let's ingest the first data set. We want to set the segment granularity to `month`, so the ingestion statement uses a `PARTITIONED BY MONTH` clause. Moreover, we enforce secondary partitioning by choosing `REPLACE` mode and by including a `CLUSTERED BY` clause. Here's the complete statement (replace the path in `baseDir` with the path you saved the sample file to):

```sql
REPLACE INTO "ad_data" OVERWRITE ALL
WITH "ext" AS (
  SELECT *
  FROM TABLE(
    EXTERN(
      '{"type":"local","baseDir":"/<my base path>","filter":"data1.json"}',
      '{"type":"json"}'
    )
  ) EXTEND ("date" VARCHAR, "ad_network" VARCHAR, "ads_impressions" BIGINT, "ads_revenue" DOUBLE)
)
SELECT
  TIME_PARSE("date") AS "__time",
  "ad_network",
  "ads_impressions",
  "ads_revenue"
FROM "ext"
PARTITIONED BY MONTH
CLUSTERED BY "ad_network"
```

## The technique

we do have UNION in msq but it is too brain damaged - we cannot filter 

so instead, use the 80s technique of doing a full outer join

give credit to john sergio 

## the merge query

note: this needs `{ "sqlJoinAlgorithm": "sortMerge" }` in the context

--> context screenshot

then you can run this query

--> quote merge sql

## putting it all together

--> replace overwrite all query

## can we be more selective?

--> replace overwrite week

--> show error

it has to be segment aligned! if not druid will not let you

this one works:

--> replace overwrite month

and so we get the update data:

--> paste table

--> screenshot explore, timeline stacked by network

## Learnings

- yada yada
- 

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
