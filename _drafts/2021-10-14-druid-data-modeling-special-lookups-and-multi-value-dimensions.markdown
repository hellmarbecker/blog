---
layout: post
title:  "Druid Data Modeling Special: Lookups and Multi-value Dimensions"
categories: blog imply druid
---

The other day, [Peter Marshall](https://www.petermarshall.io/) brought up the question: Do lookups work with multi-value dimensions in Druid? Spoiler alert: They do, in fact! But let's first take a quick look at what lookups are.

## Dimensional Modeling in Apache Druid

The general data model of Druid is that of a fully denormalized, wide table. It would look much like the data models you might know if you have been using [SPSS](https://www.ibm.com/analytics/spss-statistics-software) or [GNU Octave](https://de.wikipedia.org/wiki/GNU_Octave). This means each field value is determined at ingestion and stays that way.

But sometimes attribute values change over time. For instance, people tend to move around or get married, so their addresses and names might change. A news publisher might change the headline of an article but the content stays the same. This introduces the theory of [slowly changing dimensions](https://de.wikipedia.org/wiki/Slowly_Changing_Dimensions).

If you want to have an attribute that changes over time, you need a way to refer to that attribute's value at query time. [Druid lookups](https://druid.apache.org/docs/latest/querying/lookups.html) are a way to do this. A lookup maintains a key-value map that is stored in memory on all data nodes (historical and peon processes), and possibly updated periodically. In Druid SQL, the [LOOKUP()](https://druid.apache.org/docs/latest/querying/sql.html#string-functions) function replaces the key, which is a field or expression from your datasource, with the lookup value. This emulates what is known as a [type 1 slowly changing dimension](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/type-1/).

(It is also possible, and often makes sense, to apply a lookup at ingestion time. But thst is a story for another time.)

So, Peter's question is: can you apply lookups with [multi-value dimensions](https://blog.hellmar-becker.de/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/)?

## Real life example: 

Here's our data snippet for today's tutorial:
```json
{ "ts": "2021-10-14 10:00:00", "customer": "Gian", "basket": [ "0001", "0001", "0002", "0004" ] }
{ "ts": "2021-10-14 10:10:00", "customer": "Rachel", "basket": [ "0002", "0004", "0005" ] }
{ "ts": "2021-10-14 10:20:00", "customer": "Peter", "basket": [ "0005", "0004", "0002" ] }
{ "ts": "2021-10-14 10:30:00", "customer": "Gian", "basket": [ "0002" ] }
{ "ts": "2021-10-14 10:40:00", "customer": "Jessy", "basket": [ "0003", "0005", "0006" ] }
{ "ts": "2021-10-14 10:50:00", "customer": "Gian", "basket": [ "0005", "0006" ] }
```
And the product catalog lookup definition:
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
