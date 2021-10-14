---
layout: post
title:  "Druid Data Modeling Special: Lookups and Multi-value Dimensions"
categories: blog imply druid
---

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
