---
layout: post
title:  "Druid Data Cookbook: Array as JSON"
categories: blog apache druid imply tutorial
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

In my series about [multi-value dimensions (MVD)](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/), I took you to a small Italian restaurant where we looked at this snippet of orders:

```json
{ "date": "2021-09-25", "customer": "Fangjin", "orders": [ "pizza", "tiramisu", "espresso", "espresso" ] }
{ "date": "2021-09-25", "customer": "Gian", "orders": [ "pizza", "espresso", "tiramisu" ] }
{ "date": "2021-09-25", "customer": "Vadim", "orders": [ "pizza", "tiramisu" ] }
{ "date": "2021-09-25", "customer": "Rachel", "orders": [ "pizza", "tiramisu", "espresso" ] }
{ "date": "2021-09-25", "customer": "Xiaolan", "orders": [ "pizza", "espresso" ] }
{ "date": "2021-09-25", "customer": "Jessy", "orders": [ "pizza", "espresso", "espresso" ] }
```

We ingested the order list as an MVD, and we observed the "magic" behavior that implicitly unnests the MVD values inside a query that groups or filters by that dimension.

Since then, Druid has added more ways of representing structured data. An MVD is declared as a `STRING` dimension, but it behaves (mostly) like an array or a list of values. However, an array is also a valid JSON object and it can be ingested into Druid as a [nested column](https://druid.apache.org/docs/latest/querying/nested-columns.html). A lot of work is currently happening in this field.

This tutorial works with the Druid 25.0 quickstart.

## Nested Columns

Ingest the above data sample using the classic batch ingestion wizard in the Druid console. When you come to the `Configure schema` stage, change the type of the `orders` dimension to `json`:

![Select JSON type for column](/assets/2023-01-29-01-json.jpg)

Name the datasource `ristorante_json` and ingest the data.

## First Query

Let's just look at the table in SQL.

```sql
SELECT *
FROM "ristorante_json"
```

![Select all](/assets/2023-01-29-02-select-star.jpg)

It doesn't look too different from the MVD modeling. Only the little tree symbol next to the column name indicates that we are dealing with a nested column.

## `GROUP BY` - Where's the Magic?

Let's try to `GROUP` like before:

```sql
SELECT 
  orders, 
  COUNT(*) AS "Count"
FROM "ristorante_json"
GROUP BY 1
```

![Group by JSON error](/assets/2023-01-29-03-groupby-json.jpg)

Ouch. That did not work. We will have to extract individual fields from the nested column in order to do anything meaningful with the data. This one is legit:

```sql
SELECT 
  JSON_VALUE(orders, '$[0]'), 
  JSON_VALUE(orders, '$[1]'), 
  COUNT(*) AS "Count"
FROM "ristorante_json"
GROUP BY 1, 2
```

![Group by individual fields works](/assets/2023-01-29-04-groupby-values.jpg)

Unfortunately, there is no builtin function (yet) to iterate over all the elements of a JSON array, so the usefulness of this idiom is limited. We could also transform the JSON column into a string:

```sql
SELECT 
  TO_JSON_STRING(orders), 
  COUNT(*) AS "Count"
FROM "ristorante_json"
GROUP BY 1
```

But these are simple `STRING`s with no grouping magic.

![Transform JSON into string](/assets/2023-01-29-05-groupby-json-string.jpg)

However, this is the path to a solution ...

## Here's the Magic!

If we could parse the JSON string representation into an MVD, maybe the original grouping would work again. In an [earlier post](/2021/12/18/druid-lab---processing-movie-tag-data/) I mentioned that `STRING_TO_MV()` splits a string _by a regular expression_. This, and the ability to `TRIM` characters from the beginning and or end of a string, gives us all the tools we need.

- We will have to strip leading and trailing square brackets, and also the double quotes. Therefore the strip set for `TRIM` will be `["]`.
- We want to split only where a comma is enclosed by double quotes, possibly with whitespace after the comma. The regular expression for this is `",\s*"`.

Let's first try the transformation:

```sql
SELECT
  customer,
  orders,
  STRING_TO_MV(
    TRIM('["]' FROM TO_JSON_STRING(orders)), 
    '",\s*"') AS orders_mv
FROM "ristorante_json"
```

![Cast to MV](/assets/2023-01-29-06-mv.jpg)

Note the different type symbol next to `orders_mv`.

And now, to the finale:

```sql
SELECT
  STRING_TO_MV(
    TRIM('["]' FROM TO_JSON_STRING(orders)), 
    '",\s*"') AS order_item,
  COUNT(customer) AS num_orders
FROM "ristorante_json"
GROUP BY 1
```

![Group by MV](/assets/2023-01-29-07-mv-groupby.jpg)

The group by behavior is what we expect from an MVD!

## Learnings

- Array fields can now be ingested as nested JSON columns into Druid.
- However, one cannot `GROUP BY` a JSON column.
- Individual elements can be extracted using JSON Path expressions, but for now there is no way to iterate over all the fields or array elements.
- As a workaround, you can parse the JSON array into a multi-value dimension and use existing Druid functionality to unnest it.  

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.

