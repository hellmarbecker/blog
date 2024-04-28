---
layout: post
title:  "Druid Data Cookbook: About SQL NULL"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/2021-12-21-elf.jpg
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

In the latest versions of Druid, handling of logical expressions, and in particular of _NULL_ (unknown) values, has been changed to match the SQL standard. This leads to some behavior that may be surprising to long time Druid users. Let's take a look at some examples!

## Server configuration settings that affect _NULL_ handling

There are three configuration settings that affect 

- `druid.generic.useDefaultValueForNull`
- `druid.expressions.useStrictBooleans`
- `druid.generic.useThreeValueLogicForNativeFilters`

The _Status_ tile in the Druid web console indicates whether Druid is configured in SQL compliant mode; if you hover over the corresponding text, it shows a detailed breakdown of the settings:

![Null mode settings](/assets/2024-04-28-01-null-mode.png)



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


