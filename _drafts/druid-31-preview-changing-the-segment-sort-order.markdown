---
layout: post
title:  "Druid 31 Preview: Changing the Segment Sort Order"
categories: blog apache druid imply sql ingestion tutorial
---

Up until Druid 30, data in a Druid segment was always sorted by time. But this is about to change. Druid 31 comes with an experimental option to change the segment sort order. With [Druid Summit 2024](https://druidsummit.org/) and the announcement of Druid 31 around the corner, let's take a look at this new feature.

This blog is based on the public documentation of [this pull request](https://github.com/apache/druid/pull/16849). 

_**Disclaimer:** This tutorial uses undocumented functionality and unreleased code. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. In addition, the entire build, or execution, may fail. Your mileage may vary._

## Motivation

Why would you want to change the segment sort order? This is mainly about data compression. Druid can compress data much better if blocks of contiguous rows have the same value in a column. It is also consistent with the tendency to better organize data within Druid.

For instance, [range partitioning of data](/2022/01/25/partitioning-in-druid-part-3-multi-dimension-range-partitioning/) is now becoming the gold standard for batch ingestion. In fact, [Imply's Polaris service](https://imply.io/imply-fully-managed-dbaas-polaris/) offers range partitioning for both batch and streaming ingestion. (This is done by ingesting data into dynamic partitions first and running a preconfigured [autocompaction](https://druid.apache.org/docs/latest/data-management/automatic-compaction) job in the background that makes sure all data is partitioned according to the configured settings. You can do this in open source Druid too, but you have to configure it yourself.)

In most cases, 



Let's try it out!

