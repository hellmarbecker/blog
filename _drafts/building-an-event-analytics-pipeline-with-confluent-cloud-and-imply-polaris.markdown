---
layout: post
title:  "Building an Event Analytics Pipeline with Confluent Cloud and Imply Polaris"
categories: blog apache druid imply kafka confluent
---

![Streaming Analytics Architecture]()

A modern streaming analytics pipeline is built around two central components:

- an event streaming platform
- an event analytics platform.

The de facto standard for event streaming is Apache Kafka; for event analytics it is Apache Druid. Both are open source projects and it is comparatively straightforward to build a streaming analytics pipeline using these tools.

## Using Managed Services

With the advent of managed SaaS services this task becomes even easier.

[Confluent](https://www.confluent.io/) was founded by the creators of Kafka. Its Confluent Cloud service offers not only Kafka clusters, but also a managed version of [ksqlDB](https://ksqldb.io/), a streaming SQL framework that enables engineers to develop and deploy real time ETL applications using just SQL statements.

[Imply](https://imply.io/) was founded by the creators of Druid, and offers Imply Polaris, an integrated real time analytics platform based on Apache Druid.

Taking these two together, we have all components to build a streaming analytics pipeline.
 
## Prerequisites

- confluent cloud
  - an environment
  - a cluster
  - a KSQL application
- imply polaris
  - an environment

## Data Generation

- how to use the news data generator
