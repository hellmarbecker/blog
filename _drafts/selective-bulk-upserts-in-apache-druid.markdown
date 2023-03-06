---
layout: post
title:  "Selective Bulk Upserts in Apache Druid"
categories: blog druid imply adtech tutorial update crud
---

[Apache Druid](https://druid.apache.org/) is designed for high query speed. The [data segments](https://druid.apache.org/docs/latest/design/segments.html) that make up a Druid datasource (think table) are generally immutable: You do not update or replace individual rows of data; however you can replace an entire segment by a new version of itself.

Sometimes in analytics, you have to update or insert rows of data in a segment. This may be due to a change in status - such as an order being shipped, or canceled, or returned. Generally, you would have a _key_ column in your data, and based on that key you would update a row if it exists in the table already, and insert it otherwise. This is called `upsert`, after the name of the command that is used in many SQL dialects.

[This Imply blog](https://imply.io/blog/upserts-and-data-deduplication-with-druid/) talks about the various strategies to handle such scenarios with Druid. But today, I want to look at a special case of Upsert, where you want to update or insert a bunch of rows based on a key and time interval.

## The use case

I encountered this scenario with some of my AdTech customers. They obtain performance analytics data by issuing API calls to the ad network providers. These API calls allow only certain predefined time ranges to specified - data is downloaded in bulk. Moreover, depending on late arriving conversion data and other factors, metrics associated with the data rows may change over time.

If we want to make these data available in Druid, we will have to cut out existing data by key and interval, and transplant the new data instead, like in this diagram: 

![Combining ingestion](/assets/2023-03-05-01.png)

## Solution outline

In order to achieve this behavior in Druid, we will use a [`combining` input source](https://druid.apache.org/docs/latest/ingestion/native-batch-input-sources.html#combining-input-source) in the ingestion spec. A combining input source contains a list of delegate input sources - we will use two, but you can actually have more than two.

The ingestion process will read data from all delegate input sources and ingest them, much like what a `union all` in SQL does.

We have to make sure that all input sources have the same schema and, where that applies, the same input format. In practice this means:

- you can combine multiple external sources only if they are all parsed in the same way
- or you can combine external sources like above with any number of `druid` input sources (reindexing).

The latter is what we are going to do.

## Tutorial: How to do it in practise

In this tutorial, we will set up a bulk upsert using the combining input source technique and two stripped down sample data set.

We will:
- load an initial data sample for multiple ad networks
- show the upsert technique by replacing data for one network and a specific date range.

The tutorial can be done using the [Druid 25.0 quickstart](https://druid.apache.org/docs/latest/tutorials/index.html).

Note: Because the tutorial assumes that you are running all Druid processes on a single machine, it can work with local file system data. In a cluster  setup, you would have to use a network mount or (more commonly) cloud storage, like S3.

### Initial load

The first data sample serves to populate the table. It has one week's worth of data from three networks:

```
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

Save this sample locally to a file named `daa1.json` and ingest it using this ingestion spec (replace the path in `baseDir` with the path you saved the sample file to):

```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "local",
        "baseDir": "/<my base path>",
        "filter": "data1.json"
      },
      "inputFormat": {
        "type": "json"
      },
      "appendToExisting": false
    },
    "tuningConfig": {
      "type": "index_parallel",
      "partitionsSpec": {
        "type": "hashed"
      },
      "forceGuaranteedRollup": true
    },
    "dataSchema": {
      "dataSource": "ad_data",
      "timestampSpec": {
        "column": "date",
        "format": "iso"
      },
      "dimensionsSpec": {
        "dimensions": [
          "ad_network",
          {
            "type": "long",
            "name": "ads_impressions"
          },
          {
            "name": "ads_revenue",
            "type": "double"
          }
        ]
      },
      "granularitySpec": {
        "queryGranularity": "none",
        "rollup": false,
        "segmentGranularity": "week"
      }
    }
  }
}
```

You can create this ingestion spec by clicking through the console wizard, too. There are a few notable settings here though:

- I've used hash partitioning over all partitions here. The default in the wizard is dynamic partitioning, but you would usually use dymanic partitioning with batch data only if you want to append data to an existing data sets. In all other cases, use hash or range partitioning.
- I've configured weekly segments. This is to show that the technique works even if the updated range does not align with segment boundaries. 

### Doing the upsert

Now, let's fast-forward two days in time. We have downloaded a bunch of new and updated data from the `gaggle` network. The new data looks like this:

```json
{"date": "2023-01-03T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 4521, "ads_revenue": 378.65}
{"date": "2023-01-04T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 4330, "ads_revenue": 464.02}
{"date": "2023-01-05T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 6088, "ads_revenue": 320.57}
{"date": "2023-01-06T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 3417, "ads_revenue": 162.77}
{"date": "2023-01-07T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 9762, "ads_revenue": 76.27}
{"date": "2023-01-08T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 1484, "ads_revenue": 188.17}
{"date": "2023-01-09T00:00:00Z", "ad_network": "gaagle", "ads_impressions": 1845, "ads_revenue": 287.5}
```

Save this sample as `data2.json` and proceed to replace/insert the new data using this spec:

```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": { 
        "type": "combining",
        "delegates": [
          {
            "type": "druid",
            "dataSource": "ad_data",
            "interval": "1000/3000",
            "filter": {
              "type": "not",
              "field": {
                "type": "and",
                "fields": [
                  {
                    "type": "selector",
                    "dimension": "ad_network",
                    "value": "gaagle"
                  },
                  {
                    "type": "interval",
                    "dimension": "__time",
                    "intervals": [
                      "2023-01-03T00:00:00Z/2023-01-10T00:00:00Z"
                    ],
                    "extractionFn": null
                  }
                ]
              }
            }      
          },
          {
            "type": "local",
            "files": ["/Users/hellmarbecker/meetup-talks/upsert/data2.json"]
          }
        ]
      },
      "inputFormat": {
        "type": "json"
      }
    },
    "tuningConfig": {
      "type": "index_parallel",
      "partitionsSpec": {
        "type": "hashed"
      },
      "forceGuaranteedRollup": true,
      "maxNumConcurrentSubTasks": 2
    },
    "dataSchema": {
      "timestampSpec": {
        "column": "__time",
        "missingValue": "2010-01-01T00:00:00Z"
      },
      "transformSpec": {
        "transforms": [
          {
            "name": "__time",
            "type": "expression",
            "expression": "nvl(timestamp_parse(date), \"__time\")"
          }
        ]
      },
      "granularitySpec": {
        "rollup": false,
        "queryGranularity": "none",
        "segmentGranularity": "week",
        "intervals": ["1000/3000"]
      },
      "dimensionsSpec": {
        "dimensions": [
          {
            "name": "ad_network",
            "type": "string"
          },
          {
            "name": "ads_impressions",
            "type": "long"
          },
          {
            "name": "ads_revenue",
            "type": "double"
          }
        ]
      },
      "dataSource": "ad_data"
    }
  }
}
```

Here's the result of a `SELECT *` query after the ingestion finishes:

__time|ad_network|ads_impressions|ads_revenue
---|---|---|---
2023-01-01T00:00:00.000Z|fakebook|9646|137.85
2023-01-01T00:00:00.000Z|gaagle|2770|330.69
2023-01-01T00:00:00.000Z|twottr|1139|493.73
2023-01-02T00:00:00.000Z|fakebook|4426|170.96
2023-01-02T00:00:00.000Z|gaagle|9066|368.66
2023-01-02T00:00:00.000Z|twottr|9110|452.2
2023-01-03T00:00:00.000Z|fakebook|9494|426.37
_2023-01-03T00:00:00.000Z_|_gaagle_|_4521_|_378.65_
2023-01-03T00:00:00.000Z|twottr|4325|107.44
2023-01-04T00:00:00.000Z|fakebook|8955|254.5
_2023-01-04T00:00:00.000Z_|_gaagle_|_4330_|_464.02_
2023-01-04T00:00:00.000Z|twottr|6905|211.74
2023-01-05T00:00:00.000Z|fakebook|4870|205.84
_2023-01-05T00:00:00.000Z_|_gaagle_|_6088_|_320.57_
2023-01-05T00:00:00.000Z|twottr|1418|282.21
2023-01-06T00:00:00.000Z|fakebook|1251|265.52
_2023-01-06T00:00:00.000Z_|_gaagle_|_3417_|_162.77_
2023-01-06T00:00:00.000Z|twottr|8055|394.56
2023-01-07T00:00:00.000Z|fakebook|5848|162.96
_2023-01-07T00:00:00.000Z_|_gaagle_|_9762_|_76.27_
2023-01-07T00:00:00.000Z|twottr|9449|379.21
_2023-01-08T00:00:00.000Z_|_gaagle_|_1484_|_188.17_
_2023-01-09T00:00:00.000Z_|_gaagle_|_1845_|_287.5_

Note how all the rows in _italics_ come from the second data set. They have either been inserted (the last two rows), or they replace previous rows for the same time interval and network.

### Taking a closer look

Let's go through some interesting points in the ingestion spec.

#### The input sources

- combining input source is like a UNION
- delegates are the parts of the union, they can be any inputsource, there can be more than 2
- #1: reindexes the existing data
  - there is a filter that leaves out the interval and key we want to replace
  - filters are a kind of boolean prefix notation, they tell us which rows to _keep_
  - so here it is: not(and(network_key=gaggle, timestamp in \[interval\]))
- #2: pulls in the new data
  - logical complement of filter #1: and(network_key=gaggle, timestamp in \[interval\]) but cannot do this explicitly
- schema matches but not quite

#### Schema alignment: Timestamp definition

#### Tuning configuration

The documentation mentions that

> The secondary partitioning method determines the requisite number of concurrent worker tasks that run in parallel to complete ingestion with the Combining input source. Set this value in `maxNumConcurrentSubTasks` in `tuningConfig` based on the secondary partitioning method:
>
> - `range` or `single_dim` partitioning: greater than or equal to 1
> - `hashed` or `dynamic` partitioning: greater than or equal to 2

**This advice is to be taken seriously.** If you try to run with an insufficient number of subtasks you will get a highly misleading error message that looks like:

```
java.lang.UnsupportedOperationException: Implement this method properly if needsFormat() = true
```

Make sure you configure at least two concurrent subtasks if you are using `hashed` or `dynamic` partitioning.

and done!

## Conclusion

yada yada



