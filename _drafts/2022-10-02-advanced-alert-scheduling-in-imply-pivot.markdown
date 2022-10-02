---
layout: post
title:  "Advanced Alert Scheduling in Imply Pivot"
categories: blog imply druid alert pivot tutorial
---

Here's a neat trick when working with [Imply Pivot](https://docs.imply.io/latest/pivot-overview/). Imagine you are running an online business, and you want to be notified whenever your online sales fall below a threshold value. You configure an [alert](https://docs.imply.io/latest/alerts/) to check your sales on an hourly basis but you find a pattern like this:

![Conversion graph with alert conditions](/assets/2022-10-02-01-basegraph.jpg)

In the small hours of the night, you have low sales but that is expected and you don't want to have lots of messages during that time. Wouldn't it be nice to set a more detailed schedule for the alert to be evaluated only during certain hours of the day?

