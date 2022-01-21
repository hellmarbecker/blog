---
layout: post
title:  "Geospatial Data in Apache Druid - Generating Geohashes"
categories: blog apache druid imply
---

![Map](/assets/2021-09-05-map.png)

In [an earlier post](/2021/09/05/geospatial-data-in-apache-druid---ingestion/), I discussed how to model geospatial data in Druid. Here's a little hack that shows (as a proof of concept) how these data can be used to generate [Geohash](https://en.wikipedia.org/wiki/Geohash) codes from latitude and longitude values.

This uses a deprecated feature of Druid which lets you write your own parser in Javascript. You will have to [enable Javascript](https://druid.apache.org/docs/latest/configuration/index.html#javascript) to make this work.

It also uses the older `firehose` format for ingestion, which means you will have to write the ingestion spec in a text editor and submit it through the [Task API](https://druid.apache.org/docs/latest/operations/api-reference.html#post-5).

Here is an example with inline data:
```
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "firehose": {
        "type": "inline",
        "data": "50.09019,8.4493,Hofheim am Taunus,DE,Europe/Berlin\n52.47774,10.5511,Gifhorn,DE,Europe/Berlin\n52.53048,13.29371,Charlottenburg-Nord,DE,Europe/Berlin\n48.21644,9.02596,Albstadt,DE,Europe/Berlin\n52.53048,13.29371,Charlottenburg-Nord,DE,Europe/Berlin\n49.68369,8.61839,Bensheim,DE,Europe/Berlin\n50.64336,7.2278,Bad Honnef,DE,Europe/Berlin\n48.46458,9.22796,Pfullingen,DE,Europe/Berlin\n53.59337,9.47629,Stade,DE,Europe/Berlin\n50.80904,8.77069,Marburg an der Lahn,DE,Europe/Berlin"
      }
    },
    "tuningConfig": {
      "type": "index_parallel",
      "partitionsSpec": {
        "type": "dynamic"
      }
    },
    "dataSchema": {
      "dataSource": "geo_data_leg",
      "parser": {
        "type": "string",
        "parseSpec": {
          "format": "javascript",
          "function": "function(a){var b=a.split(\",\");var c=b[0];var d=b[1];const BITS=[16,8,4,2,1];const BASE32=\"0123456789bcdefghjkmnpqrstuvwxyz\";var e=1;var i=0;var f=[];var g=[];var h=0;var j=0;var k=12;geohash=\"\";f[0]=-90.0;f[1]=90.0;g[0]=-180.0;g[1]=180.0;while(geohash.length<k){if(e){mid=(g[0]+g[1])/2;if(d>mid){j|=BITS[h];g[0]=mid}else g[1]=mid}else{mid=(f[0]+f[1])/2;if(c>mid){j|=BITS[h];f[0]=mid}else f[1]=mid}e=!e;if(h<4)h++;else{geohash+=BASE32[j];h=0;j=0}}return{latitude:b[0],longitude:b[1],place_name:b[2],country_code:b[3],timezone:b[4],geohash:geohash}}",
          "timestampSpec": {
            "column": "!!!_no_such_column_!!!",
            "missingValue": "2010-01-01T00:00:00Z"
          },
          "columns": ["latitude","longitude","place_name","country_code","timezone","geohash"],
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
              "timezone",
              "geohash"
            ]
          }
        }
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
For the Javascript parser, I used a Geohash function that I found [here](https://github.com/davetroy/geohash-js), adapted the signature, and ran it through an obfuscator/compressor.

You can see the intermediate steps in this repo: https://github.com/hellmarbecker/druid-spatial.

Don't do this in production, but it is a fun thing to try!

