---
layout: post
title:  "Partitioning in Druid - Part 1: Dynamic and Hash Partitioning"
categories: blog apache druid imply
---

In addition to segmenting data by time, Druid allows to introduce a secondary partitioning within each segment. There are (currently) three strategies:
- dynamic partitioning
- hash partitioning
- single dim partitioning.

Let's use [the Wikipedia sample data](https://druid.apache.org/docs/latest/tutorials/index.html#step-4-load-data) to do some experiments. You can use the Druid quickstart for this tutorial.

### Dynamic Partitioning

Generally speaking, dynamic partitioning does not do a lot to the data with regards to organizing or reordering them. You will get one or more partitions per input data blob for batch ingestion, and one or more partitions per Kafka partition for realtime Kafka ingestion. Often times, the resulting segment files are smaller than desired. The upside of this strategy is that ingestion with dynamic partitioning is faster than any other strategy, and it doesn't need to buffer or shuffle data. The downside is that it doesn't do anything to optimize query performance.

For the first experiment, follow the [quickstart tutorial](https://druid.apache.org/docs/latest/tutorials/index.html) step by step, with one exception: Set the maximum number of rows per segment to 14,000, because we want to have more than one partition:

![Setting the partition size](/assets/2022-01-05-1-rows-per-segment.jpg)

Also, name the datasource `wikipedia-dynamic-1` (we will create more versions of this.) Run the ingestion, it will finish within a few seconds. Now let's look what we have:

![Segment list](/assets/2022-01-05-2-num-segments.jpg)

Because the sample has roughly 40,000 rows of data, Druid has created three partitions. Let's take a closer look at the partitions. 

In the quickstart setup, the segments live in `var/druid/segments/`, and here's the structure Druid has created for the new datasource:
```
wikipedia-dynamic-1
└── 2015-09-12T00:00:00.000Z_2015-09-13T00:00:00.000Z
    └── 2022-01-05T16:03:35.016Z
        ├── 0
        │   └── index.zip
        ├── 1
        │   └── index.zip
        └── 2
            └── index.zip
```

<pre>
datasource-name
&#x2514;&#x2500;time chunk
&nbsp;&nbsp;&#x2514;&#x2500;version timestamp
&nbsp;&nbsp;&nbsp;&nbsp;&#x2514;&#x2500;partition number
</pre>

In your shell, navigate to the path where you installed Druid, and type:
```bash
cd var/druid/segments/wikipedia-dynamic-1
```


```bash
#!/bin/bash
DSPATH=$HOME/apache-druid-0.22.1/var/druid/segments/wikipedia-dynamic-1


java -classpath "$HOME/apache-druid-0.22.1/lib/*" -Ddruid.extensions.loadList="[]" org.apache.druid.cli.Main \
  tools dump-segment \                                                                                     
  --directory "$HOME/tmpwork" \
  --out "$HOME/tmpwork/output.txt"
```




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
