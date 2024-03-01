---
layout: post
title:  "New in Druid 29: Exporting Query Results"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/2021-12-21-elf.jpg
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)


## The problem

Often my customers come to me with the requirement to extract large and/or detailed data sets from druid; they would like to store these data in a well known format for further processing by other tools. With [multi-stage query](https://druid.apache.org/docs/latest/multi-stage-query/concepts#multi-stage-query-task-engine), you can issue an asynchronous query against deep storage that handles (almost) unlimited amounts of data.

However, obtaining a result is a two step process:
- First, [submit the query](https://druid.apache.org/docs/latest/api-reference/sql-api#submit-a-query-1);
- then [poll the task endpoint](https://druid.apache.org/docs/latest/api-reference/sql-api#get-query-status) until it is done
- and finally, [retrieve the result](https://druid.apache.org/docs/latest/api-reference/sql-api#get-query-results).

Meanwhile, the data that you download in step 3 has been written to some storage location inside Druid already. You can define a path and even instruct Druid to use [durable storage](https://druid.apache.org/docs/latest/operations/durable-storage#enable-durable-storage) for query results, but: these data are is still in a Druid specific format and cannot easily be read by other tools.

What if we could skip that step (persisting the result) completely and send the result directly to a file in a format of our choice?

It turns out druid 29 can do this. For now, it is somewhat limited - it only supports csv, and can only export to local filesystem or S3. But other formats, such as Parquet, are coming.

Let's try this out with a [Druid Quickstart](https://druid.apache.org/docs/latest/tutorials/) installation!

In this tutorial, you will
- learn how to configure the settings for MSQ export
- export a sample dataset.

## Preparation

We are going to export to local storage. To limit the attack surface for malicious or inexperienced users, you have to define a specific filesystem path where Druid is allowed to store export files.

On your local machine, install Druid 29 from the [tarball](https://druid.apache.org/downloads/).

Create a directory `/tmp/druid-export` on your local disk.

In your Druid installation, edit the file `conf/druid/auto/_common/common.runtime.properties` and add the line

```properties
druid.export.storage.baseDir=/tmp/druid-export
```

at the end of the file.

Then start Druid like so, from within your Druid install directory:

```bash
bin/start-druid -m5g
```

Ingest the _wikipedia_ sample data following the instructions using either [classic batch](https://druid.apache.org/docs/latest/tutorials/#load-data) or [SQL ingestion](https://druid.apache.org/docs/latest/tutorials/tutorial-msq-extern).

Then go to the query tab in the Druid console.

## Exporting data

Run this query:

```sql
INSERT INTO 
EXTERN(local(exportPath => '/tmp/druid-export/wikipedia-export'))
AS CSV
SELECT * FROM wikipedia
```

![Screenshot of running query](/assets/2024-03-01-01.jpg)

When the query finishes, check the export directory and you will find a CSV file containing the data:

![Preview of result file in a shell window](/assets/2024-03-01-02.jpg)

Note: the target directory has to be empty, else you get an error message.

This also works for export to [S3](https://druid.apache.org/docs/latest/multi-stage-query/reference/#s3).

## Learnings

- With MSQ, you can now export query results directly to external storage.
- This is a new feature in Druid 29. It is currently limited to CSV format and either local storage or S3, but expect more options to be added soon.

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
