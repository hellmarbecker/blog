---
layout: post
title:  "Apache Druid in the Publishing Industry"
categories: blog imply druid industry
---
[Apache Druid](https://druid.apache.org/) is a high performance, distributed analytical datastore that has natural uses in many industries. Today, I am going to look at use cases in the publishing industry.

![](/assets/2021-11-19-newspaper_fire_orange.jpg)[^1]

## Challenges in the Publishing Industry

The publishing industry is experiencing a rapid change:
- Print is on the decline: In the U.S., comparing 2020 numbers to 2019, [weekday print circulation decreased 19% and Sunday print circulation decreased 14%](https://www.pewresearch.org/journalism/fact-sheet/newspapers/). Publishing is increasingly digital.
- Traditional monetization strategies using display ads are lless successful.
- With Google dominating the search market, traditional SEO is getting more difficult.
- With Mobile becoming an important delivery channel, publishers need to adapt to the smaller screen format of smartphones. This means that a small area on top of each page is prime real estate and placement of content is crucial.

To address these challenges, publishers are looking for new ways to ensure revenue:
- Paid subscriptions are becoming more important
- Cooperations with retailers are a new option, the publisher acts as a marketplace to sell certain items in a branded shop
- [Native advertising and advertorials](https://www.youtube.com/watch?v=1SmlsfSqmOw)

## Why Druid?

Here are a few unique points about Druid:

### Built to scale

Druid is in itself a distributed system with built in service discovery and resiliency. It is unique in that it scales not only for large amounts of data, but also for a large amount of concurrent queries.

This enables use cases in which a dashboard application fires lots of incremental queries [^2]

### Streaming ingestion

Druid connects directly to event streaming platforms like [Apache Kafka](https://kafka.apache.org/) and [AWS Kinesis](https://aws.amazon.com/kinesis/). With the proper frontend, this means you can get updated insights about your reader's behavior within seconds.

### Smart data lookups

Druid has the built in ability to use secondary data sources as dimensions. For instance, Druid can link your behavior data against your asset database, showing headlines and other CMS information.

### Multi-value Dimensions

With the built in [multi-value dimensions](/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/), Druid has a very performant way to model common one-to-many relationships in publishing data, like tag clouds or mapping a news item to multiple sections.

## How does Druid help the Publishing Industry?

German publisher, Ippen Digital talked at [Druid Summit](https://druidsummit.org/) this year

<iframe width="560" height="315" src="https://www.youtube.com/embed/1ceY6iXgKug" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

[^1]: Derived from: "Newspaper high contrast B&W" by NS Newsflash is licensed with CC BY 2.0. To view a copy of this license, visit https://creativecommons.org/licenses/by/2.0/ 

[^2]: lorem ipsum
