---
layout: post
title:  "Indexes in Apache Druid"
categories: blog druid
---

If you come from a traditional database background, you are probably used to creating and maintaining indexes on most of your data. In a relational database, indexes can speed up queries but at a cost of slower data insertion.

In Druid, on the other hand, you never see a `CREATE INDEX` statement. Instead, Druid automatically indexes all data, creating optimized storage segments that provide high performance for all data types - and you never need to select or manage indexes. Let's look at some of these data organization features!

## Druid Bitmap Indexes

these are created automatically on all string columns and on each subfield of a JSON column

### Forward and inverted indexes

### Bitmap indexes - why?

- fast lookup of value
- mergeable in any combination
- always segment local, thus fast to maintain
- for high cardinality and sparse data, a forward index may be faster bit see below
- why no B-tree? unlike bitmap index, a B-tree index has to be global to be fast. insertion is expensive

### Sparse indexes

### How we implement the best of forward and inverted index: Druid concise bitmaps


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
