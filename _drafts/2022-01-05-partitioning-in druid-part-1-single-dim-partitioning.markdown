---
layout: post
title:  "Partitioning in Druid - Part 1: Single Dim Partitioning"
categories: blog apache druid imply
---

In addition to segmenting data by time, Druid allows to introduce a secondary partitioning within each segment. There are (currently) three strategies:
- dynamic partitioning
- hash partitioning
- single dim partitioning.

### Dynamic Partitioning

Generally speaking, dynamic partitioning does not do a lot to the data with regards to organizing or reordering them. You will get one or more partitions per input data blob for batch ingestion, and one or more partitions per Kafka partition for realtime Kafka ingestion. Often times, the resulting segment files are smaller than desired. The upside of this strategy is that ingestion with dynamic partitioning is faster than any other strategy, and it doesn't need to buffer or shuffle data. The downside is that it doesn't do anything to optimize query performance.

### Hash Partitioning



### Single Dim Partitioning

Here's what I get with all data in one segment:
```
% jq '.["channel"]' output.txt | sort | uniq -c | sort -nr
11549 "#en.wikipedia"
9747 "#vi.wikipedia"
2523 "#de.wikipedia"
2099 "#fr.wikipedia"
1386 "#ru.wikipedia"
1383 "#it.wikipedia"
1256 "#es.wikipedia"
1126 "#zh.wikipedia"
 983 "#uz.wikipedia"
 749 "#ja.wikipedia"
 565 "#pl.wikipedia"
 533 "#ko.wikipedia"
 478 "#ca.wikipedia"
 472 "#pt.wikipedia"
 445 "#nl.wikipedia"
 423 "#ar.wikipedia"
 289 "#hu.wikipedia"
 263 "#uk.wikipedia"
 251 "#el.wikipedia"
 246 "#he.wikipedia"
 244 "#sv.wikipedia"
 244 "#fi.wikipedia"
 222 "#cs.wikipedia"
 219 "#fa.wikipedia"
 208 "#tr.wikipedia"
 169 "#no.wikipedia"
 168 "#sr.wikipedia"
 153 "#hy.wikipedia"
 110 "#id.wikipedia"
  96 "#da.wikipedia"
  76 "#ro.wikipedia"
  75 "#bg.wikipedia"
  65 "#gl.wikipedia"
  60 "#ce.wikipedia"
  52 "#et.wikipedia"
  39 "#simple.wikipedia"
  33 "#sk.wikipedia"
  33 "#la.wikipedia"
  33 "#be.wikipedia"
  26 "#nn.wikipedia"
  22 "#hr.wikipedia"
  22 "#eo.wikipedia"
  21 "#sl.wikipedia"
  20 "#lt.wikipedia"
  19 "#hi.wikipedia"
  14 "#sh.wikipedia"
  13 "#eu.wikipedia"
  11 "#ms.wikipedia"
   9 "#kk.wikipedia"
   1 "#war.wikipedia"
   1 "#min.wikipedia"
```
