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

![Ingestion](/assets/2024-04-28-02-ingest.png)

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

If you ingest this data set and list the entire table, you should get something like this:

![Select all data](/assets/2024-04-28-03-select-star.png)

Note the _NULL_ value in the _color_ column.

Let's run some more queries now!

## Comparing against single values

Since a comparison with _NULL_ is never true, _NULL_ is not even considered equal to itself. This is why the following query 

![Select null](/assets/2024-04-28-04-select-equals-null.png)

yields no rows in return. Even more, if you change the condition to `WHERE color <> NULL` you get nothing, too!

You have to use the operators `IS NULL` and `IS NOT NULL`, instead.

In a similar vein, let's get all the entries whose color is not _red_. Naïvely, we try:

![Select out single value, does not work](/assets/2024-04-28-05-select-single-value.png)

We get only the _blue_ and _green_ entries - the _NULL_ value is, again, not caught by the operator.

You _could_ construct a combined filter clause handling the _NULL_ case separately. But there is a special operator that allows one to treat _NULL_ values like regular values:

![Select out single value using IS DISTINCT FROM](/assets/2024-04-28-06-select-is-distinct-from.png)

The `IS DISTINCT FROM` operator does what we need: it treats _NULL_ values as equivalent and distinct from any other value.

## Comparing against multiple values

How about filtering multiple values with an `IN` clause? Let's try to retrieve only those rows that have color _red_ or _NULL_:

![Select with IN clause](/assets/2024-04-28-07-select-in.png)

After the previous experiments, this should not come as a surprise: The query returns only the row for _red_. But what if we invert the condition?

![Select with NOT IN clause](/assets/2024-04-28-08-select-not-in.png)

This one returns no rows at all! Supposedly, the comparison with anything that contains _NULL_ would always give a _NULL_ result, and in fact the entire filter is optimized out of the query plan.

There is a workaround though. Instead of using a list with `IN`, we can also try an array literal like so:

![Select with array](/assets/2024-04-28-09-select-array-contains.png)

This gives the same result as the `IN` filter. But if we invert the filter condition, we get something that is more along the lines of the expectation:

![Select with array, inverted](/assets/2024-04-28-10-select-not-array-contains.png)


## Learnings

- Druid's handling of unknown values has been made SQL compliant.
- This can lead to unexpected results since any comparison with a _NULL_ value yields a _NULL_ value itself: _NULL_ is equal to nothing, but is also not equal to nothing - not even to itself!
- In order to handle _NULL_ values properly, special operators exist, such as `IS NULL` and `IS DISTINCT FROM`.
- Beware of _NULL_ values in `IN ()` filter clauses! Using an array literal instead can help.   

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.

