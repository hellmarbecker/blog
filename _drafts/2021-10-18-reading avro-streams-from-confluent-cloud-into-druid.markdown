---
layout: post
title:  "Reading Avro Streams from Confluent Cloud into Druid"
categories: blog imply druid confluent kafka eventstreaming
---

Today I am going to show how to get AVRO data from a schema aware Confluent Cloud cluster into Apache Druid. Use the Druid 0.22 [micro-quickstart](https://druid.apache.org/docs/latest/tutorials/index.html) setup for this exercise.

In order to parse Avro messages, you first have to [enable](https://druid.apache.org/docs/0.22.0/development/extensions.html#loading-extensions) the Avro extension in the Druid configuration. For the `micro-quickstart` configuration, edit `conf/druid/single-server/micro-quickstart/_common/common.runtime.properties`:

```properties
# If you specify `druid.extensions.loadList=[]`, Druid won't load any extension from file system.
# If you don't specify `druid.extensions.loadList`, Druid will load all the extensions under root extension directory.
# More info: https://druid.apache.org/docs/latest/operations/including-extensions.html
druid.extensions.loadList=["druid-hdfs-storage", "druid-kafka-indexing-service", "druid-datasketches", "druid-avro-extensions"]
```

## Setting things up in Confluent Cloud

For this tutorial, I am assuming you have a [Confluent Cloud](https://confluent.cloud) account, as well as an environment and a cluster to work with. We need to set up a few things here:
- a topic that we are going to consume data from
- a data generator that adds data to the topic, which is part of the managed Kafka Connect service in Confluent Cloud
- a service account that will have access only to our tutorial topic
- an API key and secret associated with that service account
- a schema registry instance where we store the schema definition for our Avro records
- another API key and secret to access the schema registry.
- a data generator that adds data to the topic, which is part of the managed Kafka Connect service in Confluent Cloud

### Create a topic

For this tutorial, create a topic `tut-avro` with default settings. (We are only using little data, so there is no point in tuning the configuration.) You can do this in the GUI using `Topics` > `Add topic`.

### Service account and API credentials

Navigate to `Data integration` > `API keys` > `Add key`. You will be asked whether your key should be allowed global or granular access:
- _Global access_ means that the key is owned by your personal user account and shares all its access rights. It also means that, should your account ever be deleted, the API key will expire as well.
- _Granular access_ means that the key is owned by a _service account_ and can be endowed with specific privileges as needed. This is the proper way to set up machine to machine communication.

If you select that last option, you can either use an existing service account, or create a new one. Create a new service account and name it `tut-avro-service-account`. Add two ACLs to the service account so that the account can read and write the topic `tut-avro` that you just created. Finally, download the generated API key and secret for later use. Also, note down the Kafka bootstrap server URL. You can find it if you go to `Data integration` > `Clients` > `New client` > `Java`.

### Schema registry

At the bottom left of your cluster's GUI menu you will find the item `Schema Registry`. Open it and make sure Schema Registry is enabled.

Once you enable Schema Registry, you will find the API coordinates in the `Schema Registry` tab of your environment. (A schema registry is defined on environment level and can be shared by multiple clusters.) Note down the API endpoint URL and create a Schema Registry API key and secret using the Edit button in the `API credentials` section.

Once again, download the API key and secret you just created.

### Data generator

Use the menu navigation `Data integration` > `Connectors` > `Add connector` and select the `Datagen Source` connector. 

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

![](/assets/2021-10-18-1-load-gibberish.jpeg)

Press `Next: Parse data`. And ... we get an error. This is because Avro needs a schema and we haven't specified one.

## Fixing Schema Registry access


