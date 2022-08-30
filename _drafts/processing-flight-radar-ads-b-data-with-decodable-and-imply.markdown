---
layout: post
title:  "Processing Flight Radar (ADS-B) Data with Imply, Decodable, and Confluent Cloud"
categories: blog druid imply tutorial kafka streamprocessing sql flink decodable
---

![Overview Architecture](/assets/2022-08-30-01-overview.jpg)

Flight radar data are sent by most commercial and private aircraft, and can easily be received and decoded using a Raspberry Pi and a DVB-T receiver stick.

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

[Flightradar24](https://www.flightradar24.com/build-your-own) has step by step instructions how to set up a flight radar receiver. There are also various open source repositories that add functionality to the original `dump1090` software.

`dump1090` makes the data it receives available in a CSV format on port 30003, which is where we are going to pick it up for the next step.

This tutorial assumes that you have your Raspberry Pi set up to receive aircraft data.

## Sending Data to Kafka

I am using Confluent Cloud and `kcat` as a client. (The package that comes with the current release of [Raspberry Pi OS](https://www.raspberrypi.com/software/operating-systems/), which is based on Debian 11, uses the old name `kafkacat`.)

In Confluent Cloud, create two topics `adsb-raw` and `adsb-json` for the flight data. You can create them with just one partition and leave all the other default settings in place. Also you have to create an API key and assign ACLs to it that allow both read and write access to those two topics. I have covered this in detail in [an earlier post](/2021/10/19/reading-avro-streams-from-confluent-cloud-into-druid/).

On your Raspberry Pi, create a script `send_kafka.sh` like this:

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

(The `awk` command in the script is not strictly needed for this tutorial. It adds the ID of the flight as a key field to each Kafka message.)

Don't forget to make the file executable with `chmod 755 send_kafka.sh`. Let's install this script as a service with `systemd`, so that it can be started automatically on boot.

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

Sign up for a free account at Decodable. Here's what we need to create:

- a _source connector_ to connect to the raw data topic
- a _pipeline_ to transform the data
- a _sink connector_ to write the transformaed ddata out to Kafka
- _streams_ to connect all these.

### Connecting to the data source

First, create a source connector. You will be needing the Confluent cluster ID and REST endpoint (this is _not_ the broker endpoint!) You can find these in the cluster menu under `Cluster overview` > `Cluster settings`.

![Screenshot of Confluent Cloud cluster settings](/assets/2022-08-30-03-cc-settings.jpg)

Go to `Create Source` in Decodable, and pick Confluent Cloud as the source:

![Create Source](/assets/2022-08-30-02-create-source.jpg)

In the next dialog, enter the Confluent cluster settings in Decodable's configuration dialog. Also enter your API key and secret to access Confluent Cloud, and make sure you select `Raw` as the value format.

![Source Connector Settings](/assets/2022-08-30-04-source-settings.jpg)

In the next step, specify `adsb-raw` as the topic to connect to.

![Choose Topic](/assets/2022-08-30-05-topic.jpg)

Next, Decodable is going to ask which stream to connect to. You will create a new stream in this step. Select `New Stream`, and enter the name and description.

![Create New Stream](/assets/2022-08-30-06-create-stream.jpg)

A stream also needs a schema. Since the data source fromat is `Raw`, the schema has only one field, which we will conveniently name `raw`. Pick `string` as the field type.

![Define schema](/assets/2022-08-30-06-schema.jpg)

Hit `Next` and define the name and description for the connection:

![Name connection](/assets/2022-08-30-07-name-connection.jpg)

Start the connection using the `Start` button.

### Transforming the data

Next, let's create a Pipeline. Use the `Create Pipeline` button and select `stream-adsb-raw` as the input source. In the following dialog, enter this SQL code:

```sql
insert into `stream-adsb-json`
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
from `stream-adsb-raw`
```

![Decodable SQL screen](/assets/2022-08-30-08-sql.jpg)



---

<a href="https://www.flaticon.com/free-icons/radar" title="radar icons">Radar icons created by Freepik - Flaticon</a>
