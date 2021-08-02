---
layout: post
title:  "Multi-Value Dimensions in Apache Druid (Part 1)"
categories: blog apache druid imply
---
[Apache Druid](https://druid.apache.org/) is a high performance, distributed OLAP database that also has characteristics of a timeseries database. One feature that sets Druid apart from its competitors is the ability to use [multi-value dimensions](https://druid.apache.org/docs/0.21.1/querying/multi-value-dimensions.html). The documentation on this feature seems to be a bit terse though, so I am going to start a loose series of posts to shed some light on questions such as:
- What are multi-value dimensions, anyway?
- What are they good for?
- How do you build a data model using multi-value dimensions?
- How do multi-value dimensions behave in queries?
- What are some real world use cases of multi-value dimensions?

Let's get started!


