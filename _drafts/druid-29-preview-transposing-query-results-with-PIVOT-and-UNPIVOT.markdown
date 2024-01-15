---
layout: post
title:  "Druid 29 Preview: Transposing Query Results with PIVOT and UNPIVOT"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/.jpg
---

## The data

```csv
region,2022,2023
Central,215000,240000
East,350000,360000
West,415000,450000
```

## ingestion query

```sql
REPLACE INTO "sales_data" OVERWRITE ALL
WITH "ext" AS (
  SELECT *
  FROM TABLE(
    EXTERN(
      '{"type":"inline","data":"region,2022,2023\nCentral,215000,240000\nEast,350000,360000\nWest,415000,450000"}',
      '{"type":"csv","findColumnsFromHeader":true}'
    )
  ) EXTEND ("region" VARCHAR, "2022" BIGINT, "2023" BIGINT)
)
SELECT
  TIMESTAMP '2000-01-01 00:00:00' AS "__time",
  "region",
  "2022",
  "2023"
FROM "ext"
PARTITIONED BY ALL
```

## select with pivot - transpose rows to columns

```sql
SELECT *
FROM "sales_data"
PIVOT (
  SUM("2022") AS sales_2022, 
  SUM("2023") AS sales_2023 
  FOR "region" IN ('East' AS east, 'Central' AS central))
```

- needs an aggregation because it implicitly groups by the values in the pivot columns
- can have multiple aggregations as in this example
- to keep the column list finite, you have to give it a list of values for the pivot column
- define aliases for the values, those will serve as column prefixes
- you can use the generated column names in query clauses: this here is legit

```sql
SELECT east_sales_2022
FROM "sales_data"
PIVOT (
  SUM("2022") AS sales_2022, 
  SUM("2023") AS sales_2023 
  FOR "region" IN ('East' AS east, 'Central' AS central))
```

## select with unpivot - transpose columns to rows

```sql
SELECT *
FROM "sales_data"
UNPIVOT ( "sales" FOR "year" IN ("2022" AS 'previous', "2023" AS 'current') )
```

- this needs no aggregation since it only reorders the values
- you need to define two aliases:
  - the first one, `"sales"` in the example, is the column where the _values_ end up
  - the second one, `"year"` is where the column names go, expressed as strings
- again, you can define alias values for the column names

## UNPIVOT during ingestion

this ingesiton does not have a proper timestamp because the time information is in the column headers

can we use this to generate a proper timestamp?

let's see with MSQ ingestion

```sql
REPLACE INTO "sales_data_unpivot" OVERWRITE ALL
WITH "ext" AS (
  SELECT *
  FROM TABLE(
    EXTERN(
      '{"type":"inline","data":"region,2022,2023\nCentral,215000,240000\nEast,350000,360000\nWest,415000,450000"}',
      '{"type":"csv","findColumnsFromHeader":true}'
    )
  ) EXTEND ("region" VARCHAR, "2022" BIGINT, "2023" BIGINT)
)
SELECT
  TIME_PARSE("year", 'YYYY') AS "__time",
  "region",
  "sales"
FROM "ext"
UNPIVOT ( "sales" FOR "year" IN ("2022", "2023" ) )
PARTITIONED BY YEAR
```
