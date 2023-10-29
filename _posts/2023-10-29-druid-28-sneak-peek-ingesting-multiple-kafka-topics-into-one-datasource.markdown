---
layout: post
title:  "Druid 28 Sneak Peek: Ingesting Multiple Kafka Topics into One Datasource"
categories: blog apache druid imply streaming kafka tutorial
---
![Pizza](/assets/2022-11-23-00-pizza.jpg)

[Apache Druid](https://druid.apache.org/) has the concept of [supervisors](https://druid.apache.org/docs/latest/development/extensions-core/kafka-ingestion) that orchestrate ingestion jobs and handle data handoff and failure recovery. Per datasource, you can have exactly one supervisor.

Until recently, that meant that one datasource could only ingest data from one stream. But many of my customers asked whether it would be possible to multiplex several streams into populating one datasource. With Druid 28, this becomes possible!

In this quick tutorial, you will learn how to utilize the new options in Kafka ingestion so as to stream multiple topics into one Druid datasource. You will need:

- a Druid 28 preview build (see below)
- any Kafka installation
- I am using [Francesco's pizza simulator](https://github.com/Aiven-Labs/python-fake-data-producer-for-apache-kafka) for generating test data.

## Building the distribution

You can use the 30 day free trial of [Imply's Druid release](https://imply.io/download-imply/) which already contains the new features. [Documentation is also available](https://docs.imply.io/latest/druid/development/extensions-core/kafka-supervisor-reference/#ingesting-from-multiple-topics).

But if you want to build the open source version:

Clone the Druid repository, check out the 28.0.0 branch, and build the tarball:

```bash
git clone https://github.com/apache/druid.git
cd druid
git checkout 28.0.0
mvn clean install -Pdist -DskipTests
```

Then follow the [instructions](https://druid.apache.org/docs/latest/development/build.html) to locate and install the tarball, and start Druid. Make sure you are [loading the Kafka indexing extension](https://druid.apache.org/docs/latest/development/extensions-core/kafka-ingestion#load-the-kafka-indexing-service). (It is included in the quickstart but not by default in the Docker image.)

## Generating test data

I am assuming that you are running Kafka locally on the standard port and that you have enabled auto topic creation.

Clone the simulator repository, change to the simulator directory and run three instances of pizza delivery:

```bash
python3 main.py --host localhost --port 9092 --topic-name pizza-mario --max-waiting-time 5 --security-protocol PLAINTEXT --nr-messages 0 >/dev/null &
python3 main.py --host localhost --port 9092 --topic-name pizza-luigi --max-waiting-time 5 --security-protocol PLAINTEXT --nr-messages 0 >/dev/null &
python3 main.py --host localhost --port 9092 --topic-name my-pizza --max-waiting-time 5 --security-protocol PLAINTEXT --nr-messages 0 >/dev/null &
```

If you have set up Kafka differently, you may have to modify these instructions.

## Connecting Druid to the streams

Navigate your browser to the Druid GUI (in the quickstart, this is http://localhost:8888), and start configuring a streaming ingestion:

<img src="/assets/2023-10-29-01-streaming.jpg" width="35%" />

Choose Kafka as the input source. Note how there is a new option `topicPattern` in the connection settings:

![Connection screen](/assets/2023-10-29-02-pattern-setting.jpg)

This is a [regular expression](https://en.wikipedia.org/wiki/Regular_expression) that you can specify in place of the topic name. Let's try to gobble up all our pizza related topics by setting the pattern to _"pizza"_:

![Naive attempt](/assets/2023-10-29-03-naive-pattern.jpg)

Oh, this didn't work as expected. But the documentation and bubble help show us the solution: The topic pattern has to match _the entire topic name_. So, the above expression actually matches like the regular expression `^pizza$`.

Armed with this knowledge, let's correct the pattern:

![Preview with prefix match](/assets/2023-10-29-04-match-both.jpg)

This matches all topic names that start with _"pizza-"_.

## Building the data model

Let's have a look at the `Parse data` screen. Among the [Kafka metadata](/2022/11/23/processing-nested-json-data-and-kafka-metadata-in-apache-druid/), there is a new field containing the source topic for each row of data. The default column name is `kafka.topic` but this is configurable in the Kafka metadata settings on the right hand side:

![Parse screen with metadata settings](/assets/2023-10-29-05-topic-field.jpg)

Proceed to the final data model - the topic name is automatically included as a `string` column:

![Data model](/assets/2023-10-29-06-data-model.jpg)

Before kicking off the ingestion job, you may want to review and edit the datasource name

<img src="/assets/2023-10-29-07-rename-datasource.jpg" width="40%" />

because by default, the datasource name is derived from the topic pattern and may contain a lot of special characters.

Once the supervisor is running, you should see data coming in from both the `pizza-mario` and `pizza-luigi` topics:

![Query example](/assets/2023-10-29-08-query.jpg)

What if you want to pick up all 3 topics? From the above, it should be clear - you need to pad the regular expression with `.*` on _both_ sides:

<img src="/assets/2023-10-29-09-open-pattern.jpg" width="30%" />

You can try it yourself!

## Task management

Druid will pick up any topics that match the `topicPattern`, even if new topics are added during the ingestion.

How are partitions assigned to tasks?

The Supervisor will fetch the list of all partitions from all topics and assign the list of these partitions in same way as it assigns the partitions for one topic. In detail this means (quote from the [documentation](https://docs.imply.io/latest/druid/development/extensions-core/kafka-supervisor-reference/#ingesting-from-multiple-topics)):

> When ingesting data from multiple topics, partitions are assigned based on the hashcode of the topic name and the id of the partition within that topic. The partition assignment might not be uniform across all the tasks.

And looking at the code, this boils down to

```
Math.abs(31 * topic().hashCode() + partitionId) % taskCount
```

This heuristic should give a fairly uniform load, provided that the data volumes per _partition_ are comparable.

## Conclusion

- You can use `topicPattern` instead of `topic` in a Kafka Supervisor spec, to enable ingesting from multiple topics.
- `topicPattern` is a regex but the regex has to match the whole topic name
- You can have as many active ingestion tasks as the total partitions of all topics. Partitions are assigned to tasks using a hashing algorithm.

----
 <p class="attribution">"<a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/26242865@N04/5919366429">Pizza</a>" by <a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/26242865@N04">Katrin Gilger</a> is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-sa/2.0/?ref=openverse">CC BY-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>. </p> 
