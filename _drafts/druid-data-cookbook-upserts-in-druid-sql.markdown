---
layout: post
title: "Druid Data Cookbook: Upserts in Druid SQL"
categories: blog druid sql imply adtech tutorial update crud
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

In [an earlier blog](/2023/03/07/selective-bulk-upserts-in-apache-druid/), I demonstrated a technique to combine existing and new data in Druid batch ingestion in a way that more or less emulates what is usually expressed in SQL as a `MERGE` or `UPSERT` statement. This technique involves a `combine` datasource and works only in JSON-based ingestion.

Today I am going to look at a similar construction, but using the more modern SQL based ingestion mechanism that is available in newer versions of Druid.

The `MERGE` statement, in a simplified way, would work like this:

```sql
MERGE INTO druid_table
(SELECT * FROM external_table)
ON druid_table.keys = external_table.keys
WHEN MATCHED THEN UPDATE ...
WHEN NOT MATCHED THEN INSERT ...
```

So, you compare old _(druid_table)_ and new data _(external_table)_ with respect to a _matching condition_. This would entail a combination of timestamp and key fields, which in the above pseudocode is denoted by _keys_. There are three possible outcomes for any combination of _keys:_

1. If _keys_ exists only in _druid table_, leave these data untouched.
2. If _keys_ exists in both tables, replace the row(s) in _druid_table_ with those in _external_table_.
3. If _keys_ exists only in _external_table_, insert that data into _druid_table_.



first down would work like this

```sql
REPLACE INTO druid_table
SELECT * FROM druid_table
WHERE NOT filter_condition
UNION ALL
SELECT * FROM external_table
WHERE filter_condition
```

the technique demonstrated is cool, but can we do this in SQL?

while druid 28 has a union all construct in MSQ (the engine used for SQL ingestion), this is not quite powerful enough yet. but we can achieve the same effect easily. let's see how!

... prerequisites ...

## Recap: the data

yada yada prepare data to a local file

## Loading the data

first bunch of data goes into data1

we want to set up range partitioning so let's make sure to include a `CLUSTERED BY` clause

run the ingestion

--> quote sql

note we are doing monthly segments here

## the technique

we do have UNION in msq but it is too brain damaged - we cannot filter 

so instead, use the 80s technique of doing a full outer join

give credit to john sergio 

## the merge query

note: this needs `{ "sqlJoinAlgorithm": "sortMerge" }` in the context

--> context screenshot

then you can run this query

--> quote merge sql

## putting it all together

--> replace overwrite all query

## can we be more selective?

--> replace overwrite week

--> show error

it has to be segment aligned! if not druid will not let you

this one works:

--> replace overwrite month

and so we get the update data:

--> paste table

--> screenshot explore, timeline stacked by network

## Learnings

- yada yada
- 

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
