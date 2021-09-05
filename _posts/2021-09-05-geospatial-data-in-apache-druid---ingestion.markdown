---
layout: post
title:  "Geospatial Data in Apache Druid - Ingestion"
categories: blog apache druid imply
---

![Map](/assets/2021-09-05-map.png)

A lesser known feature of Apache Druid is the ability to handle [spatial data](https://docs.imply.io/latest/druid/development/geo/#spatial-indexing) directly. There are a number of built in functions that allow query filtering on spatial structures like rectangles or polygons. Also, there is a data type for spatial coordinates which you can specify in the ingestion spec.

Let's look at a simple example. I am going to take a list of German cities that I generated using the Python [Faker](https://faker.readthedocs.io/en/master/index.html) module:
```
latitude,longitude,place_name,country_code,timezone
50.09019,8.4493,Hofheim am Taunus,DE,Europe/Berlin
52.47774,10.5511,Gifhorn,DE,Europe/Berlin
52.53048,13.29371,Charlottenburg-Nord,DE,Europe/Berlin
48.21644,9.02596,Albstadt,DE,Europe/Berlin
52.53048,13.29371,Charlottenburg-Nord,DE,Europe/Berlin
49.68369,8.61839,Bensheim,DE,Europe/Berlin
50.64336,7.2278,Bad Honnef,DE,Europe/Berlin
48.46458,9.22796,Pfullingen,DE,Europe/Berlin
53.59337,9.47629,Stade,DE,Europe/Berlin
50.80904,8.77069,Marburg an der Lahn,DE,Europe/Berlin
```
I created an ingestion spec for these data:
```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "inline",
        "data": "latitude,longitude,place_name,country_code,timezone\n50.09019,8.4493,Hofheim am Taunus,DE,Europe/Berlin\n52.47774,10.5511,Gifhorn,DE,Europe/Berlin\n52.53048,13.29371,Charlottenburg-Nord,DE,Europe/Berlin\n48.21644,9.02596,Albstadt,DE,Europe/Berlin\n52.53048,13.29371,Charlottenburg-Nord,DE,Europe/Berlin\n49.68369,8.61839,Bensheim,DE,Europe/Berlin\n50.64336,7.2278,Bad Honnef,DE,Europe/Berlin\n48.46458,9.22796,Pfullingen,DE,Europe/Berlin\n53.59337,9.47629,Stade,DE,Europe/Berlin\n50.80904,8.77069,Marburg an der Lahn,DE,Europe/Berlin"
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
      "dataSource": "geo_data",
      "timestampSpec": {
        "column": "!!!_no_such_column_!!!",
        "missingValue": "2010-01-01T00:00:00Z"
      },
      "dimensionsSpec": {
        "spatialDimensions": [
          {
            "dimName": "coordinates",
            "dims": [
              "latitude",
              "longitude"
            ]
          }
        ],
        "dimensions": [
          {
            "type": "double",
            "name": "latitude"
          },
          {
            "type": "double",
            "name": "longitude"
          },
          "place_name",
          "country_code",
          "timezone"
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
Note how we have a `spatialDimensions` spec besides the regular dimensions.

Let's query the data!
![GeoQuery](/assets/2021-09-05-geo1.jpg)

Aha! The spatial data seems to be represented internally as a string dimension, but unfortunately our original coordinate fields are gone. As a general rule, in Druid, you can use each data field only once as a dimension. If you want to use the same field twice, you need to declare a logical duplicate using a transform spec.

Let's try this:
```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "inline",
        "data": "latitude,longitude,place_name,country_code,timezone\n50.09019,8.4493,Hofheim am Taunus,DE,Europe/Berlin\n52.47774,10.5511,Gifhorn,DE,Europe/Berlin\n52.53048,13.29371,Charlottenburg-Nord,DE,Europe/Berlin\n48.21644,9.02596,Albstadt,DE,Europe/Berlin\n52.53048,13.29371,Charlottenburg-Nord,DE,Europe/Berlin\n49.68369,8.61839,Bensheim,DE,Europe/Berlin\n50.64336,7.2278,Bad Honnef,DE,Europe/Berlin\n48.46458,9.22796,Pfullingen,DE,Europe/Berlin\n53.59337,9.47629,Stade,DE,Europe/Berlin\n50.80904,8.77069,Marburg an der Lahn,DE,Europe/Berlin"
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
      "dataSource": "geo_data",
      "timestampSpec": {
        "column": "!!!_no_such_column_!!!",
        "missingValue": "2010-01-01T00:00:00Z"
      },
      "dimensionsSpec": {
        "spatialDimensions": [
          {
            "dimName": "coordinates",
            "dims": [
              "latitude",
              "longitude"
            ]
          }
        ],
        "dimensions": [
          {
            "type": "double",
            "name": "latitude"
          },
          {
            "type": "double",
            "name": "longitude"
          },
          "place_name",
          "country_code",
          "timezone"
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
And this time, it works! We get both the spatial dimension and the regular fields.

In a future post, I'll look at what we can do with those data.

## Learnings

- Druid has built in support for spatial data.
- You may need a dimension transformation if you want to preserve the original data that goes into a spatial dimension.
- As of now, this needs to entered manually as a JSON spec. The ingestion wizard does not support spatial dimensions.
