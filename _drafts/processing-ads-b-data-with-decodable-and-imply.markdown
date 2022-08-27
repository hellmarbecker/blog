---
layout: post
title:  "Processing ADS-B Data with Imply, Decodable, and Confluent Cloud"
categories: blog druid imply tutorial kafka streamprocessing sql
---

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
