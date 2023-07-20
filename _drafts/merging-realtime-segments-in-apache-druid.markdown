---
layout: post
title:  "Merging Realtime Segments in Apache Druid"
categories: blog apache druid imply realtime
---

## Data Layout and Druid Performance

druid wants to be fast

but druid's performance can be influenced by multiple factors in the data layout

- segment size. the optimum size of a [data segment](link to doc) is about 500 MB. if segments are much bigger, those segments need more resources for querying and also parallelism suffers. but more often there are too many small segments, which slows down query performance.
- partitioning and sorting of data. partitioning gives an extra performance boost when you can partition the segments according to the expected query pattern; also inside a segment data is sorted by time, but then by partitioning key; this further speeds up segment scans by increasing compression ratio. for this to work you need to enable [range partitioning](link to blog).
- rollup. helps even more by pre-aggregating data. ideally you want to have perfect rollup so that each unique combination of dimension values corresponds to exactly one aggregate row. for this to work, again one has to use range or hash partitioning. in fact, with range or hash partitioning only, rollup is always perfect; with [dynamic partitioning](link to blog), rollup is best effort - the resulting table may be multiple times bigger than the optimum.

## The Problem with Streaming Data

streaming data usually does not come in neatly ordered

the point is to have these data available for analytics within a split second after an event occurs so segments are built up in memory and persisted frequently. as a result, after a [handoff](link to docs) by streaming ingestion, segments are not optimal

- segments will usually be smaller than optimum because we cannot wait long to initiate a handoff. in addition, we may have to juggle multiple time chunks simultaneously because of late arriving data
- range partitioning requires multiple steps of mapping and shuffling and merging, we cannot afford to do that during streaming ingestion so the only allowed partitioning scheme is Dynamic
- because data can always be added incrementally, rollup is best effort

## Managing the Lifecycle

autocompaction

- done automatically by the Coordinator in the background
- simple configuration
- basically a reindexing job

autocompaction can:

- set/modify the partitioning scheme
- modify rollup settings
- modify segment granularity
- modify query granularity

it also has a setting to leave the newest data alone so as not to interfere with the ongoing ingestion.

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

## Conclusion

- yada yada
- ...
- 

