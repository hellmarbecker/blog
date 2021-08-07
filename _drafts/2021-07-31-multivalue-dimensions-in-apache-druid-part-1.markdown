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

## What are multi-value dimensions?

A multi-value dimension is created when Druid ingests a dimension field that is in itself organized as a list, or an array. Some possible examples:
- In a JSON data set, the dimension value is a JSON array
- In a delimited file (CSV or TSV), values within one field are subdivided by a secondary delimiter
Currently, multi-value dimensions are supported only for string values.

## Why would you use multi-value dimensions?

There are a number of use cases where multi-value dimensions come in handy. For instance, imagine you run a video streaming service. Each movie can have multiple tags, or keywords associated with it.

Let's look at a simplified example.

Here's the data snippet we are going to use:
```
timestamp,movie,tags
2020-01-01 00:00:00,Black Widow,comedy|cartoon|mystery|murder|superhero
2020-01-01 00:01:00,Aladdin,mystery|sci-fi|romantic|comedy|murder|kids|western|murder
2020-01-01 00:02:00,High Noon,romantic|cartoon|mystery|western|western|romantic|mystery
2020-01-01 00:03:00,50 First Dates,kids|sci-fi|horror|kids|sci-fi|romantic|sci-fi|cartoon|horror
2020-01-01 00:04:00,Alien,kids|murder|comedy|superhero|romantic|western|superhero|cartoon|kids|murder
2020-01-01 00:05:00,High Noon,murder|superhero|murder|murder|comedy|cartoon|murder|western|romantic
2020-01-01 00:06:00,50 First Dates,kids|western|western|romantic|romantic
2020-01-01 00:07:00,Black Widow,romantic|sci-fi
2020-01-01 00:08:00,Alien,cartoon|sci-fi|murder|western|sci-fi|sci-fi|sci-fi|cartoon
2020-01-01 00:09:00,High Noon,murder|cartoon|horror|superhero|western|kids|superhero|comedy|cartoon
2020-01-01 00:10:00,Alien,romantic|romantic|murder
```
Somebody has been watching movies and assigned tages to them. Note that the main fields of our data set are separated by `,` while the individual tags are separated by `|`. This is one way to get multi-value dimensions into Druid.
