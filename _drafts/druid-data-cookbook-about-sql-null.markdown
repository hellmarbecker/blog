---
layout: post
title:  "Druid Data Cookbook: About SQL NULL"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/2021-12-21-elf.jpg
---

druid.generic.useDefaultValueForNull

druid.expressions.useStrictBooleans

druid.generic.useThreeValueLogicForNativeFilters

## Ingestion

```sql
REPLACE INTO "inline_data" OVERWRITE ALL
WITH "ext" AS (
  SELECT *
  FROM TABLE(
    EXTERN(
      '{"type":"inline","data":"{\"id\": 1, \"color\": \"red\"}\n{\"id\": 1, \"color\": null}\n{\"id\": 1, \"color\": \"blue\"}\n{\"id\": 1, \"color\": \"green\"}\n"}',
      '{"type":"json"}'
    )
  ) EXTEND ("id" BIGINT, "color" VARCHAR)
)
SELECT
  TIMESTAMP '2000-01-01 00:00:00' AS "__time",
  "id",
  "color"
FROM "ext"
PARTITIONED BY ALL
```


