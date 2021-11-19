---
layout: post
title:  "Apache Druid in the Publishing Industry"
categories: blog imply druid industry
---
[Apache Druid](https://druid.apache.org/) is a high performance, distributed analytical datastore that has natural uses in many industries. Today, I am going to look at use cases in the publishing industry.

## Challenges in the Publishing Industry

...

## What is so special about Druid?

Here are a few unique points about Druid:

### Built to scale

Druid is in itself a distributed system with built in service discovery and resiliency. It is unique in that it scales not only for large amounts of data, but also for a large amount of concurrent queries.

This enables use cases in which a dashboard application fires lots of incremental queries

### Streaming ingestion

Druid connects directly to event streaming platforms like [Apache Kafka](https://kafka.apache.org/) and [AWS Kinesis](https://aws.amazon.com/kinesis/). With the proper frontend, this means you can get updated insights about your reader's behavior within seconds.

### Smart data lookups

Druid has the built in ability to use secondary data sources as dimensions. For instance, Druid can link your behavior data against your asset database, showing headlines and other CMS information.

### Multi-value Dimensions

With the built in [multi-value dimensions](/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/), Druid has a very performant way to model common one-to-many relationships in publishing data, like tag clouds or mapping a news item to multiple sections.

## How does Druid help the Publishing Industry?
