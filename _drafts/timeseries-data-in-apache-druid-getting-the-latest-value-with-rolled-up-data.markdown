---
layout: post
title:  "Timeseries Data in Apache Druid: Getting the Latest Value with Rolled Up Data"
categories: blog druid imply timeseries analytics tutorial
---

![Stock ticker image]()

Imagine you are analyzing time based data and your most frequent queries are about retrieving the latest values for each time interval. One example would be stock ticker data where you would most likely want to know the closing prices for each trading day. IoT related use cases are another scenario where this would be useful.

This is easy if you use detail data. After all, Druid comes with `LATEST` and `EARLIEST` aggregators that give you exactly what you asked for. But you cannot use rollup because that would limit the granularity, and it would lead to incorrect results with imperfect rollup such as in realtime ingestion.

