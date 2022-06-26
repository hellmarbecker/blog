---
layout: post
title:  "New in Druid 0.23: Ingesting Kafka Keys and Timestamps"
categories: blog imply druid kafka eventstreaming tutorial
---

![Streaming analytics architecture](/assets/2021-10-19-0-architecture.png)

Among the many new features in [Druid 0.23](https://github.com/apache/druid/releases/tag/druid-0.23.0) is the ability to handle Kafka metadata upon ingestion. This includes

- the Kafka timestamp
- the message key
- Kafka message headers.

While customers of [Imply](https://imply.io/) have been able to take advantage of this functionality for a few months already, it is still officially considered an alpha feature. But why would we want this anyway? 

Let's have a look!

## Why Does It Matter?

Up until now, Druid's Kafka ingestion module used to look only at the Kafka message value. But if you try to optimize your Kafka partitioning strategy, you would usually set up a partitioning key that gives you the ability to both scale and parallelize, and to have a partial order guarantee in your message stream. This is important when you look at message streams that have the notion of a session or a transaction which is tied together by a common session ID field and where the order of events matters.

In that case it makes a lot of sense to make the session ID the Kafka key, and having it again inside the message value would be an unnecessary data duplication.

Modern stream processing systems like Confluent's [ksqlDB](https://ksqldb.io/) honor this fact: if you create a new stream joining two source streams (or a stream and a table, for that matter), the join key will be made the new Kafka message key, and it will not be represented in the message value unless you put a copy there explicitly. Thus, having the ability to **ingest the key** directly simplifies the preprocessing pipeline and makes it more organic.

Likewise, a Kafka message always has a **timestamp**. In the absence of other information, this would be the time when the message has been produced into Kafka. But for many cases this is good enough, and again, with pre-0.23 Druid, you would have to create another timestamp field inside the message for it to be ingested.

Lastly, **Kafka headers** are used by some system to track data provenance and lineage. In order to fit into the larger context of a data governance architecture, you would want to preserve and/or process Kafka headers as the data maakes its way into and through Druid.

I am going to present a short tutorial that shows how to handle Kafka keys and timestamps within Druid. Headers work very similary and are left as an exercise for the reader.

## Setting Up Kafka

You will need a Kafka service for this tutorial. If you don't have one ready, you can install it on your laptop using [Confluent's Community Docker images](https://docs.confluent.io/platform/current/installation/docker/image-reference.html), as I have covered in detail [in a previous post](/2022/05/26/ingesting-protobuf-messages-into-apache-druid/).

For this tutorial, you will need just a Kafka broker and Zookeeper. Here's the Docker compose file:

```
---
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.0.1
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "12181:12181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 12181
      ZOOKEEPER_TICK_TIME: 2000

  broker:
    image: confluentinc/cp-kafka:7.0.1
    hostname: broker
    container_name: broker
    depends_on:
      - zookeeper
    ports:
      - "29092:29092"
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:12181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
```
Leave the default configuration in place - among other options, this will auto-create a topic as soon as we produce into it.

You will also need [`kcat`](https://github.com/edenhill/kcat) as a Kafka client. Although this is an open source tool, you can find valuable information about it [on Confluent's pages](https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html).

I prefer `kcat` over the built-in client utilities because it is more flexible. For instance, it comes with format options to include keys and to show metadata along with the message itself in consumer mode.

Let's produce some data with keys and with implicit timestamps. Between messages, we wait a random number of seconds in order to get different timestamps:

```bash
#!/bin/bash

BROKER=localhost:9092
TOPIC=keytest

for i in {1..20}; do
   echo key$i:message$i | kcat -b $BROKER -P -t $TOPIC -K ":"
   sleep $(( $RANDOM % 10 + 1 ))
done
```

Check if it worked:

```
% kcat -b localhost:9092 -C -t keytest -f "%T:%k:%s\n" -o 0
1656233704435:key1:message1
1656233707521:key2:message2
1656233712602:key3:message3
1656233722675:key4:message4
1656233728750:key5:message5
1656233738825:key6:message6
1656233739894:key7:message7
1656233742965:key8:message8
1656233745040:key9:message9
1656233748117:key10:message10
1656233751189:key11:message11
1656233752262:key12:message12
1656233758331:key13:message13
1656233766409:key14:message14
1656233773485:key15:message15
1656233780557:key16:message16
1656233782631:key17:message17
1656233785704:key18:message18
1656233793770:key19:message19
1656233797840:key20:message20
% Reached end of topic keytest [0] at offset 20
```

This shows the timestamp (in milliseconds since Epoch), the message key and value. We are good to go!

## Setting Up Druid

## Ingesting the Data

## Learnings
