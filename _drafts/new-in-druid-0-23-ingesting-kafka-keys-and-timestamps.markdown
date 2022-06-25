---
layout: post
title:  "New in Druid 0.23: Ingesting Kafka Keys and Timestamps"
categories: blog imply druid kafka eventstreaming tutorial
---

Among the many new features in [Druid 0.23](https://github.com/apache/druid/releases/tag/druid-0.23.0) is the ability to handle Kafka metadata upon ingestion. This includes

- the Kafka timestamp
- the message key
- Kafka message headers.

While customers of [Imply](https://imply.io/) have been able to take advantage of this functionality for a few months already, it is still officially considered an alpha feature. But why would we want this anyway? 

Let's have a look at this today.

## Why Does It Matter?

Up until now, Druid's Kafka ingestion module used to look only at the Kafka message value. Now, if you try to optimize your Kafka partitioning strategy, you would usually set up a partitioning key that gives you the ability to both scale and parallelize, and to have a partial order guarantee in your message stream. This is important when you look at message streams that have the notion of a session or a transaction which is tied together by a common session ID field and where the order of events matters.

In that case it makes a lot of sense to make the session ID the Kafka key, and having it again inside the message value would be an unneccessary data duplication.

Modern stream processing systems like Confluent's [ksqlDB](https://ksqldb.io/) honor this fact: if you create a new stream joining two source streams (or a stream and a table, for that matter), the join key will be made the Kafka message key, and it will not be represented in the message value unless you put a copy there explicitly. Thus, having the ability to **ingest the key** directly simplifies the preprocessing pipeline and makes it more organic.

Likewise, a Kafka message always has a **timestamp**. In the absence of other information, this would be the time when the message has been produced into Kafka. But for many cases this is good enough, and again, with pre-0.23 Druid, you would have to create another timestamp field inside the message for it to be ingested.

Lastly, **Kafka headers** are used by some system to track data provenance and lineage. In order to fit into the larger context of a data governance architecture, you would want to preserve and/or process Kafka headers as the data maakes its way into and through Druid.

I am going to present a short tutorial that shows how to handle Kafka keys and timestamps within Druid. Headers work very similary and are left as an exercise for the reader.

## Setting up Kafka

In order to 

## Setting up Druid

## Ingesting the Data

## Learnings
