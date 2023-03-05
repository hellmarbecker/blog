---
layout: post
title:  "Selective Bulk Upserts in Apache Druid"
categories: blog druid imply adtech tutorial update crud
---

talk about the general question of updates
refer to Jad's blog

## The Use Case

describe the adtech use case:
- ad impressions and performance data are loaded from ad network's API, you can only download predefined time intervals
- data may have been updated the next day because late arriving data or some attribution magic
- because we can only download data in bulk, we have to cut out data by key and interval, and transplant the new data instead

## Solution Outline

explain combining inputSource

## Tutorial: How to do it in practise

yada yada what we will do today
- load initial data sample for multiple ad networks
- do the upsert for one network
this can be done with the current OSS druid quickstart

note, where do we place the files, local file system for a 1 node install but a cluster would use s3 or whatever

### The data sample

- sample 1: 2 weeks worth of data, 2-3 networks, possibly campaign id
- sample 2: 1 weeks worth data, 1 network, some numbers will have changed
all json, or csv who cares

### Initial load

ingest sample 1 with this ingestion spec: (paste spec)
note this can be done with the wizard

### Doing the upsert

now replace with new data (paste spec)

explain how it is done
- combining input source is like a UNION
- delegates are the parts of the union, they can be any inputsource, there can be more than 2
- #1: reindexes the existing data
  - there is a filter that leaves out the interval and key we want to replace
  - filters are a kind of boolean prefix notation, they tell us which rows to _keep_
  - so here it is: not(and(network_key=gaggle, timestamp in \[interval\]))
- #2: pulls in the new data
  - logical complement of filter #1: and(network_key=gaggle, timestamp in \[interval\])

and done!

## Conclusion

yada yada



