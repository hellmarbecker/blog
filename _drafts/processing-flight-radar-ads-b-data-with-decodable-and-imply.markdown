---
layout: post
title:  "Processing Flight Radar (ADS-B) Data with Imply, Decodable, and Confluent Cloud"
categories: blog druid imply tutorial kafka streamprocessing sql flink decodable
---

Flight radar data are sent by commercial and most private aircraft, and can easily be received and decoded using a Raspberry Pi and a DVB-T receiver stick.

In this tutorial, I am going to show you how to set up a pipeline that

- generates an event stream from flight radar data
- converts this stream into JSON format using [Decodable](https://www.decodable.co/)
- and ingests these JSON events into [Imply Polaris](https://imply.io/imply-polaris/) using the new Pull Ingestion feature that was released in August 2022. 

## Technologies Used

Imply's analytical platform, based on [Apache Druid](https://druid.apache.org/), is a great tool for fast event analytics, and with the advent of [**Imply Polaris**](https://imply.io/imply-polaris/) as a managed service these capabilities are ever easier to use.

Imply will be the final destination of the data pipeline and the visualization tool to the end user.

In order to prepare streaming data for ingestion by Druid, [Apache Flink](https://flink.apache.org/) has become a popular programming framework. [**Decodable**](https://www.decodable.co/) is a no-code streaming ETL service on top of Flink that allows data engineers to configure stream processing pipelines using a graphical interface. Instead of programming the entire pipeline in Java, Decodable has a concept of simple building blocks:

- _Connections_ are the interface to external data sources and sinks, such as streaming platforms or databases. They connect to streams.
- _Pipelines_ are processing blocks: their inputs and outputs are streams. A pipeline has a piece of SQL that defines the processing.
- _Streams_ connect pipelines to connections or to other pipelines. A stream has a schema defining the fields and their types.

If you build a processing pipeline using these blocks, Decodable compiles them into Flink code behind the scenes. I am going to use Decodable to do some parsing and preprocessing on the data.

[**Confluent Cloud**](https://confluent.cloud/) is Confluent's managed streaming service based on [Apache Kafka](https://kafka.apache.org/). This will supply the streams (topics) that tie everything together.

## Generating Data

yada yada raspberry pi

## Sending Data to Kafka

I am using Confluent Cloud and `kcat` as a client. (The package that comes with the current release of [Raspberry Pi OS](https://www.raspberrypi.com/software/operating-systems/), which is based on Debian 11, uses the old name `kafkacat`.)

You have to set up the topic for the flight data, as well as an API key and ACLs in Confluent Cloud. I have covered this in detail in [an earlier post](/2021/10/19/reading-avro-streams-from-confluent-cloud-into-druid/).

For Decodable, you will be needing the cluster ID and REST endpoint (this is _not_ the broker endpoint!) You can find these in the cluster menu under `Cluster overview` > `Cluster settings`.

![Screenshot of Confluent Cloud cluster settings](...)

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

Let's install this script as a service with `systemd`, so that it can be started automatically on boot.

Create a file `dump1090-kafka.service` with this content (you may need to adapt the file path):

```
[Unit]
Description=connector

[Service]
User=pi
ExecStart=/home/pi/plt-airt-2000/send_kafka.sh
RestartSec=30
Restart=always

[Install]
WantedBy=multi-user.target
```

and copy it to `/etc/systemd/system/`. Then run

```
sudo systemctl enable dump1090-kafka
sudo systemctl start dump1090-kafka
```


## Preprocessing the Data in Decodable

```sql
insert into `adsb-json`
select
    to_timestamp(
        split_index(`raw`, ',',  6) || ' ' || split_index(`raw`, ',',  7),
        'yyyy/MM/dd HH:mm:ss.SSS'
    ) as __time,
    split_index(`raw`, ',',  0) as message_type,
    cast(split_index(`raw`, ',',  1) as integer) as transmission_type,
    cast(split_index(`raw`, ',',  2) as integer) as session_id,
    cast(split_index(`raw`, ',',  3) as integer) as aircraft_id,
    split_index(`raw`, ',',  4) as hex_ident,
    cast(split_index(`raw`, ',',  5) as integer) as flight_id,
    split_index(`raw`, ',',  6) as date_message_generated,
    split_index(`raw`, ',',  7) as time_message_generated,
    split_index(`raw`, ',',  8) as date_message_logged,
    split_index(`raw`, ',',  9) as time_message_logged,
    split_index(`raw`, ',', 10) as callsign,
    cast(split_index(`raw`, ',', 11) as integer) as altitude,
    cast(split_index(`raw`, ',', 12) as double) as ground_speed,
    cast(split_index(`raw`, ',', 13) as double) as track,
    cast(split_index(`raw`, ',', 14) as double) as latitude,
    cast(split_index(`raw`, ',', 15) as double) as longitude,
    cast(split_index(`raw`, ',', 16) as integer) as vertical_rate,
    split_index(`raw`, ',', 17) as squawk,
    cast(split_index(`raw`, ',', 18) as integer) as alert,
    cast(split_index(`raw`, ',', 19) as integer) as emergency,
    cast(split_index(`raw`, ',', 20) as integer) as spi,
    cast(split_index(`raw`, ',', 21) as integer) as is_on_ground
from `adsb-raw`
```
