---
layout: post
title:  "Partitioning in Druid - Part 2: Single and Multi Dimension Partitioning"
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

First of all, note how the iingetsion tasks spawns multiple phases of subtasks. First, there is a task the determines the value distribution of the partition key; then the index is generated, and finally there are three tasks to shuffle the data such that every item ends up in the appropriate partition.

![Ingestion tasks](/assets/2022-01-14-2-single-tasks.jpg)

This is what comes out of the ingestion:

