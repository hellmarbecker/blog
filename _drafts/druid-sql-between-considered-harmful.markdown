---
layout: post
title:  "Druid SQL: `BETWEEN` considered harmful"
categories: blog apache druid sql
---

When querying data in Druid (or another analytical database), your query will in almost all cases include a filter on the primary timestamp. And this timestamp filter will usually take the form of an interval.

The easiest way to describe such an interval seems to be the SQL `BETWEEN` operator.

Advice from the [grug brained developer](https://grugbrain.dev/): **Don't do that.**

Here's why. Imagine you have a table like this:


You want to list

SELECT * FROM "mini_sample"
WHERE __time BETWEEN TIMESTAMP'2023-01-01' AND TIMESTAMP'2023-01-03'


SELECT * FROM "mini_sample"
WHERE __time >= TIMESTAMP'2023-01-01' AND __time < TIMESTAMP'2023-01-04'

