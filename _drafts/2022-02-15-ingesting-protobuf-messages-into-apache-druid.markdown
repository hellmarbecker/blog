---
layout: post
title:  "Ingesting Protobuf messages into Apache Druid"
categories: blog imply druid confluent kafka eventstreaming tutorial
---

Read Protobuf data from a [schema aware](https://docs.confluent.io/cloud/current/sr/schemas-manage.html) [Confluent Cloud](https://confluent.cloud) cluster into [Apache Druid](https://druid.apache.org/). Use the Druid 0.22 [micro-quickstart](https://druid.apache.org/docs/latest/tutorials/index.html) setup for this exercise.

![Streaming analytics architecture](/assets/2021-10-19-0-architecture.png)

## Setting up the Topic and all the rest

## Generating data

Use Confluent Cloud Kafka Connect

But you can also use the script that comes with the Protobuf extension

Or roll your own see [this blog](https://dzone.com/articles/how-to-use-protobuf-with-apache-kafka-and-schema-r)

## Generating data with Kafka and Kafka Connect

```bash
curl --silent --output docker-compose.yml \ 
https://raw.githubusercontent.com/confluentinc/cp-all-in-one/7.0.1-post/cp-all-in-one-community/docker-compose.yml
```

Or you can clone [the entire repository](https://github.com/confluentinc/cp-all-in-one)

Then follow the [quickstart guide](https://docs.confluent.io/platform/current/quickstart/ce-docker-quickstart.html#step-1-download-and-start-cp) by running

```bash
docker-compose up -d
```
 
Need to map ports differently because the same ports are used by Druid

|Service |Standard port |Custom port |
|:---|---:|---:|
|Zookeeper | 2181| 12181|
|Schema Registry | 8081| 18081|
|REST Proxy | 8082| 18082|
|Kafka Connect | 8083| 18083|


### Documentation of the Protobuf extension

https://github.com/apache/druid/blob/ec334a641b3f56077d2693980128e872f08d8611/docs/development/extensions-core/protobuf.md

### ProtobufBytesDecoder

this can be either `schema_registry` or `file`

https://github.com/apache/druid/blob/0e0c1a1aaf468d2e082fffa9cab8a98013f2b536/extensions-core/protobuf-extensions/src/main/java/org/apache/druid/data/input/protobuf/ProtobufBytesDecoder.java
