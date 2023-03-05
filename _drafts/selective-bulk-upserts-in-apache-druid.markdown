---
layout: post
title:  "Selective Bulk Upserts in Apache Druid"
categories: blog druid imply adtech tutorial update crud
---

[Apache Druid](https://druid.apache.org/) is designed for high query speed. The [data segments](https://druid.apache.org/docs/latest/design/segments.html) that make up a Druid datasource (think table) are generally immutable: You do not update or replace individual rows of data; however you can replace an entire segment by a new version of itself.

Sometimes in analytics, you have to update or insert rows of data in a segment. This may be due to a change in status - such as an order being shipped, or canceled, or returned. Generally, you would have a _key_ column in your data, and based on that key you would update a row if it exists in the table already, and insert it otherwise. This is called `upsert`, after the name of the command that is used in many SQL dialects.

[This Imply blog](https://imply.io/blog/upserts-and-data-deduplication-with-druid/) talks about the various strategies to handle such scenarios with Druid. But today, I want to look at a special case of Upsert, where you want to update or insert a bunch of rows based on a key and time interval.

## The Use Case

I encountered this scenario with some of my AdTech customers. They obtain performance analytics data by issuing API calls to the ad network providers. These API calls allow only certain predefined time ranges to specified - data is downloaded in bulk. Moreover, depending on late arriving conversion data and other factors, metrics associated with the data rows may change over time.

If we want to make these data available in Druid, we will have to cut out existing data by key and interval, and transplant the new data instead, like in this diagram: 

![Combining ingestion](/assets/2023-03-05-01.png)

## Solution Outline

In order to achieve this behavior in Druid, we will use a [`combining` input source](https://druid.apache.org/docs/latest/ingestion/native-batch-input-sources.html#combining-input-source) in the ingestion spec. A combining input source contains a list of delegate input sources - we will use two, but you can actually have more than two.

The ingestion process will read data from all delegate input sources and ingest them, much like what a `union all` in SQL does.

We have to make sure that all input sources have the same schema and, where that applies, the same input format. In practice this means:

- you can combine multiple external sources only if they are all parsed in the same way
- or you can combine external sources like above with any number of `druid` input sources (reindexing).

The latter is what we are going to do.

## Tutorial: How to do it in practise

In this tutorial, we will set up a bulk upsert using the combining input source technique and two stripped down sample data set.

We will:
- load an initial data sample for multiple ad networks
- show the upsert technique by replacing data for one network and a specific date range.

The tutorial can be done using the [Druid 25.0 quickstart](https://druid.apache.org/docs/latest/tutorials/index.html).

Note: Because the tutorial assumes that you are running all Druid processes on a single machine, it can work with local file system data. In a cluster  setup, you would have to use a network mount or (more commonly) cloud storage, like S3.

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



