---
layout: post
title:  "Connecting Apache Pulsar to Apache Druid"
categories: blog apache druid imply pulsar streamnative eventstreaming tutorial
---

Download pulsar https://pulsar.apache.org/en/download/

QuickStart https://pulsar.apache.org/docs/en/standalone/
```
mkdir protocols
cp ~/Downloads/pulsar-protocol-handler-kafka-2.10.0.2.nar protocols 
```
In `conf/standalone.conf`:
```
messagingProtocols=kafka
protocolHandlerDirectory=./protocols
allowAutoTopicCreationType=partitioned ##!!
kafkaListeners=PLAINTEXT://127.0.0.1:9092
kafkaAdvertisedListeners=PLAINTEXT://127.0.0.1:9092
brokerEntryMetadataInterceptors=org.apache.pulsar.common.intercept.AppendIndexMetadataInterceptor
brokerDeleteInactiveTopicsEnabled=false ##!!
kafkaTransactionCoordinatorEnabled=true 
```
https://imply.io/blog/community-spotlight-apache-pulsar-and-apache-druid-get-close/

messagingProtocols=kafka
protocolHandlerDirectory=./protocols
allowAutoTopicCreationType=partitioned
kafkaListeners=PLAINTEXT://127.0.0.1:9092
kafkaAdvertisedListeners=PLAINTEXT://127.0.0.1:9092
brokerEntryMetadataInterceptors=org.apache.pulsar.common.intercept.AppendIndexMetadataInterceptor
brokerDeleteInactiveTopicsEnabled=false
kafkaTransactionCoordinatorEnabled=true

apache-druid-0.22.1/conf/supervise/single-server/micro-quickstart-nozk.conf:

~~!p10 zk bin/run-zk conf~~

