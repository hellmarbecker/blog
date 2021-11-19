---
layout: post
title:  "Apache Druid in the Publishing Industry"
categories: blog imply druid industry
---
[Apache Druid](https://druid.apache.org/) is a high performance, distributed analytical datastore that has natural uses in many industries. Today, I am going to look at use cases in the publishing (news) industry.

![](/assets/2021-11-19-newspaper_fire_orange.jpg)[^1]

## Challenges in the Publishing Industry

The publishing industry is experiencing a rapid change:
- Print is on the decline: In the U.S., comparing 2020 numbers to 2019, [weekday print circulation decreased 19% and Sunday print circulation decreased 14%](https://www.pewresearch.org/journalism/fact-sheet/newspapers/). Publishing is increasingly digital.
- Traditional monetization strategies using display ads are less successful.
- With Google dominating the search market, traditional SEO is getting more difficult.
- With Mobile becoming an important delivery channel, publishers need to adapt to the smaller screen format of smartphones. This means that a small area on top of each page is prime real estate and placement of content is crucial.

To address these challenges, publishers are looking for new ways to ensure revenue:
- Paid subscriptions are becoming more important
- Cooperations with retailers are a new option, the publisher acts as a marketplace to sell certain items in a branded shop
- [Native advertising and advertorials](https://www.youtube.com/watch?v=1SmlsfSqmOw) as a way to mitigate display ad fatigue and to attract readers' attention
- Social media homepages (for instance on Facebook) and [dark social](https://www.brightervision.com/what-is-dark-social/) campaigns.

Publishers are constantly looking for new ways to transform their business, because the traditional business models will not work in the future.

## Why Druid?

Here are a few unique points about Druid:

## Self service analytics

Druid's principle for data can be described by the "3F" of "**F**resh, **F**ast, **F**or all". There are a number of [free](https://blog.allegro.tech/2018/10/turnilo-lets-change-the-way-people-explore-big-data.html) and [commercial](https://imply.io/post/hello-pivot) frontends that enable self service, exploratory analytics and interactive dashboarding on top of Druid - ensuring that each user gets snappy access to the data that was generated this very moment. 

### Built to scale

Druid is in itself a distributed system with built in service discovery and resiliency. It is unique in that it scales not only for large amounts of data, but also for a large amount of concurrent queries.

This enables use cases in which a dashboard application fires lots of incremental queries. Ultimately, a user of Druid can explore data much like one would nowadays explore Google Maps: as they zoom out of the data area they are viewing, or drilling into more detail, Druid will be adding only the missing bits to the view by answering incremental queries.

### Streaming ingestion

Druid connects directly to event streaming platforms like [Apache Kafka](https://kafka.apache.org/) and [AWS Kinesis](https://aws.amazon.com/kinesis/). With the proper frontend, this means you can get updated insights about your reader's behavior within seconds. The [K2D](https://imply.io/Kafka-to-Druid_stack_architecture_solution_brief.pdf) architecture is becoming a standard for fast, near-real-time analytics.

![K2D architecture overview](/assets/2021-10-19-0-architecture.png)

### Smart data lookups

Druid has the built in ability to use secondary data sources as dimensions. For instance, Druid can link your behavior data against your asset database, showing headlines and other CMS information.

### Multi-value Dimensions

With the built in [multi-value dimensions](/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/), Druid has a very performant way to model common one-to-many relationships in publishing data, like tag clouds or mapping a news item to multiple sections.

## How does Druid help the Publishing Industry?

### Measuring conversions

Druid integrateds all the sources of data, making it possible to attribute conversions, such as new paid subscriptions, to the campaign source and the content that motivated the conversion. It does theat for large amounts of data, with respoonse times of less than a second.

### Measuring teaser placement success immediately

With a small portion of the mobile page being so disproportionately valuable, it becomes crucial to beable to measure success of a teaser or headline very quickly after it is placed. If a placement is not successful, it needs to be changed quickly. Druid enables this kind of fast reaction.

### Measuring paid vs. non paid performance

How do you decide whether you place an article behind the paywall or not? Sometimes it may be beneficial to make paid content available for free for a limited time.

### Measuring SEO performance in near real time

With data made available from third party crawling services, [SEO](https://en.wikipedia.org/wiki/Search_engine_optimization) statistics can be used to optimize headlines and asset tags. Druid can ingest these data together with clickstream data from user interactions and make SEO perfromance directly visible to editors.

### Enabling analysis by editors and content specialists

Some of the publisher who use Druid have set up internal training programs, the main goal being to sensitivize non technical users for the benefits of using data driven analytics. With the interactive analytics capability of modern frontends, analytical work becomes a delightful experience.

### Hypothesis driven development

Interactive analytics with Druid is so fast that various hypotheses (_Is this placement going to be successful?_) can be tested against real data within a very short time. The editor asks a question to the data, and if the answer is unclear or unsatisfying, she would refine or modify the question - within seconds!
 
German publisher, [Ippen](https://www.ippen-digital.de/) explained how Druid enabled them to switch from opinions to insights, at [Druid Summit](https://druidsummit.org/) this year: 

<iframe width="560" height="315" src="https://www.youtube.com/embed/1ceY6iXgKug" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Conclusion

With its fast, interactive, and scalable approach, Druid enables publishers to tackle the challenges of transformation in the 21st century.

Above are but a few of the use cases of Druid in the news and publishing industry. As this sector develops, there is more to come! 

[^1]: Derived from: "Newspaper high contrast B&W" by NS Newsflash is licensed with CC BY 2.0. To view a copy of this license, visit https://creativecommons.org/licenses/by/2.0/ 

