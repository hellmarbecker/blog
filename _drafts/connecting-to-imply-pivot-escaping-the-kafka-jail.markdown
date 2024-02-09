---
layout: post
title:  "Connecting to Imply Pivot: Escaping the Kafka Jail"
categories: blog apache druid imply kafka aws security tutorial
twitter:
  image: /assets/2021-12-21-elf.jpg
---

If you want to analyze your event data, Imply Polaris is a great solution. It is powered by Apache Druid and connects natively to virtually any flavor of Kafka (including API compatible event streaming platforms, such as Azure EventHub and Redpanda (link to blog)). Polaris as a fully managed cloud service adds the figurative "Easy" button to this scenario - with just a few clicks you can connect to your Kafka topic, sample the data, and build your data model.

What I see often in my customer project is that organizations are running Kafka within a private network, 

- so how do you get out?
  
- private networks: need a proxy but then you have to reconfigure advertised listeners

- one option would be a client that consumes kafka and sends data to polaris thru push api
- but that doesn't come with delivery guarantees

- better plan: use the pattern of [data isolation](https://developers.redhat.com/articles/2023/11/13/demystifying-kafka-mirrormaker-2-use-cases-and-architecture#use_cases)

- instead of standing up your own kafka, use confluent cloud

- deploy an instance of mirror maker in the local private network

- make sure to configure the authentication right

- mention the topics to be manually created

## MirrorMaker 2 configuration 

Create a configuration file `mm.properties` following this template: 

```properties

# MirrorMaker 2 configuration 

clusters = source, destination 

source.bootstrap.servers = stream1.redbus.com:9092, stream2.redbus.com:9092, stream3.redbus.com:9092, stream4.redbus.com:9092 

destination.bootstrap.servers = <BOOTSTRAP SERVER ID>.ap-south-1.aws.confluent.cloud:9092 
destination.security.protocol=SASL_SSL 
destination.sasl.mechanism=PLAIN 
destination.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<API KEY>' password='<API SECRET>'; 

source->destination.enabled = true 

source->destination.topics = digitalfingerprintdata,mriprodstream,primoDataStream 

############################# Internal Topic Settings  ############################# 
# The replication factor for mm2 internal topics "heartbeats", "destination.checkpoints.internal" and 
# "mm2-offset-syncs.destination.internal" 
# For anything other than development testing, a value greater than 1 is recommended to ensure availability such as 3. 
checkpoints.topic.replication.factor=1
heartbeats.topic.replication.factor=1
offset-syncs.topic.replication.factor=1 

# The replication factor for connect internal topics "mm2-configs.destination.internal", "mm2-offsets.destination.internal" and 
# "mm2-status.destination.internal" 
# For anything other than development testing, a value greater than 1 is recommended to ensure availability such as 3. 
offset.storage.replication.factor=1
status.storage.replication.factor=1 
config.storage.replication.factor=1 
```

note, the replication factors are 1 because only one broker

## How to run MirrorMaker 2

```bash
bin/connect-mirror-maker.sh mm.properties 
```

This will mirror all 3 topics into topics in Confluent Cloud with the prefix source.*.  
