---
layout: post
title:  "Selective Bulk Upserts in Apache Druid: SQL Edition"
categories: blog druid sql imply adtech tutorial update crud
---

this is a sequel to [an earlier blog](/2023/03/07/selective-bulk-upserts-in-apache-druid/)

the technique demonstrated is cool, but can we do this in SQL?

we want something like

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

## Populating the staging table

we need to create a hole in the old data the shape of the new data

let's create a staging table for that purpose

--> sql replace into staging with filter

now we add the new data

--> sql insert data2 with extern

let's look at the partitioning

![](partitioning screenshot)

the insert has created a dynamic partition which is not good for pruning, we want range (link to partitioning blog)

## Reorganizing

let's swap the data back into the main table

--> sql replace

now the partitions are correct.

note: you could use an OVERWRITE WHERE ... clause to leave older segments untouched

## Learnings

- we can get the effect of a filtered union all by using an intermediate table
- the 2 step process gives us atomicity
- and also the ability to fix partitioning.

is like double buffering in graphics
