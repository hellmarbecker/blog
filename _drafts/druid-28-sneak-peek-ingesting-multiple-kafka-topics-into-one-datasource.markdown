---
layout: post
title:  "Druid 28 Sneak Peek: Ingesting Multiple Kafka Topics- into One Datasource"
categories: blog apache druid imply streaming kafka tutorial
---
![Druid Cookbook](/assets/2021-12-21-elf.jpg)

[Apache Druid](https://druid.apache.org/) has the concept of [supervisors](https://druid.apache.org/docs/latest/development/extensions-core/kafka-ingestion) that orchestrate ingestion jobs and handle data handoff and failure recovery. Per datasource, you can have exactly one supervisor.

Until recently, that meant that one datasource could only ingest data from one stream. But many of my customers asked whether it would be possible to multiplex several streams into populating one datasource. With Druid 28, this becomes possible!

In this quick tutorial, you will learn how to utilize the new options in Kafka ingestion so as to stream multiple topics into one Druid datasource. You will need:

- a Druid 28 preview build
- any Kafka installation
- I am using [Francesco's pizza simulator](https://github.com/Aiven-Labs/python-fake-data-producer-for-apache-kafka) for generating test data.

## Building the distribution

You can use the 30 day free trial of [Imply's Druid release](https://imply.io/download-imply/) which already contains the new features. But if you want to build the open source version:

Clone the Druid repository, check out the 28.0.0 branch, and build the tarball:

```bash
git clone https://github.com/apache/druid.git
cd druid
git checkout 28.0.0
mvn clean install -Pdist -DskipTests
```

Then follow the [instructions](https://druid.apache.org/docs/latest/development/build.html) to locate and install the tarball.

## Generating test data

I am assuming that you are running Kafka locally on the standard port and that you have enabled auto topic creation.

Clone the simulator repository, change to the simulator directory and run three instances of pizza delivery:

```bash
python3 main.py --host localhost --port 9092 --topic-name pizza-mario --max-waiting-time 5 --security-protocol PLAINTEXT --nr-messages 0 >/dev/null &
python3 main.py --host localhost --port 9092 --topic-name pizza-luigi --max-waiting-time 5 --security-protocol PLAINTEXT --nr-messages 0 >/dev/null &
python3 main.py --host localhost --port 9092 --topic-name my-pizza --max-waiting-time 5 --security-protocol PLAINTEXT --nr-messages 0 >/dev/null &
```

