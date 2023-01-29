---
layout: post
title:  "Druid Data Cookbook: Nested Columns vs. Multi-Value Dimensions"
categories: blog apache druid imply tutorial
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)



In my series about [multi-value dimensions](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/), I took you to a small Italian restaurant where we looked at this snippet of orders:

```json
{ "date": "2021-09-25", "customer": "Fangjin", "orders": [ "pizza", "tiramisu", "espresso", "espresso" ] }
{ "date": "2021-09-25", "customer": "Gian", "orders": [ "pizza", "espresso", "tiramisu" ] }
{ "date": "2021-09-25", "customer": "Vadim", "orders": [ "pizza", "tiramisu" ] }
{ "date": "2021-09-25", "customer": "Rachel", "orders": [ "pizza", "tiramisu", "espresso" ] }
{ "date": "2021-09-25", "customer": "Xiaolan", "orders": [ "pizza", "espresso" ] }
{ "date": "2021-09-25", "customer": "Jessy", "orders": [ "pizza", "espresso", "espresso" ] }
```

Back then, we ingested the 
