---
layout: post
title:  "Druid 26 Sneak Peek: Timeseries Interpolation"
categories: blog apache druid imply iot sql tutorial
---
![Druid Cookbook](/assets/2023-04-08-01-hotandcold.jpg)

Today I am going to look at another [new Druid 26.0 feature](https://www.linkedin.com/feed/update/urn:li:activity:7043593237915148288/).

lorem ipsum

This is a sneak peek into Druid 26 functionality. In order to use the new functions, you can (as of the time of writing) [build Druid](https://druid.apache.org/docs/latest/development/build.html) from the HEAD of the master branch:

```bash
git clone https://github.com/apache/druid.git
cd druid
mvn clean install -Pdist -DskipTests
```

Then follow the instructions to locate and install the tarball.

Alternatively, you can use [Imply Enterprise](https://imply.io/download-imply/) which ships with all the featured discussed today, and comes with a free 30 day trial license.

In this tutorial, you will 

- ingest a data sample and
- run a query to fill in missing values at regular time intervals, using a simple linear interpolation scheme.

_**Disclaimer:** This tutorial uses undocumented functionality and unreleased code. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. In addition, the entire build, or execution, may fail. Your mileage may vary._

## data sample

## query it

---

 <p class="attribution">"<a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/53575715@N02/6620214217">Hot & Cold</a>" by <a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/53575715@N02">astronomy_blog</a> is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/?ref=openverse">CC BY-NC-SA 2.0
  <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/>
  <img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/>
  <img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/>
  <img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>. </p> 
