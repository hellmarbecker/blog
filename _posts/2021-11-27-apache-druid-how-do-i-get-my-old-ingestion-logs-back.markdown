---
layout: post
title:  "Apache Druid: How Do I Get My Old Ingestion Logs Back?"
categories: blog druid imply ingestion tutorial
---
![](/assets/2021-11-27-0-banner.jpeg)

One of my clients was recently trying to ingest data into [Apache Druid](https://druid.apache.org/) and experienced some errors. The next day, he wanted to ask for help but the failed tasks and their error logs were gone frome the console. What to do?

## Looking At Task Logs

The Druid Console gives users easy access to many functions of Druid. For data ingestion, the task view is interesting. It shows a list of running supervisors in the upper half of the window, and individual ingestion tasks and subtasks in the lower half.

_Supervisors_ orchestrate realtime ingestion jobs, if you do only batch jobs this part is empty.

_Tasks_ are used for both realtime and batch jobs. If you run a batch job, it will start with a task that is typically of type `index_parallel`. This task can spawn off various subtasks, depending on your settings for partitioning and tuning. But that is a story for another time.

Each task and subtask entry has to its right two symbols: a magnifying glass and a wrench. Clicking the wrench opens the task detail view, and it is from here that you can glean a lot of interesting information such as
- task status
- payload, that is the detailed configuration of this task
- and you can follow the task log as the task proceeds.

Moreover, for a failed task you can still open up the log and find out what went wrong.

## And Then There Were None?

Unfortunately, task entries in the console are cleaned up after 24 hours. The log files are of course still available somewhere, but wouldn't it be nice to be able to bring them back to the console?

In this example, I am using a local installation of [Druid 0.22](https://www.apache.org/dyn/closer.cgi?path=/druid/0.22.0/apache-druid-0.22.0-bin.tar.gz), and I am running the [micro-quickstart](https://druid.apache.org/docs/latest/tutorials/index.html#step-2-start-up-druid-services) configuration.

I ran some ingestion processes a couple of weeks ago, but when I open the task view, it looks like this:

![](/assets/2021-11-27-1-emptylist.jpeg)

Not good for debugging.

## Bring It Back!

It turns out there is a configuration setting for this! It is [a configuration property of the Overlord process](https://druid.apache.org/docs/latest/configuration/index.html#overlord-static-configuration).

Edit the file `conf/druid/single-server/micro-quickstart/coordinator-overlord/runtime.properties`[^1] and add the line:

```properties
druid.indexer.storage.recentlyFinishedThreshold=P8W
```

[^1]: If you are running a different configuration, you may have to adjust the path of the configuration file.

The value for this setting is an [ISO 8601 interval description](https://en.wikipedia.org/wiki/ISO_8601#Time_intervals). But, not _any_ interval description will do!

I tried a setting of `P2M` and it failed. The Overlord log gives me a hint as to why:

```
Caused by: java.lang.IllegalArgumentException: Cannot construct instance of `org.apache.druid.indexing.common.config.TaskStorageConfig`, problem: Cannot convert to Duration as this period contains months and months vary in length
```

So, year or month periods are not allowed because they are not of well defined length. Weeks are okay, though.

After a service restart, the task entries are back:

![](/assets/2021-11-27-2-fulllist.jpeg)

This is extremely useful for debugging. Note though, that this is not a recommended practice for production systems! The task list is cleaned up for a reason. As the list gets longer, the console gets less responsive.

## Learnings

- Task lists in the Druid console are cleaned up after 24 hours
- However, the retention period is configurable
- Changing the retention can bring old entries back

---
