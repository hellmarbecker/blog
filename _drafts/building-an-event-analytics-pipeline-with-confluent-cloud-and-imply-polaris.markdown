---
layout: post
title:  "Building an Event Analytics Pipeline with Confluent Cloud and Imply Polaris"
categories: blog apache druid imply kafka confluent
---

![Streaming Analytics Architecture](/assets/2021-10-19-0-architecture.png)

A modern streaming analytics pipeline is built around two central components:

- an event streaming platform
- an event analytics platform.

This is conveniently achieved using [Confluent](https://www.confluent.io/) Cloud as a SaaS event streaming platform, and [Imply](https://imply.io/) Polaris as a SaaS realtime analytics database.

In this blogpost, I am going to show you how to set up a pipeline that

- generates a simulated clickstream event stream and sends it to Confluent Cloud
- processes the raw clickstream data using managed [ksqlDB](https://ksqldb.io/) in Confluent Cloud
- delivers the processed stream using Confluent Cloud
- and ingests these JSON events, using a native connection, into Imply Polaris.

But first, let's cover some of the basics.

## The Case for Streaming Analytics

### Old School Analytics

![Classical OLAP architecture](/assets/2023-01-12-01-olap-architecture.png)

This is how we used to do analytics, roughly 20 years ago. You would have your operational systems that collected data into transactional, or OLTP, databases.

OLTP databases are built to process single updates or inserts very quickly. In traditional relational modeling this means you have to normalize your data model, ideally to a point where every item exists only once in a database. The downside is when you want to run an analytical query that aggregates data from different parts of your database, these queries require complex joins and can become very expensive, hurting query times and interfering with the transactional performance of your database.

Hence another type of databases was conceived which is optimized for these analytical queries: OLAP databases. These come in different shapes and flavours, but generally a certain amount of denormalization and possibly preaggregation is applied to the data.

The process that ships data from the transactional system to the OLAP database is called ETL - Extract, Transform, Load. It is a batch process that would run on a regular basis, for instance once a night or once every week. The frequency of the batch process determines how "fresh" your analytical data is.

In the old world, where analytical users would be data analysts inside the enterprise itself, that was often good enough. But nowadays, in the age of connectivity, everyone is an analytics user. If you check your bank account balance and the list of transactions in your banking app on your smartphone, you are an analytics user. And if someone transfers funds to your account, you expect to see the result now and not two days later.

A better way of processing data for analytics was needed. And we'll look at that now.

### Big Data and the Lambda Architecture

About ten years ago, the big data craze came up around the Hadoop ecosystem. Hadoop brought with it the ability to handle historical data to a previously unknown scale, but it also already had real time [note: this is not hard real time like in embedded systems! If necessary elaborate] capability, with tooling like Kafka, Flume, and HBase.

The first approach to getting analytics more up to date was the so called lambda architecture, where incoming data would be sent across two parallel paths:

- A realtime layer with low latency and limited analytical capabilities
- A highly scalable but slower batch layer.

![Lambda Architecture](https://www.ericsson.com/48e613/assets/global/qbank/2019/01/13/lambdakappa1_1-104666resize668376crop00668376autoorientquality90stripbackground23ffffffextensionjpgid8.jpg)

This way, you would be able to retrieve at least some of the analytics results immediately, and get the full results the next day.

A common serving layer would be the single entry point for clients.

This architectural pattern did the job for a while but it has an intrinsic complexity that is somewhat hard to master. Also, when you have two different sources of results, you need to go through an extra effort to make sure that the results always match up.

### Kappa Architecture

A better way needed to be found. It was created in the form of the kappa architecture. In the kappa architecture, there is only one data path and only one result for a given query. The same processing path gives (near) real time results and also fills up the storage for historical data. 

![Kappa Architecture](https://www.ericsson.com/48e6a2/assets/global/qbank/2019/01/13/lambdakappa1_2-104667resize590332crop00590332autoorientquality90stripbackground23ffffffextensionjpgid8.jpg)

The kappa architecture handles incoming streaming data and historical data in a common, uniform way and is more robust than a lambda architecture. Ideally you still want to encapsulate the details of such an architecture and not concern the user with it. We will come to that in a moment.

## Preparing your Data: Streaming ETL

But first a few words about the part that I didn't cover in the last two slides: How to get the data out of the transactional systems into whatever analytics architecture you have.

Instead of processing batches of data, streaming ETL has to be event driven. There are two ways of processing event data in a streaming ETL pipeline:

- *Simple event processing* looks at one event at a time. Simple event processing is *stateless* which makes it easy to implement but limite the things you can do with it. This is used for format transformations, filtering, or data cleansing, for instance. An example for simple event processing is Apache NiFi.
- *Complex event processing* looks at a collection of events over time, hence it is *stateful* and has to maintain a state store in the background. With that you can do things like windows aggregations, such as sliding averages or session aggregations. You can also join various event streams (think orders and shipments), or enrich data with lookup data that is itself event based. Complex event processing is possible using frameworks like Spark Streaming, Flink, or Kafka Streams.

In this tutorial, I will use [ksqlDB](https://ksqldb.io/). ksqlDB is a community licensed SQL framework on top of Kafka Streams, by Confluent. It is also available as a managed offering in Confluent Cloud, and that is what I will be using.

With ksqlDB, you can write a complex event streaming application as simple SQL statements. ksqlDB queries are typically persistent: unlike database queries, they continue running until they are explicitly stopped, and they continue to emit new events as they process new input events in real time. ksqlDB abstracts away for the most part the detail of event and state handling.



## Prerequisites

For this tutorial, you need a Confluent Cloud account. In this account, create [an environment](https://docs.confluent.io/cloud/current/access-management/hierarchy/cloud-environments.html), [a cluster](https://docs.confluent.io/cloud/current/clusters/create-cluster.html), and [a ksqlDB application](https://docs.confluent.io/cloud/current/get-started/index.html#section-2-add-ksql-cloud-to-the-cluster).

The smallest size of cluster (`Basic`) will do.

Furthermore, you need an Imply Polaris environment. You can sign up for a free trial [here](https://signup.imply.io).

## Data Generation

I am using the [`imply-news`](https://github.com/hellmarbecker/imply-news) data generator. It simulates clickstream data from a news publisher portal. You can find setup instructions in the repository if you want to run this yourself.

Here is a sample of the data:

```json
{"timestamp": 1673512817.517501, "recordType": "click", "url": "https://imply-news.com/home/Sport/Step-left-list-discuss-up", "useragent": "Mozilla/5.0 (Windows; U; Windows NT 10.0) AppleWebKit/531.19.5 (KHTML, like Gecko) Version/5.0.5 Safari/531.19.5", "statuscode": "200", "state": "home", "statesVisited": ["home", "content", "content", "home"], "sid": 14362070, "uid": "86525", "isSubscriber": 0, "campaign": "fb-2 US Election", "channel": "display", "contentId": "Sport", "subContentId": "Step left list discuss up", "gender": "w", "age": "51-60", "latitude": "45.47885", "longitude": "133.42825", "place_name": "Lesozavodsk", "country_code": "RU", "timezone": "Asia/Vladivostok"}
{"timestamp": 1673512817.5246856, "recordType": "click", "url": "https://imply-news.com/affiliateLink/News/Argue-Congress-beautiful-go-usually-which-brother", "useragent": "Opera/9.37.(Windows CE; mni-IN) Presto/2.9.171 Version/11.00", "statuscode": "200", "state": "affiliateLink", "statesVisited": ["home", "affiliateLink"], "sid": 14357239, "uid": "59450", "isSubscriber": 0, "campaign": "fb-2 US Election", "channel": "social media", "contentId": "News", "subContentId": "Argue Congress beautiful go usually which brother", "gender": "w", "age": "26-35", "latitude": "4.96667", "longitude": "10.7", "place_name": "Tonga", "country_code": "CM", "timezone": "Africa/Douala"}
{"timestamp": 1673512730.502167, "recordType": "session", "useragent": "Mozilla/5.0 (compatible; MSIE 7.0; Windows NT 6.2; Trident/5.1)", "statesVisited": ["home", "exitSession"], "sid": 14333040, "uid": "74108", "isSubscriber": 1, "campaign": "fb-2 US Election", "channel": "social media", "gender": "m", "age": "26-35", "latitude": "-33.59217", "longitude": "-70.6996", "place_name": "San Bernardo", "country_code": "CL", "timezone": "America/Santiago", "home": 1, "content": 0, "clickbait": 0, "subscribe": 0, "plusContent": 0, "affiliateLink": 0, "exitSession": 1}
{"timestamp": 1673512817.5646698, "recordType": "click", "url": "https://imply-news.com/home/Sport/Unit-down-perform-religious-add-find-management", "useragent": "Mozilla/5.0 (Linux; Android 2.3.2) AppleWebKit/532.1 (KHTML, like Gecko) Chrome/34.0.876.0 Safari/532.1", "statuscode": "200", "state": "home", "statesVisited": ["home"], "sid": 14370083, "uid": "81655", "isSubscriber": 0, "campaign": "fb-2 US Election", "channel": "paid search", "contentId": "Sport", "subContentId": "Unit down perform religious add find management", "gender": "w", "age": "36-50", "latitude": "41.66394", "longitude": "-83.55521", "place_name": "Toledo", "country_code": "US", "timezone": "America/New_York"}
{"timestamp": 1673512817.5769033, "recordType": "click", "url": "https://imply-news.com/exitSession/Puzzle/Resource-within-author-can", "useragent": "Mozilla/5.0 (Android 5.0.2; Mobile; rv:62.0) Gecko/62.0 Firefox/62.0", "statuscode": "200", "state": "exitSession", "statesVisited": ["home", "content", "clickbait", "plusContent", "plusContent", "home", "exitSession"], "sid": 14345138, "uid": "72517", "isSubscriber": 0, "campaign": "fb-2 US Election", "channel": "social media", "contentId": "Puzzle", "subContentId": "Resource within author can", "gender": "m", "age": "51-60", "latitude": "37.60876", "longitude": "-77.37331", "place_name": "Mechanicsville", "country_code": "US", "timezone": "America/New_York"}
```

You can see that we have different kinds of objects in this topic, distinguished by the `recordType` field. These have different type and number of fields, which means we cannot just slurp up everything using off-the-shelf JSON ingestion.

(Why would you have different objects in a topic in a real life scenario? [This blog](https://www.confluent.io/blog/put-several-event-types-kafka-topic/) discusses some of the reasons and the decision criteria.)

## The ETL Pipeline


---

Lambda and Kappa architecture diagrams are linked from [this Ericsson blog](https://www.ericsson.com/en/blog/2015/11/data-processing-architectures--lambda-and-kappa).
