---
layout: post
title:  "Indexes in Apache Druid"
categories: blog druid
---

If you come from a traditional database background, you are probably used to creating and maintaining indexes on most of your data. In a relational database, indexes can speed up queries but at a cost of slower data insertion.

In Druid, on the other hand, you never see a `CREATE INDEX` statement. Instead, Druid automatically indexes all data, creating optimized storage segments that provide high performance for all data types - and you never need to select or manage indexes. Let's look at some of these data organization features!

## Druid Bitmap Indexes

Druid uses ***[bitmap indexes](https://en.wikipedia.org/wiki/Bitmap_index)***. These are created automatically on all string columns and on each subfield of a JSON column. Let's look at this design choice in some more detail.

### Types of indexes in a relational database

Relational databases use a B-tree index as their primary index type. A relational table often has a primary key that can be used to uniquely identify a row in the table. A B-tree index maps individual keys to the rows that contain them. Its use cases are:

- enforcing uniqueness of a key during inserting
- quickly looking up a single value for updates, inserts, and (sometimes) join queries.

A B-tree index is not a good choice for analytical queries where you have, as a rule, many rows with the same value, and you want to  retrieve and aggregate data in bulk. It is also to be noted that due to the structure of a B-tree index, lookups are _O(log n)_ complexity, which may be impractical for large tables.

### Bitmap indexes - why?

***Bitmap indexes*** came up as relational databases were enhanced with analytical features. A bitmap index stores, for each value, a bit array where the position of each row that has a _1_ bit and all the other positions are _0_. It can be thought of as an ***inverted index*** that maps not a row number to a value, but a value to a collection of rows where the value occurs.

This has a number of advantages:

- Fast lookup of all rows for a value. Because the bitmap index is an array, such lookups are _O(1)_. 
- Even better, bitmap indexes are mergeable in any combination. To model logical conditions such as the union or intersection of filters, just apply bitwise logical _OR_ and _AND_ operations to the bitmap.
- Bitmaps are always segment local and thus fast to maintain. If your data is partitioned or sharded, the bitmap index is partitioned in the same way.

For high cardinality and sparse data, a forward index such as a B-tree may be faster but there are ways to get the best of both worlds. I'll get to that in a moment.

Why doesn't Druid use B-tree indexes as a general option? Unlike a bitmap index, a B-tree index has to be global to be fast. (A global index spans the whole table, disregarding any partitioning.) This makes insertion and index maintenance quite expensive.

### How Druid implements the best of forward and inverted index: Druid roaring bitmaps

Let's talk about _sparse indexes_ for a moment. Contrary to a widespread belief, regular bitmaps are best for columns with medium cardinality. If the cardinality of a column is very low, the index is not very selective and you need to read a lot of data anyway. If the cardinality is very high, you have a different problem: Each value is only present in a small fraction of rows, so you would waste a lot of space storing zeroes for each value.

This is why Druid does not just implement bitmap indexes. Instead, bitmap indexes are by default compressed using [Roaring bitmap indexes](https://www.roaringbitmap.org/). The roaring bitmap algorithm cuts up the bitmap into pages of 2<sup>16</sup> rows. If the page has very few _1_ bits, it stores a list of row IDs instead.

Roaring bitmaps also support run-length encoding of pages, which is particularly effective when indexing a dimension that is also used to pre-sort the data - more about this later.

### Bitmap indexes and multi-value dimensions

[Multi-value dimensions](https://blog.hellmar-becker.de/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/) go nicely with bitmap indexes. A multi-value field would just have a bit set for every value that occurs in the cell. That is another reason to prefer bitmap indexes.

## Colocating Data: Partitioning and Clustering

In relational data modeling, the main abstraction is that you look at the table as a whole. There is no implicit ordering in the way the data is laid out. It has long been known that this is not the best model for analytical queries. That is why there are options in Druid that inform the physical layout of the data.

### Time partitioning, granularities and sorting by time

All data in Druid is partitioned and sorted by time. Each row has a primary timestamp, and part of the data modeling process is to define a _segment granularity_ and _query granularity_.

_Segment granularity_ is defined by the `PARTITIONED BY` clause in SQL based ingestion and it translates directly into the time chunks that define the segment timeline. (Within each time chunk, there may be multiple segments.) Within a segment, data is sorted by primary timestamp. This creates the equivalent of a ***timeseries index***.

_Query granularity_ is defined by truncating the primary timestamp in the ingestion query. Druid uses query granularity to deliberately define the time resolution such that data can be rolled up efficiently. This can greatly improve query performance and storage use.

### Special case: Multiple time granularities

If you want to achieve primary sorting by another column than time, you should set segment and query granularity to the same value. If you still need detailed timestamps, you can define the detailed time as a [secondary timestamp](https://druid.apache.org/docs/latest/ingestion/schema-design.html#secondary-timestamps). The main criteria for this design decision is if you expect to be running predominantly analytical queries that do not have timeseries characteristics, but you want to retain the ability to run some timeseries queries. The number of timestamp fileds is in principle not limited.

### Secondary partitioning: Pruning and range queries

Below the timestamp level, there is _secondary partitioning_, which is usually implemented as [range partitioning](https://blog.hellmar-becker.de/2022/01/25/partitioning-in-druid-part-3-multi-dimension-range-partitioning/). This defines a list of dimension fields to partition by. In SQL based ingestion, this correspinds to the `CLUSTERED BY` clause. You want to order your partitioning columns first in the ingestoin query, too. Then your data will be sorted according to the partitioning columns, and like values will be grouped together physically. If you filter by the partitioning key in a query, Druid uses this information to determine which data segments to look at, even before scanning any data. This is called ***partition pruning*** and is a great way to speed up queries.

### How Druid implements composite index functionality

With multi-dimension range partitioning, Druid achieves the same functionality as a ***composite index***. In an RDBMS, you would use a composite index whenever you have a combination of columns that you use to filter or group by in most of the queries that you typically run.

That being said, because we use bitmap indexes on all columns, we also achieve composite index functionality by margin bitmap indexes across columns.

### How Druid implements range index functionality

Another advantage of multi-dimension range partitioning is where you query for a range of values. Because the partitioning key also determines sort order, values within a range are grouped together. This achieves the functionality of a ***range index***.

### Be extra space efficient: Front coding

In addition to range sorting, Druid implements _front coding_ for character data. All data is represented by a dictionary (which can be thought of as a ***forward index***), and common prefixes are shared between entries. That way, we optimize space usage without sacrificing speed.

## Structured Data: Nested Columns

For nested (JSON) columns, Druid creates a bitmap index _for each nested field_. With that, you get the functionality of a ***document (JSON) index***. Again, Druid does the right thing automatically without requiring any explicit configuration.

## Conclusion

In this article, I gave a quick tour of data organization and indexing features in Apache Druid. What have we learned?

- You might be asking: where are the indexes? A lot of index functionality is done in Druid with features that are not technically indexes, but achieve the same effect.
- For analytical queries, bitmap indexes are the best choice for many scenarios. Druid creates bitmap indexes on all (string) columns by default.
- Bitmap indexes allow merging and logical operations, and thus support arbitrary column combinations, superseding composite indexes.
- Our implementation of Roaring bitmaps uses forward lookup for sparse columns: this optimizes both query speed and storage.
- Time partitioning aids pruning in time based queries.
- Time sorting is great for time series and time range queries.
- Secondary partitioning replaces composite and range indexes.
- Each field inside a nested column (document column) has its own bitmap index so JSON index functionality is achieved.
