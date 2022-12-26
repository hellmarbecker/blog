---
layout: post
title:  "Ingesting Data Sketches into Apache Druid"
categories: blog imply druid tutorial
---

[Data Sketches](https://youtu.be/Hpd3f_MLdXo?t=398) are a powerful way to get fast approximations for a bunch of measures that are very expensive to compute precisely. This includes [distinct counts](link) and [quantiles](link).

While data sketches can be computed directly within Druid, some of my customers prefer to roll up the data and compute the sketches in the preprocessing stage. Today, I am going to look at a few scenarios around ingesting these sketches.

This is a sneak peek into Druid 25 functionality. In order to use the new functions, you can (as of the time of writing) [build](https://druid.apache.org/docs/latest/development/build.html) the current release candidate:

```bash
git clone git@github.com:apache/druid.git
cd druid
git checkout druid-25.0.0-rc1
mvn clean install -Pdist -DskipTests
```

Then follow the instructions to locate and install the tarball.

_**Disclaimer:** This tutorial uses undocumented functionality. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. Your mileage may vary._

## Generating the Data Sample

First, let's generate our data set that has precomputed data sketches. I am going to use Druid for this; in a real project, you might use Spark or whatever preprocessing you have in place.

I am using the Wikipedia sample data that comes with each Druid version. This is one day's worth of Wikipedia edits, which for the purpose of this tutorial, I will roll up by hour. I am interested in unique users by hour and channel, and I will model those both as HLL sketches and theta sketches. Here is the ingestion spec for this:

```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "http",
        "uris": [
          "https://druid.apache.org/data/wikipedia.json.gz"
        ]
      },
      "inputFormat": {
        "type": "json"
      }
    },
    "dataSchema": {
      "granularitySpec": {
        "segmentGranularity": "day",
        "queryGranularity": "hour",
        "rollup": true
      },
      "dataSource": "wikipedia-rollup-00",
      "timestampSpec": {
        "column": "timestamp",
        "format": "iso"
      },
      "dimensionsSpec": {
        "dimensions": [
          "channel"
        ]
      },
      "metricsSpec": [
        {
          "name": "__count",
          "type": "count"
        },
        {
          "name": "theta_user",
          "type": "thetaSketch",
          "fieldName": "user"
        },
        {
          "type": "HLLSketchBuild",
          "name": "hll_user",
          "fieldName": "user"
        }
      ]
    },
    "tuningConfig": {
      "type": "index_parallel",
      "partitionsSpec": {
        "type": "range",
        "partitionDimensions": [
          "channel"
        ],
        "targetRowsPerSegment": 5000000
      },
      "forceGuaranteedRollup": true
    }
  }
}
```

Let's export these data into a newline delimited JSON file. This can be done using Druid's SQL API. There are a few caveats:

- By default, Druid returns the reault set in a JSON array. For playing back the data, we need newline delimied JSON. A quick incantation of `jq` fixes this.
- If you select any sketch field in a query, it is automatically converted into a base 64 encoded string and surrounded with an extra set of double quotes. These are then escaped as `'\"'` in the result set. The quick and dirty way to remove those is with `sed`.

Here is the complete command line:

```bash
curl -XPOST -H "Content-Type: application/json" http://localhost:8888/druid/v2/sql/ -d'{ "query": "SELECT * FROM \"wikipedia-rollup-00\"" }' | jq -c '.[]' | sed -e 's/\\\"//g' >wikipedia-rollup-00.json
```

