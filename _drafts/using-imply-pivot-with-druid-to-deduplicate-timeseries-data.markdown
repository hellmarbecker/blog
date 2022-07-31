---
layout: post
title:  "Using Imply Pivot with Druid to Deduplicate Timeseries Data"
categories: blog druid imply analytics tutorial
---

For instance, imagine you are running a solar or wind power plant. You are measuring the power output, among other parameters, on a regular basis, but the measurements are not evenly distributed along the time axis. Sometimes you get several measurements for the same unit of time, sometimes there is only one. How do you measure the average power output without giving undue weight to those time periods that had more measurements?

Let's consider a very simplified example:

```csv
ts,var,val
2022-07-31T10:01:00,p,31
2022-07-31T10:02:00,p,33
2022-07-31T10:18:00,p,48
```

Here we have two measurements of power output between 10:00 and 10:15, but only one between 10:15 and 10:30. Ingest those data into Druid with a datasource name of `power_data`. Here is the ingestion spec:

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

You should 
