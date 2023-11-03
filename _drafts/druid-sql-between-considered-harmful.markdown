---
layout: post
title:  "Druid SQL: `BETWEEN` considered harmful"
categories: blog apache druid sql
---

When querying data in Druid (or another analytical database), your query will in almost all cases include a filter on the primary timestamp. And this timestamp filter will usually take the form of an interval.
