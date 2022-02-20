---
layout: post
title:  "Ingesting Protobuf messages into Apache Druid"
categories: blog imply druid confluent kafka eventstreaming tutorial
---

Since [version 0.22](https://github.com/apache/druid/releases/tag/druid-0.22.0), Druid supports reading Protobuf data from a Kafka stream. Let's look at this in practice.

Read Protobuf data from a [schema aware](https://docs.confluent.io/cloud/current/sr/schemas-manage.html) [Confluent Cloud](https://confluent.cloud) cluster into [Apache Druid](https://druid.apache.org/). Use the Druid 0.22 [micro-quickstart](https://druid.apache.org/docs/latest/tutorials/index.html) setup for this exercise.

![Streaming analytics architecture](/assets/2021-10-19-0-architecture.png)

## Setting up the Topic and all the rest

## Generating data

There are various ways to create Protobuf data and stream them into Druid. One method is described in [the Protobuf extension documentation](https://druid.apache.org/docs/0.22.1/development/extensions-core/protobuf.html), involving a custom Python script. There's also [this blog](https://dzone.com/articles/how-to-use-protobuf-with-apache-kafka-and-schema-r), which shows how to do it in Java. I am going to look at two other options here:
- using Confluent Cloud and
- using Confluent Platform and Kafka Connect locally.

### Generating data with Confluent Cloud

The easiest way would be to use Confluent Cloud - I described this in [my post about AVRO integration](/2021/10/19/reading-avro-streams-from-confluent-cloud-into-druid/). Instead of AVRO data, choose to generate Protobuf, that's all. You will also have to set up security in Druid according to the instructions in that post.

But you can also use the script that comes with the Protobuf extension

Or roll your own see [this blog](https://dzone.com/articles/how-to-use-protobuf-with-apache-kafka-and-schema-r)

### Generating data with Kafka and Kafka Connect

In order to generate data and stream them through Kafka, I am going to use the Community Edition of Confluent Platform, loosely following Confluent's [quickstart](https://docs.confluent.io/platform/current/quickstart/ce-docker-quickstart.html) instructions.

First, let's download the docker compose file for the Community Edition.

```bash
curl --silent --output docker-compose.yml \ 
https://raw.githubusercontent.com/confluentinc/cp-all-in-one/7.0.1-post/cp-all-in-one-community/docker-compose.yml
```

Alternatively, you can clone [the entire repository](https://github.com/confluentinc/cp-all-in-one).

If you try to start this version alongside Apache Druid, you will notice that some of the same port numbers that Confluent Platform wants to use are also claimed by Druid. Fortunately, this is easy to fix. Using a text editor of your choice, replace each of the offending port numbers by a free port. Here's my list.

|Service |Standard port |Custom port |
|:---|---:|---:|
|Zookeeper | 2181| 12181|
|Schema Registry | 8081| 18081|
|REST Proxy | 8082| 18082|
|Kafka Connect | 8083| 18083|

Note down the port numbers that you assigned, because you will need them later.

Then follow the [quickstart guide](https://docs.confluent.io/platform/current/quickstart/ce-docker-quickstart.html#step-1-download-and-start-cp) by running

```bash
docker-compose up -d
```

You can check the deployment with the `docker ps` command. Or query the Kafka metadata using [`kcat`](https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html):
```
kcat -b localhost:9092 -L
```
If Kafka is up and running, this will return a list of topics and partitions.


### Documentation of the Protobuf extension

https://github.com/apache/druid/blob/ec334a641b3f56077d2693980128e872f08d8611/docs/development/extensions-core/protobuf.md

### ProtobufBytesDecoder

this can be either `schema_registry` or `file`

https://github.com/apache/druid/blob/0e0c1a1aaf468d2e082fffa9cab8a98013f2b536/extensions-core/protobuf-extensions/src/main/java/org/apache/druid/data/input/protobuf/ProtobufBytesDecoder.java
