---
layout: post
title:  "Building an Event Analytics Pipeline with Confluent Cloud and Imply Polaris"
categories: blog apache druid imply kafka confluent
---

![Streaming Analytics Architecture]()

A modern streaming analytics pipeline is built around two central components:

- an event streaming platform
- an event analytics platform.

This is conveniently achieved using [Confluent](https://www.confluent.io/) Cloud as a SaaS event streaming platform, and [Imply](https://imply.io/) Polaris as a SaaS realtime analytics database.

In this tutorial, I am going to show you how to set up a pipeline that

- generates a simulated clickstream event stream and sends it to Confluent Cloud
- processes the raw clickstream data using managed [ksqlDB](https://ksqldb.io/) in Confluent Cloud
- delivers the processed stream using Confluent Cloud
- and ingests these JSON events, using a native connection, into Imply Polaris.

## Prerequisites

For this tutorial, you need a Confluent Cloud account. In this account, create [an environment](https://docs.confluent.io/cloud/current/access-management/hierarchy/cloud-environments.html), [a cluster](https://docs.confluent.io/cloud/current/clusters/create-cluster.html), and [a ksqlDB application](https://docs.confluent.io/cloud/current/get-started/index.html#section-2-add-ksql-cloud-to-the-cluster).

The smallest size of cluster (`Basic`) will do.

Furthermore, you need an Imply Polaris environment.

## Data Generation

- how to use the news data generator
