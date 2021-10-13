---
layout: post
title:  "Druid Data Modeling Special: Lookups and Multi-value Dimensions"
categories: blog imply druid
---

Here's our data snippet for today's tutorial:
```
ts,customer,basket
2021-10-13 10:00:00,Gian,0001|0001|0002|0004
2021-10-13 10:10:00,Rachel,0002|0004|0005
2021-10-13 10:20:00,Peter,0005|0004|0002
2021-10-13 10:30:00,Gian,0002
2021-10-13 10:40:00,Jessy,0003|0005|0006
2021-10-13 10:50:00,Gian,0005|0006
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
