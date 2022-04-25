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
- We also need a data simulator. I am using a siimple script here


## Installing Pulsar and KoP

Download pulsar https://pulsar.apache.org/en/download/

QuickStart https://pulsar.apache.org/docs/en/standalone/

download kop from streamnative github https://github.com/streamnative/kop/releases

install KoP in `apache-pulsar-2.10.0` directory:

```
mkdir protocols
cp ~/Downloads/pulsar-protocol-handler-kafka-2.10.0.2.nar protocols 
```

now add the necessary config settings

In `conf/standalone.conf`:
```
messagingProtocols=kafka
protocolHandlerDirectory=./protocols
allowAutoTopicCreationType=partitioned     # !! overrides the default setting !!
kafkaListeners=PLAINTEXT://127.0.0.1:9092
kafkaAdvertisedListeners=PLAINTEXT://127.0.0.1:9092
brokerEntryMetadataInterceptors=org.apache.pulsar.common.intercept.AppendIndexMetadataInterceptor
brokerDeleteInactiveTopicsEnabled=false    # !! overrides the default setting !!
kafkaTransactionCoordinatorEnabled=true    # this is not in the docs but required for Druid
```
https://imply.io/blog/community-spotlight-apache-pulsar-and-apache-druid-get-close/

## Installing and Preparing Druid

let's just use Pulsar's zookeeper

apache-druid-0.22.1/conf/supervise/single-server/micro-quickstart-nozk.conf:

~~`!p10 zk bin/run-zk conf`~~

bin/start-micro-quickstart-nozk:

`exec "$WHEREAMI/supervise" -c "$WHEREAMI/../conf/supervise/single-server/micro-quickstart-nozk.conf"`
