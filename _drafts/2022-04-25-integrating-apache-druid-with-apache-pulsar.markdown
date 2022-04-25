---
layout: post
title:  "Integrating Apache Druid with Apache Pulsar"
categories: blog apache druid imply pulsar streamnative eventstreaming tutorial
---

![Pulsar to Druid](/assets/2022-04-25-01-banner.png)

With companies adopting the [event streaming pattern](https://medium.com/capital-one-tech/event-streaming-an-additional-architectural-style-to-supplement-api-design-703c4f801722), analytics has to become more "realtime" too. A great database for event analytics is [Apache Druid](https://druid.apache.org). Druid connects natively to various event streaming systems such as Kafka and AWS Kinesis.

One of the most modern event streaming platforms is [Apache Pulsar](https://pulsar.apache.org). Pulsar has a modern, cloud native architecture that separates the storage layer from the message brokers, claiming unprecendented scalability and flexibility. Sure it would be great to stream events directly from Pulsar into Druid!

The idea has been discussed before [by the Imply and StreamNative teams](https://imply.io/blog/community-spotlight-apache-pulsar-and-apache-druid-get-close/), but up until recently no turnkey solution existed. Pulsar offered a drop-in client library with call compatibility to the Kafka libraries, but using it would usually require rebuilding the entire application, which is not for everyone. 

Recently, the folks at [StreamNative](https://streamnative.io/) released  [Kafka-on-Pulsar (KoP)](https://streamnative.io/blog/tech/2020-03-24-bring-native-kafka-protocol-support-to-apache-pulsar/), which is a plugin that makes Pulsar look like a Kafka cluster from an application perspective! Since the compatibility works on the network protocol level, existing clients should continue to work.

If we could set up a Pulsar cluster with KoP enabled, we should be able to leverage the existing Kafka integration in Druid to ingest data directly from Pulsar. Let's try it out!

We will need to install
- Apache Pulsar and KoP
- Apache Druid
- We also need a data simulator. This will be a simple script that uses the Pulsar commandline client, so we will also see the mapping between Kafka topic names and Pulsar topic names.

I am running all components directly on my laptop.

## Installing Pulsar and KoP

Download Pulsar from [the Apache Pulsar download website](https://pulsar.apache.org/en/download/), and untar into your home directory. At the time of this writing, the latest release is 2.10.0.

Download KoP from [StreamNative's GitHub repository](https://github.com/streamnative/kop/releases). Make sure that the release number of KoP you download matches your Pulsar release. I am using  version 2.10.0.2.

Install KoP in `apache-pulsar-2.10.0` directory - you need to create a `protocols` directory and copy the nar file into it:

```bash
cd $HOME/apache-pulsar-2.10.0
mkdir protocols
cp ~/Downloads/pulsar-protocol-handler-kafka-2.10.0.2.nar protocols 
```

### Configuration for KoP

You have to add a few necessary configuration settings to `conf/standalone.conf` as per https://github.com/streamnative/kop/blob/master/docs/kop.md:

```
messagingProtocols=kafka
protocolHandlerDirectory=./protocols
allowAutoTopicCreationType=partitioned     # !! overrides the default setting !!
kafkaListeners=PLAINTEXT://127.0.0.1:9092
kafkaAdvertisedListeners=PLAINTEXT://127.0.0.1:9092
brokerEntryMetadataInterceptors=org.apache.pulsar.common.intercept.AppendIndexMetadataInterceptor
brokerDeleteInactiveTopicsEnabled=false    # !! overrides the default setting !!
```

Note that the settings for `allowAutoTopicCreationType` and `brokerDeleteInactiveTopicsEnabled` override the default settings, you have to find the lines with the default settings and edit or remove them.

Druid uses Kafka transactions, so we need one more option to make KoP work with Druid:

```
kafkaTransactionCoordinatorEnabled=true    # this is not in the docs but required for Druid
```

### Topic Mapping

While Kafka has a flat namespace, Pulsar has a naming scheme `tenant`/`namespace`/`topic`. You can set the default tenant and namespace for Kafka topics like so: (In `conf/standalone.conf`)

```
kafkaTenant=kop
kafkaNamespace=kop
```

Finally, start Pulsar according to the [standalone quickstart](https://pulsar.apache.org/docs/en/standalone/) instructions:

```
bin/pulsar standalone
```

## Installing and Preparing Druid

let's just use Pulsar's zookeeper

apache-druid-0.22.1/conf/supervise/single-server/micro-quickstart-nozk.conf:

~~`!p10 zk bin/run-zk conf`~~

bin/start-micro-quickstart-nozk:

`exec "$WHEREAMI/supervise" -c "$WHEREAMI/../conf/supervise/single-server/micro-quickstart-nozk.conf"`
