---
layout: post
title:  "Druid Data Modeling Special: Lookups and Multi-value Dimensions"
categories: blog imply druid
---

The other day, [Peter Marshall](https://www.petermarshall.io/) brought up a question: Do lookups work with multi-value dimensions in Druid? Spoiler alert: They do, in fact! But let's first take a quick look at what lookups are.

## Dimensional Modeling in Apache Druid

The general data model of Druid is that of a fully denormalized, wide table. It would look much like the data models you might know if you have been using [SPSS](https://www.ibm.com/analytics/spss-statistics-software) or [GNU Octave](https://de.wikipedia.org/wiki/GNU_Octave). This means each field value is determined at ingestion and stays that way.

But sometimes attribute values change over time. For instance, people tend to move around or get married, so their addresses and names might change. A news publisher might change the headline of an article but the content stays the same. This introduces the theory of [slowly changing dimensions](https://de.wikipedia.org/wiki/Slowly_Changing_Dimensions).

If you want to have an attribute that changes over time, you need a way to refer to that attribute's value at query time. [Druid lookups](https://druid.apache.org/docs/latest/querying/lookups.html) are a way to do this. A lookup maintains a key-value map that is stored in memory on all data nodes (historical and peon processes), and possibly updated periodically. In Druid SQL, the [LOOKUP()](https://druid.apache.org/docs/latest/querying/sql.html#string-functions) function replaces the key, which is a field or expression from your datasource, with the lookup value. This emulates what is known as a [type 1 slowly changing dimension](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/type-1/).

(It is also possible, and often makes sense, to apply a lookup at ingestion time. But that is a story for another time.)

So, Peter's question is: can you apply lookups with [multi-value dimensions](/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/)?

## Real life example

Let's assume I am running an ecommerce shop. There's a number of items in my portfolio, and each of them has a SKU number and product name like so:
```json
{
  "0001": "Mug O'Grog",
  "0002": "Red Herring",
  "0003": "Root Beer",
  "0004": "Staple Remover",
  "0005": "Breath Mints",
  "0006": "Fabulous Idol"
}
```
In fact, this is just the format for defining an inline lookup. From the Druid console, navigate to the `Lookup` wizard, choose `Add lookup`, and paste the snippet from above into the `Map` input field. Name the lookup `eshop_sku`:

<img src="/assets/2021-10-14-1-create-lookup.jpg" width="50%" />

(In real life, you might populate a lookup from a file that you keep for instance in Amazon S3, or from a database via JDBC. You can even have [a lookup that automatically receives up-to-date information via Kafka](https://druid.apache.org/docs/latest/development/extensions-core/kafka-extraction-namespace.html)!)

Now, let's enter some transactions.

Here's our data snippet for today's tutorial:
```json
{ "ts": "2021-10-14 10:00:00", "customer": "Gian", "basket": [ "0001", "0001", "0002", "0004" ] }
{ "ts": "2021-10-14 10:10:00", "customer": "Rachel", "basket": [ "0002", "0004", "0005" ] }
{ "ts": "2021-10-14 10:20:00", "customer": "Peter", "basket": [ "0005", "0004", "0002" ] }
{ "ts": "2021-10-14 10:30:00", "customer": "Gian", "basket": [ "0002" ] }
{ "ts": "2021-10-14 10:40:00", "customer": "Jessy", "basket": [ "0003", "0005", "0006" ] }
{ "ts": "2021-10-14 10:50:00", "customer": "Gian", "basket": [ "0005", "0006" ] }
```
Ingest these data into Druid, set the segment granularity to `day`, and name the datasource `eshop`. Time for some fun queries!

The first query I am going to run is a plain scan query. Here is the SQL:

![Scan query with lookup](/assets/2021-10-14-2-query-plain.jpeg)

Surprisingly, this works! Every single SKU in the basket is replaced with the product name, the result is an array again.

Next, let's `GROUP BY` the basket dimension. If you have been following my blog, you know that this breaks down the multi-value field as if you had applied an `EXPLODE` or `UNNEST` function, before doing the aggregation. In Druid, this compiles into a TopN query.

![Group by query with lookup](/assets/2021-10-14-3-query-groupby.jpeg)

Again, every single item in the basket dimension is replaced by the looked-up value and then the query continues as expected.

Can we `JOIN` the multi-value field against the lookup? Alas, no. While in most other scenarios the `JOIN` and `LOOKUP` syntax give the same result, `JOIN` with a multi-value field is unsupported and causes the query to fail.

It is however possible to create an exploded view of the main table by `GROUP`ing by all the fields in the table. You can then join the result and query it like this:

![Nested query with Join](/assets/2021-10-14-4-query-nested.jpeg)

That you can, doesn't necessarily mean you should, though. Complex joins and subqueries often use a lot of resources and in many cases, can be restated in such a way that the capabilities of the Druid engine come to their best use.

## Learnings

- Druid lookups can be used to emulate type 1 slowly changing dimensions.
- They work with multi-value dimensions.
- This comes in handy when you have requirements such as shopping basket analysis.


