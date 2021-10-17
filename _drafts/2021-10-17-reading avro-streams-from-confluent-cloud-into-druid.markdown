---
layout: post
title:  "Reading Avro Streams from Confluent Cloud into Druid"
categories: blog imply druid
---

- Use Druid 0.22 micro-quickstart

In order to parse Avro messages, you first have to [enable](https://druid.apache.org/docs/0.22.0/development/extensions.html#loading-extensions) the Avro extension in the Druid configuration. For the `micro-quickstart` configuration, edit `conf/druid/single-server/micro-quickstart/_common/common.runtime.properties`:

```properties
# If you specify `druid.extensions.loadList=[]`, Druid won't load any extension from file system.
# If you don't specify `druid.extensions.loadList`, Druid will load all the extensions under root extension directory.
# More info: https://druid.apache.org/docs/latest/operations/including-extensions.html
druid.extensions.loadList=["druid-hdfs-storage", "druid-kafka-indexing-service", "druid-datasketches", "druid-avro-extensions"]
```

## Setting things up in Confluent Cloud

For this tutorial, I am assuming you have a Confluent Cloud account, as well as an environment and a cluster to work with. We need to set up a few things here:
- a topic that we are going to consume data from
- a data generator that adds data to the topic, which is part of the managed Kafka Connect service in Confluent Cloud
- a service account that will have access only to our tutorial topic
- an API key and secret associated with that service account
- a schema registry instance where we store the schema definition for our Avro records
- another API key and secret to access the schema registry.

- set up confluent cloud
  - create topic tut-avro with default parameters
  - create API key
    - choose granular key
    - create service account
    - set 2 acl's on the service account so that the account can read and write the topic
    - note down the Kafka API key and secret, as well as the bootstrap server URL. You can find these if you go for example `Data integration` > `Clients` > `New client` > `Java`.
  - enable schema registry, otherwise avro won't work
    - create an API key and secret for SR too
    - note these down, and also the URL for SR
  - set up Kafka Connect with the managed Datagen connector. For this tutorial, we use the `CLICKSTREAM` data generator. Also, we want to generate Avro data, so select the format to be `AVRO`.

If you have everything configured, you can peek into the topic in the Confluent Cloud GUI and verify that data is arriving.

## First attempt to ingest these data into Druid

in Druid, start the ingestion

Since Confluent Cloud secures access to Kafka, you need to paste the consumer properties into the little window in the wizard

```json
{
  "bootstrap.servers": "<KAFKA BOOTSTRAP SERVER>",
  "security.protocol": "SASL_SSL",
  "sasl.mechanism": "PLAIN",
  "sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule  required username=\"<KAFKA API KEY>\" password=\"<KAFKA SECRET KEY>\";"
} 
```
This will automatically populate the bootstrap server field too. Enter `tut-avro` as the Kafka topic name and hit `Apply`. Druid does its best to give you a preview of the data but since it's a binary format the result looks like gibberish. Press `Next: Parse data`. And ... we get an error. This is because Avro needs a schema and we haven't specified one

![](/assets/2021-10-17-1-load-gibberish.jpeg)

Press `Next: Parse data`. And ... we get an error. This is because Avro needs a schema and we haven't specified one.

## Fixing Schema Registry access


