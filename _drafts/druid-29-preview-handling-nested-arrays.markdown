---
layout: post
title:  "Druid 29 Preview: Handling Nested Arrays"
categories: blog apache druid imply sql tutorial
---
![Druid Cookbook](/assets/2021-12-21-elf.jpg)

This is a sneak peek into Druid 26 functionality. In order to use the new functions, you can (as of the time of writing) [build Druid](https://druid.apache.org/docs/latest/development/build.html) from the HEAD of the master branch:

```bash
git clone https://github.com/apache/druid.git
cd druid
mvn clean install -Pdist -DskipTests
```

Then follow the instructions to locate and install the tarball.

All this is still under development so it is undocumented, and hidden behind a secret query context option. (We will look at that in a moment). Also notice, that window functions only work within `GROUP BY` queries, and there are still some other limitations. But it is fast progressing work.

In this tutorial, you will 

- ingest a data sample and
- do a quick cumulative report using window functions.

_**Disclaimer:** This tutorial uses undocumented functionality and unreleased code. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. In addition, the entire build, or execution, may fail. Your mileage may vary._
