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

There are three configuration settings that affect _NULL_ handling:

- `druid.generic.useDefaultValueForNull`
- `druid.expressions.useStrictBooleans`
- `druid.generic.useThreeValueLogicForNativeFilters`

The _Status_ tile in the Druid web console indicates whether Druid is configured in SQL compliant mode; if you hover over the corresponding text, it shows a detailed breakdown of the settings:

![Null mode settings](/assets/2024-04-28-01-null-mode.png)

What do these settings do? Let's look at them in a bit more detail.

### useDefaultValueForNull

Originally, Druid did not have a separate representation for empty or _NULL_ values. A _NULL_ value would be simply treated as an empty string, or as a numeric 0 (zero); and it would be equivalent to these default values for all intents and purposes.

The new default for this setting is _false_.

### useStrictBooleans

This setting now defaults to _true_. If set, it forces the result of all logical expressions to be 0 or 1. Older versions of Druid would keep the original values of input parameters, since anything that was not 0 or an empty string would be considered a _true_ value. (This resembles the way logical expressions are handled in some scripting languages.)

### useThreeValueLogicForNativeFilters

This setting defaults to _true_, too. Its effect is that _NULL_ is kept as a distinct value that is separate from either _true_ or _false_, in all logical evaluations. Any expression that contains a _NULL_ value would yield a result of _NULL_, too. This has a number of interesting ramifications. Let's look at some of them today!

This tutorial can be done using the [Druid 29.0.1 quickstart](https://druid.apache.org/docs/latest/tutorials/).

## Ingestion

First of all, let's create a very simple data set.

The table I am going to use has but four rows of data. There is a column _color_, which can be either a string, or _NULL_.

Here is the Druid SQL to create the sample data set:

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


