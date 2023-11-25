---
layout: post
title: "Druid Data Cookbook: Upserts in Druid SQL"
categories: blog druid sql imply adtech tutorial update crud
---

this is a sequel to [an earlier blog](/2023/03/07/selective-bulk-upserts-in-apache-druid/)

the technique demonstrated is cool, but can we do this in SQL?

we want something like

```sql
MERGE INTO druid_table
(SELECT * FROM external_table)
ON druid_table.keys = external_table.keys
WHEN MATCHED THEN UPDATE ...
WHEN NOT MATCHED THEN INSERT ...
```

first down would work like this

```sql
REPLACE INTO druid_table
SELECT * FROM druid_table
WHERE NOT filter_condition
UNION ALL
SELECT * FROM external_table
WHERE filter_condition
```

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
