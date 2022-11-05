---
layout: post
title:  "Druid Data Cookbook: Cumulative Sums in Druid SQL"
categories: blog apache druid imply sql
---

An often asked feature that is currently not present in Druid SQL are [window functions](https://www.sqltutorial.org/sql-window-functions/). With these, you can create aggregations on levels other than what you are `GROUP`ing by, and you can related rows within the same table to one another without resorting to a costly [cartesian join](https://en.wikipedia.org/wiki/Join_(SQL)#Cross_join).
