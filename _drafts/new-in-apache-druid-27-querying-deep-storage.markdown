---
layout: post
title:  "New in Apache Druid 27: Querying Deep Storage"
categories: blog druid imply query storage
---

yada yada

Ingest the _wikipedia_ example data set. We want to have a bunch of segments so let's partition by hour:

```sql
REPLACE INTO "wikipedia" OVERWRITE ALL
WITH "ext" AS (SELECT *
FROM TABLE(
  EXTERN(
    '{"type":"http","uris":["https://druid.apache.org/data/wikipedia.json.gz"]}',
    '{"type":"json"}'
  )
) EXTEND ("isRobot" VARCHAR, "channel" VARCHAR, "timestamp" VARCHAR, "flags" VARCHAR, "isUnpatrolled" VARCHAR, "page" VARCHAR, "diffUrl" VARCHAR, "added" BIGINT, "comment" VARCHAR, "commentLength" BIGINT, "isNew" VARCHAR, "isMinor" VARCHAR, "delta" BIGINT, "isAnonymous" VARCHAR, "user" VARCHAR, "deltaBucket" BIGINT, "deleted" BIGINT, "namespace" VARCHAR, "cityName" VARCHAR, "countryName" VARCHAR, "regionIsoCode" VARCHAR, "metroCode" BIGINT, "countryIsoCode" VARCHAR, "regionName" VARCHAR))
SELECT
  TIME_PARSE("timestamp") AS "__time",
  "isRobot",
  "channel",
  "flags",
  "isUnpatrolled",
  "page",
  "diffUrl",
  "added",
  "comment",
  "commentLength",
  "isNew",
  "isMinor",
  "delta",
  "isAnonymous",
  "user",
  "deltaBucket",
  "deleted",
  "namespace",
  "cityName",
  "countryName",
  "regionIsoCode",
  "metroCode",
  "countryIsoCode",
  "regionName"
FROM "ext"
PARTITIONED BY HOUR
```

first set of retention rules:

```json
[
  {
    "interval": "2016-06-27T12:00:00.000Z/2020-01-01T00:00:00.000Z",
    "tieredReplicants": {
      "_default_tier": 2
    },
    "useDefaultTierForNull": true,
    "type": "loadByInterval"
  },
  {
    "type": "dropForever"
  }
]
```

query this:

```
curl -L -H 'Content-Type: application/json' localhost:8888/druid/v2/sql/statements -d'{
    "query": "SELECT DATE_TRUNC('\''hour'\'', __time), COUNT(*) FROM \"wikipedia\" GROUP BY 1 ORDER BY 1",
    "context":{
        "executionMode":"ASYNC"
    }
}'
{"queryId":"query-db8b79ae-f28b-466e-b876-3f987d0e87fc","state":"ACCEPTED","createdAt":"2023-09-06T11:33:39.839Z","schema":[{"name":"EXPR$0","type":"TIMESTAMP","nativeType":"LONG"},{"name":"EXPR$1","type":"BIGINT","nativeType":"LONG"}],"durationMs":-1}
```

get the result:

```
curl -L -H 'Content-Type: application/json' localhost:8888/druid/v2/sql/statements/query-db8b79ae-f28b-466e-b876-3f987d0e87fc
{"queryId":"query-db8b79ae-f28b-466e-b876-3f987d0e87fc","state":"SUCCESS","createdAt":"2023-09-06T11:33:39.839Z","schema":[{"name":"EXPR$0","type":"TIMESTAMP","nativeType":"LONG"},{"name":"EXPR$1","type":"BIGINT","nativeType":"LONG"}],"durationMs":13944,"result":{"numTotalRows":10,"totalSizeInBytes":374,"dataSource":"__query_select","sampleRecords":[[1467028800000,1219],[1467032400000,1211],[1467036000000,1353],[1467039600000,1422],[1467043200000,1442],[1467046800000,1339],[1467050400000,1321],[1467054000000,1175],[1467057600000,1213],[1467061200000,603]],"pages":[{"id":0,"numRows":10,"sizeInBytes":374}]}}
```

