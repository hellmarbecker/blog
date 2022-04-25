---
layout: post
title:  "Integrating Apache Druid with Apache Pulsar"
categories: blog apache druid imply pulsar streamnative eventstreaming tutorial
---

![Pulsar to Druid](/assets/2022-04-25-01-banner.png)

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
