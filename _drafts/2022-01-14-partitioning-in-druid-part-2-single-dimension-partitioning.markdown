---
layout: post
title:  "Partitioning in Druid - Part 2: Single Dimension Partitioning"
categories: blog apache druid imply tutorial
---
![Test Tubes](/assets/2022-01-14-0-test-tubes.jpg)

This is part 2 of the miniseries about data partitioning in [Apache Druid](https://druid.apache.org/). Part 1 can be found [here](/2022/01/06/partitioning-in-druid-part-1-dynamic-and-hash-partitioning/).

## Recap

In the previous installment we discussed dynamic partitioning and hash partitioning in Druid. We foud out that hash partitioning is great to ensure uniform partition size, but it would not usually help query performance a lot. This is why single dimension (or, short, "single dim") partitioning is available in Druid.

## Single Dim Partitioning

Single Dim Partitioning cuts the partition key dimension into intervals and assigns partitions based on these intervals. Let's see what this looks like, in practice.

![Configure single dim partitioning](/assets/2022-01-14-1-single.jpg)

We just chose a different partitioning type. The partition dimension is still `channel`, and the target size stays the same. Name the datasource `wikipedia-single`.

First of all, note how the ingestion tasks spawns multiple phases of subtasks. First, there is a task the determines the value distribution of the partition key; then the index is generated, and finally there are three tasks to shuffle the data such that every item ends up in the appropriate partition.

![Ingestion tasks](/assets/2022-01-14-2-single-tasks.jpg)

This is what comes out of the ingestion, in terms of key distribution:
```
Partition 0:
2523 #de.wikipedia
 478 #ca.wikipedia
 423 #ar.wikipedia
 251 #el.wikipedia
 222 #cs.wikipedia
Partition 1:
11549 #en.wikipedia
2099 #fr.wikipedia
1383 #it.wikipedia
1256 #es.wikipedia
 749 #ja.wikipedia
Partition 2:
9747 #vi.wikipedia
1386 #ru.wikipedia
1126 #zh.wikipedia
 983 #uz.wikipedia
 263 #uk.wikipedia
```
(I have used the same script as in [part 1](/2022/01/06/partitioning-in-druid-part-1-dynamic-and-hash-partitioning/) and adjusted the datasource name.)

We can see how the channel values are now neatly sorted into alphabetic buckets. There is actually an even better way to view these data by issuing a [segment metadata query](https://druid.apache.org/docs/latest/querying/segmentmetadataquery.html). This is a type of query that is used internally by a Druid cluster all the time in order to know which data lives where, but there is no reason why we wouldn't issue such a query directly.



