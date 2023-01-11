---
layout: post
title:  "Building an Event Analytics Pipeline with Confluent Cloud and Imply Polaris"
categories: blog apache druid imply kafka confluent
---

![Streaming Analytics Architecture]()

A modern streaming analytics pipeline is built around two central components:

- an event streaming platform
- an event analytics platform.

This is conveniently achieved using [Confluent](https://www.confluent.io/) Cloud as a SaaS event streaming platform, and [Imply](https://imply.io/) Polaris as a SaaS realtime analytics database.

In this tutorial, I am going to show you how to set up a pipeline that

- generates a simulated clickstream event stream and sends it to Confluent Cloud
- processes the raw clickstream data using managed [ksqlDB](https://ksqldb.io/) in Confluent Cloud
- delivers the processed stream using Confluent Cloud
- and ingests these JSON events, using a native connection, into Imply Polaris.

But first, let's cover some of the basics.

## The Case for Streaming Analytics

### Old School Analytics

![Classical OLAP architecture]()

This is how we used to do analytics, roughly 20 years ago. You would have your operational systems that collected data into transactional, or OLTP, databases.

OLTP databases are built to process single updates or inserts very quickly. In traditional relational modeling this means you have to normalize your data model, ideally to a point where every item exists only once in a database. The downside is when you want to run an analytical query that aggregates data from different parts of your database, these queries require complex joins and can become very expensive, hurting query times and interfering with the transactional performance of your database.

Hence another type of databases was conceived which is optimized for these analytical queries: OLAP databases. These come in different shapes and flavours, but generally a certain amount of denormalization and possibly preaggregation is applied to the data.

The process that ships data from the transactional system to the OLAP database is called ETL - Extract, Transform, Load. It is a batch process that would run on a regular basis, for instance once a night or once every week. The frequency of the batch process determines how "fresh" your analytical data is.

In the old world, where analytical users would be data analysts inside the enterprise itself, that was often good enough. But nowadays, in the age of connectivity, everyone is an analytics user. If you check your bank account balance and the list of transactions in your banking app on your smartphone, you are an analytics user. And if someone transfers funds to your account, you expect to see the result now and not two days later.

A better way of processing data for analytics was needed. And we'll look at that now.

### Big Data and the Lambda Architecture

About ten years ago, the big data craze came up around the Hadoop ecosystem. Hadoop brought with it the ability to handle historical data to a previously unknown scale, but it also already had real time [note: this is not hard real time like in embedded systems! If necessary elaborate] capability, with tooling like Kafka, Flume, and HBase.

The first approach to getting analytics more up to date was the so called lambda architecture, where incoming data would be sent across two parallel paths:

- A realtime layer with low latency and limited analytical capabilities
- A highly scalable but slower batch layer.

This way, you would be able to retrieve at least some of the analytics results immediately, and get the full results the next day.

A common serving layer would be the single entry point for clients.

This architectural pattern did the job for a while but it has an intrinsic complexity that is somewhat hard to master. Also, when you have two different sources of results, you need to go through an extra effort to make sure that the results always match up.

### Kappa Architecture

A better way needed to be found. It was created in the form of the kappa architecture. In the kappa architecture, there is only one data path and only one result for a given query. The same processing path gives (near) real time results and also fills up the storage for historical data. 

The kappa architecture handles incoming streaming data and historical data in a common, uniform way and is more robust than a lambda architecture. Ideally you still want to encapsulate the details of such an architecture and not concern the user with it. We will come to that in a moment.

## Preparing your Data: Streaming ETL

But first a few words about the part that I didn't cover in the last two slides: How to get the data out of the transactional systems into whatever analytics architecture you have.

Instead of processing batches of data, streaming ETL has to be event driven. There are two ways of processing event data in a streaming ETL pipeline:

- *Simple event processing* looks at one event at a time. Simple event processing is *stateless* which makes it easy to implement but limite the things you can do with it. This is used for format transformations, filtering, or data cleansing, for instance. An example for simple event processing is Apache NiFi.
- *Complex event processing* looks at a collection of events over time, hence it is *stateful* and has to maintain a state store in the background. With that you can do things like windows aggregations, such as sliding averages or session aggregations. You can also join various event streams (think orders and shipments), or enrich data with lookup data that is itself event based. Complex event processing is possible using frameworks like Spark Streaming, Flink, or Kafka Streams.

In this tutorial, I will use ksqlDB. ksqlDB is a community licensed SQL framework on top of Kafka Streams, by Confluent. It is also available as a managed offering in Confluent Cloud, and that is what I will be using.

With ksqlDB, you can write a complex event streaming application as simple SQL statements. ksqlDB queries are typically persistent: unlike database queries, they continue running until they are explicitly stopped, and they continue to emit new events as they process new input events in real time. ksqlDB abstracts away for the most part the detail of event and state handling.



## Prerequisites

For this tutorial, you need a Confluent Cloud account. In this account, create [an environment](https://docs.confluent.io/cloud/current/access-management/hierarchy/cloud-environments.html), [a cluster](https://docs.confluent.io/cloud/current/clusters/create-cluster.html), and [a ksqlDB application](https://docs.confluent.io/cloud/current/get-started/index.html#section-2-add-ksql-cloud-to-the-cluster).

The smallest size of cluster (`Basic`) will do.

Furthermore, you need an Imply Polaris environment. You can sign up for a free trial [here](https://signup.imply.io).

## Data Generation

- how to use the news data generator

## Setting up ksqlDB

## Setting up Imply Polaris

