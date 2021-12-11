---
layout: post
title:  "Semi-Structured Data in Apache Druid: Loading HN comments"
categories: blog druid imply ingestion tutorial snowflake bigquery
---

Today I came across this tweet by a Snowflake engineer:

https://twitter.com/felipehoffa/status/1469392108452679680

So, he is exporting semi structured data and sending them to a data warehouse. [Apache Druid](https://druid.apache.org/) is perfectly suited for semi-structured data.

Druid can easily adapt to new or changing fields in the data, and it comes with built-in full indexing of all dimensions.

Let's give it a try!

## Preparing the data

You can follow Felipe's [instructions](https://hoffa.medium.com/loading-all-hacker-news-comments-into-snowflake-in-less-than-1-minute-728100f38272) on how to export the data from Google Big Query - I exported everything as gzipped Parquet files.

Ingest these data into your Druid instance. Any Druid version will do, but make sure you have the [Parquet](https://druid.apache.org/docs/latest/development/extensions-core/parquet.html) and [Google](https://druid.apache.org/docs/latest/development/extensions-core/google.html) enabled. Also, follow the instructions on setting up GCS access in the docs if you don't want to make your data publicly accessible.

## Loading the data

Go to the `Load data` wizard and select `Google Cloud Storage` as the source:

![](/assets/2021-12-11-1.jpg)

Enter the Google Cloud location where your data resides, in the `Google Cloud prefixes` field. You will see a (garbled) preview:

![](/assets/2021-12-11-2.jpg)

But once you configure the Parquet parser, it all looks fine:

![](/assets/2021-12-11-3.jpg)

## Flexible schemas!

Because the data is semi-structured, we want to be able to [pick up any new or changed fields as they occur](/2021/08/13/experiments-with-schema-evolution-in-apache-druid/). This is called [schemaless ingestion](https://druid.apache.org/docs/latest/ingestion/ingestion-spec.html#inclusions-and-exclusions) in Druid and it is the most flexible way to handle semi-structured data. You can flexibly ingest any new fields, and they will be automatically indexed too!

In order to achieve this, in the `Configure schema` screen, switch the `Explicitly specify dimension list` toggle to the off position:

![](/assets/2021-12-11-4.jpg)

Configure monthly segments and leave dynamic partitioning in place. You could improve things even more by changing the partitioning strategy, but that's another story for another blogpost.

![](/assets/2021-12-11-6.jpg)

Reviewing the ingestion spec, this is what a schemaless spec looks like:

![](/assets/2021-12-11-7.jpg)

You can query the data directly from the Druid console, or use a tool like [Pivot](https://imply.io/product/imply-pivot) to explore the data:

![](/assets/2021-12-11-9.jpg)

## Learnings

- Apache Druid is built to handle semi-structured data in a very natural way.
- With automatic schema evolution, new fields are picked up seamlessly.
- As a bonus, all the fields are indexed.
- This allows for super fast data exploration, and the ability to power web apps!
