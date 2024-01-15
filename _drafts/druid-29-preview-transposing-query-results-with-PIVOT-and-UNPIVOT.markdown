---
layout: post
title:  "Druid 29 Preview: Transposing Query Results with PIVOT and UNPIVOT"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/.jpg
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

Imagine that you are tasked with getting a spreadsheet of sales data into Druid that looks like this:

![Raw data in table format](/assets/2024-01-15-01-rawdata-table.png)

You've got the sales figures in the cells, and the regions down and the years across. While you _can_ work with the data as it is now in Druid, this is maybe not the best option. And Druid 29 brings two new SQL functions that can help with transforming the data into a format that is better suited for analytics. Let's see how that works!

## Getting set up

This is a sneak peek into Druid 29 functionality. In order to use the new functions, you can (as of the time of writing) [build Druid](https://druid.apache.org/docs/latest/development/build.html) from the HEAD of the master branch:

```bash
git clone https://github.com/apache/druid.git
cd druid
mvn clean install -Pdist -DskipTests
```

Then follow the instructions to locate and install the tarball. Make sure you have [the `druid-multi-stage-query` extension enabled](https://druid.apache.org/docs/latest/multi-stage-query/#load-the-extension).

In this tutorial, you will 

- learn how to use the `PIVOT` and `UNPIVOT` functions to transpose rows into columns and vice versa
- and use this knowledge to transform a dataset during ingestion in Druid.

_**Disclaimer:** This tutorial uses undocumented functionality and unreleased code. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. In addition, the entire build, or execution, may fail. Your mileage may vary._

## The data

The dataset is very simple and looks like this:

```csv
region,2022,2023
Central,215000,240000
East,350000,360000
West,415000,450000
```

## Ingestion query

Run the ingestion wizard; make sure to give a meaningful name to the target datasource. Or you can paste the below SQL directly into a query window: 

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

## Conclusion

- yada yada

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.

PARTITIONED BY YEAR
```
