---
layout: post
title:  "Streaming Events from Confluent Cloud into Imply Polaris using Nifi"
categories: blog druid imply polaris saas apache nifi eventstreaming confluent kafka
---

![Banner](/assets/2022-04-02-01-banner.png)

[Imply Polaris](https://imply.io/polaris-signup) is a fully managed database-as-a-service (DBaaS) based on [Druid](https://druid.apache.org/). It has been designed to give a hassle free experience of blazing fast OLAP, without bothering about the technical details.

One of the strengths of Druid is the ability to ingest and analyze event data in near real time, encapsulating what we used to call a [Kappa Architecture](https://medium.com/@devin.bost/the-kappa-architecture-8105a3c10f98) in the Big Data hype period. Polaris supports this paradigm by exposing an [Event Streaming REST API](https://docs.imply.io/polaris/api-stream/). But how would we connect an event stream that exists in, say, Kafka, to Polaris today?

For today's tutorial I have set up a topic in [Confluent Cloud](https://www.confluent.io/confluent-cloud/), which generates quite a lot of simulated clickstream data. 

![Confluent Cloud screenshot](/assets/2022-04-02-01c-confluent.jpg)

The data has a field called `timestamp` which is in seconds since Epoch, so some transformation will be needed. What we have to do:

- Read data out of Confluent Cloud
- Batch events up to a convenient size (not too small but less than 1 MB)
- Rename the timestamp field to `__time` and transform it to either ISO format or milliseconds since Epoch, according to [the Polaris requirements](https://docs.imply.io/polaris/supported-formats/#supported-time-formats)
- Apply some filtering because my topic might spew out more data than we can handle

A great tool to perform all these tasks is [Apache NiFi](https://nifi.apache.org/), the Swiss Army knife of streaming data integration. Today I am going to show you how to build a flow like this:

![NiFi screenshot](/assets/2022-04-02-01a-flow.jpg)

Let's get to work!

## Prerequisites

### Preparing the Polaris Table

First of all, we are going to need a target table in Polaris. The best way to create this table is to take a sample of the topic data, save it into a file and upload this file to Polaris. Create a schema and ingest the sample according to [these steps](https://docs.imply.io/polaris/schema/). Once you have the table ready, go to the table detail page and obtain the endpoint for the Push Event API:

![Get Table ID and API endpoint](/assets/2022-04-02-07-table-id.jpg)

Note this down - we will need it later.

### Preparing Confluent Cloud

You will also need:
- you _Kafka endpoint URL (Broker URL)_ from Confluent Cloud
- a _service account_ that will have access to your Kafka topic
- a _Kafka API key and secret_ associated with that service account
- a _consumer group ID_ that you can choose freely, but you need to create
- an _ACL_ for your consumer group in Confluent Cloud to give your service account read access to that consumer group

I've covered this in a bit more detail in [my post about Confluent Cloud integration with Druid](/2021/10/19/reading-avro-streams-from-confluent-cloud-into-druid/).

## Components of the NiFi Flow

### Controller Services

I am using Nifi 1.15.3 for this tutorial, and I am going to make use of the Record processors. Each record processor needs a Record Reader and a Record Writer. Since we are going to read whatever JSON we get at every stage, the Record Reader is generally just going to be a `JSONTreeReader` with the default configuration. 

For the Record Writer, consider that Polaris expects data to arrive in [ndjson](http://ndjson.org/) format (1 JSON object per line). This is  conveniently achieved by setting the `Output Grouping` attribute to `One Line Per Object`:

![Record Writer Configuration](/assets/2022-04-02-03-recordwriter.jpg)

If you don't do this, the output will be a JSON array and Polaris will not be happy.

### Kafka Consumer


