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

- 
