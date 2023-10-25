---
layout: post
title:  "Druid 28 Sneak Peek: Ingesting Multiple Kafka Topics- into One Datasource"
categories: blog apache druid imply sql tutorial
---
![Druid Cookbook](/assets/2021-12-21-elf.jpg)

## Building the distribution

Clone the Druid repository, check out the 28.0.0 branch, and build the tarball:

```
git clone https://github.com/apache/druid.git
cd druid
git checkout 28.0.0
mvn clean install -Pdist -DskipTests
```

Then follow the [instructions](https://druid.apache.org/docs/latest/development/build.html) to locate and install the tarball.
