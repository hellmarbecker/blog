---
layout: post
title:  "Indexes in Apache Druid"
categories: blog druid
---

If you come from a traditional database background, you are probably used to creating and maintaining indexes on most of your data. In a relational database, indexes can speed up queries but at a cost of slower data insertion.

In Druid, on the other hand, you never see a `CREATE INDEX` statement. Instead, Druid automatically indexes all data, creating optimized storage segments that provide high performance for all data types - and you never need to select or manage indexes. Let's look at some of these data organization features!

## Druid Bitmap Indexes

Druid uses [bitmap indexes](https://en.wikipedia.org/wiki/Bitmap_index). These are created automatically on all string columns and on each subfield of a JSON column

### Types of indexes in a relational database

Relational databases use a B-tree index as their primary index type. A relational table often has a primary key that can be used to uniquely identify a row in the table. A B-tree index maps individual keys to the rows that contain them. Its use cases are:

- enforcing uniqueness of a key during inserting
- quickly looking up a single value for updates, inserts, and (sometimes) join queries.

A B-tree index is not a good choice for analytical queries where you have, as a rule, many rows with the same value, and you want to  retrieve and aggregate data in bulk. It is also to be noted that due to the structure of a B-tree index, lookups are _O(n)_ complexity, which may be impractical for large tables.

### Bitmap indexes - why?

Bitmap indexes came up as relational databases were enhanced with analytical features. A bitmap index stores, for each value, a bit array where the position of each row that has a _1_ bit and all the other positions are _0_. This has a number of advantages:

- Fast lookup of all rows for a value. Because the bitmap index is an array, such lookups are _O(1)_. 
- Even better, bitmap indexes are mergeable in any combination. To model logical conditions such as the union or intersection of filters, just apply bitwise logical _OR_ and _AND_ operations to the bitmap.
- Bitmaps are always segment local and thus fast to maintain. If your data is partitioned or sharded, the bitmap index is partitioned in the same way.

For high cardinality and sparse data, a forward index such as a B-tree may be faster but there are ways to get the best of both worlds. I'll get to that in a moment.

Why doesn't Druid use B-tree indexes as a general option? Unlike a bitmap index, a B-tree index has to be global to be fast. (A global index spans the whole table, disregarding any partitioning.) This makes insertion and index maintenance quite expensive.

### Sparse indexes

### How we implement the best of forward and inverted index: Druid roaring bitmaps


## Colocating Data: Partitioning and Clustering

the main abstraction in an RDBMS is that you look at the table as a whole. there is no implicit ordering in the way the data is laid out. it has long been known that this is not the best model for analytical queries. 

### Time partitioning, granularities and sorting by time

### Special case: Multiple time granularities

### Secondary partitioning: Pruning and range queries

### How we implement composite index functionality

### How we implement range index functionality

### Be extra space efficient: Front coding


## Structured Data: Nested Columns

### Indexes on nested data

### How we implement JSON (document) index functionality


## Bonus: Bloom filters

yada yada

## Conclusion

- you might be asking: where are the indexes. a lot of index functionality is done in druid with features that are not technically indexes
- bitmap index is cool
- bitmap index allows merge and thus arbitrary column combi
- concise bitmaps uses forward lookup for sparse columns so speeds up andd reduces storage
- time partitioning aids pruning in time based queries
- time sorting is great for time series and time range queries
- secondary partitioning replaces composite and range indexes
- each field inside a nested column (document column) has its own bitmap index so JSON index functionality is achieved
