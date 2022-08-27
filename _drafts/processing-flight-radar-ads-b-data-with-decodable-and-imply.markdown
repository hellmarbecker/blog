---
layout: post
title:  "Processing Flight Radar (ADS-B) Data with Imply, Decodable, and Confluent Cloud"
categories: blog druid imply tutorial kafka streamprocessing sql flink decodable
---

Imply's analytical platform, based on [Apache Druid](https://druid.apache.org/), is a great tool for fast event analytics, and with the advent of [Imply Polaris](https://imply.io/imply-polaris/) as a managed service these capabilities are ever easier to use.

In order to prepare streaming data for ingestion by Druid, [Apache Flink](https://flink.apache.org/) has become a popular programming framework.

Flight radar data are sent by commercial and most private aircraft, and can easily be received and decoded usin a Raspberry Pi and a DVB-T receiver stick.

In this tutorial, I am going to show you how to set up a pipeline that

- generates an event stream from flight radar data
- converts this stream into JSON format using [Decodable](https://www.decodable.co/)
- and ingests these JSON events into [Imply Polaris](https://imply.io/imply-polaris/) using the new Pull Ingestion feature that was released in August 2022. 

## Generating Data

## Sending Data to Kafka

I am using Confluent Cloud and `kcat` as a client. (The package that comes with the current release of [Raspberry Pi OS](https://www.raspberrypi.com/software/operating-systems/), which is based on Debian 11, uses the old name `kafkacat`.)

You have to set up the topic for the flight data, as well as an API key and ACLs in Confluent Cloud. I have covered this in detail in [an earlier post](/2021/10/19/reading-avro-streams-from-confluent-cloud-into-druid/).

```bash
#!/bin/bash

CC_BOOTSTRAP="<confluent cloud bootstrap server>"
CC_APIKEY="<api key>"
CC_SECRET="<secret>"
CC_SECURE="-X security.protocol=SASL_SSL -X sasl.mechanism=PLAIN -X sasl.username=${CC_APIKEY} -X sasl.password=${CC_SECRET}"
TOPIC_NAME="adsb-raw"

nc localhost 30003 \
    | awk -F "," '{ print $5 "|" $0 }' \
    | kafkacat -P -t ${TOPIC_NAME} -b ${CC_BOOTSTRAP} -K "|" ${CC_SECURE}
```
