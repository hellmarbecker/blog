---
layout: post
title:  "Merging Realtime Segments in Apache Druid"
categories: blog apache druid imply realtime
---

## Data Layout and Druid Performance

So, you want your realtime analytical queries to be really fast, and that's why you selected [Apache Druid](https://druid.apache.org/)! Today, let's have a look at another aspect of how Druid achieves its amazing performance.

Druid's query performance can be influenced by multiple factors in the data layout:

- **Segment size**. The optimum size of a [data segment](https://druid.apache.org/docs/latest/design/segments.html) is about 500 MB. If segments are much bigger than that, those segments need more resources for querying and also parallelism suffers. More often you encounter the opposite problem: there are too many small segments, which slows down query performance.
- **Partitioning and sorting of data**. Partitioning gives an extra performance boost when you can partition the segments according to the expected query pattern; also, inside a segment data is sorted by time, but then by partitioning key; this further speeds up segment scans by increasing compression ratio. For this to work you need to enable [range partitioning](https://blog.hellmar-becker.de/2022/01/25/partitioning-in-druid-part-3-multi-dimension-range-partitioning/).
- **Rollup**. This reduces both storage and query needs by pre-aggregating data. Ideally you want to have [perfect rollup](https://druid.apache.org/docs/latest/ingestion/rollup.html#perfect-rollup-vs-best-effort-rollup) so that each unique combination of dimension values corresponds to exactly one aggregate row. For this to work, again one has to use range or hash partitioning. In fact, with range or hash partitioning only, rollup is always perfect; with [dynamic partitioning](https://blog.hellmar-becker.de/2022/01/06/partitioning-in-druid-part-1-dynamic-and-hash-partitioning/), rollup is best effort - the resulting table may be multiple times bigger than the optimum.

## The Problem with Streaming Data

All the above factors can be addressed eaily in batch processing. Streaming data, however, usually does not come in neatly ordered. The point in streaming ingestion is to have these data available for analytics within a split second after an event occurs: and so, segments are built up in memory and persisted frequently. As a result, after a _hand-off_ (the process of persisting a segment to deep storage) by streaming ingestion, segments are not optimal:

- Segments will usually be _smaller than optimum_ because we cannot wait long to initiate a handoff. in addition, we may have to juggle multiple time chunks simultaneously because of late arriving data
- Range partitioning requires multiple steps of mapping and shuffling and merging. This is not possible to do during streaming ingestion so _the only allowed partitioning scheme is Dynamic_.
- Because data can always be added incrementally, rollup is _best effort_.

## Managing the Lifecycle

This is where many databases would add an external maintenance process that reorganizes data. It is the beauty of Druid that it handles this reorganization largely automatically by a process called _autocompaction_. Here are a few notes in passing about autocompaction and its capabilities.

I discussed autocompaction briefly in [my blog about data lifecycle management in Druid](https://blog.hellmar-becker.de/2023/01/22/apache-druid-data-lifecycle-management/). It is a data compaction process that:

- is done automatically by the Coordinator in the background
- has a simple configuration, either through the Druid API or through the Unified Console GUI
- it is basically a reindexing job - it takes all the segments for a given time chunk and re-ingests them into the same datasource, creating a new version.

Autocompaction can:

- make sure segments have a **size close to the target value**;
- set/modify the **partitioning scheme**;
- modify **rollup** settings;
- modify **segment granularity**;
- modify **query granularity**.

It also has a setting to leave the newest data alone so as not to interfere with the ongoing ingestion.

### Set partitioning scheme

if you would like to achieve perfect rollup, use either hash or range

if you know typical query patterns in advance, range is recommended

### Modify rollup settings

you can go from a detail to a rollup table yada yada but mostly leave this alone

### Modify segment granularity

use with caution, this can make sense if your data volume is too low to have even one big enough segment per time chunk, or if you want to force secondary partitioning into effect. (if there is only one segment per time chunk, secondary partitioning will do essentially nothing.) make sure segment granularities roll up into each other neatly (so no week to month), or else you are in for some surprises. (link to data lifecycle blog)

### Modify query granularity

this is a data lifecycle operation, which actually aggregates data further. this is not normally what you do when configuring a segment merge autocompaction, leave it alone

### Configure grace period for recent data

skipOffsetFromLatest

default P1D (one day)

this is an ISO 8601 time period

## Configuring it in the wizard

this can be configured using the wizard or [API](link to API doc)

![Screenshot of autocompaction wizard]()

## Outlook

in 28 - concurrent ingestion and autocompaction

## Conclusion

- yada yada
- ...
- 

